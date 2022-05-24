---
title: "JavaのCLIをGradleとGitHub ActionsとHomebrew tapで継続的に配布する"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kotlin", "gradle", "cli", "githubactions", "homebrew"]
published: true
---

JavaやKotlinで書いたCLIをサクッと継続的に配布したいと思った時に、真似するだけで簡単にできる情報が意外と見つけられなかったのでまとめておきます。

CLIの配布方法としてはいくつかの選択肢が考えられますが、この記事では一番上の方法を解説します。

* jarファイルと起動用のスクリプトが含まれたアーカイブファイルを配布する
* 単体で動作するjarファイルを配布する
* GraalVMで生成したネイティブバイナリを配布する
* ソースコードを配布してユーザーがビルドできるようにする

継続的に配布するための全体の流れは次の通りです。

1. jarファイルと起動用のスクリプトが含まれたアーカイブファイルを作成する
2. GitHub Actionsでタグをつけたときにリリースを作る
3. HomebrewのTapを作成して簡単にインストールできるようにする
4. GitHub ActionsでHomebrewのFormulaを継続的にアップデートする

## 前提

`ktcodeshift` というコマンド名のCLIを例に説明します。次のようなマルチプロジェクト構成になっており、CLIは `ktcodeshift-cli` というサブプロジェクトの成果物です。マルチプロジェクトを使わない場合は、パスを適宜読み替えてください。

```
ktcodeshift
├── ktcodeshift-cli
│   └── build.gradle.kts
├── ktcodeshift-dsl
│   └── build.gradle.kts
├── build.gradle.kts
└── settings.gradle.kts
```

ソースコードは以下の場所に置いてあります。記事タイトルにはJavaとありますが、ビルドしたらほぼ同じなので実際にはKotlinで書いてます。Gradleの設定もKotlin DSLを使っています。

CLI:
https://github.com/orangain/ktcodeshift

HomebrewのTap:
https://github.com/orangain/homebrew-tap


利用したバージョンは次のとおりです。

- macOS Monterey 12.4
- Gradle 7.4.2
- OpenJDK Eclipse Temurin 17.0.1
- Kotlin 1.6.21
- Homebrew 3.4.11

## 1. jarファイルと起動用のスクリプトが含まれたアーカイブファイルを作成する

Gradleの[applicationプラグイン](https://docs.gradle.org/current/userguide/application_plugin.html) を利用すると、jarファイルと起動用のスクリプトが含まれたアーカイブファイルを作成できます。デフォルトでは `tar` 形式と `zip` 形式が作られますが、設定を追加すると `tar` を `tar.gz` にできます。

バージョン番号は手でメンテナンスするとやや面倒ですが、 [Git-Version](https://github.com/palantir/gradle-git-version) というGradleプラグインを使うと、Gitの情報からいい感じにバージョン番号を生成してくれて便利です。

この2つのプラグインを使うGradleの設定 (`ktcodeshift-cli/build.gradle.kts`) は次のようになります。

```kts
plugins {
    // 略
    application
    id("com.palantir.git-version") version "0.15.0"
}

// バージョン番号をGitの情報から生成する
val gitVersion: groovy.lang.Closure<String> by extra
version = gitVersion()

// 略

application {
    // アプリケーションのメインクラス
    mainClass.set("ktcodeshift.AppKt")
    // アプリケーション（コマンド）名。プロジェクト名と異なる場合は要設定。
    applicationName = "ktcodeshift"
}

// `gradle run` で実行した時にカレントディレクトリを引き継ぐ（開発時に便利だけど必須ではない）
tasks.run.get().workingDir = File(System.getProperty("user.dir"))

// .tarファイルの代わりに.tar.gzファイルを生成する
tasks.distTar {
    compression = Compression.GZIP
    archiveExtension.set("tar.gz")
}
```

`./gradlew assemble` を実行すると `ktcodeshift-cli/build/distributions` 以下にアーカイブファイルが生成されます。

```
% ls ktcodeshift-cli/build/distributions/
ktcodeshift-0.1.0.tar.gz	ktcodeshift-0.1.0.zip
```

ここでは `0.1.0` というタグのコミットがチェックアウトされた状態で実行したので、ファイル名に `0.1.0` というバージョン番号が付けられていますが、他のコミットではコミットハッシュなどを含む番号になります。

アーカイブファイルの中身を知りたいときは展開してもいいですが、 `./gradlew installDist` を実行すると `build/install` 以下にアーカイブファイルの中身に相当するファイルが置かれるため、これで中身を確認すると楽です。

```
% tree ktcodeshift-cli/build/install
ktcodeshift-cli/build/install
└── ktcodeshift
    ├── bin
    │   ├── ktcodeshift
    │   └── ktcodeshift.bat
    └── lib
        ├── aether-connector-basic-1.1.0.jar
        ├── aether-transport-file-1.1.0.jar
        ├── aether-transport-wagon-1.1.0.jar
        ├── animal-sniffer-annotations-1.14.jar
        ├── annotations-13.0.jar
        ...
```

:::message

サードパーティのjarファイルを一緒に配布することになりますが、OSSのライセンスファイルは基本的に各jarファイルに含まれていることが多いため、ここでは別途一覧などは用意しません。

一覧を用意する場合は、次のGradleプラグインが便利です。自分のアプリケーションのライセンスと互換性がないライブラリが含まれていないかの確認にも役立ちます。

jk1/Gradle-License-Report: A plugin for generating reports about the licenses of third party software using Gradle
https://github.com/jk1/Gradle-License-Report
:::


## 2. GitHub Actionsでタグをつけたときにリリースを作る

`.github/workflows/java_ci.yaml` を次の内容で作成することで、普段のpush時にビルドを実行しつつ、タグをつけたときには追加でリリースを作成して、アーカイブファイルをアップロードできます。最後の `Create release on tag` のステップに `if` を設定し、タグをつけた時だけ実行されるようにしています。

リリースの作成を行うアクションはいろいろあるようですが、ここでは [marvinpinto/action-automatic-releases](https://github.com/marvinpinto/action-automatic-releases) を使いました。 `files` の部分はディレクトリ名に合わせて置き換えてください。

```yaml
name: Java CI

on: [ push ]

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'adopt'
      - name: Validate Gradle wrapper
        uses: gradle/wrapper-validation-action@v1
      - name: Setup Gradle
        uses: gradle/gradle-build-action@v2
      - name: Execute Gradle build
        run: ./gradlew build

      - name: Create release on tag
        if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/')
        uses: "marvinpinto/action-automatic-releases@v1.2.1"
        with:
          repo_token: "${{ secrets.GITHUB_TOKEN }}"
          prerelease: false
          files: |
            ktcodeshift-cli/build/distributions/*
```

成功すると、このようにリリースが作成されます。

![作成されたリリース](/images/distribute-java-cli-continuously/cli-release.png)

## 3. HomebrewのTapを作成して簡単にインストールできるようにする

Homebrewでは [Tap](https://docs.brew.sh/Taps) を使うことで、本家にPull Requestを送らずともユーザーにインストールして使ってもらえるようになります。

### Tapを作成する

任意のディレクトリで次のコマンドを実行すると、Tapを作成できます。 `orangain` はGitHubのユーザー名に置き換えてください。

```
brew tap-new orangain/homebrew-tap
```

これによって `/usr/local/Homebrew/Library/Taps/orangain/homebrew-tap` ^[実際に作成されるパスは `$(brew --prefix)` によって変わります。]にGitリポジトリが作られます。

GitHubに同じ名前 (`homebrew-tap`) で中身が空のリポジトリを作成し、生成されたGitリポジトリをpush^[pushするコマンドはデフォルトのブランチ名に依存するので、GitHubで表示されたコマンドを実行してください。]しておきます。

```
git remote add origin git@github.com:orangain/homebrew-tap.git
git branch -M main
git push -u origin main
```

### Formulaを作成する

次のように、手順2で公開したファイルのURLを引数に与えてコマンドを実行すると、Tapに新しいFormulaを作成できます。

```
brew create --tap orangain/homebrew-tap https://github.com/orangain/ktcodeshift/releases/download/0.1.0/ktcodeshift-0.1.0.tar.gz
```

`/usr/local/Homebrew/Library/Taps/orangain/homebrew-tap/Formula/ktcodeshift.rb` にFormulaが作られ、エディターで開かれます。

`install` メソッドを次のように変更することで、インストールできるようになります。この処理は、[GradleのFormula](https://github.com/Homebrew/homebrew-core/blob/master/Formula/gradle.rb)を大いに参考にしました。

```rb
  def install
    rm_f Dir["bin/*.bat"]
    libexec.install %w[bin lib]
    env = Language::Java.overridable_java_home_env
    (bin/"ktcodeshift").write_env_script libexec/"bin/ktcodeshift", env
  end
```

### インストールする

次のコマンドを実行すると、ローカルで編集したFormulaでインストールできます。

```
brew install orangain/tap/ktcodeshift
```

問題なくインストールしてコマンドを実行できたら、Formulaをコミットしてpushします。ここまでやれば、他のmacからでも上記のコマンドでインストールできるようになるはずです。

インストールして使うだけであればこれだけでも問題ありませんが、 `brew test orangain/tap/ktcodeshift` や `brew audit orangain/tap/ktcodeshift` の実行結果を見て、これらのコマンドが成功するように書き換えると、より行儀が良いFormulaと言えるでしょう。

## 4. GitHub ActionsでHomebrewのFormulaを継続的にアップデートする

HomebrewのFormulaにはバージョン番号とsha256ハッシュが含まれており、継続的に配布するためにはここのアップデートも自動化したいところです。

GitHub Actionsで [mislav/bump-homebrew-formula-action](https://github.com/mislav/bump-homebrew-formula-action) を使うと、次の手順でリリース時にhomebrew-tapリポジトリのFormulaが自動で更新されるようになります。

1. GitHubの[Personal Access Token](https://github.com/settings/tokens)を `public_repo` の権限で作成する
2. アプリケーションのリポジトリの Settings → Secrets → Actions → New repository secretで `COMMITTER_TOKEN` という名前で作成したトークンを保存する
3. アプリケーションのリポジトリのGitHub Actionsの `.github/workflows/java_ci.yaml` に次のステップを追加する
    ```yaml
          - name: Extract version
            id: extract-version
            run: |
              printf "::set-output name=%s::%s\n" tag-name "${GITHUB_REF#refs/tags/}"
          - name: Update formula on tag
            if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/')
            uses: "mislav/bump-homebrew-formula-action@v2"
            with:
              download-url: https://github.com/orangain/ktcodeshift/releases/download/${{ steps.extract-version.outputs.tag-name }}/ktcodeshift-${{ steps.extract-version.outputs.tag-name }}.tar.gz
              formula-name: ktcodeshift
              homebrew-tap: orangain/homebrew-tap
            env:
              COMMITTER_TOKEN: ${{ secrets.COMMITTER_TOKEN }}
    ```

上記の設定を行うと、アプリケーションのリポジトリでタグをつけたときに、Homebrew Tapのリポジトリに次のようなコミットが自動で作られるようになります。

[![作成されたコミット](/images/distribute-java-cli-continuously/bump-formula-commit.png)](https://github.com/orangain/homebrew-tap/commit/f1ac644c74ab9073d60b00491a24557d46ebd0e4)

試していませんが、Personal Access Tokenを作ったユーザーがリポジトリにプッシュする権限を持たない場合は、代わりにPull Requestが作られるようです。

## まとめ

この記事では、次の流れでJava (Kotlin) のCLIを配布する方法を解説しました。

1. jarファイルと起動用のスクリプトが含まれたアーカイブファイルを作成する
2. GitHub Actionsでタグをつけたときにリリースを作る
3. HomebrewのTapを作成して簡単にインストールできるようにする
4. GitHub ActionsでHomebrewのFormulaを継続的にアップデートする

Homebrew向けだけではありますが、継続的なリリースを行う環境が整えられ、開発に集中できるようになったかなと思います。もっと手軽な方法があれば、ぜひ教えていただけると嬉しいです。

## 参考文献

https://blog.ottijp.com/2020/05/23/homebrew-make-formula/
→ ソースコードを配布し、ユーザーの手元でGradleでビルドするアプローチ

https://zenn.dev/kesin11/scraps/954ecc429c5096
→ 単体で動作するjarファイルを配布し、ユーザーの手元で起動用スクリプトを生成するアプローチ

https://github.com/Homebrew/homebrew-core/blob/master/Formula/gradle.rb
→ GradleのFormula

https://dev.classmethod.jp/articles/gradle-auto-versioning-with-git-2020/
→ Git-Versionプラグインの紹介

https://stackoverflow.com/questions/58475748/github-actions-how-to-check-if-current-push-has-new-tag-is-new-release
→ GitHub Actionsでタグがつけられた場合だけステップを実行する方法
