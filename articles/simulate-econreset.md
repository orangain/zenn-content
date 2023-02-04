---
title: "HTTPレスポンスボディを返す途中でわざとECONRESETを発生させる"
emoji: "🤕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs", "http"]
published: true
---

あるHTTPのプロキシサーバーをテストする際に、レスポンスボディを返し始めた後にTCPレベルのエラーが発生するという状況を再現したくなり、そのような挙動をするHTTPサーバーをNode.jsで作ってみました。

なお、Node.js v16.17.0, v18.3.0 以降で追加された [socket.resetAndDestroy()](https://nodejs.org/api/net.html#socketresetanddestroy) 関数を使っています。動作は v16.19.0 で確認しました。

```js
const net = require('net');

const port = 9000
net.createServer()
  .on('connection', handleConnection)
  .on('listening', () => {console.log(`listening on post ${port}`)})
  .listen(port);

function handleConnection(socket) {
  // クライアントからのリクエストをコンソールに表示する。
  socket.on('data', (chunk) => {
    console.log('Received chunk:\n', chunk.toString());
  });
  // Content-Length: 13のうち最初の6バイトだけ返す。
  socket.write('HTTP/1.1 200 OK\r\nServer: my-web-server\r\nContent-Length: 13\r\n\r\nHello,');
  // RSTパケットを送信してTCPコネクションをクローズする。
  socket.resetAndDestroy();
}
```

適当に `broken-server.js` などと名前をつけて実行するとポート9000で起動します。

```
% node broken-server.js
listening on post 9000
```

このサーバーに対してcurlでリクエストを送ると、レスポンスボディを処理する途中でエラーが発生していることがわかります。

```
% curl -i http://localhost:9000/
HTTP/1.1 200 OK
Server: my-web-server
Content-Length: 13

curl: (56) Recv failure: Connection reset by peer
Hello,%
```

## 参考

- [Let's code a web server from scratch with NodeJS Streams\! \| Codementor](https://www.codementor.io/@ziad-saab/let-s-code-a-web-server-from-scratch-with-nodejs-streams-h4uc9utji)
  - Node.jsのnetモジュールを使ってHTTPサーバーを作る方法を参考にさせてもらいました。
- [Endpoints that simulates connection errors · Issue \#420 · postmanlabs/httpbin · GitHub](https://github.com/postmanlabs/httpbin/issues/420)
  - [httpbin](https://httpbin.org/)にエラーを発生させる機能がないか調べたときに参考にさせてもらいました。pathodが含まれているとされている[mitmproxy](https://mitmproxy.org/)のAddonについて調べたものの、TCPレベルのエラーを簡単に発生させる方法は見つけられませんでした。

