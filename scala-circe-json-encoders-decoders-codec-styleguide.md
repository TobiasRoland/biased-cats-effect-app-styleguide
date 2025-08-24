# Circe JSON Encoding and Decoding Style Guide

## Overview

This guide establishes patterns for JSON codec implementation in Scala using Circe, prioritizing long-term
maintainability; API stability and explicit contracts over the initial perceived convenience of automatic
derivation. Handrolling encoders/decoders don't take many seconds, but ensures that the JSON structure is
predictable and stable.

Circe plays most other libraries, and even in projects without "typelevel" stack dependencies, I recommend it as the
default JSON library for Scala.

## Core Principles

Overarching principle: Your model *is not* your JSON, your JSON *is not* your model - they translate into each other
via the circe Encoder and Decoder.

1. **Never derive codecs automatically** - All codecs must be written manually
2. **Explicit contracts** - Ensure stable API between domain models and JSON
3. **Minimal code** - Define only what's necessary for encoding/decoding
4. **Clear field mapping** - Use named arguments and explicit key mapping
5. **Json is decoded as early as possible in application code** - avoid passing untyped
   json between code constructs and layers, for your own sanity - tracking down a bug related to a missing or wrong JSON
   property is a lot easier if it happens as soon as
   you receive the incoming json in a route than trying to fix it after it's ended
   up in your database as 'raw' json.
6. **Json is encoded as late as possible in application code** - avoid passing untyped
   json around and constructing it by introspecting the JSON; responses should remain in domain model form up until it's
   time to produce the final JSON response.
7. The models decoded/encoded should be immutable and devoid of side-effects and runtime exceptions.
8. Serialization models (ones with Encoder/Decoder) should be separate from your business logic models; don't add logic
   at your serialization/deserialization layer.

## Required dependencies

Assuming `V.circe` is the version of circe:

```
    // ✅ recommended baseline dependencies
    "io.circe" %% "circe-core"    % V.circe,
    "io.circe" %% "circe-literal" % V.circe,
    "io.circe" %% "circe-parser"  % V.circe,
    
    // 🚫 avoid both -generic and -semiauto dependencies
    "io.circe" %% "circe-generic"        % V.circe,
    "io.circe" %% "circe-generic-extras" % V.circe,
    "io.circe" %% "circe-semiauto"       % V.circe,
```

## Standard Imports

Always use wildcard imports to ensure all necessary implicits are in scope:

```scala 3
import io.circe.*
import io.circe.syntax.*
// if constructing hardcoded JSON:
import io.circe.literal.*
```

**Never import:**

```scala 3
// 🚫 Auto-derivation (forbidden)
import io.circe.generic.auto.*
import io.circe.semiauto.*

// 🚫 Individual imports (unnecessary unless namespace conflicts)
import io.circe.Encoder
import io.circe.Decoder
```

## Encoder Pattern

**Template:**

```scala 3
object ModelName {
  given modelNameEncoder: Encoder[ModelName] = x => Json.obj(
    "jsonKey1" := x.field1,
    "jsonKey2" := x.field2
  )
}
```

**Key Rules:**

- Place in companion object as `given` instance
- Use anonymous function with parameter named `x`
- Construct with `Json.obj` and `:=` operator
- Map each field explicitly

### Examples

**Single field:**

```scala 3
final case class Owner(address: String)

object Owner {
  given ownerEncoder: Encoder[Owner] = x => Json.obj(
    "address" := x.address
  )
}

final case class Pet(
  name: String, 
  age: Int, 
  owner: Owner
)

object Pet {
  given petEncoder: Encoder[Pet] = x => Json.obj(
    "name" := x.name,
    "age" := x.age,
    "owner" := x.owner
  )
}
```

## Decoder Pattern

**Single argument template:**

```scala 3
object ModelName {
  given modelNameDecoder: Decoder[ModelName] =
    _.downField("jsonKey").as[FieldType].map(ModelName.apply)
}
```

**Multiple arguments template:**

```scala 3
object ModelName {
  given modelNameDecoder: Decoder[ModelName] = c => for {
    field1 <- c.downField("jsonKey1").as[FieldType1]
    field2 <- c.downField("jsonKey2").as[FieldType2]
  } yield ModelName(
    field1 = field1,
    field2 = field2  
  )
}
```

**Key Rules:**

- Place in companion object as `given` instance
- Single arg: chain `_.downField`, `as`, and `map`
- Multi arg: use `for-comprehension` with anonymous function parameter `c`
- Always use named arguments on separate lines following the for-comprehension's `yield ModelName(`

### Examples

**Single field:**

```scala 3
object Owner {
  given ownerDecoder: Decoder[Owner] =
    _.downField("address").as[String].map(Owner.apply)
}
```

**Multiple fields:**

```scala 3
object Pet {
  given petDecoder: Decoder[Pet] = c => for {
    name <- c.downField("name").as[String]
    age <- c.downField("age").as[Int]
    owner <- c.downField("owner").as[Owner]
  } yield Pet(
    name = name,
    age = age,
    owner = owner
  )
}
```

## Complete Model Template

When both Encoding and Decoding are required:

```scala 3
final case class ModelName(
  field1: Type1,
  field2: Type2,
  field3: Type3
)

object ModelName {
  given modelNameEncoder: Encoder[ModelName] = x => Json.obj(
    "field1" := x.field1,
    "field2" := x.field2,
    "field3" := x.field3
  )
  
  given modelNameDecoder: Decoder[ModelName] = c => for {
    field1 <- c.downField("field1").as[Type1]
    field2 <- c.downField("field2").as[Type2]
    field3 <- c.downField("field3").as[Type3]
  } yield ModelName(
    field1 = field1,
    field2 = field2,
    field3 = field3
  )
}
```

## Dealing with Enums

* Avoid using the derived enum names; if code is refactored, the serialised value will change.

**Template: Enums**

```scala 3
enum Status {
  case Active, Inactive, Pending
}

object Status {
  
  given statusEncoder: Encoder[Status] = {
  // Explicit, non-derived naming so refactoring code won't refactor the serialised value.
    case Status.Active   => "active".asJson 
    case Status.Inactive => "inactive".asJson  
    case Status.Pending  => "pending".asJson
  }
  
  given statusDecoder: Decoder[Status] =
    // Explicit, non-derived naming so refactoring code won't refactor the expected serialised value.
    Decoder[String].emap {
      case "active"   => Right(Status.Active)
      case "inactive" => Right(Status.Inactive)
      case "pending"  => Right(Status.Pending)
      case otherwise  => Left(s"Unknown status, only supports 'active', 'inactive' and 'pending', was: '$otherwise'")
    }
}
```

## Dealing with Model Hierarchies (ASTs)

For sealed traits hierarchies, use the following guidelines:

* Define `sealed trait` with all possible cases
* Define case object only for the sealed trait "root"
* Define a private encoder and/or decoder (if required) for each case class in the case object for the sealed trait
* Define a public encoder and/or decoder (if required) for sealed trait that utilize the private encoders/decoders.
* If there already exists a discriminator field in the domain model, that can be utilized, otherwise, add a new field "
  type", "discriminator" or "kind" as the default name for discriminator fields in the public Encoder/Decoder.
* Use the discriminator field to pick which decoder to utilize. If not possible, fall back to
  "combined" encoders decoders that will try each encoder in turn until one succeeds (or all fail).

**Template: Encoding and Decoding AST with discriminator**

```scala 3
sealed trait Hierarchy
final case class ChildTypeA(field1: String, field2: String)       extends Hierarchy
final case class ChildTypeB(fieldA: Int, fieldB: String)          extends Hierarchy
final case class ChildTypeC(fieldX: String, fieldY: Option[Long]) extends Hierarchy

object Hierarchy {
  // --- Encoding ---
  // Define a private given encoder for each case class in the hierarchy.
  // There's nothing specific, apart from being `private` that differs from non-AST encoding. 
  private given childTypeAEncoder: Encoder[ChildTypeA] = x => Json.obj(
    "field1" := x.field1,
    "field2" := x.field2
  )
  private given childTypeBEncoder: Encoder[ChildTypeB] = x => Json.obj(
    "fieldA" := x.fieldA,
    "fieldB" := x.fieldB
  )
  private given childTypeCEncoder: Encoder[ChildTypeC] = x => Json.obj(
    "fieldX" := x.fieldX,
    "fieldY" := x.fieldY
  )
  
  // Define a public Encoder for the root trait that delegates to the private encoders/decoders that adds a discriminator field.
  given hierarchyEncoder: Encoder[Hierarchy] = {
    // we tack on an extra JSON field (typically named "type", "discriminator" or "kind") in the encoder for the root of the hierarchy.
    // this makes it possible for consumers of the raw JSON to match on the field and identify which model it is receiving. 
    case x: ChildTypeA => x.asJson.mapObject(_.add("type", "childTypeA".asJson)) // NOTE: the discriminator field is added to the JSON object, not the domain model itself.
    case x: ChildTypeB => x.asJson.mapObject(_.add("type", "childTypeB".asJson))
    case x: ChildTypeC => x.asJson.mapObject(_.add("type", "childTypeC".asJson))
  }
  // --- Decoding ---
  // Define a private given Decoder for each case class in the hierarchy. There's nothing specific in the child decoders that
  // differ from non-AST decoding. 
  private given childTypeADecoder: Decoder[ChildTypeA] = c => for {
    field1 <- c.downField("field1").as[String]
    field2 <- c.downField("field2").as[String]
  } yield ChildTypeA(
    field1 = field1,
    field2 = field2
  )  
  private given childTypeBDecoder: Decoder[ChildTypeB] = c => for {
    fieldA <- c.downField("fieldA").as[Int]
    fieldB <- c.downField("fieldB").as[String]
  } yield ChildTypeB(
    fieldA = fieldA,
    fieldB = fieldB
  )
  private given childTypeCDecoder: Decoder[ChildTypeC] = c => for {
    fieldX <- c.downField("fieldX").as[String]
    fieldY <- c.downField("fieldY").as[Option[Long]]
  } yield ChildTypeC(
    fieldX = fieldX,
    fieldY = fieldY
  )
  
  // Define a public Decoder for the root trait that delegates to the private encoders/decoders that delegates to the private decoders
  // by checking the discriminator field.
  given hierarchyDecoder: Decoder[Hierarchy] = c => for {
    discriminator <- c.downField("type").as[String]
    result        <- discriminator match {
      case "childTypeA" => c.as[ChildTypeA]
      case "childTypeB" => c.as[ChildTypeB]
      case "childTypeC" => c.as[ChildTypeC]
      case otherwise    => Decoder.failedWithMessage[Hierarchy](s"Unknown discriminator value '$otherwise', expected one of: childTypeA, childTypeB, childTypeC")(c)
    }
  } yield result
}
```

**Template: Decoding without discriminator**

```scala 3
// In case where there is no discriminator field, we can use a combined encoder/decoder that tries each encoder in turn until one succeeds (or all fail).
// This is subpar, and if you're in this situation, optimize so the most likely decoder is first in the list.
given hierarchyDecoder: Decoder[Hierarchy] = 
  List[Decoder[Hierarchy]](
    childTypeADecoder.widen, // we need to help Scala a bit by using "widen" on the type to make it a Decoder[Hierarchy] instead of a Decoder[ChildTypeA]
    childTypeBDecoder.widen,
    childTypeCDecoder.widen
  ).reduceLeft(_ or _) // we "combine" them all
```

## Decoding ASTs when there is no discriminator field

## Key Method Reference

| Method                    | Context             | Purpose                                                  | Example                                                                                      |
|---------------------------|---------------------|----------------------------------------------------------|----------------------------------------------------------------------------------------------|
| `Json.obj`                | Encoders            | Create JSON object                                       | `Json.obj("key" := value)`                                                                   |
| `:=`                      | Encoders            | Map field to JSON key                                    | `"name" := x.name`                                                                           |
| `downField("key")`        | Decoders            | Navigate to JSON field                                   | `c.downField("name")`                                                                        |
| `.as[Type]`               | Decoders            | Decode as specific type                                  | `.as[String]`                                                                                |
| `.map(Constructor.apply)` | Single-arg decoders | Transform decoded value                                  | `.map(Owner.apply)`                                                                          |
| `.emap`                   | Decoder             | validation/transformation with potential simple failures | `Decoder[Int].emap(i => if (i > 0) Right(i) else Left(s"int needs to be positive, got $i"))` |
| `.contramap`              | Encoders            | Encoder transformation without failure                   | `Encoder[Long].contramap[UserId](_.value)`                                                   |

## Forbidden Patterns

### ❌ Automatic Derivation

```scala 3
// Never use these imports
import io.circe.generic.auto.*
import io.circe.semiauto.*

// Never use these methods
deriveEncoder[Pet]
deriveDecoder[Pet]
```

### ❌ ForProductN Helpers

```scala 3
// Avoid - vulnerable to field reordering, and less readable as argument count goes up
Encoder.forProduct3("name", "age", "owner")(p => (p.name, p.age, p.owner))
Decoder.forProduct3("name", "age", "owner")(Pet.apply)
```

### ❌ Map-based Construction

```scala 3
// Avoid - loses type safety
Encoder.instance[Pet] { pet =>
  Json.fromFields(Map(
    "name" -> pet.name.asJson,
    "age" -> pet.age.asJson
  ))
}
```

## Advanced Patterns

### Custom JSON Field Names

```scala 3
final case class User(firstName: String, lastName: String)

object User {
  given userEncoder: Encoder[User] = x => Json.obj(
    "first_name" := x.firstName,  // Snake case in JSON
    "last_name" := x.lastName
  )
  
  given userDecoder: Decoder[User] = c => for {
    firstName <- c.downField("first_name").as[String]
    lastName <- c.downField("last_name").as[String]
  } yield User(
    firstName = firstName,
    lastName = lastName
  )
}
```

### Optional Fields

```scala 3
final case class Product(
  name: String, 
  description: Option[String]
)

object Product {
  given productEncoder: Encoder[Product] = x => Json.obj(
    "name" := x.name,
    "description" := x.description  // Circe handles Option automatically
  )
  
  given productDecoder: Decoder[Product] = c => for {
    name        <- c.downField("name").as[String]
    description <- c.downField("description").as[Option[String]]
  } yield Product(
    name = name,
    description = description
  )
}
```

### Collections

```scala 3
final case class Team(
  name: String, 
  members: List[Member]
)

object Team {
  given teamEncoder: Encoder[Team] = x => Json.obj(
    "name" := x.name,
    "members" := x.members  // Circe handles List[A] automatically when an encoder for A is in scope
  )
  
  given teamDecoder: Decoder[Team] = c => for {
    name    <- c.downField("name").as[String]
    members <- c.downField("members").as[List[Member]] // Circe handles List[A] automatically when a decoder for A is in scope
  } yield Team(
    name = name,
    members = members
  )
}
```

## Decoding/Encoding Nested JSON into Flat Case class structure

When working with APIs that return nested JSON structures, you often need to flatten them into simpler case classes.
Use chained `.downField` calls to navigate deep into the JSON structure.

**Pattern for nested navigation:**

```scala 3
c.downField("parent").downField("child1").downField("child2").as[Type]
```

### Example - API response with nested user data into flat structure

```scala 3
// Target JSON structure:
// {
//   "user": {
//     "profile": {
//       "name": "John Doe",
//       "email": "john@example.com"
//     },
//     "settings": {
//       "theme": "dark"
//     }
//   },
//   "timestamp": "2024-01-01T12:00:00Z"
// }

// Flat case class we want to decode into for convenience / less mess
final case class UserSummary(
  name: String,
  email: String,
  theme: String,
  timestamp: String
)

object UserSummary {
  given userSummaryEncoder: Encoder[UserSummary] = x => 
    Json.obj(
      "user" := Json.obj(
        "profile" := Json.obj(
          "name" := x.name,
          "email" := x.email
        ),
        "settings" := Json.obj(
          "theme" := x.theme
        )
      ),
      "timestamp" := x.timestamp
    )

  given userSummaryDecoder: Decoder[UserSummary] = c => for {
    name      <- c.downField("user").downField("profile").downField("name").as[String]
    email     <- c.downField("user").downField("profile").downField("email").as[String]
    theme     <- c.downField("user").downField("settings").downField("theme").as[String]
    timestamp <- c.downField("timestamp").as[String]
  } yield UserSummary(
    name = name,
    email = email,
    theme = theme,
    timestamp = timestamp
  )
}
```

## Advanced Codec Transformations

Encoder transformations are useful for:

* **Reuse Existing Codecs**: Leverage built-in encoders/decoders for primitive types
* **Clear Error Messages**: .emap provides descriptive validation failures
* **Type Safety**: Wrap primitive types in value classes with validation

### Using .emap for Decoder Validation

Use `.emap` to add validation logic or transform decoded values with potential failures:

#### Pattern:

```scala 3
given decoder: Decoder[Type] = 
  Decoder[RawType].emap(rawValue => 
    // validation/transformation logic that returns Either[String, Type]
  )
```

#### Examples:

```scala 3
final case class PositiveInt(value: Int)

object PositiveInt {
  given positiveIntEncoder: Encoder[PositiveInt] = x => Json.fromInt(x.value)
  
  given positiveIntDecoder: Decoder[PositiveInt] =
    Decoder[Int].emap { int =>
      if (int > 0) Right(PositiveInt(int))
      else Left(s"Expected positive integer, got: $int")
    }
}
```

### Using .contramap for Encoder Transformation

Use .contramap to transform a value before encoding, leveraging existing encoders:

#### Pattern:

```scala 3
given encoder: Encoder[Type] = 
  Encoder[BaseType].contramap[Type](_.extractBaseValue)
```

#### Example:

```scala 3
final case class UserId(value: Long)

object UserId {
  given userIdEncoder: Encoder[UserId] = Encoder[Long].contramap(_.value)
  given userIdDecoder: Decoder[UserId] = Decoder[Long].map(UserId.apply)
}

final case class Temperature(celsius: Double) {
  def fahrenheit: Double = celsius * 9.0 / 5.0 + 32.0
}

object Temperature {
  // Encode as Fahrenheit but store as Celsius internally
  given temperatureEncoder: Encoder[Temperature] = 
    Encoder[Double].contramap(_.fahrenheit)
  
  // Decode from Fahrenheit to Celsius
  given temperatureDecoder: Decoder[Temperature] = 
    Decoder[Double].map(f => Temperature((f - 32.0) * 5.0 / 9.0))
}
```

### Full example with validation and transformation

```scala 3
import java.time.LocalDate
import java.time.format.DateTimeFormatter
import java.time.format.DateTimeParseException

final case class DateWrapper(date: LocalDate)

object DateWrapper {
  private val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd")
  
  given dateWrapperEncoder: Encoder[DateWrapper] = 
    Encoder[String].contramap(_.date.format(formatter))
  
  given dateWrapperDecoder: Decoder[DateWrapper] = 
    Decoder[String].emap { str =>
      try {
        Right(DateWrapper(LocalDate.parse(str, formatter)))
      } catch {
        case _: DateTimeParseException => 
          Left(s"Invalid date format, expected yyyy-MM-dd: $str")
      }
    }
}
```

## Benefits

- **API Stability**: Internal model changes don't break JSON contract
- **Explicit Mapping**: Clear relationship between fields and JSON keys. Allows deviation between (sometimes very weird)
  JSON and the Scala model (e.g. snake_case vs camelCase vs kebab-case vs etc etc etc); keeps the scala models "sane",
  and keeps the JSON specific madness isolated to the JSON encoding/decoding.
- **Type Safety**: Compile-time verification prevents runtime errors
- **Field Ordering Independence**: Named arguments prevent field swap bugs
- **Maintainability**: Consistent patterns make code easy to understand
- **Refactoring Safety**: JSON structure remains stable during model evolution

This approach ensures consistent, maintainable, and stable JSON serialization across Scala applications using Circe