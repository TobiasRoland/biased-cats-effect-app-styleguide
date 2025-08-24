# Scala Circe JSON Testing Style Guide

## Overview

This guide establishes patterns for testing JSON codecs in Scala using Circe, prioritizing comprehensive verification
of encoder/decoder contracts through both property-based testing and explicit examples.

The core philosophy is on testing your codecs, not Circe itself, ensuring that your JSON serialization contracts remain
stable and correct.

This guide complements the [Circe JSON Encoding and Decoding Style Guide](circe-json-codec-styleguide.md) and assumes
you are following those implementation patterns.

## Core Principles

1. **Test your codecs, not Circe** - Focus on your encoder/decoder logic, not the library
2. **Roundtrip property testing** - Use ScalaCheck generators when both encoder and decoder exist
3. **Explicit JSON examples** - Use hardcoded examples with circe-literal for concrete verification
4. **Comprehensive error testing** - Test all failure modes your codecs can produce
5. **Single test class per model** - Group encoder and decoder tests for the same model together
6. **Exact structure verification** - Assert precise JSON structure, not just presence of fields
7. **Simple assertion patterns** - Use direct equality assertions, avoid unpacking Either values to keep things simple
8. **Reliable generators** - Use specific ScalaCheck methods, avoid `.suchThat` predicates

## Required Test Dependencies

Add these to your `build.sbt`:

```scala 3
libraryDependencies ++= Seq(
  // Core Circe (from main styleguide)
  "io.circe" %% "circe-core" % V.circe,
  "io.circe" %% "circe-literal" % V.circe,
  "io.circe" %% "circe-parser" % V.circe,
  
  // Testing dependencies
  "org.scalacheck" %% "scalacheck" % V.scalacheck % Test,
  
  // Choose ONE testing framework, munit or scalatest (depending on what the project already uses)
  "org.scalatest" %% "scalatest" % V.scalatest % Test,
  "org.scalatestplus" %% "scalacheck-1-17" % V.scalatestScalacheck % Test,
  // OR
  "org.scalameta" %% "munit" % V.munit % Test,
  "org.scalameta" %% "munit-scalacheck" % V.munit % Test
)
```

## Standard Test Imports

```scala 3
import io.circe.*
import io.circe.syntax.*
import io.circe.literal.*
import io.circe.testing.ArbitraryInstances.*
import org.scalacheck.{Arbitrary, Gen}

// ScalaTest
import org.scalatest.funspec.AnyFunSpec
import org.scalatest.matchers.should.Matchers
import org.scalatestplus.scalacheck.ScalaCheckPropertyChecks

// OR MUnit
import munit.{FunSuite, ScalaCheckSuite}
import org.scalacheck.Prop
```

## File Naming and Organization

**File naming:**

- `{ModelName}JsonSpec.scala` for individual models
- `{HierarchyName}JsonSpec.scala` for sealed trait hierarchies
- Place in `src/test/scala` mirroring the package structure of your codecs

**Test class structure:**

```scala 3
class ModelNameJsonSpec extends TestFramework {
  // ScalaCheck generators/arbitrary instances
  // Hardcoded examples  
  // Encoder tests
  // Decoder tests
  // Roundtrip tests
  // Error case tests
}
```

## ScalaCheck Generators for the tests

### ✅ Reliable Generator Patterns

```scala 3
given Arbitrary[User] = Arbitrary[User] {
  // Good - specific, reliable generators
  for {
    name          <- Gen.nonEmptyStringOf(Gen.alphaNumChar)                         // Never empty
    email         <- Gen.nonEmptyStringOf(Gen.alphaNumChar).map(_ + "@example.com")
    age           <- Gen.choose(18, 99)                                             // Constrained range
    preferences   <- Gen.nonEmptyListOf(Gen.alphaStr)                               // Never empty list
    optionalField <- Gen.option(Gen.posNum[Int])                                    // May be None
  } yield User(
    name = name, 
    email = email, 
    age = age, 
    preferences = preferences, 
    optionalField = optionalField, 
  )
  
  given Arbitrary[Hierarchy] = Arbitrary[Hierarchy] {
    // Generators for each child type
    val genChildA: Gen[ChildTypeA] = ??? // Define generator(s) here if small, or define it as a val in the file
    val genChildB: Gen[ChildTypeB] = ??? 
    val genChildC: Gen[ChildTypeC] = ??? 
    // Use Gen.oneOf to randomly pick one of the child generators
    Gen.oneOf(genChildA, genChildB, genChildC)
  }

// The property test itself
property("Hierarchy encoder/decoder should maintain equality through roundtrips") {
  Prop.forAll { (hierarchy: Hierarchy) =>
    val json = hierarchy.asJson
    val decoded = json.as[Hierarchy]
    decoded == Right(hierarchy)
  }

property("Hierarchy encoder/decoder should maintain equality through roundtrips") {
  Prop.forAll { (hierarchy: Hierarchy) =>
    val json = hierarchy.asJson
    val decoded = json.as[Hierarchy]
    decoded == Right(hierarchy)
  }
}

}  
```

### ❌ Avoid These Patterns

```scala 3
// Bad - can cause ScalaCheck to give up
name <- Gen.alphaNumStr.suchThat(_.nonEmpty)    // Use Gen.nonEmptyStringOf instead
age  <- Gen.posNum[Int].suchThat(_ < 100)       // Use Gen.choose(1, 99) instead
list <- Gen.listOf(gen).suchThat(_.length > 5)  // Use Gen.listOfN or specific generators
```

### Common Reliable Generator Methods

| Instead of `.suchThat`                 | Use this reliable method                 |
|----------------------------------------|------------------------------------------|
| `Gen.alphaNumStr.suchThat(_.nonEmpty)` | `Gen.nonEmptyStringOf(Gen.alphaNumChar)` |
| `Gen.listOf(gen).suchThat(_.nonEmpty)` | `Gen.nonEmptyListOf(gen)`                |
| `Gen.posNum[Int].suchThat(_ < 100)`    | `Gen.choose(1, 99)`                      |
| `Gen.alphaStr.suchThat(_.length == 5)` | `Gen.stringOfN(5, Gen.alphaChar)`        |

## Hardcoded JSON Examples

### Short JSON (< 250 characters) - Use circe-literal

```scala 3
private val validUserJson = json"""{
  "name": "John Doe",
  "email": "john@example.com",  
  "age": 30,
  "preferences": ["reading", "coding"],
  "score": 85
}"""

private val validUser = User(
  name = "John Doe",
  email = "john@example.com",
  age = 30,
  preferences = List("reading", "coding"),
  score = Some(85)
)
```

### Long JSON (≥ 250 characters) - Use test resources

```scala 3
// src/test/resources/test-data/complex-user-example.json
private lazy val complexUserJson: Json = {
  val jsonString = Source.fromResource("test-data/complex-user-example.json").mkString
  parser.parse(jsonString).getOrElse(fail("Failed to parse test JSON"))
}
```

## Testing Patterns by Framework

### ScalaTest with Should Matchers

```scala 3
class UserJsonSpec extends AnyFunSpec with Matchers with ScalaCheckPropertyChecks {
  
  describe("User encoder") {
    it("should produce expected JSON structure") {
      validUser.asJson.spaces2 should === (validUserJson.spaces2)
    }
    
    it("should handle optional fields when None") {
      userWithoutScore.asJson.spaces2 should === (userWithoutScoreJson.spaces2)
    }
  }
  
  describe("User decoder") {
    it("should parse valid JSON correctly") {
      validUserJson.as[User] should === (Right(validUser))
    }
  }
  
  describe("User roundtrip property") {
    it("should maintain equality through encode/decode cycles") {
      forAll { (user: User) =>
        val json = user.asJson
        val decoded = json.as[User]
        decoded should === (Right(user))
      }
    }
  }
  
  describe("User decoder error cases") {
    it("should fail for missing required name field") {
      val result = jsonMissingName.as[User]
      result shouldBe a[Left[_, _]]
      result.left.getOrElse(fail()).message should include ("name")
    }
  }
}
```

### MUnit

```scala 3
class UserJsonSpec extends ScalaCheckSuite {
  
  test("User encoder should produce expected JSON structure") {
    assertEquals(validUser.asJson.spaces2, validUserJson.spaces2)
  }
  
  test("User decoder should parse valid JSON correctly") {
    assertEquals(validUserJson.as[User], Right(validUser))
  }
  
  property("User encoder/decoder should maintain equality through roundtrip cycles") {
    Prop.forAll { (user: User) =>
      val json = user.asJson
      val decoded = json.as[User]
      decoded == Right(user)
    }
  }
  
  test("User decoder should fail for missing required name field") {
    val result = jsonMissingName.as[User]
    assert(result.isLeft)
    assert(result.left.getOrElse(fail("Expected Left")).message.contains("name"))
  }
}
```

## Core Testing Patterns

### 1. Encoder Testing - Exact Structure Verification

```scala 3
test("Model encoder should produce expected JSON structure") {
  val model = Model(field1 = "value1", field2 = 42)
  val expectedJson = json"""{
    "field1": "value1", 
    "field2": 42
  }"""
  
  // Assert exact JSON structure using .spaces2 for consistent formatting
  assertEquals(model.asJson.spaces2, expectedJson.spaces2)
}
```

### 2. Decoder Testing - Direct Right Equality

```scala 3
test("Model decoder should parse valid JSON correctly") {
  val json = json"""{
    "field1": "value1",
    "field2": 42  
  }"""
  
  val expectedModel = Model(field1 = "value1", field2 = 42)
  
  // Assert equality with Right(expectedValue), don't unpack the Either
  assertEquals(json.as[Model], Right(expectedModel))
}
```

### 3. Roundtrip Property Testing

```scala 3
property("Model encoder/decoder should maintain equality through roundtrip cycles") {
  Prop.forAll { (model: Model) =>
    val json = model.asJson
    val decoded = json.as[Model]
    decoded == Right(model) // Property must return Boolean
  }
}
```

### 4. Error Case Testing - Focus on Your Logic

```scala 3
test("Model decoder should fail for custom validation errors") {
  val invalidJson = json""""invalid_enum_value""""
  
  val result = invalidJson.as[Status]
  assert(result.isLeft)
  // Only test error messages you explicitly provide
  assert(result.left.getOrElse(fail()).message.contains("Unknown status"))
}
```

## Testing Enum Codecs

```scala 3
class StatusJsonSpec extends ScalaCheckSuite {
  
  test("Status encoder should produce correct string values") {
    assertEquals(Status.Active.asJson.spaces2, json""""active"""".spaces2)
    assertEquals(Status.Inactive.asJson.spaces2, json""""inactive"""".spaces2) 
    assertEquals(Status.Pending.asJson.spaces2, json""""pending"""".spaces2)
  }
  
  test("Status decoder should parse all valid values correctly") {
    assertEquals(json""""active"""".as[Status], Right(Status.Active))
    assertEquals(json""""inactive"""".as[Status], Right(Status.Inactive))
    assertEquals(json""""pending"""".as[Status], Right(Status.Pending))
  }
  
  test("Status decoder should fail for invalid enum values") {
    val result = json""""invalid"""".as[Status]
    assert(result.isLeft)
    assert(result.left.getOrElse(fail()).message.contains("Unknown status"))
  }
}
```

## Testing AST Hierarchies (Sealed Traits)

```scala 3
class HierarchyJsonSpec extends ScalaCheckSuite {
  
  // Test each branch individually
  test("ChildTypeA encoder should add discriminator field") {
    val child = ChildTypeA(field1 = "value1", field2 = "value2")
    val expectedJson = json"""{
      "field1": "value1",
      "field2": "value2", 
      "type": "childTypeA"
    }"""
    
    assertEquals(child.asJson.spaces2, expectedJson.spaces2)
  }
  
  test("ChildTypeA decoder should parse correctly") {
    val json = json"""{
      "field1": "value1",
      "field2": "value2",
      "type": "childTypeA" 
    }"""
    
    val expected = ChildTypeA(field1 = "value1", field2 = "value2")
    assertEquals(json.as[Hierarchy], Right(expected))
  }
  
  // Test discriminator validation
  test("Hierarchy encoder should add correct discriminator for all types") {
    val childA = ChildTypeA(field1 = "a", field2 = "b")
    val childB = ChildTypeB(fieldA = 42, fieldB = "test")  
    val childC = ChildTypeC(fieldX = "x", fieldY = Some(123L))
    
    assertEquals(childA.asJson.hcursor.get[String]("type"), Right("childTypeA"))
    assertEquals(childB.asJson.hcursor.get[String]("type"), Right("childTypeB")) 
    assertEquals(childC.asJson.hcursor.get[String]("type"), Right("childTypeC"))
  }
  
  // Test error cases
  test("Hierarchy decoder should fail for invalid discriminator") {
    val json = json"""{
      "field1": "value1",
      "field2": "value2",
      "type": "invalidType"
    }"""
    
    val result = json.as[Hierarchy]
    assert(result.isLeft)
    assert(result.left.getOrElse(fail()).message.contains("Unknown discriminator"))
  }
  
  test("Hierarchy decoder should fail for missing discriminator") {
    val json = json"""{
      "field1": "value1", 
      "field2": "value2"
    }"""
    
    val result = json.as[Hierarchy]
    assert(result.isLeft)
  }
}
```

## Advanced Patterns

### Testing Custom Transformations (.emap, .contramap)

```scala 3
class UserIdJsonSpec extends ScalaCheckSuite {
  
  test("UserId encoder should encode as underlying Long value") {
    val userId = UserId(12345L)
    val expectedJson = json"12345"
    
    assertEquals(userId.asJson.spaces2, expectedJson.spaces2)
  }
  
  test("UserId decoder should validate positive values") {
    assertEquals(json"12345".as[UserId], Right(UserId(12345L)))
    
    val negativeResult = json"-1".as[UserId]
    assert(negativeResult.isLeft)
    assert(negativeResult.left.getOrElse(fail()).message.contains("positive"))
  }
}
```

### Testing Nested JSON Structures

```scala 3
test("UserSummary should flatten nested API response") {
  val nestedJson = json"""{
    "user": {
      "profile": {
        "name": "John Doe",
        "email": "john@example.com"
      },
      "settings": {
        "theme": "dark"
      }
    },
    "timestamp": "2024-01-01T12:00:00Z"
  }"""
  
  val expected = UserSummary(
    name = "John Doe",
    email = "john@example.com", 
    theme = "dark",
    timestamp = "2024-01-01T12:00:00Z"
  )
  
  assertEquals(nestedJson.as[UserSummary], Right(expected))
}
```

## What NOT to Test

### ❌ Don't Test Circe's Built-in Behavior

```scala 3
// Don't test these - they're Circe's responsibility
test("should handle malformed JSON") { /* Circe handles this */ }
test("should fail for wrong primitive types") { /* Circe handles this */ }
test("should encode Lists correctly") { /* Circe handles this */ }
```

### ❌ Don't Test Implementation Details

```scala 3
// Don't test internal Circe mechanics
test("should use specific Circe decoder methods") { /* Implementation detail */ }
test("should create HCursor correctly") { /* Internal to Circe */ }
```

### ✅ DO Test Your Logic

```scala 3
// Test these - they're your responsibility  
test("should validate business rules in .emap") { /* Your validation logic */ }
test("should map field names correctly") { /* Your field mapping */ }
test("should handle your custom discriminator logic") { /* Your AST logic */ }
test("should encode/decode your domain models correctly") { /* Your codecs */ }
```

## Complete Example Template

```scala 3
class ModelJsonSpec extends ScalaCheckSuite {
  
 
  given Arbitrary[Model] = Arbitrary[Model] {
    for {
      field1 <- Gen.nonEmptyStringOf(Gen.alphaNumChar)
      field2 <- Gen.choose(1, 100)
      field3 <- Gen.option(Gen.nonEmptyStringOf(Gen.alphaChar))
    } yield Model(field1, field2, field3)
  }
  
  // Hardcoded examples
  private val validModelJson = json"""{
    "field1": "test",
    "field2": 42,
    "field3": "optional"
  }"""
  
  private val validModel = Model("test", 42, Some("optional"))
  
  // Encoder tests - exact structure
  test("Model encoder should produce expected JSON structure") {
    assertEquals(validModel.asJson.spaces2, validModelJson.spaces2)
  }
  
  // Decoder tests - direct Right equality
  test("Model decoder should parse valid JSON correctly") {
    assertEquals(validModelJson.as[Model], Right(validModel))
  }
  
  // Roundtrip property tests
  property("Model encoder/decoder should maintain equality through roundtrips") {
    Prop.forAll { (model: Model) =>
      val json = model.asJson
      val decoded = json.as[Model]
      decoded == Right(model)
    }
  }
  
  // Error case tests - focus on your logic
  test("Model decoder should fail for missing required fields") {
    val incompleteJson = json"""{"field2": 42}"""
    val result = incompleteJson.as[Model]
    assert(result.isLeft)
    assert(result.left.getOrElse(fail()).message.contains("field1"))
  }
}
```

## Key Benefits

- **Comprehensive Coverage**: Both property-based and example-based testing
- **Contract Verification**: Ensures JSON structure matches expectations exactly
- **Regression Prevention**: Roundtrip testing catches breaking changes
- **Error Validation**: Confirms your error handling works as designed
- **Maintainable**: Consistent patterns make tests easy to read and modify
- **Reliable**: Proper generators avoid flaky test failures
- **Documentation**: Hardcoded examples serve as living documentation of your JSON contracts

This approach ensures your JSON serialization remains stable and correct as your codebase evolves, providing confidence
in your API contracts and data persistence layers.

## LLM Usage Notes

When generating test code based on this guide:

1. **Always use reliable generators** - prefer `Gen.nonEmptyStringOf`, `Gen.choose`, `Gen.listOfN` over `.suchThat`
   predicates
2. **Generate complete test classes** - include generators, examples, and all test categories (encoder, decoder,
   roundtrip, errors)
3. **Match the user's testing framework** - adapt patterns for ScalaTest vs MUnit syntax
4. **Focus on user's codec logic** - don't generate tests for Circe's built-in functionality
5. **Use exact JSON structure verification** - always use `.spaces2` for consistent formatting
6. **Follow naming conventions** - `{ModelName}JsonSpec.scala` for files, descriptive test method names
7. **Include error cases** - test the failure modes that the user's codecs explicitly handle
