Circe
=====

#### A Quick Introduction
[Benen Cahill](https://github.com/benen)

---

## What is Circe?

* A library for interacting with Json using FP paradigms
  - exposes a JSON AST (Abstract Syntax Tree)
  - provides decoders instead of parsers
  - decouples parsing implementation (JAWN vs Jackson)

---

## What is Circe?

* Exposes Json through Types
  - better encapsulation of what is and isn't serializable
  - many runtime implementations rely on undocumented behaviors

---

## What is Circe?

* Developed with Performance in mind
  - maintainers not afraid to use mutable state and imperative paradigm where it makes sense

Note: 

* Encoder and Decoders are Type Classes

---

## Parsing Json

```scala
import io.circe._, io.circe.parser._

val rawJson = """{"id": 1, "name": "Benen"}"""

val parsedJson: Either[ParsingFailure, Json] =  parse(rawJson) 
```

Note: 

* At it's more basic level, provides a Json AST
* No runtime exceptions, all encapsulated in Either
* Coder decides whether to fail fast or accumlate all errors

---

## Traversing Json

```scala
  val doc = io.circe.parser.parse(
    """{
      |  "id": 1,
      |  "name":{
      |    "firstName":"Benen",
      |    "surname":"Cahill"
      |  }
      |}""".stripMargin)

  val cursor: HCursor = doc.getOrElse(Json.Null).hcursor

  val name: Decoder.Result[String] =
    cursor.downField("name").downField("firstName").as[String]

  val reversed: ACursor =
    cursor.downField("name").downField("firstName").withFocus(_.mapString(_.reverse))

  println(reversedNameCursor.top)
```

---

## Encoders And Decoders

* Type classes
* `Encoder[A]` provides `A => Json`
* Can be derived in different ways
  - Manual specification
  - Semi-automatic derivation
  - Fully-automatic derivation
  - Annotations

Note:

* Annotations are bad as they implicitly couple models to dependencies.  
* If you must use them, decouple the json model from your domain model!

---

## Manual Specification

```scala
implicit val encodeUser: Encoder[User] = new Encoder[User] {
  final def apply(user: User): Json = Json.obj(
    ("id", Json.fromInt(user.id)),
    ("name", Json.fromString(user.name))
  )
}

// or using productN helpers
implicit val encodeUser: Encoder[User] =
  Encoder.forProduct2("id", "name")(u => (u.id, u.name))
```

---

## Manual Specification

```scala
implicit val decoderUser: Decoder[User] = new Decoder[User] {
  final def apply(c: HCursor): Decoder.Result[User] = 
    for {
      id <- c.downField("id").as[Int]
      name <- c.downField("name").as[String]
    } yield User(id, name)
}

// or using productN helpers
implicit val decodeUser: Decoder[User] =
    Decoder.forProduct2("id", "name")(User.apply)
```

---

## Fully Automatic Derivation

```scala
import io.circe.generic.auto._

case class User(id: Int, name: String)

User(1, "Benen").asJson
```

Note: 

* Can be slow to compile, doesn't necessarily cache decoders
* Relies on shapeless under the hood
  - Uses Shapeless to derive a HList initially
  - Uses Shapeless Generics to derive typeclass from HList
  - Uses Shapeless `Lazy` to define recursive types
  - Uses Shapeless `LowPriority` to resolve implicit conflicts
* An implementation using macros possible, but hard to maintain

---

## Semi-Automatic Derviation

```scala
implicit val encodeUser: Encoder[User] = deriveEncoder[User]

implicit val decodeUser: Decoder[User] = deriveDecoder[User]

User(1, "Benen").asJson

```

Note: 

* You explicitly specify which encoders you want
* Little bit of boiler plate
* Feels less magical
* Results in the enoders being cached
* Reduces compilations quite significantly

---

## Annotations driven Derivation

```scala
import io.circe.generic.JsonCodec, io.circe.syntax._

@JsonCodec case class User(id: Int, name: String)

User(1, "Benen").asJson
```

Relies on an external plugin called [Macro Paradise](https://docs.scala-lang.org/overviews/macros/paradise.html)

---

## Optics

```scala
import io.circe.optics.JsonPath._

val firstNameOptic = root.name.firstName.string

val json = io.circe.parser.parse(
    """{
      |  "id": 1,
      |  "name":{
      |    "firstName":"Benen",
      |    "surname":"Cahill"
      |  }
      |}""".stripMargin)

val firstName: Option[String] = firstName.getOption(json)
```

Note: 

* Uses monacle under the hood

---

## Incomplete Decoders

```scala
case class PartialUser(name: String) {
  def toUser(id: Int) = User(id, name)
}

val posted = """{"name":"Benen"}"""

val partial = decode[Int => User](posted)

val benen = partial.map(_(1)) 
```

---

## Why use Circe? 

* Circe promotes safer typing
  - Jackson and most other json make extensive usage of _type casing_ (i.e. `_.asInstanceOf[T]`)
  - Circe uses type parameterization and derives type classes instead

---

## Why use Circe?

* Circe promotes better exception handling
  - Jackson and other libraries throw runtime exceptions
  - Circe exposes all parsing attempted as `Either[ParsingFailure, T]`
  - Circe makes it easy to implement _fail fast_ or error accumulation

---

## Why use Circe?

* Circe supports idiomatic Scala
  - Incomplete decoders: `Decoder[Int => User]`
  - Default arguments: (`circe-extras`)
  - Discrimators: `{"id": 1, "name":"Benen", "type":"User"}`
  - Enumeration ADTs

Note: 

* Circe uses a right biased Either, 2.10 and 2.11 will need to import cats Either

---

## References

* Official Circe Documentation 
  https://circe.github.io/circe/
* Generic Derivation: The Hard Parts
  https://www.youtube.com/watch?v=80h3hZidSeE 
