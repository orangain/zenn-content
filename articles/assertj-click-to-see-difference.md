---
title: "IntelliJ IDEAでAssertJを使うとClick to see differenceが表示されない場合の対処法"
emoji: "👀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["intellij", "assertj", "junit", "gradle"]
published: true
---

## はじめに

IntelliJ IDEAには、ユニットテストが失敗したときに assertEquals のexpected (期待した値) とactual (実際の値) の[差分をわかりやすく表示する機能](https://pleiades.io/help/idea/viewing-and-exploring-test-results.html#assert-equals)があります。

テストのログから差分が検出されると「Click to see difference」というリンクが表示され、それをクリックすると差分が左右に表示されます。

![Click to see differenceというリンク](/images/assertj-click-to-see-difference/click-to-see-difference.png)

![差分の表示](/images/assertj-click-to-see-difference/comparison-failure.png)

これは非常に便利な機能ですが、AssertJとJUnit 5とGradleを組み合わせた場合、次のように「Click to see difference」が表示されないことがあります。

![Click to see differenceが表示されない](/images/assertj-click-to-see-difference/problem.png)

## 対応策

この場合、以下のいずれかの対応をすると表示されるようになります。

1. AssertJではなくJUnit 5のアサーション (assertEquals) を使う。
2. JUnit 5ではなく、JUnit 4を使う。
3. テストランナーとして、Gradleではなく[IntelliJ IDEAを使う](https://pleiades.io/help/idea/work-with-tests-in-gradle.html#configure_gradle_test_runner)。

1を行った場合、次の画像のようにエラーメッセージの形式が変わります。Click to see differenceの実装には、ハードコードされた正規表現が使われているよう [^1] で、JUnit 5のエラーメッセージ形式はこの正規表現にマッチするため、差分が表示されます。逆に言うと、AssertJはエラーメッセージをJUnitのものよりわかりやすくしているために、IntelliJの正規表現にマッチしなくなっていたのです。

![対応策1](/images/assertj-click-to-see-difference/solution1.png)

2を行った場合も、1と同様にエラーメッセージの形式が変わり、差分が表示されるようになります。AssertJでは、JUnit 5がクラスパス上に見つからず、かつJUnit 4が見つかる場合に、JUnit 4の形式 (`org.junit.ComparisonFailure`) でエラーを出力するという実装 [^2] になっているためです。

![対応策2](/images/assertj-click-to-see-difference/solution2.png)

3を行った場合、IntelliJは差分をログの文字列から検知するのではなく、例外オブジェクト `org.opentest4j.AssertionFailedError` から直接expectedとactualの値を取得するようになるようです。これによって、エラーメッセージの形式を変更することなく、差分を検知できるわけです。ただし、テストランナーを変更すると、コンパイル速度が速くなる代わりに、予期しない差異が生まれる可能性があることに注意が必要です。

![対応策3](/images/assertj-click-to-see-difference/solution3.png)

## まとめ

以下のIssueにある通り、特別に対応しなくても差分が表示される方が嬉しいですが、正規表現で差分を検出するアプローチは多様なライブラリへの対応が難しいので、望みは薄いかもしれません。開発環境とCI環境で差異が出るのはなるべく避けたいではありますが、 DXを考えると上記の3の対応が現実的かなと考えています。

* [<Click to see diference> does not appear with org.opentest4j.AssertionFailedError when running tests with Gradle : IDEA-275065](https://youtrack.jetbrains.com/issue/IDEA-275065)
* [<Click to see difference> is not present in test output when running tests with Gradle and assertj assertions : IDEA-277216](https://youtrack.jetbrains.com/issue/IDEA-277216)


## 環境

試した環境は以下の通りです。

* IntelliJ IDEA 2021.2.3 (Community Edition)
* AdoptOpenJDK 14.0.2 
* Gradle 7.1
* AssertJ 3.21.0
* JUnit 5.8.1 / 4.13.2

## 参考

* [Support opentest4j in all tests not just JUnit 5 : IDEA-179482](https://youtrack.jetbrains.com/issue/IDEA-179482)
* [Use org.opentest4j.AssertionFailedError to report error when available · Issue #1072 · assertj/assertj-core](https://github.com/assertj/assertj-core/issues/1072)
* [IntelliJ doesn't show <Click to see difference> link with Groovy 2.5 · Issue #1364 · assertj/assertj-core](https://github.com/assertj/assertj-core/issues/1364)
* [Remove space in default error message for ShouldBeEqual to help IntelliJ diff detection · Issue #2199 · assertj/assertj-core](https://github.com/assertj/assertj-core/issues/2199)
* [Formatting output so that Intellij Idea shows diffs for two texts - Stack Overflow](https://stackoverflow.com/questions/10934743/formatting-output-so-that-intellij-idea-shows-diffs-for-two-texts)


[^1]: https://github.com/JetBrains/intellij-community/blob/3f7e93e20b7e79ba389adf593b3b59e46a3e01d1/plugins/testng/src/com/theoryinpractice/testng/model/TestProxy.java#L50

[^2]: https://github.com/assertj/assertj-core/blob/assertj-core-3.21.0/src/main/java/org/assertj/core/error/ShouldBeEqual.java#L123-L124
