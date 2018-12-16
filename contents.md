# Scala Code Reading
## spray-json 編

</br>

2018-12-18

@orepuri

---

### 本日の内容

</br>

spray-jsonでJSON変換がどう実装されているかを見る

---

### なぜライブラリのコードを読むのか

</br>

開発時間の短縮にライブラリを使った開発が必須

ライブラリの使い方を学習するのにはコストがかかる

コードを読むことでライブラリの深い理解</br>
(使い方や隠れた機能)が得られる

優れた設計やイデオムを学習し, 自分のコードに活かすことができる

---

### spray-json

https://github.com/spray/spray-json

</br>

ScalaのJSONライブラリ

シンプル/非外部依存/型クラス

16ファイル 1,644行 (v1.3.5)

---

### spray-jsonの変換

</br>

JSON文字列 ↔ JSON AST ↔ 独自クラス

<img src="./images/pic1.png" />

---

### JSON AST (抽象構文木)

```scala
// JSON値を表す抽象クラス
sealed abstract class JsValue { ... }

// 文字列のJSON値
case class JsString(value: String) extends JsValue { ... }

// 数値のJSON値
case class JsNumber(value: BigDecimal) extends JsValue { ... }

// 真偽のJSON値
sealed abstract class JsBoolean extends JsValue { ... }
case object JsTrue extends JsBoolean { ... }
case object JsFalse extends JsBoolean { ... }
```

---

### JSON AST (抽象構文木)

```scala
// nullのJSON値
case object JsNull extends JsValue

// オブジェクトのJSON値
// キー(文字列) と 値(任意のJSON値) のMapで表現
case class JsObject(fields: Map[String, JsValue]) extends JsValue { ... }

// 配列のJSON値
// 値(任意のJSON値) のVectorで表現
case class JsArray(elements: Vector[JsValue]) extends JsValue { ... }

```

---

### JSON AST (抽象構文木)

```scala
import spray.json._

JsArray(JsString("orepuri"), JsNumber(38))
// spray.json.JsArray = ["orepuri",38]

JsObject(Map(
  "name" -> JsString("orepuri"), 
  "age" -> JsNumber(38), 
  "isEngineer" -> JsTrue
))
// spray.json.JsObject = {"name":"orepuri","age":38,"isEngineer":true}

```

---

### JSON文字列 → JSON AST

```scala
import spray.json._

val jsonString = """{"name":"orepuri","age":38,"isEngineer":true}"""
// String = {"name":"orepuri","age":38,"isEngineer":true}

jsonString.parseJson
// spray.json.JsValue = {"name":"orepuri","age":38,"isEngineer":true}
```

jsonStringの型はString

ScalaのStringクラスにparseJsonは定義されてない

---

### parseJsonの定義

```scala
// spray/json/package.scala
package object json {
  ...
  implicit def enrichString(string: String) = new RichString(string)
  ...
  private[json] class RichString(string: String) {
    ...
    def parseJson: JsValue = JsonParser(string) // JsonParser.apply(string)
    ...
  }
```

Scalaの暗黙の型変換によりRichString#parseJsonが呼ばれる

JsonParserがJsValueに変換する

---

### parseJsonの呼び出し

```scala
// enrichStringやRichStringをimport
import spray.json._

val jsonString = """{"name":"orepuri","age":38,"isEngineer":true}"""

jsonString.parseJson
// => enrichString(jsonString).parseJson
// => new RichString(jsonString).parseJson
```

---

### JSON文字列 ← JSON AST

```scala
import spray.json._

val user = JsObject(Map(
  "name" -> JsString("orepuri"), 
  "age" -> JsNumber(38), 
  "isEngineer" -> JsTrue
))
// spray.json.JsObject = {"name":"orepuri","age":38,"isEngineer":true}

user.compactPrint // CompactPrinterクラスで実装
// String = {"name":"orepuri","age":38,"isEngineer":true}

user.toString // compactPrintと同じ
// String = {"name":"orepuri","age":38,"isEngineer":true}
```
---

### JSON文字列 ← JSON AST

```scala
user.prettyPrint // PrettyPrinterクラスで実装
// String =
{
  "name": "orepuri",
  "age": 38,
  "isEngineer": true
}

user.sortedPrint // SortedPrinterクラスで実装
// String =
{
  "age": 38,
  "isEngineer": true,
  "name": "orepuri"
}
```

---

### JSON文字列 ← JSON AST

```scala
sealed abstract class JsValue {
  ...
  def toString(printer: (JsValue => String)) = printer(this)
```

```scala
val myPrinter: JsValue => String = {
  case JsObject(keyValues) =>
    keyValues.map { case (key,value) => s"$key = $value" }.mkString(",")
}
user.toString(myPrinter)
// String = name = "orepuri",age = 38,isEngineer = true
```
独自の変換も実装できる

---

### JSON AST ↔ 独自クラス

```scala
import spray.json._

// AST
val userAst = JsObject(Map(
  "name" -> JsString("orepuri"), 
  "age" -> JsNumber(38), 
  "isEngineer" -> JsTrue
))
// spray.json.JsObject = {"name":"orepuri","age":38,"isEngineer":true}

// 独自クラス
case class User(name: String, age: Int, isEngineer: Boolean)
```
---

### JSON AST ↔ 独自クラス

convertTo, toJsonを使う

```scala
// AST to User defined class
userAst.convertTo[User]

// User defined class to AST
val user = User("orepuri", 38, true)
user.toJson

// error: Cannot find JsonReader or JsonFormat type class for User
// error: Cannot find JsonWriter or JsonFormat type class for User

```

JsonReader / JsonWriter / JsonFormat ???

---

### JsonReader/JsonWriter/JsonFormat

```
trait JsonReader[T] {
  def read(json: JsValue): T
}

trait JsonWriter[T] {
  def write(obj: T): JsValue
}

trait JsonFormat[T] extends JsonReader[T] with JsonWriter[T]

```

JSON AST(JsValue) と 任意の型T との相互変換

convertTo, toJsonにはJsonFormatが必要

---

### JsonFormatを定義する

```scala
implicit val userFormat: JsonFormat[User] = new JsonFormat[User] {
  override def read(json: JsValue): User = {
    json.asJsObject.getFields("name", "age", "isEngineer") match {
      case Seq(JsString(name), JsNumber(age), JsBoolean(isEngineer)) =>
        new User(name, age.toInt, isEngineer)
      case _ =>
        throw new DeserializationException(
          "User must have name, age and isEngineer"
        )
    }
  }
  ...
```
---

### JsonFormatを定義する

```scala
  override def write(user: User): JsValue = {
    JsObject(Map(
      "name" -> JsString(user.name),
      "age" -> JsNumber(user.age),
      "isEngineer" -> JsBoolean(user.isEngineer)
    ))
  }

```
---

### JSON AST ↔ 独自クラス

```scala
// import userFormat

val userAst = JsObject(Map(
  "name" -> JsString("orepuri"), 
  "age" -> JsNumber(38), 
  "isEngineer" -> JsTrue
))

val user = userAst.convertTo[User]
// User = User(orepuri,38,true)

user.toJson
// spray.json.JsValue = {"name":"orepuri","age":38,"isEngineer":true}
```

---

### convertTo (JSON AST → ユーザー定義クラス)

```scala
sealed abstract class JsValue {
  ...
  def convertTo[T :JsonReader]: T = jsonReader[T].read(this)

```

[T :JsonReader] はContext boundと呼ばれる</br>
シンタックスシュガーで下記と同等

```scala
  def contertTo[T](implicit ev: JsonReader[T]): T = jsonReader[T].read(this)
```

jsonReaderの定義

```scala
// in json package object
  ...
  def jsonReader[T](implicit reader: JsonReader[T]) = reader
```

---

### convertTo (JSON AST → ユーザー定義クラス)

```scala
  // オリジナルの実装
  def convertTo[T :JsonReader]: T = jsonReader[T].read(this)

  // Context boundをimplicitパラメータに書き換え
  def convertTo[T](implicit ev: JsonReader[T]): T = jsonReader(ev).read(this)

  // jsonReaderを置き換え
  def convertTo[T](implicit ev: JsonReader[T]): T = ev.read(this)
```

convertToはimplicitパラメータのJsonReader[T]#readを呼ぶ

なのでconvertToにはJsonReader(or JsonFormat)が必要

---

### toJson (JSON AST ← ユーザー定義クラス)

```scala
case class User(name: String, age: Int, isEngineer: Boolean)

val user = User("orepuri", 38, true)

user.toJson
```

UserクラスにはtoJsonが定義されてない


---

### toJson (JSON AST ← ユーザー定義クラス)

```scala
// spray/json/package.scala
package object json {

  implicit def enrichAny[T](any: T) = new RichAny(any)

...
package json {

  private[json] class RichAny[T](any: T) {
    def toJson(implicit writer: JsonWriter[T]): JsValue = writer.write(any)
  }
```

JSON文字列 → JSON ASTで見たparseJson(RichString)と似た構造

toJsonはimplicitパラメータのJsonWriter[T]#writeを呼ぶ

なのでtoJsonにはJsonWriter(or JsonFormat)が必要

---

### toJson (JSON AST ← ユーザー定義クラス)

```scala
// UserのJsonFormat
implicit val userFormat: JsonFormat[User] = new JsonFormat[User] { ... }

val user = User("orepuri", 38, true)
user.toJson

// は下記のコードになる
new RichAny(user).toJson(userFormat)
```

---

### まとめ

</br>

spray-jsonのJSONの変換がどう実装されているかみた

JSON文字列はJsonParserによりASTに変換

ユーザー定義クラスはJsonFormatによりASTとの相互変換

暗黙の型変換により実装

---

### 今日話せなかったこと

</br>

RootJsonFormat

JsonFormatの自動生成(jsonFormatN)

型クラスとしてのJsonFormat

なぜこのようなインタフェースや実装になっているか


