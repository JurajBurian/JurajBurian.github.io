---
title: "Building application in Scala Cats"
date: 2025-10-28T00:00:00
draft: false
categories: [Scala]
tags: [Cats, Cats Effect, Doobie, Tapir, JWT, Bcrypt, Munit]
---

## Introduction

In this article, we'll build a simple application using the Cats ecosystem. The application will expose a REST API for user management, including an OpenAPI documentation endpoint.<br>
Some endpoints will require JWT token authentication, also we'll use the `Bcrypt` library to generate password hashes. We'll demonstrate both unit testing and integration testing using `testcontainers` to run a `Postgres` database in Docker.

The GitHub repository for this project is available [here](https://github.com/JurajBurian/cats-user-demo).

## Services

In a Cats-based application, services are typically defined as traits with higher-kinded type abstraction, commonly referred to as `F[_]`. If you're not familiar with this technique, I recommend reading about the `Tagless Final` pattern [here](https://rockthejvm.com/articles/tagless-final-in-scala).

Our application will consist of four main services:

- `UserRepository`: Handles user management (CRUD) operations in the database
- `JwtService`: Manages generation and validation of JWT tokens
- `PasswordService`: Handles password hashing and verification
- `UserService`: Implements user-related business logic


Here is definition of all services located in io.github.jb.domain.Services.scala :
```Scala
trait UserRepository[F[_]] {
  def create(userCreate: UserCreate, passwordHash: String): F[User]
  def findByEmail(email: String): F[Option[User]]
  def findById(id: UUID): F[Option[User]]
  def updateStatus(id: UUID, isActive: Boolean): F[Boolean]
  def findActive(offset: Long, count: Long): F[List[User]]
}

trait JwtService[F[_]] {
  def generateAccessToken(user: User): F[String]
  def generateRefreshToken(userId: UUID): F[String]
  def validateAndExtractAccessToken(token: String): F[Option[AccessTokenClaims]]
  def validateAndExtractRefreshToken(token: String): F[Option[RefreshTokenClaims]]
}

trait PasswordService[F[_]] {
  def hashPassword(password: String): F[String]
  def verifyPassword(password: String, hash: String): F[Boolean]
}

trait UserService[F[_]] {
  def createUser(userCreate: UserCreate): F[UserResponse]
  def login(loginRequest: LoginRequest): F[AuthResponse]
  def refreshTokens(refreshToken: String): F[AuthResponse]
  def getUser(id: UUID): F[UserResponse]
  def updateUserStatus(id: UUID, isActive: Boolean): F[Boolean]
  def validateUserForAccess(token: String): F[User]
  def listActiveUsers(offset: Long, count: Long): F[List[UserResponse]]
}
```
In short, `F[_]` is an abstraction over a monad. In our case, we use the [cats.effect.IO](https://typelevel.org/cats-effect/api/3.x/cats/effect/IO.html) monad in the production code.

While we won't cover all services or source code in detail, the following sections will highlight particularly interesting or principle-demonstrating parts.

A straightforward example is the `PasswordService` implementation:

```scala
class PasswordServiceImpl[F[_]](cost: Int)(using sync: Sync[F]) extends PasswordService[F] {

  def hashPassword(password: String): F[String] =
    sync.delay {
      BCrypt.withDefaults().hashToString(cost, password.toCharArray)
    }

  def verifyPassword(password: String, hash: String): F[Boolean] =
    sync.delay {
      BCrypt.verifyer().verify(password.toCharArray, hash).verified
    }
}
```
This is a basic implementation that currently ignores potential exceptions. For production code, we would need to implement proper error handling, which we'll address later.

Our implementation doesn't know in which context it will be used. It can be used in the IO monad or in other monads. We know that our final monad must have a `delay` operation.

## Simple unit testing with munit

Let look to the test, to see how can be instantiated and also tested.

```scala
package io.github.jb.service

import cats.effect.SyncIO
import munit.FunSuite

class PasswordServiceTest extends FunSuite {

  val passwordService = new PasswordServiceImpl[SyncIO](12)

  test("hash and verify password successfully") {
    (for {
      hash <- passwordService.hashPassword("testPassword123")
      isValid <- passwordService.verifyPassword("testPassword123", hash)
    } yield assert(isValid)).unsafeRunSync()
  }
  // ... rest of the test
}
```
We use `SyncIO` as the monad for testing purposes, while in the production code, we use the `IO` monad.

## Error Handling in Cats Effect

In Cats Effect applications, error handling primarily revolves around the `ApplicativeError` and `MonadError` type classes. Here are the key aspects:

- **Error Types**: 
   - By default, effects like `IO` and `SyncIO` use `Throwable` as their error type
   - This aligns with the JVM's exception model while providing a pure functional interface

- **Common Patterns**:
   - `handleError`/`handleErrorWith` for recovery
   - `attempt` to lift errors into the value channel as `Either[Throwable, A]`
   - `onError` for side-effecting error handling
   - `raiseError` to explicitly signal errors

- **Best Practices**:
   - Use `ApplicativeError.raiseWhen` for conditional error raising
   - Prefer typed errors using ADTs for domain-specific errors
   - Consider `EitherT` or `IO[Either[E, A]]` for explicit error handling
   - Use `Resource` for safe resource acquisition and release

#### Exploring Alternative Approaches with Cats MTL

Is there another way to handle errors? The answer is yes! In a [recent article](https://typelevel.org/blog/2025/09/02/custom-error-types.html), Daniel Spiewak describes how to use `Cats MTL` to manage errors differently. Let's examine this approach in our codebase.

In `io.github.jb.domain.Domain.scala`, we've defined a simplified error model using a sealed trait:
```scala
sealed trait ApiError extends Product with Serializable {
  def message: String
}

object ApiError {
  case class UserAlreadyExists(email: String, override val message: String = "User with this email already exists")extends ApiError
  case class InvalidCredentials(override val message: String = "Invalid credentials") extends ApiError
  case class InvalidRefreshToken(override val message: String = "Invalid refresh token") extends ApiError
  case class UserNotFound(id: UUID, override val message: String = "User not found") extends ApiError
  case class InvalidOrExpiredToken(override val message: String = "Invalid or expired token") extends ApiError
  case class AccountDeactivated(override val message: String = "Account has been deactivated") extends ApiError
  case class InternalServerError(cause: String, override val message: String = "Internal server error") extends ApiError
}
```

#### Raising Errors

To directly propagate errors, let's examine `UserServiceImpl` to see how it works:

```scala
class UserServiceImpl[F[_]](
    userRepo: UserRepository[F],
    jwtService: JwtService[F],
    passwordService: PasswordService[F]
)(using M: Monad[F], R: Raise[F, ApiError])
    extends UserService[F] {

  import cats.mtl.syntax.raise.given // use sufix syntax with raise
  
  // ... other methods ...
  
  def refreshTokens(refreshToken: String): F[AuthResponse] = {
    for {
      refreshClaimsOpt <- jwtService.validateAndExtractRefreshToken(refreshToken)
      refreshClaims <- refreshClaimsOpt match {
        case Some(claims) => claims.pure[F]
        case None         => ApiError.InvalidRefreshToken().raise
      }

      userOpt <- userRepo.findById(refreshClaims.userId)
      user <- userOpt match {
        case Some(user) => user.pure[F]
        case None       => ApiError.UserNotFound(refreshClaims.userId).raise
      }

      _ <- validateUserStatus(user)

      newAccessToken <- jwtService.generateAccessToken(user)
      newRefreshToken <- jwtService.generateRefreshToken(user.id)
      userResponse = toUserResponse(user)
    } yield AuthResponse(Tokens(newAccessToken, newRefreshToken), userResponse)
  }

  private def validateUserStatus(user: User): F[Unit] =
    if (!user.isActive) ApiError.AccountDeactivated().raise else Monad[F].unit

  private def toUserResponse(user: User): UserResponse =
    UserResponse(
      id = user.id,
      email = user.email,
      username = user.username,
      firstName = user.firstName,
      lastName = user.lastName,
      isActive = user.isActive,
      createdAt = user.createdAt
    )

  // ... other methods ...
}
```
The key to understanding error propagation here is the `Raise` trait. In the `UserServiceImpl` constructor, we see `R: Raise[F, ApiError]`, and we import `cats.mtl.syntax.raise.given` to enable the `*.raise` syntax for error propagation.

> **Note:** The error type isn't visible at the API level.

> The same strategy is also implemented in `DoobieUserRepository` for consistent error handling.

#### Handling Errors

While our services produce `ApiError` values, we need to handle these errors appropriately. In our case, we use `tapir` for both OpenAPI definitions and endpoint implementations.

To manage error handling consistently, we've implemented a `CustomHandle` class:

```scala
package io.github.jb.service

import cats.Applicative
import cats.effect.IO
import cats.mtl.Handle
import io.github.jb.domain.ApiError

class CustomHandle extends Handle[IO, ApiError] {

  private case class InternalError(apiError: ApiError) extends Throwable(apiError.message)

  override def applicative: Applicative[IO] = cats.effect.IO.asyncForIO

  override def handleWith[A](fa: IO[A])(f: ApiError => IO[A]): IO[A] =
    fa.recoverWith {
      case e: InternalError =>
        f(e.apiError)
      case other =>
        f(ApiError.InternalServerError(other.getMessage)) // re-raise non-ApiError as unknown error
    }

  override def raise[E2 <: ApiError, A](e: E2): IO[A] =
    IO.raiseError(new InternalError(e))
}
```

The `Handle` type class serves two purposes: handling errors and raising them (since `Handle` extends `Raise[F, E]`).

In Cats applications, `Throwable` is typically used for error handling. While we can't avoid this entirely, we use a private `InternalError` case class to wrap our `ApiError` and maintain type safety.

Let's examine how this works in `Endpoints.scala`:

```scala
class Endpoints[F[_]](userService: UserService[F])(using M: Monad[F], H: Handle[F, ApiError]) {

  private val basePath = "api" / "v1"
  private val bearerTokenHeader = auth.bearer[String]()

  // Simple error handling - just convert to Either
  private def handleServiceError[T](result: F[T]): F[Either[ApiError, T]] = H.attempt(result)

  // Define specific error variants for each ApiError type
  private val invalidCredentialsError =
    oneOfVariant(StatusCode.Unauthorized, jsonBody[InvalidCredentials])
  private val invalidRefreshTokenError =
    oneOfVariant(StatusCode.Unauthorized, jsonBody[InvalidRefreshToken])
  private val invalidOrExpiredTokenError =
    oneOfVariant(StatusCode.Unauthorized, jsonBody[InvalidOrExpiredToken])
  private val accountDeactivatedError =
    oneOfVariant(StatusCode.Forbidden, jsonBody[AccountDeactivated])
  private val userNotFoundError =
    oneOfVariant(StatusCode.NotFound, jsonBody[UserNotFound])
  private val userAlreadyExistsError =
    oneOfVariant(StatusCode.Conflict, jsonBody[UserAlreadyExists])
  private val internalServerError =
    oneOfVariant(StatusCode.InternalServerError, jsonBody[InternalServerError])

  // Each endpoint specifies exactly which errors it can return
  val loginEndpoint: ServerEndpoint[Any, F] =
    endpoint.post
      .in(basePath / "auth" / "login")
      .in(jsonBody[LoginRequest])
      .out(jsonBody[AuthResponse])
      .errorOut(
        oneOf(
          invalidCredentialsError,
          accountDeactivatedError,
          internalServerError
        )
      )
      .serverLogic(loginRequest => handleServiceError(userService.login(loginRequest)))

  // ... other endpoints ... and final list of endpoints
}
```
Handling is pretty strait forward, method `private def handleServiceError[T](result: F[T]): F[Either[ApiError, T]] = H.attempt(result)` convert result to `Either[ApiError, T]` that is used directly in `serverLogic` method. 

> **Remark:** unfortunately we must declare all possible errors in `errorOut` method. This is not the best solution, but it is the only one that I found.   

## Running Integration Tests

Our test suite includes integration tests in `UserServiceTest.scala`, which uses `testcontainers` to run a `Postgres` database in Docker. Here's how it's implemented:
```Scala
package io.github.jb.service

import cats.{Applicative, MonadError}
import cats.effect.IO
import cats.effect.Resource
import cats.mtl.Handle
import cats.syntax.all.*
import munit.CatsEffectSuite
import com.dimafeng.testcontainers.PostgreSQLContainer
import org.testcontainers.utility.DockerImageName
import doobie.util.ExecutionContexts
import doobie.hikari.HikariTransactor
import doobie.implicits.{toConnectionIOOps, toSqlInterpolator}
import org.flywaydb.core.Flyway
import io.github.jb.domain.*
import io.github.jb.config.*
import io.github.jb.repository.DoobieUserRepository

import java.util.UUID
import scala.concurrent.duration.DurationInt

class UserServiceTest extends CatsEffectSuite {

  given Handle[IO, ApiError] = new CustomHandle()

  lazy val dbContainer: PostgreSQLContainer = {
    val pg = new PostgreSQLContainer(Some(DockerImageName.parse("postgres:17")))
    pg.start()
    pg
  }

  private lazy val transactorResource: Resource[IO, HikariTransactor[IO]] =
    HikariTransactor.newHikariTransactor[IO](
      "org.postgresql.Driver",
      dbContainer.jdbcUrl,
      dbContainer.username,
      dbContainer.password,
      ExecutionContexts.synchronous
    )

  private lazy val (transactor: HikariTransactor[IO], cleanup) = transactorResource.allocated.unsafeRunSync()

  private lazy val userRepo: DoobieUserRepository[IO] =
    new DoobieUserRepository[IO](transactor)

  private lazy val jwtService: JwtServiceImpl[IO] =
    new JwtServiceImpl[IO](
      JwtConfig("test-secret-key-for-testing-only-very-long", "15 minutes", "30 days")
    )

  private lazy val passwordService: PasswordServiceImpl[IO] =
    new PasswordServiceImpl[IO](12)

  private lazy val userService: UserServiceImpl[IO] = {
    new UserServiceImpl[IO](userRepo, jwtService, passwordService)
  }

  private lazy val testServices: (DoobieUserRepository[IO], UserServiceImpl[IO]) = (userRepo, userService)

  private def handleServiceError[A](io: IO[A]): IO[Either[ApiError, A]] =
    Handle[IO, ApiError].attempt(io)

  override def beforeAll(): Unit = {
    // execute migrations
    val flyway = Flyway
      .configure()
      .dataSource(dbContainer.jdbcUrl, dbContainer.username, dbContainer.password)
      .locations("classpath:db/migration")
      .load()
    flyway.migrate()
  }

  override def afterAll(): Unit = {
    // Clean up transactor
    cleanup.unsafeRunSync()
    // Stop container
    dbContainer.stop()
  }

  private def withServices[A](
      test: (DoobieUserRepository[IO], UserServiceImpl[IO]) => IO[A]
  ): IO[A] =
    IO(testServices)
      .flatMap { services =>
        import doobie.*
        // before each test clean database
        sql"DELETE from users".update.run.transact(transactor).void.map(_ => services)
      }
      .flatMap { case (userRepo, userService) => test(userRepo, userService) }

  test("create user successfully") {

    withServices { case (userRepo, userService) =>
      val userCreate = UserCreate(
        email = "newuser@example.com",
        username = "newuser",
        password = "securePassword123",
        firstName = Some("New"),
        lastName = Some("User")
      )

      for {
        userResponse <- userService.createUser(userCreate)
        foundUser <- userRepo.findByEmail("newuser@example.com")
      } yield {
        assertEquals(userResponse.email, "newuser@example.com")
        assertEquals(userResponse.username, "newuser")
        assertEquals(userResponse.firstName, Some("New"))
        assertEquals(foundUser.map(_.email), Some("newuser@example.com"))
        assert(foundUser.isDefined)
      }
    }
  }

  // ... other tests
}
```
The test setup involves assembling the entire service stack and managing resources, including a Postgres database in Docker. Here's a breakdown of the key components:

- **Error Handling**: `given Handle[IO, ApiError] = new CustomHandle()` - Required in scope for `Raise` used in `UserService` and `DoobieUserRepository`
- **Database**: `dbContainer` initializes a Postgres database in Docker
- **Connection Management**: The `(transactor: HikariTransactor[IO], cleanup)` tuple creates and manages the Doobie transactor for database connections, with proper cleanup
- **Service Dependencies**:
  - `passwordService` and `jwtService` are used to create the `userService` instance
  - `handleServiceError` manages service errors using the `Handle` monad
- **Test Lifecycle**:
  - `beforeAll`: Executes Flyway migrations to set up the database schema
  - `afterAll`: Ensures proper cleanup of resources and stops the database container

The `withServices` method wraps test code to provide:
1. A clean database state before each test
2. Properly propagate `userRepo` and `userService` instances

> This approach allows us to validate the database state after each test using `userRepo`, ensuring our service layer works correctly with the persistence layer.

## Application Entry Point and Resource Management

The `Main` object serves as the entry point for our Cats Effect application, extending `IOApp.Simple` to provide a pure functional runtime.

Here is the code of the `Main` object:

```scala
object Main extends IOApp.Simple {

  given LoggerFactory[IO] = Slf4jFactory.create[IO]

  // Get the logger instance
  private val logger = org.typelevel.log4cats.slf4j.Slf4jLogger.getLogger[IO]

  private def transactor(config: DatabaseConfig): Resource[IO, HikariTransactor[IO]] = {
    val connectEC = ExecutionContext.global

    val hikariConfig = {
      val cfg = new HikariConfig()
      cfg.setDriverClassName(config.driver)
      cfg.setJdbcUrl(config.url)
      cfg.setUsername(config.user)
      cfg.setPassword(config.password)
      cfg.setConnectionTimeout(2000)
      cfg.setInitializationFailTimeout(2000)
      cfg.setValidationTimeout(2000)
      cfg.setConnectionTestQuery("SELECT 1")
      cfg
    }

    HikariTransactor.fromHikariConfig[IO](hikariConfig)
  }

  private def initialize(transactor: HikariTransactor[IO]): IO[Unit] = transactor.configure { dataSource =>
    IO {
      val flyway =
        Flyway
          .configure()
          .dataSource(dataSource)
          .lockRetryCount(10)
          .baselineOnMigrate(true)
          .validateMigrationNaming(true)
          .ignoreMigrationPatterns("*:pending")
          .schemas("public")
          .load()
      flyway.migrate()
      ()
    }
  }

  given Handle[IO, ApiError] = new CustomHandle

  private def createUserService(xa: HikariTransactor[IO], jwt: JwtConfig, bcrypt: BcryptConfig) = {

    val userRepo = new DoobieUserRepository[IO](xa)
    val jwtService = new JwtServiceImpl[IO](jwt)
    val passwordService = new PasswordServiceImpl[IO](bcrypt.cost)
    new UserServiceImpl[IO](userRepo, jwtService, passwordService)
  }

  override def run: IO[Unit] = {

    val serverResource = for {
      // Load configuration
      config <- Resource.eval(Config.load)
      _ <- Resource.eval(logger.info("Starting User Management API..."))

      // Initialize database
      xa <- transactor(config.database)
      _ <- Resource.eval(initialize(xa))

      // Assemble services and endpoints
      endpoints = new Endpoints[IO](createUserService(xa, config.jwt, config.bcrypt))

      // Create OpenAPI documentation
      openApiEndpoints = endpoints.allEndpoints.map(_.endpoint)
      docEndpoints = sttp.tapir.swagger.bundle
        .SwaggerInterpreter()
        .fromEndpoints[IO](openApiEndpoints, "User Management API", "1.0")

      // Start http server
      server <- NettyCatsServer
        .io()
        .flatMap { server =>
          Resource.make(
            server
              .host(config.http.host)
              .port(config.http.port)
              .addEndpoints(endpoints.allEndpoints ++ docEndpoints)
              .start()
          )(_.stop())
        }
    } yield (server, config) // Return both server and config

    // Run the application
    serverResource
      .use { case (server, config) =>
        logger.info(s"Server started at http://${config.http.host}:${config.http.port}") *>
          logger.info(s"API documentation available at http://${config.http.host}:${config.http.port}/docs") *>
          IO.never
      }
  }
}
```
### Resource Management and Application Lifecycle

The application uses Cats Effect's `Resource` type to manage the lifecycle of various components, ensuring proper acquisition and release of resources like database connections and server instances. The entire application follows this lifecycle:

#### Application Lifecycle Stages

1. **Configuration Loading**
    - `Resource.eval(Config.load)` loads application configuration

2. **Database Setup**
    - `transactor(config.database)` creates a connection pool
    - `initialize(xa)` runs Flyway migrations

3. **Service Assembly**
    - `createUserService` wires together all service dependencies

4. **API Construction**
    - Business endpoints are created from services
    - OpenAPI documentation is generated automatically

5. **Server Startup**
    - Netty server starts with all endpoints

6. **Runtime Execution**
    - Application runs until termination signal