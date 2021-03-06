AccessorRegistry
================

The [AccessorRegistry](http://mediamath.github.io/scala-json/doc/index.html#json.tools.AccessorRegistry)
gives you a way of 'pickling' objects based on their underlying type. Unlike when
using the accessors implicitly, this requires a runtime lookup of the class type of a serialized/deserialized object to
locate the accessor. This can be especially handy in message-passing systems that may need to marshal
a blob without knowing its source/destination type. When combined with pattern matching, this can provide you
with an actor-like message passing system.

```tut
import json._
import json.tools.AccessorRegistry

@accessor case class TestClass(a: Int, b: Option[Int], c: String = "", d: Option[Int] = None)

AccessorRegistry.add[TestClass]
```

The AccessorRegistry also exposes itself as a selfless trait if you would like to encapsulate your own registry.

```tut
object MyRegistry extends AccessorRegistry
MyRegistry.add[TestClass]
```

You can also add singleton objects (these are bound by their name classname, case objects work best)

```tut
case object StartCommand
MyRegistry.add(StartCommand)
```

The AccessorRegistry also provides an accessor for the Scala
type ```Any```. When used carefully, this can allow you to serialize to and from the type ```Any``` assuming
the base types have been registered in the AccessorRegistry.

```tut
case class Envelope(msg: Any)
implicit val envAccessor = {
  import AccessorRegistry.anyAccessor //or MyRegistry.anyAccessor if using your own registry

  ObjectAccessor.create[Envelope]
}
AccessorRegistry.add[Envelope]

val inst = Envelope(TestClass(1, None))
val regularJS = inst.js
val pickeledJS = inst.js(AccessorRegistry.anyAccessor)
require(pickeledJS.toObject[Any](AccessorRegistry.anyAccessor) == inst)
require(regularJS.toObject[Envelope] == inst)
```