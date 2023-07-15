---
title: "curlの1コマンドでWebサーバーの起動を待つ"
emoji: "🕓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["curl"]
published: true
---

CI環境など、シェルスクリプトでWebサーバーの起動を待ちたいことがあります。Bashのループなどを使う方法も考えられますが、もっと簡単に実現したいと思って調べてみたところ、curlのオプションだけでシンプルに実現できることがわかりました。

次のコマンドで、エラーなくHTTPレスポンスを得られるまで、1秒間隔で最大30回リトライできます。

```
curl --retry 30 --retry-delay 1 --retry-all-errors http://localhost:8080
```

それぞれのオプションの意味は次のとおりです。

* [--retry <num>](https://curl.se/docs/manpage.html#--retry) リトライ回数を指定します。初回分は含まないので、上記だと最大31回リクエストします。
* [--retry-delay <seconds>](https://curl.se/docs/manpage.html#--retry-delay): リトライまでの秒数を指定します。指定しない場合はデフォルトの指数バックオフが使われます。
* [--retry-all-errors](https://curl.se/docs/manpage.html#--retry-all-errors): 全てのエラー時にリトライします。

中でも `--retry-all-errors` は比較的新しいオプションで、curl 7.71.0以降で利用できます。

なお、上記のコマンドではHTTPステータスコードが4xxや5xxでもエラー扱いにはならず、リトライされません。HTTPステータスコードに応じてリトライしたい場合は、[--fail](https://curl.se/docs/manpage.html#-f) オプションをつけます。

## 参考

* [How to create a loop in bash that is waiting for a webserver to respond? \- Stack Overflow](https://stackoverflow.com/questions/11904772/how-to-create-a-loop-in-bash-that-is-waiting-for-a-webserver-to-respond/71522504#71522504)
  * この回答で使われている `--retry-connrefused` だと、DockerでHTTPサーバーを起動している場合に `curl: (52) Empty reply from server` のエラーでリトライできないことがありました。 `--retry-all-errors` に置き換えたところ、確実にリトライできました。
