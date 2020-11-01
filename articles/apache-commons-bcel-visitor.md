---
title: "Apache Commons BCELでclassファイルのバイトコードをvisitする"
emoji: "🌲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kotlin", "jvm", "apachecommonsbcel"]
published: true
---
[Apache Commons BCEL](https://commons.apache.org/proper/commons-bcel/)でclassファイルをVisitorパターンで走査するコードを書きたかったのですが、意外とサンプルコードが見つからなかったので残しておきます。

## サンプルコード

サンプルコードはKotlin 1.4で書きました。次の3つのファイルを紹介します。

- build.gradle.kts
- ExampleVisitor.kt
- Main.kt

### build.gralde.kts

Gradleのdependenciesに[Apache Commons BCEL](https://mvnrepository.com/artifact/org.apache.bcel/bcel)を追加します。バージョン6.5.0を使いました。

```kt
dependencies {
    implementation("org.apache.bcel:bcel:6.5.0")
}
```

### ExampleVisitor.kt

Visitorを作るには、 [org.apache.bcel.classfile.Visitor](https://commons.apache.org/proper/commons-bcel/apidocs/org/apache/bcel/classfile/Visitor.html) インターフェイスを実装します。このインターフェイスを空のメソッドで実装した [org.apache.bcel.classfile.EmptyVisitor](https://commons.apache.org/proper/commons-bcel/apidocs/org/apache/bcel/classfile/EmptyVisitor.html) クラスが用意されているので、このEmptyVisitorを継承するのが楽です。

```kt
import org.apache.bcel.classfile.*
import org.apache.bcel.generic.ConstantPoolGen
import org.apache.bcel.generic.InstructionList
import org.apache.bcel.generic.InvokeInstruction

class ExampleVisitor : EmptyVisitor() {

    override fun visitJavaClass(jc: JavaClass) {
        println("Class: ${jc.className}")
    }

    override fun visitField(field: Field) {
        println("Field: ${field.name} ${field.signature}")
    }

    override fun visitMethod(method: Method) {
        println("Method: ${method.name}${method.signature}")
    }

    override fun visitCode(code: Code) {
        println("Code:")
        // メソッドの中身を表すCodeより下の階層は、Visitorのメソッドとしては用意されていないようなので、InstructionListを使ってアクセスする。
        val instructions = InstructionList(code.code)
        // 名前を取得する処理ではConstantPoolGenオブジェクトが必要になる。
        val constantPoolGen = ConstantPoolGen(code.constantPool)
        instructions.forEach { instructionHandle ->
            when (val instruction = instructionHandle.instruction) {
                // instructionの型に応じて処理を分岐する。
                is InvokeInstruction -> {
                    // 例: メソッド呼び出しの場合
                    println("  ${instruction.name}: ${instruction.getReferenceType(constantPoolGen)}.${instruction.getMethodName(constantPoolGen)}${instruction.getSignature(constantPoolGen)}")
                }
                else -> {
                    // その他の場合
                    println("  ${instruction.name}")
                }
            }
        }
    }

    override fun visitInnerClasses(innerClasses: InnerClasses) {
        innerClasses.innerClasses.forEach { innerClass ->
            println("InnerClass: ${innerClass.toString(innerClasses.constantPool)}")
        }
    }
}
```

### Main.kt

このVisitorを使って実際に階層を辿ってくれるのが、 [org.apache.bcel.classfile.DescendingVisitor](https://commons.apache.org/proper/commons-bcel/apidocs/org/apache/bcel/classfile/DescendingVisitor.html) です。 [org.apache.bcel.classfile.ClassParser](https://commons.apache.org/proper/commons-bcel/apidocs/org/apache/bcel/classfile/ClassParser.html) を使ってパースしたJavaClassと、Visitorのインスタンスを与えてDescendingVisitorのインスタンスを作成し、visitメソッドで走査します。

```kt
import org.apache.bcel.classfile.ClassParser
import org.apache.bcel.classfile.DescendingVisitor

fun main() {
    val classParser = ClassParser("/path/to/build/classes/kotlin/main/target/TargetKt.class")
    val javaClass = classParser.parse()
    val descendingVisitor = DescendingVisitor(javaClass, ExampleVisitor())
    descendingVisitor.visit()
}
```

## visit対象のファイル

例として、次のファイル `Target.kt` をビルドした結果のclassファイル `TargetKt.class` をvisitします。このサンプルコードは、JDBC URLをソースコードに書くことを推奨するものではありません。

```kt
package target

import org.jdbi.v3.core.Jdbi
import org.jdbi.v3.core.kotlin.mapTo
import java.time.LocalDate

fun main() {
    val jdbcUrl = "jdbc:postgresql://localhost:5432/dvdrental?user=postgres&password="
    val jdbi = Jdbi.create(jdbcUrl).installPlugins()

    jdbi.useHandle<Exception> { handle ->
        val sql = "SELECT * FROM customer"
        val result = handle.createQuery(sql)
                .mapTo<Result>()
                .list()
        println(result)
    }
}

data class Result(
    val customerId: Int,
    val storeId: Int,
    val firstName: String,
    val lastName: String,
    val createDate: LocalDate,
)
```

## 実行結果

実行結果は次のようになりました。Kotlinで定義しているmain関数は一つですが、TargetKtクラスにはシグネチャが異なる2つのメソッドがあることがわかります。

実行結果の最終行から、jdbi.useHandleの引数に指定しているラムダ式は、InnerClassとして別のクラスファイルにあることがわかります。同じファイルで定義していたdata classのResultも別のクラスファイルにあり、この実行結果には現れていません。

```
Class: target.TargetKt
Method: main()V
Code:
  ldc
  astore_0
  aload_0
  invokestatic: org.jdbi.v3.core.Jdbi.create(Ljava/lang/String;)Lorg/jdbi/v3/core/Jdbi;
  invokevirtual: org.jdbi.v3.core.Jdbi.installPlugins()Lorg/jdbi/v3/core/Jdbi;
  astore_1
  aload_1
  getstatic
  checkcast
  invokevirtual: org.jdbi.v3.core.Jdbi.useHandle(Lorg/jdbi/v3/core/HandleConsumer;)V
  return
Method: main([Ljava/lang/String;)V
Code:
  invokestatic: target.TargetKt.main()V
  return
InnerClass:   static final (anonymous)=class target.TargetKt$main$1
```

## 最後に

実装してから気づきましたが、classファイルのバイトコードに存在する階層構造は限られており、Visitorパターンを使うメリットはそれほどありません。ですが、何かの参考になるかもしれないので残しておきます。

## 参考
- [Apache Commons BCEL™ – The BCEL API](https://commons.apache.org/proper/commons-bcel/manual/bcel-api.html)
- [Apache BCEL でクラスファイルの情報を取得してみる \- プログラム日記](http://a4dosanddos.hatenablog.com/entry/2015/08/11/010336)
