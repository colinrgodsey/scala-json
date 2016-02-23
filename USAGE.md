Usage and Examples
------------------

The intermediate type used by scala-json is the JValue. All intermediate JSON values extend [JValue](http://mediamath.github.io/scala-json/doc/index.html#json.JValue).
JValue types can be serialized to a JSON string by just using [JValue#toString](http://mediamath.github.io/scala-json/doc/index.html#json.JValue@toString(settings:json.JSONBuilderSettings,lvl:Int):String).
[JValue.fromString](http://mediamath.github.io/scala-json/doc/index.html#json.JValue$@fromString(str:String):json.JValue)
is used to create a JValue from a JSON string.

* Import the json package
```scala
scala> import json._
import json._

scala> JValue fromString "[1,2,3,4,5]" //parse JSON
res0: json.JValue = [1, 2, 3, 4, 5]

scala> JArray(JNull, JTrue) //create JSON
res1: json.JArray = [null, true]
```
* Implicit conversion to JValue types
```scala
scala> "hello".js
res2: json.JString = "hello"

scala> true.js
res3: json.JBoolean = true

scala> 1.7.js
res4: json.JNumber = 1.7

scala> def testMap = Map("hey" -> "there")
testMap: scala.collection.immutable.Map[String,String]

scala> val testMapJs = testMap.js
testMapJs: json.JObject =
{
  "hey": "there"
}

scala> Map("key" -> Seq.fill(3)(Set(Some(false), None))).js //even complex types
res5: json.JObject =
{
  "key": [[false, null], [false, null], [false, null]]
}

scala> testMap.keySet.headOption.js
res6: json.JValue = "hey"

scala> testMap.get("nokey").js
res7: json.JValue = null

scala> testMap.js.toDenseString
res8: String = {"hey":"there"}
```
* JS-like dynamic select
```scala
scala> require(testMapJs("nokey") == JUndefined)
```
* JS-like boolean conditions
```scala
scala> if(testMapJs("nokey")) sys.error("unexpected")
```
* JArrays as scala collections
```scala
scala> JArray(1, 2, 3, 4).map(_.toJString)
res11: json.JArray = ["1", "2", "3", "4"]

scala> JArray(1, 2, 3, 4).map(_.num) //drops down to Seq when working with non JValue types
res12: scala.collection.immutable.IndexedSeq[Double] = Vector(1.0, 2.0, 3.0, 4.0)

scala> JArray(1, 2, 3, 4) ++ JArray(5)
res13: json.JArray = [1, 2, 3, 4, 5]

scala> JArray(1, 2, 3, 4) ++ Seq(JNumber(5)) //can append any Iterable[JValue]
res14: json.JArray = [1, 2, 3, 4, 5]

scala> JArray(JObject.empty, JArray.empty) ++ Seq("nonjval") //adding a non JValue results in a normal Seq
res15: scala.collection.immutable.IndexedSeq[Object] = Vector({}, [], nonjval)
```
* JObjects as scala collections
```scala
scala> JObject("foo" -> 1.js, "a" -> false.js) ++ Map("bar" -> true).js - "a" //extends MapLike
res16: json.JObject =
{
  "foo": 1,
  "bar": true
}
```
* Compile-time case class marshalling
```scala
scala> case class TestClass(@name("FIELD_A") a: Int, b: Option[Int], c: String = "", d: Option[Int] = None) {
     |     @ephemeral def concat = a.toString + b + c + d //ephemeral fields get written but never read
     | }
defined class TestClass

scala> implicit val acc = ObjectAccessor.create[TestClass] //macro expands here to create the accessor
acc: json.internal.CaseClassObjectAccessor[TestClass] = CaseClassObjectAccessor

scala> val testClassJs = TestClass(1, None).js
testClassJs: json.JObject =
{
  "FIELD_A": 1,
  "b": null,
  "c": "",
  "d": null,
  "concat": "1NoneNone"
}

scala> val testClassJsString = testClassJs.toDenseString
testClassJsString: String = {"FIELD_A":1,"b":null,"c":"","d":null,"concat":"1NoneNone"}

scala> JValue.fromString(testClassJsString).toObject[TestClass]
res17: TestClass = TestClass(1,None,,None)
```
* Streamlined compile-time case class marshalling (requires [macro-paradise](#dependencies))
```scala
scala> @json.accessor case class SomeModel(a: String, other: Int)
defined object SomeModel
defined class SomeModel

scala> implicitly[JSONAccessor[SomeModel]] //accessor available in scope via hidden implicit
res18: json.JSONAccessor[SomeModel] = CaseClassObjectAccessor

scala> SomeModel("foo", 22).js
res19: json.JObject =
{
  "a": "foo",
  "other": 22
}
```
* Intermediate JObject from case class
```scala
scala> require(testClassJs("concat") != JUndefined) //ephemeral field exists

scala> JObject("FIELD_A" -> 23.js).toObject[TestClass] //using FIELD_A as renamed via @name annotation
res21: TestClass = TestClass(23,None,,None)

scala> TestClass(1, None).js + ("blah" -> 1.js) - "FIELD_A" //JObject supports MapLike operations
res22: json.JObject =
{
  "b": null,
  "c": "",
  "d": null,
  "concat": "1NoneNone",
  "blah": 1
}
```
* Dynamic field access
```scala
scala> val seqJson = Seq(TestClass(1, None), TestClass(1, Some(10), c = "hihi")).js
seqJson: json.JArray =
[{
  "FIELD_A": 1,
  "b": null,
  "c": "",
  "d": null,
  "concat": "1NoneNone"
}, {
  "FIELD_A": 1,
  "b": 10,
  "c": "hihi",
  "d": null,
  "concat": "1Some(10)hihiNone"
}]

scala> seqJson.dynamic(1).c.value
res23: json.JValue = "hihi"

scala> seqJson.dynamic.length
res24: json.JDynamic = 2

scala> require(seqJson.d == seqJson.dynamic)
```
* Typed exceptions with field data
```scala
scala> try {
     |   JObject("FIELD_A" -> "badint".js).toObject[TestClass]
     |   sys.error("should fail before")
     | } catch {
     |   case e: InputFormatException =>
     |     //returns all field exceptions for a parse, not just the first one!
     |     e.getExceptions.map {
     |       //detailed exception classes very useful for form validation
     |       case fieldEx: InputFieldException if fieldEx.fieldName == "FIELD_A" =>
     |         fieldEx.getMessage
     |       case x => sys.error("unexpected error " + x)
     |     }.mkString
     | }
res26: String = numeric expected but found json.JString (of value "badint")
```
