# Humans codestyle guide
Before starting to read this guide, you should be confident that you are familiar with basic aspects of Go codestyle:
- [Effective Go](https://golang.org/doc/effective_go.html)
- [CodeReview Comments](https://github.com/golang/go/wiki/CodeReviewComments)

## Contents
- [Go code](#go-code)
    - [Files](#files)
    - [Packages](#packages)
    - [File structure](#file-structure)
    - [Imports](#imports)
    - [Variables](#variables)
    - [Functions and methods](#functions-and-methods)
    - [Operators](#operators)
    - [Constants](#constants)
    - [Errors](#errors)
    - [Interfaces](#interfaces)
    - [Logs](#logs)
    - [Advices](#advices)
- [Go tests](#go-tests)
- [Proto](#proto)
- [GraphQL](#graphql)
    - [Errors](#graphql errors)
- [Indentation](#indentation)


## Go code
### Files
- Filename should identify it's content. If you call it `entities` it shouldn't contain functions.

- Filenames start with lowercase and consist only of lowercase, underscore and numbers.

  Use numbers only if it needed to identify content (`sha256`, `cvv2`)

- Filenames do not contain the package name unless they repeat it (`entity/user`, not `entities/entity_user`)

### Packages
- All lower-case. No capitals or underscores. For example, `masterslave`, not `master_slave`

  One exception: when package name is created from some another names (variables or paths) and it could be too long.
  Use underscore `_` in this case.

- Be original. Does not need to be renamed using named imports at most call sites. Never use package names from SDK.

- Short and succinct. Remember that the name is identified in full at every call site.

- Not plural. For example, `net/url`, not `net/urls`.

- Not `common`, `util` or `lib`. These are bad, uninformative names.

- Packages defines domain, not an implementation of something. Packages are not groups, they are layers of application.

  *Packages are non-cyclical so if you can't think of how you'd stack your packages together like layers on a cake then your structure is probably off.* (c) Ben Johnson


If you haven't read it, just do it:
[Package names from authors](https://blog.golang.org/package-names)

### File structure
1. Types declaration
2. Types constructors
3. Exported methods
4. Exported functions
5. Unexported methods
6. Unexported functions

### Imports
- One `import` block on file

- Imports should be separated into 3 blocks: SDK, third-party packages, internal packages.

- Rename imported packages only in case when their names are identical or name have incorrect format (e.g. `some-package` or `somePackage`).

- Alias should be original and shouldn't be like a variable name (`grpcclient`, not `client`)

For simple understanding how to write imports:
1. Write all imports in one block
2. Use `goimports -w -local pkg.humans.net .`

### Variables
- No globals other than const and errors are allowed.

- One letter variable can be only in very small scope (e.g. in operators scope)

- No variable shadowing.

  Bad:
    ```go
    const defaultRetryAmount = 2
    ...

    var retryAmount int
    if retryAmount, err := repo.GetRetryAmountByUser(userID); err != nil {
        retryAmount = defaultRetryAmount
        ...
    }
    fmt.Println(retryAmount) // We'll see 0 instead of 2
    ```

  Good:
    ```go
    const defaultRetryAmount = 2
    ...

    retryAmount, err := repo.GetRetryAmountByUser(userID)
    if err != nil {
        retryAmount = defaultRetryAmount
        ...
    }
    ```

- Declaration:
    - Default value
        ```go
        var amount int
    
        var WelcomeMsg Message 
        ```
    - Concrete value:
        ```go
        amount := 10
    
        WelcomeMsg := Message{
            From: "Me",
            To: somebody,
            Text: "Hi, bla-bla"
        }
        ```

  Period. No another variants.

- Declare variables as close to the first usage as possible.

  A rule of thumb:
  *The greater the distance between a name's declaration and its uses,
  the longer the name should be.*

- Group similar declaration. More than two `var`-words in a row is a bad style.

- At the top level, use the standard `var` keyword. Do not specify the type, unless it is not the same type as the expression.

  Bad:
    ```go
    var A bool = true
    ```

  Good:
    ```go
    var A = true
    ```

- Reduce scope of variables as possible. Especially, in operators.

  Bad:
    ```go
    err := someFunc()
    if err != nil {
    ...
    }
    ```

  Good:
    ```go
    if err := someFunc(); err != nil {
    ...
    }
    ```
- Use `is` prefix for boolean variables (e.g. `isAdmin`, not `admin`)

- Don't use words `slice`, `array`, `map`, `chan` etc. in variable names.

  Think carefully about what variable really represents.

- Use blank identifiers `_` to mark all variables that are not used.

- If code lines count is more 10+, use one style structure initialization:

  Inline:
    ```go
    isTestRequest := false
    if env == "stage" {
        isTestRequest = true
    }
    req := Request{
        UserID: id.New(),
        IsTest: isTestRequest,
    }
    ```

  Or sequentially:
   ```go
    var req Request // (not req := new(Request) and not req := Request{})
    if env == "stage" {
        req.IsTest = true
    }
    ```

  Don't use both variants:
    ```go
    req := Request{
        UserID: id.New(),
    }
    if env == "stage" {
        req.IsTest = true
    }
    ```

#### Functions and methods
- No init functions (maybe, only for `prometheus` as an exception).

- Use `New()` or `NewSmth()` (if it's not obvious from package name) as constructors

- Use `smth.Init()` for initialisation of already allocated/constructed variables. Don't use `Init()` as global function.

- Use named return parameters carefully

  Name return parameters only if the types do not give enough information about what function or method actually returns.

- Avoid shallow functions which uses one-two times

  Bad:
    ```go
    ...
        doAndLog(ctx, param)
    ...
    }

    func doAndLog(ctx context.Context, param string) {
       if err := do(); err != nil {
          log.FromContext(ctx).With(err).Error("do failed")
       }
    }
    ```
  Better:
    ```go
    ...
        if err := do(); err != nil {
            log.FromContext(ctx).With(err).Error("do failed")
        }
    ...
    }
    ```

- Functions should not be too long. (e.g. 100+ lines - it's already *too long*)

- Well-named functions is more (**MORE**) preferable than well-commented

- Also, well-named helper functions are preferable rather than code blocks with comments

  Bad:
    ```go
    const defaultName = "John Doe"
    func SendMultiple(s Sender, msg string, users []User) error {
        // This is used because in another service name is required
        // it should be equal or more than 8 letters
        var contacts []Contact 
        for _, u := range users {
            if len(u.Name) == 0 {
                u.Contact.Name = defaultName
            }
            contacts = append(contacts, u.Contact)
        }

        s.SendTo(msg, contacts)
    }
    ```

  Good:
    ```go
    const defaultName = "John Doe"
    func SendMultiple(s Sender, msg string, users []User) error {
        contacts := prepareContactsForSender(users)
        s.SendTo(msg, contacts)
    }

    func prepareContactsForSender(users []User) []Contact {
        var contacts []Contact 
        for _, u := range users {
            if len(u.Name) == 0 {
                u.Contact.Name = defaultName
            }
            contacts = append(contacts, u.Contact)
        }

        return contacts
    }
    ```

- Prefer function/method definitions with arguments in a single line. If it's too wide, put each argument on a new line.

    ```go
    func function(
        argument1 int,
        argument2 string,
        argument3 time.Duration,
        argument4 SomeType,
    ) (int, error) {
    ...
    }
    ```

  One exception would be when you expect the variadic (e.g. `...string`) arguments to be filled in pairs.

- Functions in a file should be grouped by a receiver.

- If you declare a `Printf`-style function, make sure that `go vet` can detect it and check the format string.

- Any kind of arguments are passed to the func expected to be not nil values. E.g. pointers, interfaces, funcs, etc.

    ```go
    func Method(ctx context.Context, d Doer, data *Data, fn func(), opts ...MethodOption) {
        // it's safe to use any of the passed argument
    }
    ```

  Exceptions could be made for the funcs used in 3rd party libraries. In this case received nil values should be handled properly to avoid panics.

- Nilable values should be checked before they are going to be passed to a func. Exceptions could be made for a func wich used especially for validation.

  Bad:
  ```go
  s := Struct{Data: nil}
  obj.Method(ctx, s.Data)
  ```

  Good:
  ```go
  s := Struct{Data: nil}
  var data *Data
  if s.Data == nil{
      return fmt.Errorf("data is nil") // or fill data variable with a value.
  }

  obj.Method(ctx, data)
  ```

- Function calls should not contain nil values as an arguments. Nillable arguments could be replaced by non pointer values
  or options pattern for example or something else with obvious agrument value.

  Bad:
  ```go
  obj.Method(ctx, nil)
  ```

  Good(an example, not rule):
  ```go
  type NilableType struct {
      Valid bool
      Value Type
  }
  obj.Method(ctx, NilableType{})

  // or for a case when nil struct is valid value and the struct could be used anyway
  var nilValue *Type
  obj.Method(ctx, nilValue)
  ```

#### Operators
- Reduce Nesting

  Code should reduce nesting where possible by handling error cases/special conditions first and returning early or continuing the loop.
  Reduce the amount of code that is nested multiple levels.

- Else is unnecessary. Always.

- Avoid forever loops. Better to add restrictions in operator definition than in body.

#### Constants
- Start enums from `1`. Zero - for incorrect or unknown values.

- When you name global constants, remember about package name.

  Maybe some part of the constant name is unnecessary, or it may look incorrect outside the package.

#### Errors
- Don't forget to handle all returned errors

  It's easy to forget to check the error returned by a `Close` method that we deferred.
    ```go
    f, err := os.Open(...)
    if err != nil {
        // handle..
    }
    defer f.Close() // What if an error occurs here?
    
    // Write something to file... etc.
    ```
  Unchecked errors like this can lead to major bugs.

  Consider the above example: the `*os.File` Close method can be responsible for actually flushing to the file, so if an error occurs at that point, the **whole write might be aborted**!

- Init errors correctly

  We use only two ways to initiate errors:
    - When we don't have any parameters (rarely)
    ```go
    err := errors.New("some error")
    ```

    - When we have some of them (and even when we don't have too)
    ```go
    err := fmt.Errorf("some error with %q because of: %w", stringParam, errAnother)
    ```
- Base `errors` package is enough for everything.

  **Don't use another libraries for wrapping errors like `pkg/errors`.**

- Don't Panic

  Code running in production must avoid panics.
  Panics are a major source of cascading failures.
  If an error occurs, the function must return an error and allow the caller to decide how to handle it.

- Try not to use unnecessary words

  For example `can't`, `shouldn't`, `must be` is unnecessary in many cases.
  We already know that it's an error and message like:
    ```go
    return fmt.Errorf("saving user: %w", err)
    // or
    return errors.New("parse argument: %v is not true", truth)
    ```
  is enough for understanding.

#### Interfaces

- Keep interfaces as narrow as possible.

  Bad:
    ```go
    type InputHandler interface {
        Parse(SomeInput) (string, error)
        ParseAnother(AnotherInput) (string, error)
        SendSomewhere(SomeInput) error
    }
    ```

  Better:
    ```go
    type SomeHandler interface {
        Parse(SomeInput) (string, error)
        SendSomewhere(SomeInput) error
    }

    type AnotherParser interface {
        ParseAnother(AnotherInput) (string, error)
    }
    ```

  Good:
    ```go
    type Parser interface {
        ParseSome(SomeInput) (string, error)
        ParseAnother(AnotherInput) (string, error)
    }

    type Sender interface {
        SendSomewhere(SomeInput) error
    }
    ```

  Interfaces for generating mocks can be exception (like services, repos etc.).  
  But better to avoid huge interfaces where it can be useful.

- Naming.

  Never use words `Interface` or another special prefixes/suffixes (`I`, `Abstract` etc.) in their names.

  Name of interfaces should describe what the interface DO. It describes behavior instead of entity.

  Names in `Doer`-style (parser, saver, stringer...) is very useful for it.
  You can even use fictional or not really suitable words for this (eg stringer is not a bowstring).

  Bad:
    ```go
    type String interface {
        IntToString(int) string
        BoolToString(bool) string
        StringToInt(string) (int, error)
        StringToBool(string) (bool, error)
    }
    ```

  Better:
    ```go
    type Stringer interface {
        StringInt(int) string
        StringBool(bool) string
    }
    type Parser interface {
        ParseInt(string) (int, error)
        ParseBool(string) (bool, error)
    }
    ```

- Avoid pointers to interfaces.

  You almost never need a pointer to an interface. If you really need it, think about this twice.

- Verify interface compliance.
  For avoiding situations when you forgot to add changes to your interface, you may to add verification in your code.
    ```go
    type Handler struct {
      // ...
    }
    
    var _ http.Handler = (*Handler)(nil)
    
    func (h *Handler) ServeHTTP(
      w http.ResponseWriter,
      r *http.Request,
    ) {
      // ...
    }
    ```

#### Logs
- Use `snake_case` for logs variable

  Bad:
    ```go
    log.FromContext(ctx).With(log.MsgParams(zap.String("userID", uuid))).Debug("user ID")
    ```
  Good:
    ```go
    log.FromContext(ctx).With(log.MsgParams(zap.String("user_id", uuid))).Debug("user ID")
    ```

- All logging parameters should be wrapped in `log.MsgParams` and this wrapper should be used once per logger-call

#### Advices

- Avoid to use Reflect and Unsafe packages. Use those only for very specific, critical cases.

  Especially `reflect` tend to be very slow.

- Use raw string literals to avoid escaping. It's better for readability.

  Bad:
    ```go
    message := "unknown name: \"Ikakiy\""
    ```

  Good:
    ```go
    message := `unknown name: "Vasiliy"`
    ```

  Another Good:
    ```go
    message := fmt.Sprintf("unknown name: %q", name)
    ```

- Nil - valid slice

  It means that no needs to initiate empty slice if you don't fill it.

  Also, you can get `len(slice)` with no fears, that it will be `nil`.

  One problematic case - `reflect.DeepEqual`, from this function you will get, that they are not equal.

- Remember boyscout rule: "Leave the code cleaner than you found it."

### Go tests

- Make it.

### Proto

- Use CamelCase for message/service/rpc names and for enum elements
- Use snake_case for message field names
- Add validation rulles for message fields where possible. Dont use complex rules which could affect perfomance
- `Id` field is mostly ULID as UUID4 string but not necessary
- Group request/response per rpc method:
    ```proto
    message GetLabelsByUser {
        message Request {
            string user_id = 1 [(validate.rules).string.len = 36];
        }
        message Response {
            repeated Label labels = 1;
        }
    }
    ```

## GraphQL

### GraphQL Errors

Use graphql standard errors for unauthenticated, unavailable, not allowed and other transport and framework errors. e.g:
```json
{
  "errors": [
    {
      "message": "Unavailable",
      "path": [
        "me",
        "profile",
        "fintechUZ"
      ],
      "extensions": {
        "error_code": "Unavailable",
        "trace_id": "d787c98b1c45f7dc"
      }
    }
  ]
}
```

Use typed schema based on union of success result and business error for business cases.
```graphql
extend type Mutation {
    submitDelivery(input: SubmitDeliveryInput!): SubmitDeliveryResult!
}

union SubmitDeliveryResult = SubmitDeliveryOutput | SubmitDeliveryError

type SubmitDeliveryOutput {
    orderID: ID!
}

type SubmitDeliveryError {
    code: SubmitDeliveryErrorCode!
}

enum SubmitDeliveryErrorCode {
    ContactNotVerified
}
```

If error is not matched to business one - `return gqlerr.From(ctx, err)`
```go
    return SubmitDeliveryError{}, gqlerr.From(ctx, err)
```

### Mutations

- We place mutations in the `Mutation` object using schema stitching  e.g.
```graphql
extend type Mutation {
  doSmth(input: DoSmthInput!): DoSmthResult!
}
```

- Every mutation must accept single mandatory param with type postfix `Input` e.g.
```graphql
input DoSmthInput {
  id: ID!
}
````
- Every mutation must return union with at least success result or at most business error. e.g.:
```graphql
union DoSmthResult = DoSmthOutput | DoSmthError

type DoSmthOutput {
    id: ID!
}

type DoSmthError {
    code: DoSmthErrorCode!
}

enum DoSmthErrorCode {
    SmthBusinessSpecific
}
```
- We don't use prefixes or postfixes in naming at the moment.
- We choose humans readable names for the mutations. e.g.
```
#### Bad
```graphql
extend type Mutation {
  profileCreate(input: ProfileCreateInput!): ProfileCreateOutput!
}
```
#### Good
```graphql
extend type Mutation {
  createProfile(input: CreateProfileInput!): CreateProfileOutput!
}
```
- We group queries by domain.
#### Bad
```graphql
extend type Query {
  bankCardsList: List!
  bankCardPreferences: Preferences!
}
```
#### Good
```graphql
type BankCard {
  list: List!
  preferences: Preferences!
}

extend type Query {
  bankCard: BankCard!
}
```

## Indentation

Indentation with tabs where tab size is set to 4 spaces has to be used in source code in Go, SQL and other languages used in project
