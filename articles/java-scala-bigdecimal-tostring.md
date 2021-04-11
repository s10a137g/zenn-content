---
title: "JavaのBigDecimalとScalaのBigDecimalで.toStringの表記が少し違う"
emoji: "👀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["java"|"scala"]
published: false
---


この前ScalaでBigDecimalをいじっていて、JavaのBigDecimalとで、toStringの出力結果が少し違うことに気づいたのでメモ。

```scala
val scalaBd1 = BigDecimal("9342342342342342342342342342342342342342342342342342342342342")
val scalaBd2 = BigDecimal("9999342342342342342342342342342342342342342342342342342342465")
val scalaBd3 = BigDecimal("19341684684684684684684684684684684684684684684684684684684807")

println((scalaBd1 + scalaBd2).toString) // 1.934168468468468468468468468468468468468468468468468468468481E+61 と表示される
println((scalaBd3).toString)            // 19341684684684684684684684684684684684684684684684684684684807 と表示される

val javaBd1 = new java.math.BigDecimal("9342342342342342342342342342342342342342342342342342342342342")
val javaBd2 = new java.math.BigDecimal("9999342342342342342342342342342342342342342342342342342342465")
val javaBd3 = new java.math.BigDecimal("19341684684684684684684684684684684684684684684684684684684807")
println((javaBd1.add(javaBd2)).toString)  // 19341684684684684684684684684684684684684684684684684684684807 と表示される
println((javaBd3.toString))               // 19341684684684684684684684684684684684684684684684684684684807 と表示される
```



以下の記事を発見。

[JavaのBigDecimalをtoStringしたときに指数表記(E)が出ないようにする](https://qiita.com/shotana/items/e02357516798e6bc658e)



記事中にJavaAPIのドキュメントリンクが貼ってある。

https://docs.oracle.com/javase/jp/8/docs/api/java/math/BigDecimal.html#toString--



Javaではスケールと指数で、表記が決定するらしい。

Scalaについても一応ドキュメントはあるけど、細かくは書いていなかった。

https://www.scala-lang.org/api/2.12.2/scala/math/BigDecimal.html#toString():String



ちなみにJavaだと、 `toEngineeringString` と `toPlainString` が用意されていて、指数表記とそうでない表記を分けて出力することもできるらしい。

特にtoStringで困ることもなさそうだけれど、でもあると便利なのかな。



(補足)
ScalaのBigDecimalは、JavaのBigDecimalのラッパークラスだけど、一部デフォルト設定値に注意しないとJavaとScalaでBigDecimalの結果結果が異なる場合もあるそうで、
ここらへんの精度の設定値差分あたりで表記も変わっているのかなーと少し思いました。
というかここらへんが理由だと出力結果が異なるというよりかは、計算結果が異なるんではという気が、、、後でもう少ししっかり調べます。

ScalaでBigDecimal使うときは要注意。

[Scala の BigDecimal と Java の BigDecimal とで計算結果が異なる](https://tnoda-scala.tumblr.com/post/105794370493/the-default-mathcontext-of-scala-math-bigdecimal)

