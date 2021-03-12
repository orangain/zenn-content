---
title: "macOSで小文字のUUIDv4を生成してコピーするワンライナー"
emoji: "1️⃣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mac"]
published: true
---

DBの主キーをUUIDにしていると、テストコードなどでしばしばランダムなUUIDv4を生成したくなります。macOSには `uuidgen` コマンドがありますが、さらに手間を省くのが次のワンライナーです。

```
uuidgen | tr '[A-Z]' '[a-z]' | tr -d '\n' | pbcopy
```

このワンライナーは次のことをやってくれるので、UUIDが必要な箇所に貼り付けるだけです。

* アルファベットを小文字に変換
* 末尾の改行を除去
* クリップボードにコピー

次のように `.zshrc` などでエイリアスを定義しておくと、簡単に呼び出せます。

```
alias uuid="uuidgen | tr '[A-Z]' '[a-z]' | tr -d '\n' | pbcopy"
```

## 環境

* macOS Catalina 10.15.7
* zsh 5.7.1 (x86_64-apple-darwin19.0)
