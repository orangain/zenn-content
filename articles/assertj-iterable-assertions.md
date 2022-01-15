---
title: "AssertJでリストや配列の中身をチェックするアサーションのまとめ"
emoji: "🔀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["assertj"]
published: true
---

## はじめに

[AssertJ](https://assertj.github.io/doc/) はJVM系言語向けのアサーションライブラリです。AssertJでIterableやArrayを比較する場合、通常の `assertThat(actual).isEqualTo(expected)` の形式でも比較できますが、状況によっては専用のアサーションメソッドを利用した方が便利なこともあります。

IterableやArray用のアサーションについては、ドキュメントの [2.5.11. Iterable and array assertions](https://assertj.github.io/doc/#assertj-core-group-assertions) に記述があります。なかでも、リストや配列の中身をチェックするためのアサーションメソッドは、次の箇所にまとめられています。

https://assertj.github.io/doc/#assertj-core-group-assertions

ただ、メソッド名から正確な挙動を推測するのは難しく、それぞれの違いに頭を悩ませることがあるので、改めて整理してみます。

## 対象範囲

この記事では、上記のリンク先の表に書かれている次のメソッドとその変種、および [isEqualTo](https://www.javadoc.io/doc/org.assertj/assertj-core/latest/org/assertj/core/api/AbstractIterableAssert.html#isEqualTo(java.lang.Object)) を対象にします。

- [contains](https://www.javadoc.io/doc/org.assertj/assertj-core/latest/org/assertj/core/api/AbstractIterableAssert.html#contains(ELEMENT...))
- [containsOnly](https://www.javadoc.io/doc/org.assertj/assertj-core/latest/org/assertj/core/api/AbstractIterableAssert.html#containsOnly(ELEMENT...))
- [containsExactly](https://www.javadoc.io/doc/org.assertj/assertj-core/latest/org/assertj/core/api/AbstractIterableAssert.html#containsExactly(ELEMENT...))
- [containsExactlyInAnyOrder](https://www.javadoc.io/doc/org.assertj/assertj-core/latest/org/assertj/core/api/AbstractIterableAssert.html#containsExactlyInAnyOrder(ELEMENT...))
- [containsSequence](https://www.javadoc.io/doc/org.assertj/assertj-core/latest/org/assertj/core/api/AbstractIterableAssert.html#containsSequence(ELEMENT...))
- [containsSubsequence](https://www.javadoc.io/doc/org.assertj/assertj-core/latest/org/assertj/core/api/AbstractIterableAssert.html#containsSubsequence(ELEMENT...))
- [containsOnlyOnce](https://www.javadoc.io/doc/org.assertj/assertj-core/latest/org/assertj/core/api/AbstractIterableAssert.html#containsOnlyOnce(ELEMENT...))
- [containsAnyOf](https://www.javadoc.io/doc/org.assertj/assertj-core/latest/org/assertj/core/api/AbstractIterableAssert.html#containsAnyOf(ELEMENT...))

ドキュメントの2.5.11に書かれている、これ以降のコンテンツ（Satisy, Match, Navigating, Filtering, Exracting, Mapping, Comparatorなど）は対象外です。

検証した環境は次のとおりです。

- OpenJDK Runtime Environment Temurin-17.0.1+12
- Kotlin 1.6.10
- AssertJ 3.22.0

## アサーションの系統図

リストや配列の中身をチェックするためのアサーションの系統図を作ってみました。上のものほど比較が緩く、下に行くほど厳密な比較になります。図中の `actual` と `expected` はそれぞれ `assertThat(actual).メソッド(expected)` に入る値です。

緑色のメソッドは、対応する黄色のメソッドの変種で、expectedに複数の引数で要素を指定する代わりに、expectedをIterableにしたものです。

![アサーションの系統図](/images/assertj-iterable-assertions/assertion-tree.jpg)

系統図の左から2列目にあるメソッドを詳しく比較すると次の表のようになります。下に行くほど厳密な比較になっていることがわかると思います。

| メソッド                  | expectedの要素が多い | expectedの要素が少ない | 重複した要素をexpectedに1つだけ指定 | expectedと順序が違う |
|---------------------------|--------------------|----------------------|---------------------------|------------|
| containsAnyOf             | **OK**             | **OK**               | **OK**                    | **OK**     |
| contains                  | NG                 | **OK**               | **OK**                    | **OK**     |
| containsOnly              | NG                 | NG                   | **OK**                    | **OK**     |
| containsExactlyInAnyOrder | NG                 | NG                   | NG                        | **OK**     |
| containsExactly           | NG                 | NG                   | NG                        | NG         |

系統図の一番左側の列を比較すると次のようになります。こちらも下に行くほど厳密な比較になっています。

| メソッド            | expectedと順序が違う | expectedの要素間に別の値がある | expectedの方が少ない | 重複した要素をexpectedに1つだけ指定 |
|---------------------|------------|------------------------------|----------------------|---------------------------|
| contains            | **OK**     | **OK**                       | **OK**               | **OK**                    |
| containsSubsequence | NG         | **OK**                       | **OK**               | **OK**                    |
| containsSequence    | NG         | NG                           | **OK**               | **OK**                    |
| containsExactly     | NG         | NG                           | NG                   | NG                        |

## 実際のコードによる比較

実際のコードを見た方がわかりやすい場合もあるので、それぞれの違いがわかるようなテストケースを作成しました。Kotlinで記述しています。コメントアウトしていないアサーションはすべて成功し、コメントアウトしてあるアサーションはすべて失敗します。

```kt
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class IterableTest {
    private val list = listOf(0, 1, 1, 2, 3, 5)

    @Test
    fun testContainsAnyOf() {
        assertThat(list)
            .containsAnyOf(1)
            .containsAnyOf(4, 3, 2)
            //.containsAnyOf(4) // fail
            .containsAnyElementsOf(listOf(4, 3, 2))
            .doesNotContainAnyElementsOf(listOf(4, 6))
    }

    @Test
    fun testContains() {
        assertThat(list)
            .contains(1)
            .contains(5, 3, 2)
            //.contains(4, 3, 2) // fail
            .doesNotContain(4, 6)
            .doesNotContainAnyElementsOf(listOf(4, 6))
    }

    @Test
    fun testContainsOnly() {
        assertThat(list)
            .containsOnly(5, 3, 2, 1, 0)
            //.containsOnly(5, 3, 2) // fail
    }

    @Test
    fun testContainsExactlyInAnyOrder() {
        assertThat(list)
            .containsExactlyInAnyOrder(5, 3, 2, 1, 1, 0)
            //.containsExactlyInAnyOrder(5, 3, 2, 1, 0) // fail
            .containsExactlyInAnyOrderElementsOf(listOf(5, 3, 2, 1, 1, 0))
    }

    @Test
    fun testContainsExactly() {
        assertThat(list)
            .containsExactly(0, 1, 1, 2, 3, 5)
            //.containsExactly(5, 3, 2, 1, 1, 0) // fail
            .containsExactlyElementsOf(listOf(0, 1, 1, 2, 3, 5))
    }

    @Test
    fun testIsEqualTo() {
        assertThat(list)
            .isEqualTo(listOf(0, 1, 1, 2, 3, 5))
            //.isEqualTo(arrayOf(0, 1, 1, 2, 3, 5)) // fail
    }

    @Test
    fun testContainsSubsequence() {
        assertThat(list)
            .containsSubsequence(1, 1, 2, 5)
            //.containsSubsequence(3, 2, 1) // fail
            .doesNotContainSequence(3, 2, 1)
    }

    @Test
    fun testContainsSequence() {
        assertThat(list)
            .containsSequence(1, 1, 2, 3)
            //.containsSequence(1, 1, 2, 5) // fail
            .doesNotContainSequence(3, 2, 1)
    }

    @Test
    fun testContainsOnlyOnce() {
        assertThat(list)
            .containsOnlyOnce(0, 2, 3)
            //.containsOnlyOnce(1) // fail
            .containsOnlyOnceElementsOf(listOf(0, 2, 3))
    }
}
```

ソースコード全体は以下のリポジトリに置いてあります。

https://github.com/orangain/assertj-iterable-assertions

## まとめ

AssertJのリストや配列の中身をチェックするアサーションを整理してみました。個人的によく使うのは、順序を気にせず比較する **`containsExactlyInAnyOrder`** と、そのexpectedをIterableにした **`containsExactlyInAnyOrderElementsOf`** です。

これらよりも緩く比較したい場面はあまり多くなく、順序が決まっている場合は `containsExactly` や `containsExactlyElementsOf` よりも、 **`isEqualTo`** を使うことが多いです。
