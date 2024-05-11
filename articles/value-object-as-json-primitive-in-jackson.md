---
title: "Jacksonで単一プロパティを持つdata classをJSONのプリミティブ型として表現する"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["jackson", "kotlin"]
published: false
published_at: "2020-04-12"
---

Value Objectとして使っている、単一プロパティを持つdata classを、JacksonでシンプルなJSONとして表現するのにちょっと手こずったのでまとめておきます。

## 問題設定

例として、次のようなクラス定義を考えます。

```kt
data class User(val userId: UserId, val userName: UserName)
data class UserId(val value: Int)
data class UserName(val value: String)
```

UserオブジェクトをJacksonでシリアライズした場合、普通だと次のようになります。

```json
{"userId": {"value": 1}, "userName": {"value": "sato"}}
```

これを次のようにシリアライズした上で、元の形にデシリアライズしたいというのが、この記事の主題です。

```json
{"userId": 1, "userName": "sato"}
```

なお、 `UserId` や `UserName` は `User` 以外のクラスからも使われる可能性があります。これらを使うクラス間で挙動を統一できるよう、クラス定義に手を加えるなら `User` よりも `UserId` や `UserName` に加えたいです。

## 結論

次のように、  `UserId` や `UserName` にアノテーション `@JsonCreator` と `@JsonValue` をつけると実現できます。

```kt
data class User(val userId: UserId, val userName: UserName)
data class UserId @JsonCreator(mode = JsonCreator.Mode.DELEGATING) constructor(@JsonValue val value: Int)
data class UserName @JsonCreator(mode = JsonCreator.Mode.DELEGATING) constructor(@JsonValue val value: String)
```

## 解説

まず、 [@JsonValue](http://fasterxml.github.io/jackson-annotations/javadoc/2.9/com/fasterxml/jackson/annotation/JsonValue.html) はシリアライズに影響を与えるアノテーションで、シリアライズ結果を取得するメソッドに付けます。

次に、 [@JsonCreator](http://fasterxml.github.io/jackson-annotations/javadoc/2.9/com/fasterxml/jackson/annotation/JsonCreator.html) はデシリアライズに影響を与えるアノテーションで、JSONの値からオブジェクトを作るためのコンストラクターやファクトリーメソッドに付けます。Javaの場合はコンストラクターに `@JsonCreator` をつけるだけでいいのですが、Kotlin Moduleを使っている場合はうまく行かないので、 `mode = JsonCreator.Mode.DELEGATING` が必要です [^1]。

なお、Kotlinでプライマリコンストラクターにアノテーションをつけるには、 `constructor` キーワードが必要です [^2]。

## コード

実際のコードは次のようになります。

build.gradleの一部

```gradle
dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk8"
    implementation "com.fasterxml.jackson.core:jackson-databind:2.9.10.4"
    implementation "com.fasterxml.jackson.module:jackson-module-kotlin:2.9.10"
}
```

Main.kt

```kt
import com.fasterxml.jackson.annotation.JsonCreator
import com.fasterxml.jackson.annotation.JsonValue
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue

fun main() {
    val u = User(UserId(1), UserName("sato"))
    println(u)

    val mapper = jacksonObjectMapper()
    val json = mapper.writeValueAsString(u)
    println(json)

    val parsed: User = mapper.readValue(json)
    println(parsed)
}

data class User(val userId: UserId, val userName: UserName)
data class UserId @JsonCreator(mode = JsonCreator.Mode.DELEGATING) constructor(@JsonValue val value: Int)
data class UserName @JsonCreator(mode = JsonCreator.Mode.DELEGATING) constructor(@JsonValue val value: String)
```

実行結果

```
User(userId=UserId(value=1), userName=UserName(value=sato))
{"userId":1,"userName":"sato"}
User(userId=UserId(value=1), userName=UserName(value=sato))
```

## 参考

* java - How to instruct Jackson to serialize a field inside an Object instead of the Object it self? - Stack Overflow
https://stackoverflow.com/questions/11031110/how-to-instruct-jackson-to-serialize-a-field-inside-an-object-instead-of-the-object-it-self
* java - Is there a generic way to deserialize single-valued value objects (without custom Deserializer) in Jackson - Stack Overflow
https://stackoverflow.com/questions/55492657/is-there-a-generic-way-to-deserialize-single-valued-value-objects-without-custo


[^1]: @jsonCreator on primary constructor is not working · Issue #305 · FasterXML/jackson-module-kotlin
https://github.com/FasterXML/jackson-module-kotlin/issues/305
[^2]: https://kotlinlang.org/docs/reference/annotations.html#usage