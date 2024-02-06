---
title: "GradleでMaven Centralにライブラリを公開する（2024年版）"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gradle", "mavencentral"]
published: true
---

## はじめに

Maven Central Repository (以降、Maven Central) はJavaなどのライブラリを公開するための代表的なMavenリポジトリです。これまでMaven Centralにライブラリを公開するためには、 issues.sonatype.org というJIRAでの申請が必要でしたが、このシステムが2024年2月1日に[廃止された](https://central.sonatype.org/news/20240109_issues_sonatype_org_deprecation/)ことに伴い、Maven Centralへの新しいライブラリ公開手順が追加されました。

以前の手順ではOSSRHと呼ばれるシステムが使用されていましたが、新しい手順では[Central Portal](https://central.sonatype.com/)からライブラリを公開できます。GitHubアカウントでのログインに対応しているため、ライブラリのグループIDとして `io.github.{アカウント名}` で始まるものを利用する場合には、公開がだいぶ楽になりました。

ただし、この記事の執筆時点では、[公式のライブラリ公開手順](https://central.sonatype.org/publish-ea/publish-ea-guide/)としてMavenまたは手動アップロードを使う方法のみが用意されています。Gradleに関しては、コミュニティが開発するプラグインへのリンクがあるのみなので、Gradleを使う場合の公開方法を記しておきます。

## 大まかな流れ

ライブラリを公開するための大まかな流れは次の通りです。

1. Central Portalにアカウントを作成し、Namespaceを取得する
2. GPG/PGPを準備する
3. Central PortalでUser Tokenを生成する
4. Gradleの設定を追加する
5. ライブラリを公開する
6. (おまけ) GitHub Actionsを設定する

なお、Gradleを利用した公式の手順や紹介するコミュニティプラグインは絶賛整備中の模様で、この記事の4以降の内容はすぐに古くなるかもしれません。最新の情報は以下の公式ドキュメントを確認してください。

https://central.sonatype.org/register/central-portal/

## 注意事項

Maven Centralに公開するライブラリには[満たすべき要件](https://central.sonatype.org/publish/requirements/)があります。大まかに言えば次の通りです。

- ライブラリのjarに加えてsourcesJarとjavadocJarを提供する
- ファイルのチェックサムを提供する
- GPG/PGPによるファイルの署名を提供する
- pom.xmlでライブラリのメタデータを提供する

基本的には後述の手順に従うことでこれらの要件は満たせますが、詳しくはドキュメントを参照してください。

また、一度公開したライブラリの成果物は[削除できません](https://central.sonatype.org/publish/requirements/immutability/)。開発中を表すSNAPSHOTバージョンを公開することもできないので、公開前によく確認してください。

## 1. Central Portalにアカウントを作成し、Namespaceを取得する

公式手順: [Register to Publish Via the Central Portal \- The Central Repository Documentation](https://central.sonatype.org/register/central-portal/)

[Central Portal](https://central.sonatype.com/) にアクセスし、右上からSign Inします。GoogleまたはGitHubのアカウント、またはユーザ名・メールアドレス・パスワードでサインアップできます。

GitHubアカウントでログインすると、 `io.github.{アカウント名}` のNamespaceが自動で取得できるので、このNamespaceを使う予定の場合には、GitHubアカウントでログインするのがオススメです。GitHubアカウントでログインした後、右上のメニューから「View Namespaces」を選択すると、Namespaceが表示されるはずです。

GitHub以外のアカウントを使用する場合や、他の名前空間を使う場合はNamespaceを取得する必要があるので、上記の公式手順を参照してください。

## 2. GPG/PGPの準備

公式手順: [Working with PGP Signatures \- The Central Repository Documentation](https://central.sonatype.org/publish/requirements/gpg/)

GPGの鍵ペアを持っていない場合、GPG/PGPで署名するための準備が必要です。ここでは上記の公式手順に従ってGnuPGを使います。

1. GnuPGのインストール

    環境に合わせてインストールします。以下はmacOSでHomebrewを使う場合の例です。
    ```sh
    brew install gnupg
    ```

2. GPG鍵ペアの生成
    名前とメールアドレス、パスフレーズを入力して鍵ペアを生成します。表示された鍵IDは後のコマンドで使用します。
    
    ```sh
    gpg --gen-key
    ```

3. 公開鍵の送信
    公開鍵をキーサーバーに送信して、他のユーザーが公開鍵を取得できるようにします[^1]。
    ```sh
    gpg --keyserver keyserver.ubuntu.com --send-keys {鍵ID}
    ```

[^1]: この手順を実施する代わりに、後のSonatype Central Uploadプラグインの設定でpublicKeyを指定することも可能ですが、この手順を実施する方が楽だと思います。

## 3. Central PortalでUser Tokenを生成する

公式手順: [Publish Portal API \- The Central Repository Documentation](https://central.sonatype.org/publish/publish-portal-api/#authentication-authorization)

Central Portalに戻り、右上のメニュー → 「View Account」から「Generate User Token」でUser Tokenを生成します。usernameとpasswordが表示されるので控えておきます。

## 4. Gradleの設定を追加する

公式手順: [Publish Portal Gradle \- The Central Repository Documentation](https://central.sonatype.org/publish/publish-portal-gradle/)

執筆時点では、Gradleを使った公式の手順は用意されておらず、上記のページではコミュニティによるプラグインが紹介されているのみです。

ここでは [Maven Publishプラグイン](https://docs.gradle.org/current/userguide/publishing_maven.html) と、 [Sonatype Central Uploadプラグイン](https://github.com/Im-Fran/SonatypeCentralUpload) の2つを使用します。Maven Publishプラグインは、POMファイルの生成だけに使用し、Central Portalへの公開処理はSonatype Central Uploadプラグインを使います。

ライブラリを公開するために `build.gradle.kts` に4つの設定を追加します。

1. プラグインの有効化

    Maven PublishプラグインとSonatype Central Uploadプラグインを追加します。

    ```kts
    plugins {
        `maven-publish`
        id("cl.franciscosolis.sonatype-central-upload") version "1.0.2"
    }
    ``` 

2. sourcesJarとjavadocJarを生成するための設定を追加

    ```kts
    java {
        withSourcesJar()
        withJavadocJar()
    }
    ```

3. pom.xmlを生成するための設定を追加

    値は例なので、実際のプロジェクトに合わせて設定してください。なお、グループID、アーティファクトID、バージョンはGradleプロジェクトの設定から自動で取得されます。

    ```kts
    publishing {
        publications {
            register<MavenPublication>("maven") {
                pom {
                    name.set(project.name)
                    // 説明
                    description.set("Custom assertion to check JSON string matches pattern.")
                    // URL
                    url.set("https://github.com/orangain/json-fuzzy-match")
                    // ライセンス
                    licenses {
                        license {
                            name.set("MIT License")
                            url.set("https://github.com/orangain/json-fuzzy-match/blob/master/LICENSE")
                            distribution.set("repo")
                        }
                    }
                    // 開発者
                    developers {
                        developer {
                            id.set("orangain")
                            name.set("Kota Kato")
                            email.set("orangain@gmail.com")
                        }
                    }
                    // バージョン管理システム
                    scm {
                        url.set("https://github.com/orangain/json-fuzzy-match")
                    }
                }
            }
        }
    }
    ```

4. Sonatype Central Uploadプラグインの設定[^2]を追加

    秘密の値は環境変数で渡すことを想定した設定です。

    ```kts
    tasks.named<SonatypeCentralUploadTask>("sonatypeCentralUpload") {
        // 公開するファイルを生成するタスクに依存する。
        dependsOn("jar", "sourcesJar", "javadocJar", "generatePomFileForMavenPublication")

        // Central Portalで生成したトークンを指定する。
        username = System.getenv("SONATYPE_CENTRAL_USERNAME")
        password = System.getenv("SONATYPE_CENTRAL_PASSWORD")

        // タスク名から成果物を取得する。
        archives = files(
            tasks.named("jar"),
            tasks.named("sourcesJar"),
            tasks.named("javadocJar"),
        )
        // POMファイルをタスクの成果物から取得する。
        pom = file(
            tasks.named("generatePomFileForMavenPublication").get().outputs.files.single()
        )

        // PGPの秘密鍵を指定する。
        signingKey = System.getenv("PGP_SIGNING_KEY")
        // PGPの秘密鍵のパスフレーズを指定する。
        signingKeyPassphrase = System.getenv("PGP_SIGNING_KEY_PASSPHRASE")
    }
    ```

[^2]: この記事の執筆時点では、Sonatype Central UploadプラグインのREADMEにはバージョン1.0.0の設定方法が記載されていますが、最新の1.0.2だとそのままでは動作しません。この記事では1.0.2が想定していると思われる設定方法を記載しています。


設定の全体像は、この手順で実際に公開したライブラリのリポジトリを参考にしてください。

https://github.com/orangain/json-fuzzy-match

## 5. ライブラリを公開する

次のように環境変数を設定して、Gradleのタスク `sonatypeCentralUpload` を実行することでライブラリを公開できます。

```sh
export SONATYPE_CENTRAL_USERNAME='{username}'
export SONATYPE_CENTRAL_PASSWORD='{password}'
export PGP_SIGNING_KEY=$(gpg --armor --export-secret-keys {鍵ID})  # GPG鍵ペア生成時のパスフレーズが要求される
export PGP_SIGNING_KEY_PASSPHRASE='{秘密鍵のパスフレーズ}'

./gradlew sonatypeCentralUpload
```

このタスクでは、必要な成果物をまとめてアップロードするためのZIPファイルが生成され、Central Portalにアップロードされます。アップロードする単位をDeploymentと呼び、Deploymentの作成後には、要件を満たすかどうかの検証が行われます。

Deploymentの状態はCentral Portalのメニューの「View Deployments」から確認でき、検証に失敗した場合のエラー内容もここで確認できます。検証が成功すると次のように表示されて、DeploymentはPUBLISHING状態になります。このまま数分待つと、PUBLISHED状態になって公開されます。

```
> Task :sonatypeCentralUpload
[Sonatype Central Upload] Uploading to Sonatype Central...
[Sonatype Central Upload] Successfully uploaded to Sonatype Central. Deployment ID: 08e78430-3b0e-41d1-b7d2-ee76db8badd9
[Sonatype Central Upload] Checking status of deployment...
[Sonatype Central Upload] Deployment status: PENDING. Waiting 5 seconds before checking status again...
[Sonatype Central Upload] Checking status of deployment...
[Sonatype Central Upload] Deployment status: PUBLISHING. Waiting 5 seconds before checking status again...
[Sonatype Central Upload] Checking status of deployment...
[Sonatype Central Upload] Deployment status: PUBLISHING. Waiting 5 seconds before checking status again...
[Sonatype Central Upload] Checking status of deployment...
[Sonatype Central Upload] Deployment status: PUBLISHING. Assuming everything is ok. For further information please check your Sonatype Central account (usually it takes 5-7 minutes to publish).

BUILD SUCCESSFUL in 1m 17s
8 actionable tasks: 8 executed
```

公開後は、Central Portalにライブラリのページが作成され、検索できるようになります。

## 6. (おまけ) GitHub Actionsを設定する

おまけとして、GitHub Actionsを使ってタグをつけたときに自動でライブラリを公開するための設定を紹介します。以下のような設定を `.github/workflows/publish.yml` に追加します。

この例では、productionという名前の[Environment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)を使用しており、Environment Secretsに手順5の環境変数と同じ名前と値でシークレットを設定していることを前提としています。

```yml
name: Publish to Central Repository

on:
  push:
    tags:
      - '*'

jobs:
  publish:
    runs-on: ubuntu-latest
    environment:
      name: production

    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: 11
          distribution: temurin
      - name: Publish to Central Repository
        run: ./gradlew sonatypeCentralUpload
        env:
          SONATYPE_CENTRAL_USERNAME: ${{ secrets.SONATYPE_CENTRAL_USERNAME }}
          SONATYPE_CENTRAL_PASSWORD: ${{ secrets.SONATYPE_CENTRAL_PASSWORD }}
          PGP_SIGNING_KEY: ${{ secrets.PGP_SIGNING_KEY }}
          PGP_SIGNING_KEY_PASSPHRASE: ${{ secrets.PGP_SIGNING_KEY_PASSPHRASE }}
```

## まとめ

Gradleを使ってMaven Centralにライブラリを公開する手順を紹介しました。GitHubを使っている場合、以前よりも簡単にライブラリを公開できるようになり、嬉しい限りです。繰り返しますが、Gradleを利用した手順やプラグインは発展途上のため、最新の情報は[公式ドキュメント](https://central.sonatype.org/register/central-portal/)を確認してください。

## 参考

- [Register to Publish Via the Central Portal \- The Central Repository Documentation](https://central.sonatype.org/register/central-portal/)
- [Im\-Fran/SonatypeCentralUpload: A gradle plugin to upload artifacts to Sonatype Central](https://github.com/Im-Fran/SonatypeCentralUpload)
- [グループIDがio\.github\.xxxのアーティファクトをMavenセントラルへ公開する with GitHub Actions](https://zenn.dev/backpaper0/articles/bc3c4854151f9a)
- [Yukiの枝折: Maven Central Repositoryにライブラリを公開する](https://yuki312.blogspot.com/2021/08/maven-central-repository.html)



