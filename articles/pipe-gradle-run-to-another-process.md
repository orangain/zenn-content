---
title: "gradle runの実行結果を別コマンドにパイプして出力する"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gradle"]
published: true
---

`./gradlew run` で起動したアプリケーションの標準出力を別のコマンドにパイプして出力したいという欲求がありました。具体的には、Webアプリケーションから標準出力にJSON形式で出力されたログを、ローカル実行時だけ [pino-pretty](https://github.com/pinojs/pino-pretty) コマンドを使って読みやすく整形して出力したいというものです。

## 検討したけど採用しなかった案

CLIで実行するだけであれば、次のようにパイプすれば良いですが、IntelliJ IDEAからアプリケーションを起動した場合にもログを整形したかったので採用しませんでした。IntelliJ IDEAから起動したアプリケーションに対して、パイプするプロセスを指定する方法は見つけられませんでした。

```
./gradlew run | pino-pretty
```

また、IntelliJ IDEAからShell Scriptで上記のコマンドを起動するという案もありましたが、アプリケーションをデバッグ実行できないため見送りました。

`pino-pretty` にそこまでこだわりはないので、IntelliJ IDEAでJSON形式のログを整形して表示するプラグインとかあるかなーと思って探してみましたが、特に見つかりませんでした。

アプリケーション内で開発時だけ整形して出力するという案は、Twelve Factor Appの[Dev/prod parity](https://12factor.net/dev-prod-parity)の観点からあまり好ましくないと考えているので採用しませんでした。

## 実装

最終的に `build.gradle.kts` で次のように実装することで、 `./gradlew run` を実行するだけで `pino-pretty` コマンドを使って整形できるようになりました。IntelliJで起動した場合も同様の出力が得られます。

`run` タスクの実行前にProcessBuilderを使って `pino-pretty` を起動し、 `run` で起動するアプリケーションの標準出力を `pino-pretty` の標準入力にパイプするというものです。

```kts
import kotlin.concurrent.thread
import java.io.InputStreamReader
import java.nio.charset.StandardCharsets

tasks.named<JavaExec>("run") {
    doFirst {
        val process = ProcessBuilder("pino-pretty", "--colorize").start()
         // 起動するアプリケーションの標準出力をpino-prettyの標準入力にパイプする。
        standardOutput = process.outputStream

        thread {
            InputStreamReader(process.inputStream, StandardCharsets.UTF_8).use { reader ->
                var value: Int
                while (reader.read().also { value = it } != -1) {
                    print(value.toChar())
                }
            }
        }
    }
}
```

※上記のコードでは省略しましたが、実際には `-PnoPretty=1` をつけた場合はdoFirst自体を実行しないという処理を入れて、`pino-pretty` コマンド無しでも起動できるようにしています。

## 実行イメージ

IntelliJ IDEAでの実行イメージは次の通りです。

![](https://storage.googleapis.com/zenn-user-upload/5pbt7z2oagc3vxrwup9jmjbnuroc)

## わかってないこと

[ProcessBuilder](https://docs.oracle.com/javase/jp/8/docs/api/java/lang/ProcessBuilder.html)について調べると、 `.redirectOutput(ProcessBuilder.Redirect.INHERIT)` とすることで現在の標準出力と同じになるとありましたが、これを指定して `pino-pretty` を起動しても標準出力には何も表示されませんでした。仕方ないので、別スレッドで `pino-pretty` の標準出力を読み取っては `print` する処理を実行しています。

`ps` で見る限りプロセス自体は起動しているので、Gradleが標準出力を捨ててるのかなーと推察していますが、同じ箇所で `println` した文字列は出力されており、いまいちよくわかっていません。

## 最後に

ニッチな欲求かと思いますが、調べてもすぐに見つからなかったので、書いておきました。上記のわかってないことを含め、もう少し良い方法がありそうなので、もしご存知の方は教えていただけると幸いです。

# 参考

* [Java SE 7のProcessBuilderでリダイレクト \- n\-agetsumaの日記](http://n-agetsuma.hatenablog.com/entry/2014/02/12/215321)
* [io \- Easy way to write contents of a Java InputStream to an OutputStream \- Stack Overflow](https://stackoverflow.com/questions/43157/easy-way-to-write-contents-of-a-java-inputstream-to-an-outputstream)
