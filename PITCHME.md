
Building a JSON API with Scala and Play

`Alexis Hernandez`


---

Topics

* Scala
* Play Framework

+++

Also, talk about

* Functional programming
* Asynchronous functions
* Configuration
* Error handling
* Testing

+++

Stack

* Google Guice
* PostgreSQL
* Anorm
* Scalactic
* Typesafe Config
* Scalatest
* Docker



---

Why another language?

+++

Let's see some examples

+++

```java
...
int createUser(String email) {
  User createdUser = userDAO.create(email);
  return createdUser.id;
}
...
```

+++

The Billion Dollar Mistake!

NullPointerException

+++

```java
void processCSV(List<CSVRow> rowList) {
  CSVRow header = rowList.get(0);
  ...
}
```

+++

ArrayIndexOutOfBoundsException / NoSuchElementException

+++

```java
void createUser(String username, String fullname) {
  ...
}

createUser(fullname, username);
```

+++

Welcome to the world of weak types!

+++

```java
enum UserStatus {
  INACTIVE, ACTIVE, DELETED
}

sendOffers(User user, String offersText) {
  if (user.status == UserStatus.INACTIVE) {
    throw new RuntimeException("Can't send campaing");
  } else if (user.status == UserStatus.ACTIVE)
    emailService.sendOffersEmail(user, offersText)
  }
}
```

+++

Non exhaustive matching!

+++

"The sooner you detect a bug, the easier it is to find and fix"

+++

Catch them all by the compiler!



















---

Why another framework?

+++

Plenty of built-in features to get productive fast!

+++

Performance

+++

Type safe

+++

Flexible / Extensible













---

Creating a route supporting JSON

+++

conf/routes
```scala
POST   /hello   controllers.HelloController.hello()
```

+++

Create the controller

```scala
class HelloController @Inject() (cc: ControllerComponents)
    extends AbstractController(cc) {

  def hello() = Action(parse.json) { request =>
    val input = request.body.as[HelloJson]
    val msg = s"Hello ${input.name}, you are ${input.age} years old!"
    val output = HelloJsonResponse(msg)
    Ok(Json.toJson(output))
  }
}
```

+++

Create the input model

```scala
case class HelloJson(name: String, age: Int)
object HelloJson {
  implicit val reads: Reads[HelloJson] = Json.reads[HelloJson]
}
```

+++

Create the output model

```scala
case class HelloJsonResponse(message: String)
object HelloJsonResponse {
  implicit val writes: Writes[HelloJsonResponse] = {
    Json.writes[HelloJsonResponse]
  }
}
```

+++

Result

```bash
> http POST localhost:9000/hello name=Alex age:=18
HTTP/1.1 200 OK
Content-Type: application/json
...
{
    "message": "Hello Alex, you are 18 years old!"
}
```

+++

In the real life, a lot of things could go wrong!

+++

How do we handle ...

+++

Malformed JSON?

+++

No JSON at all?

+++

Wrong JSON?

+++

Exceptions?

+++

Paginated query / result?

+++

Accumulating errors?

+++

Authentication?

+++

Consistent error responses?

+++

More?
...

---















Presenting Crypto Coin Alerts API
- https://github.com/AlexITC/crypto-coin-alerts

+++

Goals

+++

Type safety

+++

i18n support

+++

Clean controllers

+++

Error accumulation

+++

Pagination support

+++

Automatic error mapping

+++

Consistent error responses

+++

Simple authentication

+++

Async code with no callback hell

+++

Well tested code!

















---

Let's see the basics

+++

app/com/alexitc/coinalerts/errors/ApplicationError.scala

```scala
sealed trait ApplicationError
sealed trait InputValidationError extends ApplicationError
sealed trait ConflictError extends ApplicationError
sealed trait NotFoundError extends ApplicationError
sealed trait AuthenticationError extends ApplicationError
sealed trait PrivateError extends ApplicationError {
  def cause: Throwable
}
```

+++

app/com/alexitc/coinalerts/commons/package.scala

```scala
package object commons {

  type ApplicationErrors = Every[ApplicationError]
  type ApplicationResult[+A] = A Or ApplicationErrors
  type FutureApplicationResult[+A] = Future[ApplicationResult[A]]
}
```

+++

```scala
def createUser(model: Model): Future[User Or Every[ApplicationError]]]

/* Or */

def createUser(model: Model): FutureApplicationResult[User]
```

+++
app/com/alexitc/coinalerts/commons/FutureOr.scala

```scala
object FutureOr {
  object Implicits {
    implicit class FutureOps[+A](val future: FutureApplicationResult[A]) extends AnyVal {
      def toFutureOr: FutureOr[A] = {
        new FutureOr(future)
      }
    }
  }
  ...
}
```

+++

```scala
def create(model: Model) = {
  userDataHandler.createUser(model).flatMap { createdUser =>
    userDataHandler.createToken(createdUser.id).flatMap { token =>
      emailService.sendVerificationEmail(user.email, token).map { _ =>
        createdUser
      }
    }
  }
}
```

+++

Or...
```scala
def create(model: Model) = {
  for {
    createdUser <- userDataHandler.createUser(model).toFutureOr
    token <- userDataHandler.createToken(createdUser.id).toFutureOr
    _ <- emailService.sendVerificationEmail(createUser.email token)
  } yield createdUser
}
```

+++

app/com/alexitc/coinalerts/commons/RequestContext.scala

```scala
sealed trait RequestContext {
  def lang: Lang
}
sealed trait HasModel[T] {
  def model: T
}

final case class PublicRequestContext(lang: Lang) extends RequestContext

final case class PublicRequestContextWithModel[T](model: T, lang: Lang)
    extends RequestContext with HasModel[T]

...
```

+++

conf/application.conf

```hocon
...
db.default {
  driver = "org.postgresql.Driver"
  host = "localhost:5432"
  database = "crypto_coin_alerts"
  username = "postgres"
  password = ""

  host = ${?CRYPTO_COIN_ALERTS_PSQL_HOST}
  database = ${?CRYPTO_COIN_ALERTS_PSQL_DATABASE}
  username = ${?CRYPTO_COIN_ALERTS_PSQL_USERNAME}
  password = ${?CRYPTO_COIN_ALERTS_PSQL_PASSWORD}

  url = "jdbc:postgresql://"${db.default.host}"/"${db.default.database}
}
...
```

---

Defining the routes

+++

conf/routes

```scala
POST   /users                           controllers.UsersController.create()
POST   /verification-tokens/:token      controllers.UsersController.verifyEmail(token: com.alexitc.coinalerts.models.UserVerificationToken)
POST   /users/login                     controllers.UsersController.loginByEmail()
GET    /users/me                        controllers.UsersController.whoAmI()

POST   /fixed-price-alerts              controllers.FixedPriceAlertsController.create()
GET    /fixed-price-alerts              controllers.FixedPriceAlertsController.getAlerts(query: com.alexitc.coinalerts.core.PaginatedQuery)

POST   /daily-price-alerts              controllers.DailyPriceAlertsController.create()
GET    /daily-price-alerts              controllers.DailyPriceAlertsController.getAlerts(query: com.alexitc.coinalerts.core.PaginatedQuery)
```

+++

Defining a controller

+++

```scala
class FixedPriceAlertsController @Inject() (
    components: JsonControllerComponents,
    alertService: FixedPriceAlertService)
    extends JsonController(components)
    with PlayRequestTracing {

  def create() = authenticatedWithInput(Created) {
      context: AuthCtxModel[CreateFixedPriceAlertModel] =>

    alertService.create(context.model, context.userId)
  }

  def getAlerts(query: PaginatedQuery) = authenticatedNoInput {
      context: AuthCtx =>

    alertService.getAlerts(context.userId, query)
  }
}
```

+++

Defining our service

+++

```scala
class FixedPriceAlertService @Inject() (
    alertValidator: FixedPriceAlertValidator,
    paginatedQueryValidator: PaginatedQueryValidator,
    alertFutureDataHandler: FixedPriceAlertFutureDataHandler)(
    implicit ec: ExecutionContext) {

   ...
}
```

+++

The create method

+++

```scala
def create(
    createAlertModel: CreateFixedPriceAlertModel,
    userId: UserId): FutureApplicationResult[FixedPriceAlert] = {

  val result = for {
    validatedModel <- alertValidator.validateCreateModel(createAlertModel).toFutureOr
    createdAlert <- alertFutureDataHandler.create(validatedModel, userId).toFutureOr
  } yield createdAlert

  result.toFuture
}
```

+++

```scala
def getAlerts(
    userId: UserId,
    query: PaginatedQuery): FuturePaginatedResult[FixedPriceAlert] = {

  val result = for {
    validatedQuery <- paginatedQueryValidator.validate(query).toFutureOr
    paginatedResult <- alertFutureDataHandler.getAlerts(userId, validatedQuery).toFutureOr
  } yield paginatedResult

  result.toFuture
}
```

+++

The validator

+++

```scala
class FixedPriceAlertValidator @Inject() (marketBookValidator: MarketBookValidator) {
  ...
}
```

+++

```scala
def validateCreateModel(
    createModel: CreateFixedPriceAlertModel): ApplicationResult[CreateFixedPriceAlertModel] = {

  Accumulation.withGood(
    validatePrice(createModel.price),
    validateBasePrice(createModel.basePrice),
    marketBookValidator.validate(createModel.book, createModel.market)) { (_, _, _) =>

    createModel
  }
}

private def validatePrice(price: BigDecimal): ApplicationResult[BigDecimal] = {
  if (price > 0) {
    Good(price)
  } else {
    Bad(InvalidPriceError).accumulating
  }
}

...
```

+++

The data handler definition

+++

```scala
trait FixedPriceAlertDataHandler[F[_]] {
  def create(
      createAlertModel: CreateFixedPriceAlertModel,
      userId: UserId): F[FixedPriceAlert]

  ...

  def getAlerts(
      userId: UserId,
      query: PaginatedQuery): F[PaginatedResult[FixedPriceAlert]]
}
```

---

Testing our new routes!

+++

```scala
class FixedPriceAlertsControllerSpec extends PlayAPISpec {

  import PlayAPISpec._

  implicit val alertDataHandler: FixedPriceAlertBlockingDataHandler =
      new FixedPriceAlertInMemoryDataHandler {}

  val application: Application = guiceApplicationBuilder
      .overrides(bind[FixedPriceAlertBlockingDataHandler].to(alertDataHandler))
      .build()

  ...
}
```

+++

FixedPriceAlertsControllerSpec

```scala
...
"POST /fixed-price-alerts" should {
  val url = "/fixed-price-alerts"

  "Create an alert" in {
    val body =
      """
        | {
        |   "market": "BITTREX",
        |   "book": "BTC_ETH",
        |   "isGreaterThan": false,
        |   "price": "0.123456789"
        | }
      """.stripMargin

    val user = createVerifiedUser()
    val token = jwtService.createToken(user.id)
    val response = POST(url, Some(body), token.toHeader)
    status(response) mustEqual CREATED

    val json = contentAsJson(response)
    (json \ "id").asOpt[Long].isDefined mustEqual true
    (json \ "market").as[String] mustEqual "BITTREX"
  }
  ...
}
```

+++

```scala
"GET /fixed-price-alerts" should {
  val url = "/fixed-price-alerts"

  "Return a paginated result based on the query" in {
    val user = createVerifiedUser()
    val token = jwtService.createToken(user.id)
    createFixedPriceAlert(user.id)
    createFixedPriceAlert(user.id)

    val query = PaginatedQuery(Offset(1), Limit(10))
    val response = GET(url.withQueryParams(query), token.toHeader)
    val json = contentAsJson(response)
    status(response) mustEqual OK
    (json \ "total").as[Int] mustEqual 2

    val alertJsonList = (json \ "data").as[List[JsValue]]
    alertJsonList.length mustEqual 1
  }
  ...
}
...
```

















---

Let's see the models!

+++

```scala
case class PaginatedQuery(offset: Offset, limit: Limit)
```

+++

```scala
case class PaginatedResult[T](
  offset: Offset, limit: Limit, total: Count, data: List[T])

object PaginatedResult {
  implicit def writes[T](implicit writesT: Writes[T]): Writes[PaginatedResult[T]] =
  OWrites[PaginatedResult[T]] { result =>
    Json.obj(
      "offset" -> result.offset,
      "limit" -> result.limit,
      "total" -> result.total,
      "data" -> result.data
    )
  }
}
```

+++

```scala
case class FixedPriceAlert(
    id: FixedPriceAlertId,
    userId: UserId,
    market: Market,
    book: Book,
    isGreaterThan: Boolean,
    price: BigDecimal,
    basePrice: Option[BigDecimal] = None
)

object FixedPriceAlert {
  implicit val writes: Writes[FixedPriceAlert] = Json.writes[FixedPriceAlert]
}
```

+++

```scala
case class FixedPriceAlertId(long: Long) extends AnyVal with WrappedLong
```

+++

```scala
case class CreateFixedPriceAlertModel(
    market: Market,
    book: Book,
    isGreaterThan: Boolean,
    price: BigDecimal,
    basePrice: Option[BigDecimal]
)

object CreateFixedPriceAlertModel {

  implicit val reads: Reads[CreateFixedPriceAlertModel] = {
    Json.reads[CreateFixedPriceAlertModel]
  }
}
```

+++

```scala
sealed abstract class Market(val string: String)
object Market {

  case object BITTREX extends Market("BITTREX")
  case object BITSO extends Market("BITSO")
  case class UNKNOWN(override val string: String) extends Market(string)

  private val fromStringPF: PartialFunction[String, Market] = {
    case BITTREX.string => BITTREX
    case BITSO.string => BITSO
  }

  ...

  implicit val reads: Reads[Market] = {
    JsPath.read[String].collect(JsonValidationError("error.market.unknown"))(fromStringPF)
  }

  implicit val writes: Writes[Market] = Writes[Market] { market => JsString(market.string) }
}
```

+++

```scala
case class Book(major: String, minor: String) {
  val string = s"${major}_$minor".toUpperCase
}
object Book {
  ...

  implicit val reads: Reads[Book] = {
    JsPath.read[String].collect(JsonValidationError("error.book.invalid")) {
      case string if fromString(string).isDefined => fromString(string).get
    }
  }

  implicit val writes: Writes[Book] = Writes[Book] { book => JsString(book.string) }
}
```

---

How do we render errors?

+++

```scala
sealed trait CreateFixedPriceAlertError extends ApplicationError
case object InvalidPriceError extends CreateFixedPriceAlertError with InputValidationError
case object InvalidBasePriceError extends CreateFixedPriceAlertError with InputValidationError
case object UnknownBookError extends CreateFixedPriceAlertError with InputValidationError
```

+++

JsonController

```scala
...
private def renderErrors(
    errors: ApplicationErrors)(
    implicit lang: Lang): Result = {

  // detect response status based on the first error
  val status = errors.head match {
    case _: InputValidationError => Results.BadRequest
    case _: ConflictError => Results.Conflict
    case _: NotFoundError => Results.NotFound
    case _: AuthenticationError => Results.Unauthorized
    case _: PrivateError => Results.InternalServerError
  }
  ...
}
```

+++

JsonErrorRenderer

```scala
def toPublicErrorList(
    error: ApplicationError)(
    implicit lang: Lang): Seq[PublicError] = error match {
  ...
  case error: CreateFixedPriceAlertError =>
    List(renderCreateFixedPriceAlertError(error))
  ...
}
...
```

+++

JsonErrorRenderer

```scala
private def renderCreateFixedPriceAlertError(
    error: CreateFixedPriceAlertError)(
    implicit lang: Lang) = error match {

  case InvalidPriceError =>
    val message = messagesApi("error.price.invalid")
    FieldValidationError("price", message)

  case InvalidBasePriceError =>
    val message = messagesApi("error.basePrice.invalid")
    FieldValidationError("basePrice", message)

  case UnknownBookError =>
    val message = messagesApi("error.book.unknown")
    FieldValidationError("book", message)
  ...
}
```

---

### Thanks

- Github: AlexITC (https://github.com/AlexITC)
- Linkedn: https://www.linkedin.com/in/alexis-hernandez/

- We are hiring!

