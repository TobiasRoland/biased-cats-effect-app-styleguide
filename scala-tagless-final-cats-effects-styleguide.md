# Instantiating business logic in a Cats Effects Service

The overarching concept here is to keep traits devoid of implementation details, and only express which **capabilities**
are required to instantiate the trait when you implement that trait.

## Core Principles

* **Traits Define the Contract, Not the Implementation:** Traits should be kept simple and abstract, declaring methods
  within a generic effect `F[_]`. They must not contain logic or specify effect capabilities (e.g., [F[_]: Parallel]).
* **Implementations Live in Companion Objects:** To prevent dependency on concrete classes, business logic is
  instantiated via methods (commonly apply) on the trait's companion object.
* **Request Minimal Power:** The implementation's constructor method explicitly declares the capabilities it needs via
  typeclass constraints (e.g., `def apply[F[_]: Async]`). This tells readers exactly what to expect from the code (e.g.,
  asynchronicity) and what not to expect (e.g., parallelism, if `: Parallel` is absent).
* **Instantiation is Pure:** The constructor (apply) method should return the service instance directly (
  `MyService[F]`), not wrapped in an effect (`F[MyService[F]]`), nor Resource (`Resource[F, MyService[F]]`). Effectful
  setup should be handled externally to business logic instantiation, and passed in as explicit dependencies.

## How to create application logic

Readers of the code will then be able to quickly look at the function signature, say:

```scala 3
// trait - devoid of logic, only defines the shape of the 'contract'
trait MyBusinesLogic[F[_]] { // this F[_] should be read as "effect" - implementers of this trait can decide what effect they want to use
  def process(input: String): F[Int] // ... and we return values wrapped in the effect. We haven't yet determined what that effect is!
}

object MyBusinessLogic {
  // Now, when it comes time to instantiate the trait, we can see that we need access to parallel processing. 
  // We can also see it ISN'T going to be dealing with asynchronicity, because it doesn't have `: Async` as a capability of the F.
  // we're limiting our toolbox for the implementation to just have parallel processing - we're asking for the minimum amount of power we need to implement something,
  // and future readers can immediately see what they can expect from the following code.
  def apply[F[_]: Parallel]( 
    someDependency: SomeDependency[F]
  ): MyBusinesLogic[F] = new MyBusinesLogic[F] {
      override def process(input: String): F[Int] = ???
    }
}
```

Code asks for as few capabilities as possible, and they do that via the constraints (the
`F[_]: Capability1: Capability2: Capability3]`).

So to summarise, we read the apply method as "This coming block of code has access to an effect (`F[_]`), and that
effect is the ability to do parallelism (`: Parallel`). It also tells readers, by the **absence** of an effect, what NOT
to expect - for example, we don't see `: Async` as a capability for the `F[_]`, so this code won't be dealing with
asynchronicity.

## On instantiating logic

When it comes to instantiating business logic, prefer traits with companion objects.

The fact that this trait does something effectful is expressed at the trait level completely generically with a `F[_]` -
we aren't
at the trait saying that all implementations of this trait are going to be a specific kind of effectful.

```scala 3
// === Do ===
// Do specify that "This trait is effectful" with a [F[_]]
trait ContentPublisher[F[_]] {    // Do this
  def publishEpisode(episode: Episode): F[Either[PublishingError, Unit]]
}

// ===  DO NOT ===
trait ContentPublisher {
  // do not specify the F[_] at the method level
  def publishEpisode[F[_]](episode: Episode): F[Either[PublishingError, Unit]] // Don't do this
}

// === DO NOT ===
// do not specify typeclasses in traits
trait ContentPublisher[F[_]: Parallel] {
  def publishEpisode(episode: Episode): F[Either[PublishingError, Unit]]
}
```

as you can see, traits should be completely devoid of logic, no specific capabilities provided at the trait level.

We save that for the implementation. For implementation, prefer a method on a companion object to a class. Classes are
easy to accidentally code against. A common review note for graduates and juniors is that they've accidentally added a
dependency against an implementation (`publisher: ContentPublisherImpl`) instead of the trait (
`publisher: ContentPublisher`), and with a trait+companion-object-with-apply pattern, we ensure that coders can only
depend on the trait; because there simply *is* no`...Impl` to accidentally rely on.

The reason I find this helpful, is that it gently guides the coder towards a certain type of useage, which in turn leads
to a very tangible consistency throughout the application.

```scala 3
/** Because there is only one implementation
* in this microservice of ContentPublisher... */
object ContentPublisher {

  /** ... then we use the `apply` method, since
  * there's no need to give a specific name to the implementation
  *
  * Note that we only put the smallest effects necessary to
  * implement the business code in the context bound on F[_] below.
  * This helps us see at a glance how "powerful" this code is going to
  * be, it sets the context for what we are about to read.
  * - typically, the only types that "go on the F" are from cats/cats-effects.
  * This keeps the cognitive burden for a newcomer to the code low(er).
  *
  * Because this implementation doesn't live as a
  * `class ContentPublisherImplementation {}`,
  * it is impossible for other people to specify the dependency on this
  * specific implementation - they can only specify a dependency on the interface (trait).
  */
  def apply[F[_]: MinimalEffectNeeded: MaybeSomeOtherEffectToo](
    /* all the other dependencies are passed explicitly */
    publisher: Publisher[F],
    translator: EpisodeTranslator[F],

   /* the `apply` method does NOT return `F[ContentPublisher]` nor
     `Resource[F, ContentPublisher[F]]` - just `ContentPublisher[F]`.
     If it WAS being returning something wrapped in Resource or F,
     that would indicates a good candidate for refactoring.
     If the ContentPublisher would need to do something effectful
     to instantiate itself, that's a sign that you should extract that
     out and pass it in as an argument like the publisher or translator above instead. */
  ): ContentPublisher[F] = new ContentPublisher[F] {
      /*  */
      override def publishEpisode(episode: Episode): F[Either[PublishingError, Unit]] =
        translator
          .translateEpisode(episode)  // F[EpisodeMessage]
          .flatMap(publisher.publish) // F[Either[PublishingError, Unit]]
    }
}
```

In cases where there are multiple implementations of the same trait, we abandon the singular `apply` - since there now
actually ARE different ways to instantiate the trait - and instead provide specifically named methods. If you manage to
keep your traits fairly small, just putting them in the companion object is fine:

```scala 3
object ContentPublisher {
   def kafka[F[_]: FlatMap](
     someDependency: SomeDependency[F]
   ): ContentPublisher[F] = ???

   def sqs[F[_]: FlatMap: Parallel](
     someOtherDependency: SomeOtherDependency[F]
   ): ContentPublisher[F] = ???
}
```

...but for legibility as the project grows, you may want to specify separate objects:

```scala 3
object KafkaContentPublisher {
   def apply[F[_]: FlatMap](
     someDependency: SomeDependency[F]
   ): ContentPublisher[F] = ???
}
```

.. possibly even in separate files, if these grow large.

```scala 3
object SqsContentPublisher
   def apply[F[_]: FlatMap: Parallel](
     someOtherDependency: SomeOtherDependency[F]
   ): ContentPublisher[F] = ???
}

```

# I've tricked you into understanding Tagless Final now

Some will by now have recognised this as a pattern as what is known as 'tagless final', or even more jargon-y as '
tagless final encoding'. I think this naming is unfortunate, and one of the haskellisms we've inherited that makes it
fairly hard to talk to "normal" coders. Once you know the term, you can (and should!) use it to talk to other people who
know the term, but when dropping it into a conversation with a more general population of coders, it comes across as an
arcane invocation of jargon. Be careful of this, especially with large teams and with people who aren't from a hardcore
FP-background.

I typically try and teach the mechanics of it, before revealing to people that that is what tagless final is - because
outside of its mathsy name, it simply just specifies the capabilities that you need on the `F[_]`. Ignore the scary name
and think of it as how to write abstract (tagless) code and supplying the concrete details about where those abstract
capabilities come from later(final)

ℹ️ 'Tagless Final' - is the general term for this pattern. If you look at the `apply` methods in the code above, it is
Tagless in the sense that our apply-method is defined not against a specific "tag" (typically "IO"), but abstractly
against JUST the *capabilities* or *effects* we require to implement our logic. We only ask for as much power as we need
to do our implementation (for instance, `F[_]: Parallel` if we need access to parallel processing functionality in our
implementation). Our traits are even less 'tagged', being completely oblivious to any typeclasses on their `F[_]`.

For our purposes, then:

* **Tagless**: writing code against an abstract effect type `F[_]` and specifying required capabilities via
  typeclasses (e.g., `[F[_]: Parallel]`)
* **Final**: Providing the concrete effect type (the "tag," most commonly cats.effect.IO) at the application's entry
  point.

The "Final" part of name is because we only specify what our tag is at the "end of the world", when we supply it at
the 'final' edge of our application - in our `Main.scala`.

# Do and Don't:

## Trait Design

### ✅ DO: Keep traits abstract and capability-free

```scala 3
trait UserService[F[_]] {
  def findUser(id: UserId): F[Option[User]]
  def createUser(userData: CreateUserRequest): F[Either[UserError, User]]
}
```

### ❌ DON'T: Add capabilities to traits

```scala 3
// Don't do this - traits should be capability-agnostic
trait UserService[F[_]: Async: Parallel] {
  def findUser(id: UserId): F[Option[User]]
}

// Don't do this - method-level generics
trait UserService {
  def findUser[F[_]: Async](id: UserId): F[Option[User]]
}
```

## Implementation Patterns

### ✅ DO: Use companion objects with minimal capabilities

```scala 3
object UserService {
  def apply[F[_]: Async](
    repository: UserRepository[F],
    validator: UserValidator[F]
  ): UserService[F] = new UserService[F] {
    override def findUser(id: UserId): F[Option[User]] = ???
    override def createUser(userData: CreateUserRequest): F[Either[UserError, User]] = ???
  }
}
```

### ❌ DON'T: Create concrete implementation classes

```scala 3
// Don't do this - creates tight coupling risk
class UserServiceImpl[F[_]: Async](
  repository: UserRepository[F]
) extends UserService[F] {
  // implementation
}

// Don't do this - over-specify capabilities
def apply[F[_]: Async: Parallel: Temporal: Concurrent](
  repository: UserRepository[F]
): UserService[F] = // if it only needs Async to accomplish the implementation goals, only Async should be specified
```

## Multiple Implementations

### ✅ DO: Use named methods for multiple implementations

```scala 3
object NotificationService {
  def email[F[_]: Async](
    emailClient: EmailClient[F]
  ): NotificationService[F] = ???
  
  def sms[F[_]: Async](
    smsClient: SmsClient[F]
  ): NotificationService[F] = ???
  
  def push[F[_]: Async: Parallel](  // Only push needs Parallel
    pushClient: PushClient[F]
  ): NotificationService[F] = ???
}
```

### ❌ DON'T: Use single apply for different implementations

```scala 3
// Don't do this - unclear which implementation you get
object NotificationService {
  def apply[F[_]: Async](
    emailClient: Option[EmailClient[F]] = None,
    smsClient: Option[SmsClient[F]] = None
  ): NotificationService[F] = ??? // Which one is it?
}
```

## Dependency Injection

### ✅ DO: Pass dependencies explicitly to apply methods

```scala 3
object OrderService {
  def apply[F[_]: Async](
    paymentService: PaymentService[F],
    inventoryService: InventoryService[F],
    emailService: EmailService[F]
  ): OrderService[F] = new OrderService[F] {
    override def processOrder(order: Order): F[Either[OrderError, OrderResult]] =
      for {
        _ <- inventoryService.reserve(order.items)
        payment <- paymentService.charge(order.total)
        _ <- emailService.sendConfirmation(order.customerId)
      } yield Right(OrderResult(order.id))
  }
}
```

### ❌ DON'T: Return wrapped effects or resources from apply

```scala 3
// Don't do this - pass only the dependencies you need, not a wrapper of all dependencies
def apply[F[_]: Async](deps: Dependencies): OrderService[F]] = ???

// Don't do this - apply should return the service directly
def apply[F[_]: Async](deps: Dependencies): F[OrderService[F]] = ???
def apply[F[_]: Async](deps: Dependencies): Resource[F, OrderService[F]] = ???

// Don't do this - hidden dependencies. Never instantiate anything but the trait in your construction methods.
def apply[F[_]: Async](): OrderService[F] = {
  val paymentService = PaymentService.hardcoded() // Hidden!
  // ...
}
```

## Error Handling

### ✅ DO: Use consistent error types in return signatures when you have predictable errors

```scala 3
trait ProductService[F[_]] {
  def findProduct(id: ProductId): F[Either[ProductError, Product]]
  def updateStock(id: ProductId, quantity: Int): F[Either[ProductError, Unit]]
}

sealed trait ProductError
case object ProductNotFound extends ProductError
case class InvalidQuantity(msg: String) extends ProductError
```

### ❌ DON'T: Mix error handling approaches

```scala 3
// Don't do this - inconsistent error handling
trait ProductService[F[_]] {
  def findProduct(id: ProductId): F[Option[Product]]  // Option for errors
  def updateStock(id: ProductId, quantity: Int): F[Unit]  // Throws exceptions
  def deleteProduct(id: ProductId): F[Either[ProductError, Unit]]  // Either
}
```

## Capability Specification

### ✅ DO: Request only the capabilities you actually need

```scala 3
object MetricsService {
  // Only needs Sync for simple operations
  def counter[F[_]: Sync](name: String): MetricsService[F] = ???
  
  // Needs Async for HTTP calls
  def http[F[_]: Async](endpoint: String): MetricsService[F] = ???
  
  // Needs Parallel for batch operations
  def batch[F[_]: Async: Parallel](endpoints: List[String]): MetricsService[F] = ???
}
```

### ❌ DON'T: Over-specify or under-specify capabilities

```scala 3
// Don't do this - way too many capabilities for simple logic
def apply[F[_]: Async: Parallel: Temporal: Concurrent: Resource.Make](
  simple: SimpleRepository[F]
): SimpleService[F] = ???

// Don't do this - not enough capabilities for async work
def apply[F[_]: Functor](  // Needs at least Async to implement the logic
  database: DatabaseClient[F]
): UserService[F] = ???
```

## Resource Management

### ✅ DO: Take explicit arguments

```scala 3
object UserService {
  // database client is passed in explicitly, not wrapped in Resource, not instantiated inside this logic
  def apply[F[_]: Async](databaseClient: DatabaseClient[F]): UserService[F]] = new UserService[F] { /* ... */ }
}
```

### ❌ DON'T: Mix resource management with business logic

```scala 3
// Don't do this - business logic shouldn't instantiate resources
object UserService {
  def apply[F[_]: Async](config: DatabaseConfig): Resource[F, UserService[F]] =
    DatabaseClient.resource(config).map { client =>
      // Business logic mixed with resource instantiation
      new UserService[F] { /* ... */ }
    }
}
```

```scala 3
// Don't do this - business logic shouldn't handle resources
object UserService {
  def apply[F[_]: Async](databaseResource: Resource[F, DatabaseClient[F])): Resource[F, UserService[F]] =
    databaseResource.map { client =>
      // Business logic mixed with resource management
      new UserService[F] { /* ... */ }
    }
}
```