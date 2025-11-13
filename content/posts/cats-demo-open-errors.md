---
title: "Open error model in Scala Cats applications"
date: 2025-11-13T00:00:00
draft: false
categories: [Scala]
tags: [Cats, Cats Effect, Error handling]
---
## Introduction

In a previous [article](https://jurajburian.github.io/posts/cats-user-demo/), 
I demonstrated a simple application using Cats and Cats Effect. We used an approach where errors were not explicitly represented in the API types.<br>This approach works well for small services with simple APIs.

Our final service was defined like this:

```scala
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

We used the `Raise`/`Handle` approach, and our error model was defined as a sealed trait hierarchy (though one could use enums as well).

In this article, I am going to refactor the application to use open errors. 
The refactored application is located in a new branch:<br> [https://github.com/JurajBurian/cats-user-demo/tree/open-errors](https://github.com/JurajBurian/cats-user-demo/tree/open-errors)

## Motivation
Generally, we want to provide precise descriptions of result types, including error types, at the API level.

One approach is to create sealed trait hierarchies (or enumerations) for errors. We can implement this in two ways:
1. A single hierarchy for the entire service's error model
2. Separate hierarchies for each service method

The first approach lacks precision since individual methods typically only generate a subset of possible errors. While the second approach provides better error model quality, it often results in significant code duplication and verbosity.  


## Open Error Model

> An open error model uses a set of independent error types that are not related through inheritance. Errors are defined as standalone entities.

In Scala 3, we can use type operators like `&` (intersection) and `|` (union) to define types. For example, we can specify that a result will be of type `Int | String`. This powerful abstraction allows us to define precise descriptions of result types and function parameters. 

But can we effectively use this mechanism to describe errors at the API level?

> _Remark:_ Open errors address Scala's type generalization in unions. When errors inherit from a common parent, the compiler collapses distinct error types into their shared parent type. This defeats the purpose of using union types for precise error handling. 

In the Cats ecosystem, we have the `cats.data.EitherT` monad transformer, which allows us to combine an arbitrary effect type `F` with the behavior of `Either`. This becomes particularly powerful when combined with union types in Scala 3. 


Let's define our error model with these independent case classes:

```scala
case class UserAlreadyExists(email: String,
    message: String = "User with this email already exists"
)
case class InvalidCredentials(
    message: String = "Invalid credentials"
)
case class InvalidOrExpiredToken(
    message: String = "Invalid or expired token"
)
case class InvalidOrExpiredRefreshToken(
   message: String = "Invalid refresh token"
)
case class UserNotFound(id: UUID, 
    message: String = "User not found"
)
case class AccountDeactivated(
    message: String = "Account has been deactivated"
)
case class InternalServerError(cause: String,
    message: String = "Internal server error"
)
```
Let's examine how we can define our services:
```scala
trait UserRepository[F[_]] {
  def create[E](userCreate: UserCreate, passwordHash: String):
    EitherT[F,  E | InternalServerError, User]
  def findByEmail[E](email: String): 
    EitherT[F, E | InternalServerError, Option[User]]
  def findById[E](id: UUID):
    EitherT[F, E | InternalServerError, Option[User]]
  def updateStatus[E](id: UUID, isActive: Boolean):
    EitherT[F, E | InternalServerError, Boolean]
  def findActive[E](offset: Long, count: Long):
    EitherT[F, E | InternalServerError, List[User]]
}

trait JwtService[F[_]] {
  def generateAccessToken[E](user: User): EitherT[F, E, String]
  def generateRefreshToken[E](userId: UUID): EitherT[F, E, String]
  def generateTokens[E](userId: User): EitherT[F, E, Tokens]
  def validateAndExtractAccessToken(token: String):
    EitherT[F, InvalidOrExpiredToken, AccessTokenClaims]
  def validateAndExtractRefreshToken(token: String):
    EitherT[F, InvalidOrExpiredRefreshToken, RefreshTokenClaims]
}

trait PasswordService[F[_]] {
  def hashPassword[E](password: String): EitherT[F, E, String]
  def verifyPassword[E](password: String, hash: String): EitherT[F, E, Boolean]
}

trait UserService[F[_]] {
  def createUser(userCreate: UserCreate):
    EitherT[F, InternalServerError | UserAlreadyExists, UserResponse]
  def login(loginRequest: LoginRequest):
    EitherT[F, InternalServerError | AccountDeactivated | InvalidCredentials, AuthResponse]
  def refreshTokens(refreshToken: String): 
    EitherT[F,
      InternalServerError | AccountDeactivated | InvalidOrExpiredRefreshToken | UserNotFound, 
      AuthResponse
   ]

  def getUser[E](id: UUID):
    EitherT[F, E | InternalServerError | UserNotFound, UserResponse]
  def updateUserStatus[E](id: UUID, isActive: Boolean):
    EitherT[F, E | InternalServerError, Boolean]
  def listActiveUsers[E](offset: Long, count: Long):
    EitherT[F, E | InternalServerError, List[UserResponse]]
  def validateUserForAccess[E](token: String): 
    EitherT[F, 
      E | InternalServerError | InvalidOrExpiredToken | AccountDeactivated | UserNotFound,
      UserResponse
    ]
}
```
Let's break down the key aspects of this refactoring:
* `EitherT` is used to combine an arbitrary effect type `F` on API level
* Union types are used to define error for each method
* Single type is used as result described in `EitherT`
* `E` - error is extension of each union type if the method can be combined with other methods  
  
### Understanding the `E` Type Parameter

The `E` type parameter in our service methods serves a crucial role in enabling flexible error composition. Let's examine why it's necessary and how it works.

When we chain operations in a for-comprehension, the compiler needs to unify the error types of all operations. The `E` type parameter allows us to:

1. **Preserve Error Information**: It preserves the ability to incorporate more specific error types when composing methods.
2. **Enable Flexible Composition**: It allows the same method to be used in different contexts where different error types might be relevant.
3. **Prevent Type Mismatches**: It helps the compiler verify that error types are properly propagated through the call chain.


### Composing operations example

Consider this implementation of `createUser`:

```scala
def createUser(userCreate: UserCreate): 
  EitherT[F, InternalServerError | UserAlreadyExists, UserResponse] = {
  for {
    existingUser <- userRepo.findByEmail(userCreate.email)
    _ <- EitherT.cond(existingUser.isEmpty, (), UserAlreadyExists(userCreate.email))
    passwordHash <- passwordService.hashPassword(userCreate.password)
    user <- userRepo.create(userCreate, passwordHash)
  } yield toUserResponse(user)
}
```
Here's the desugared version of the code:

```scala
def createUser(userCreate: UserCreate): 
  EitherT[F, 
    InternalServerError | UserAlreadyExists, UserResponse
  ] = {
    userRepo.findByEmail(userCreate.email).flatMap { existingUser =>
      EitherT.cond(existingUser.isEmpty, (), UserAlreadyExists(userCreate.email))
      .flatMap { _ =>
         passwordService.hashPassword(userCreate.password)
        .flatMap { passwordHash =>
           userRepo.create(userCreate, passwordHash)
           .map { user => toUserResponse(user) }
       }
     } 
  }
}
```
If we look on signature of `EitherT:flatMap` method:
`def flatMap[AA >: A, D](f: B => EitherT[F, AA, D])(implicit F: Monad[F]): EitherT[F, AA, D]`
we can see that there is AA>:A - thus the error type can only grow (become more general) through `flatMap` chains

The compiler handles this through type constraint solving:

- Final type: `InternalServerError | UserAlreadyExists`
- userRepo methods contribute: `E | InternalServerError`
- `EitherT.cond` contributes: `UserAlreadyExists`
- Solution: E = UserAlreadyExists

> _Note:_ The `[E]` type parameter is required for successful compilation.


#### When to Use [E] Type Parameter

- Composition & Propagation
    - Are likely to be composed with other operations in for-comprehensions
    - Need to propagate errors from callers or dependent operations
    - Act as building blocks in larger computation chains
- Flexibility & Reusability
    - Should work in different contexts with different error requirements
    - May be used by multiple callers with varying error hierarchies
    - Serve as generic utilities across different domains
- Library & Infrastructure Code
    - Are part of reusable libraries or framework components
    - Need to preserve error types through middleware/decorators
    - Require testability with different error scenarios

## Transforming results
The `EitherT[F, ..., UserResponse]` type often needs conversion to `F[Either[..., UserResponse]]`, as shown in `Endpoints.scala`:

```scala
class Endpoints[F[_]](userService: UserService[F])(using M: Monad[F]) {

  private val basePath = "api" / "v1"
  private val bearerTokenHeader = auth.bearer[String]()

  // Define specific error variants for each ApiError type
  // ...
  private val invalidOrExpiredTokenError =
    oneOfVariant(StatusCode.Unauthorized, jsonBody[InvalidOrExpiredToken])
  private val accountDeactivatedError =
    oneOfVariant(StatusCode.Forbidden, jsonBody[AccountDeactivated])
  private val userNotFoundError =
    oneOfVariant(StatusCode.NotFound, jsonBody[UserNotFound])
  private val internalServerError =
    oneOfVariant(StatusCode.InternalServerError, jsonBody[InternalServerError])
  // ...
  val getUserEndpoint: ServerEndpoint[Any, F] =
    endpoint.get
      .in(basePath / "users" / path[UUID]("id"))
      .in(bearerTokenHeader)
      .out(jsonBody[UserResponse])
      .errorOut(
        oneOf(
          invalidOrExpiredTokenError,
          userNotFoundError,
          accountDeactivatedError,
          internalServerError
        )
      )
      .serverLogic { case (id, accessToken) =>
        (for {
          _ <- userService.validateUserForAccess(accessToken)
          user <- userService.getUser(id)
        } yield user).value
      }
      //...
}
```
The `.value` method converts `EitherT[F, ..., UserResponse]` to `F[Either[..., UserResponse]]`.<br>
> _Remark:_ This example demonstrates method composition by chaining `validateUserForAccess` and `getUser` operations. This is why we use the `[E]` type parameter at the API level for these operations.

## Catching error
The `UserRepository` service implementation provides a good example:
```scala
class DoobieUserRepository[F[_]: Async](xa: Transactor[F]) extends UserRepository[F] {

  extension [A](fa: F[A])
  private def attemptDb[E]: EitherT[F, E | InternalServerError, A] =
    fa.attemptT.leftMap(ex => InternalServerError(ex.getMessage))

  def create[E](userCreate: UserCreate, passwordHash: String): EitherT[F, E | InternalServerError, User] =
    sql"""
         |INSERT INTO users (email, username, password_hash, first_name, last_name)
         |VALUES (${userCreate.email}, ${userCreate.username}, $passwordHash,
         |       ${userCreate.firstName}, ${userCreate.lastName})
         |RETURNING id, email, username, password_hash, first_name, last_name,
                is_active, created_at, updated_at""".stripMargin
      .query[User]
      .unique
      .transact(xa)
      .attemptDb[E]
  // ...
}
```
Here we catch database errors and convert them to `InternalServerError` (error handling is simplified for demonstration purposes) in the `attemptDb` extension method. 

## Caveats

* Use `.value` only at the end of the chain. The compiler may encounter issues when this transformation is used within intermediate method calls, as it can lead to type inference problems. For example:   

```scala
  def login2(
      loginRequest: LoginRequest
  ): F[Either[InternalServerError | AccountDeactivated | InvalidCredentials, AuthResponse]] = {
    (for {
      userOpt <- userRepo.findByEmail(loginRequest.email)
      user <- EitherT.fromOption(userOpt, InvalidCredentials())
      _ <- isActive(user)
      isValid <- passwordService.verifyPassword(loginRequest.password, user.passwordHash)
      _ <- EitherT.cond(isValid, (), InvalidCredentials())
      tokens <- jwtService.generateTokens(user)
    } yield AuthResponse(tokens, toUserResponse(user))).value
  }
```
causes compiler error:
```
[error] -- Error: .../UserServiceImpl.scala:44:6 
[error] 44 |      _ <- EitherT.cond(isValid, (), InvalidCredentials())
[error]    |      ^
[error]    |Recursion limit exceeded.
[error]    |Maybe there is an illegal cyclic reference?
[error]    |If that's not the case, you could also try to increase the stacksize using the -Xss JVM option.
[error]    |For the unprocessed stack trace, compile with -Xno-enrich-error-messages.
[error]    |A recurring operation is (inner to outer):
[error]    |
[error]    |  traversing for avoiding local references io.github.jb.domain.InvalidCredentials | io.github.jb.domain.InternalServerError
[error]    | E
[error]    |  traversing for avoiding local references io.github.jb.domain.InvalidCredentials | io.github.jb.domain.InternalServerError
...
```

* The compiler cannot detect unused error types declared at the API level. However, it will catch cases where an error type is used but not declared in the method signature
* Using a parent type in the error model prevents distinguishing between child types in union types. For example, if we have an `Error` parent with two children `Child1` and `Child2`, and another error `Error2` outside the hierarchy, then `Child1 | Child2 | Error2` is equivalent to `Error | Error2`
* Error transformations can be performed using the `leftMap` method
* When to use open error model?
  * Teams comfortable with advanced Scala type system features
  * Services requiring precise error documentation
  * Applications where different components have largely disjoint error sets
  * Greenfield projects in Scala 3 with the possibility to use open error model from the start
 
## Conclusion

The open error model represents a trade-off between type precision and complexity. It's a powerful pattern when used appropriately, but requires careful consideration of your team's expertise and project requirements.

Let's consider the ZIO monad for a moment. In ZIO, the IO monad is defined as `ZIO[R, E, A]` where:
- `R` represents the environment/requirements
- `E` represents the error type
- `A` represents the success type

This means error handling is built into ZIO's type system - it's a bifunctor with distinct success and failure channels. In contrast, Cats Effect's `IO[A]` is simpler, where `A` represents the successful result type, and error handling is not part of the type parameters. Using EitherT on API level partially mimic this behavior.