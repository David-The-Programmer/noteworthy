# How to handle errors in Gleam

In many programming languages, throwing exceptions are used as a way to handle errors. However, in Gleam, you handle errors by returning a value known as the `Result` type.

## What is the `Result` type?

A `Result(value, error)` is returned by functions that could have errors upon execution. It can have one of two values, either `Ok(value)` or `Error(error)`. 

Note that `value` and `error` can be any data type, like `Int`, `Bool`, `String`, `List(a)`, etc.

For example, let's say you are trying to find a specific element in a list. We can use the `find` function from the `gleam/list` module to do so. The function signature of `find` is as such:

```gleam
pub fn find(
  list: List(a),
  is_desired: fn(a) -> Bool,
) -> Result(a, Nil)
```
Note that the value returned here would be either `Ok(a)`, (where `a` is an element in `list`) if `a` causes `is_desired` to return `True`, or `Error(Nil)`if all elements in `list` causes `is_desired` to return `False`.

## How to handle the `Result` type?

In Gleam, we can use a few ways: pattern matching, `result.map`, `result.map_error` and `result.try`.

### Pattern matching

Building on the previous example, we can use pattern matching shown below:

```gleam
import gleam/list
import gleam/int
import gleam/io

pub fn main() {
  let find_result = list.find([1, 2, 3], fn(a) { a == 4 })
  case find_result {
    Ok(a) -> io.println(int.to_string(a))
    Error(Nil) -> io.println("Cannot find element")
  }
}
```
The code above will print `Cannot find element` in the console. If we change `is_desired` to `fn(a) { a == 3 }`, `3` will be printed instead.

Let's use a more realistic example. Let's say we are using a SQlite database and we want to query some rows of user data from the database. We will use the [`sqlight`](https://github.com/lpil/sqlight) library module.

Using just pattern matching alone, without using `result.try`, we will write something like this:
```gleam

import sqlight

type User {
  User(id: String, name: String)
}

type Error {
    UserNotFound(user_id: String, issue: sqlight.Error)
    InternalError(issue: sqlight.Error)
}

pub fn get_user_data(user_id: String, db_conn: sqlight.Connection) -> Result(User, Error) {
  let user_decoder = { 
    //...decoder implementation (not relevant to this example)
  }
  let sql = "
    SELECT id, name 
    FROM users
    WHERE id = ?;"
  let query_result = sqlight.query(sql, db_conn, [sqlight.string(user_id)], user_decoder)
  case query_result {
    Ok(user) -> Ok(user) // I know its not the most optimal, this is just an example
    Error(error) -> {
      case error.code {
        Notfound -> UserNotFound(user_id, error) // we wrap the sqlight.Error with our own to be able to trace where the error comes from
        _ -> InternalError(error)
      }
    }
  }
}

pub fn main() -> Nil {
  case sqlight.open("/path/to/sqlite/db/file") {
    Ok(db_conn) -> {
      case get_user_data("<some_user_id>", db_conn) {
        Ok(user) -> { 
          echo "Hello " <> user.name // usage of echo is purely for the example
          Nil
        }
        Error(err) -> {
          echo err
          Nil
        }
      }
    }
    Error(err) -> {
      echo err
      Nil
    }
  }
}
```
### Using `result.try` and `result.map_error`

Well, the code doesn't look that bad, but it would be a mess if more functions are needed to be called when the previous function succeeds, i.e, returns `Ok(a)` . 

Here is where `result.try` can help, `result.try` has the signature shown below:

```gleam
pub fn try(
  result: Result(a, e),
  fun: fn(a) -> Result(b, e),
) -> Result(b, e)
```
In essense, `result.try` takes in the `result` argument, and executes the `fun` function if `result` is `Ok(a)`, i.e, `fun` will never execute if `result` is an `Error(e)`. This allows us to chain results together when each result requires the previous result to be `Ok`. This behaviour is exactly what we need.

So, using `result.try`, we can refactor the code above.

Let's refactor the code in `main`.

```gleam
pub fn main() -> Nil {
  let result = result.try(
    sqlight.open("/path/to/sqlite/db/file"),
    fn(db_conn) {
      get_user_data("<some_user_id>", db_conn)
    }
  )
  case result {
    Ok(user) -> { 
      echo "Hello " <> user.name // usage of echo is purely for the example
      Nil
    }
    Error(err) -> {
      echo err
      Nil
    }
  }
}
```
Looks better, but we will get a compiler error telling us that the type expected for `result` variable is `Result(User, Error)`, but got `Result(User, sqlight.Error)` instead. This is because `sqlight.open` returns `Result(sqlight.Connection, sqlight.Error)`, but `get_user_data`returns `Result(User, Error)`.
Thus, one thing to note: **the error type returned for both the `result` parameter and the `fun` function parameter has to be the same.**

This is where error handling gets slightly annoying, **for whatever different types of errors you have to handle, you have to make your own error type to "wrap" those types of errors.**
Since `sqlight.open` returns the error type `sqlite.Error`, we need to create another variant of our `Error` type to "wrap" it.

```gleam
type Error {
    OpenDBError(issue: sqlight.Error) // new variant
    UserNotFound(user_id: String, issue: sqlight.Error)
    InternalError(issue: sqlight.Error)
}
```

Let's create a new function to wrap `sqlight.open` to fix the compilation error.

```gleam
pub fn open_db(db_file: String) -> Result(sqlight.Connection, Error) {
  case sqlite.open(db_file) {
    Error(err) -> OpenDBError(err)
    x -> x
  }
}

pub fn main() -> Nil {
  let result = result.try(
    open_db("/path/to/sqlite/db/file")
    fn(db_conn) {
      get_user_data("<some_user_id>", db_conn)
    }
  )
  case result {
    Ok(user) -> { 
      echo "Hello " <> user.name // usage of echo is purely for the example
      Nil
    }
    Error(err) -> {
      echo err
      Nil
    }
  }
}
```

Let's do the same for `get_user_data`. But if we look at this function again, `result.try`would not really help as we are mainly manipulating the errors, not the values in `Ok(a)`

This is where we can use `result.map_error`, which has the following signature:
```gleam
pub fn map_error(
  result: Result(a, e),
  fun: fn(e) -> f,
) -> Result(a, f)
```

In short, `result.map_error` allows us to transform the value `e` in the `Error(e)` from the `result` parameter given to some other value `f` using the `fun` parameter. 

Note that `result.map_error` only executes `fun` if `result` is of `Error(e)` type.
Also, if the `result` parameter is not an error, i.e, `Ok(a)`, then `result.map_error` will return `Ok(a)`.

This allows us to refactor the `get_user_data` function as such from this:

```gleam

pub fn get_user_data(user_id: String, db_conn: sqlight.Connection) -> Result(User, Error) {
  let user_decoder = { 
    //...decoder implementation (not relevant to this example)
  }
  let sql = "
    SELECT id, name 
    FROM users
    WHERE id = ?;"
  let query_result = sqlight.query(sql, db_conn, [sqlight.string(user_id)], user_decoder)
  case query_result {
    Ok(user) -> Ok(user) // I know its not the most optimal, this is just an example
    Error(error) -> {
      case error.code {
        Notfound -> UserNotFound(user_id, error)
        _ -> InternalError(error)
      }
    }
  }
}
```

into this:

```gleam
pub fn get_user_data(user_id: String, db_conn: sqlight.Connection) -> Result(User, Error) {
  let user_decoder = { 
    //...decoder implementation (not relevant to this example)
  }
  let sql = "
    SELECT id, name 
    FROM users
    WHERE id = ?;"
  sqlight.query(sql, db_conn, [sqlight.string(user_id)], user_decoder)
  |> result.map_error(
        fn(error) {
          case error.code {
          Notfound -> UserNotFound(user_id, error)
          _ -> InternalError(error)
        }
      }
    )
}
```
If we look at our `open_db` function, we can also use `result.map_error` since we only care about wrapping the error from `sqlight.open`

So we can change it from this:
```gleam

pub fn open_db(db_file: String) -> Result(sqlight.Connection, Error) {
  case sqlite.open(db_file) {
    Error(err) -> OpenDBError(err)
    x -> x
  }
}
```
into this:

```gleam

pub fn open_db(db_file: String) -> Result(sqlight.Connection, Error) {
  sqlight.open(db_file)
  |> result.map_error(fn(err) { OpenDBError(err) })
}
```

So, now our entire code looks like this:

```gleam
import sqlight
import gleam/result

type User {
  User(id: String, name: String)
}

type Error {
    OpenDBError(issue: sqlight.Error) // new variant
    UserNotFound(user_id: String, issue: sqlight.Error)
    InternalError(issue: sqlight.Error)
}

pub fn open_db(db_file: String) -> Result(sqlight.Connection, Error) {
  sqlight.open(db_file)
  |> result.map_error(fn(err) { OpenDBError(err) })
}

pub fn get_user_data(user_id: String, db_conn: sqlight.Connection) -> Result(User, Error) {
  let user_decoder = { 
    //...decoder implementation (not relevant to this example)
  }
  let sql = "
    SELECT id, name 
    FROM users
    WHERE id = ?;"
  sqlight.query(sql, db_conn, [sqlight.string(user_id)], user_decoder)
  |> result.map_error(
        fn(error) {
          case error.code {
          Notfound -> UserNotFound(user_id, error)
          _ -> InternalError(error)
        }
      }
    )
}

pub fn main() -> Nil {
  let result = result.try(
    open_db("/path/to/sqlite/db/file")
    fn(db_conn) {
      get_user_data("<some_user_id>", db_conn)
    }
  )
  case result {
    Ok(user) -> { 
      echo "Hello " <> user.name // usage of echo is purely for the example
      Nil
    }
    Error(err) -> {
      echo err
      Nil
    }
  }
}
```
Looks not too bad, but can be better.

### Using the `use` keyword

Let's say, we are coding some APIs for the backend of an online retail store, and one of the APIs needs to return products recommended to the user for purchase (like how amazon recommends products to purchase when viewing a product)

```gleam
import sqlight
import gleam/result

type User {
  User(id: String, name: String)
}

type Product {
  Product(id: String, name: String, price: Float)
}

type Error {
    OpenDBError(issue: sqlight.Error)
    UserNotFound(user_id: String, issue: sqlight.Error)
    InternalError(issue: sqlight.Error)
}
pub fn open_db(db_file: String) -> Result(sqlight.Connection, Error) {
  // same implementation as above
}

pub fn get_user_data(user_id: String, db_conn: sqlight.Connection) -> Result(User, Error) {
  // same implementation as above
}

pub fn get_recommended_products(user_data: User) -> Result(List(Product), Error) {
  // some other implementation
}

pub fn main() -> Nil {
  let products_result = 
    result.try(
      open_db("/path/to/sqlite/db/file"),
      fn(db_conn) {
        result.try(get_user_data("<some_user_id>", db_conn), fn(user_data) {
          get_recommended_products(user_data)
        })
      }
    )
  case products_result {
    Ok(products) -> { 
      echo products
      Nil
    }
    Error(err) -> {
      echo err
      Nil
    }
  }
}
```

Even with `result.try`, the nesting is getting a bit too unwieldy. We basically now have callback hell. However, we can solve this using the `use` keyword.

Let's zoom in on this snippet of the code above:
```gleam
result.try(
  open_db("/path/to/sqlite/db/file"), 
  fn(db_conn) {
    // inner body...
  }
)
```

With `use`, it becomes: 
```gleam
use db_conn <- result.try(open_db("/path/to/sqlite/db/file"))
// inner body...
```
What happened? 
1. The `db_conn` parameter is now "pulled" to the left side.
2. The inner body of the `fn(db_conn)` function argument now becomes everything below the statement containing `use`
3. The `db_conn` parameter (not `Ok(db_conn)`) can now be freely used below the statement containing `use`.

So, we this powerful syntatic sugar, we can rewrite our `main` function code to be:

```gleam
pub fn main() -> Nil {
  // have to wrap in a block
  let product_result = {
    use db_conn <- result.try(open_db("/path/to/sqlite/db/file"))
    use user_data <- result.try(get_user_data("<some_user_id>", db_conn))
    get_recommended_products(user_data)
  }
  case product_result {
    Ok(products) -> { 
      echo products
      Nil
    }
    Error(err) -> {
      echo err
      Nil
    }
  }
}
```
Having `use` with `result.try` actually replicates early error returns in other procedural languages, such as Go. Thus, if the line:

```gleam
use db_conn <- result.try(open_db("/path/to/sqlite/db/file"))
``` 
fails, i.e, return `Error(e)`, the lines below will not execute and that `Error(e)` is returned, thus `product_result` will be the value `Error(e)`.

Also note that when using `use` with `result.try`, you would have to return a `Result` type at the end.

```gleam
// ...
  let product_result = {
    use db_conn <- result.try(open_db("/path/to/sqlite/db/file"))
    use user_data <- result.try(get_user_data("<some_user_id>", db_conn))
    get_recommended_products(user_data) // have to call this function to return a Result
  }
// ...
```

**So, for me at least, I realised when doing error handling in Gleam, always try to use the `use` keyword with `result.try`. Then do the pattern matching towards the very end.**

### Why `Result` type is better than throwing exceptions

The main reason is because, you know exactly if and how a function will fail once executed.

Many people probably do not know that `JSON.parse` in JavaScript actually throws an Exception if unable to parse whatever json string you give it. The only way to know is to read the API documentation. But, most people do not do that for every function that they will use. Thus, you truly never know if and when a function will throw an Exception, which in the context of building highly reliable and scalable code, is terrible for.

Thus, it is much better for the function signature to return a `Result`, as if to say: "Hey, I can either succeed and return the value you want, or fail and return an Error type that is properly defined". Hence, the user of the function knows explicitly if the function will fail and all possible error values to handle.

### Bonus: Making your own `defer` function

I have to give full credit to [Issac Harris-Holt](https://www.youtube.com/@IsaacHarrisHolt) for this, I only knew about making my own `defer` function after watching his video on the `use` keyword. Link to the video found [here](https://www.youtube.com/watch?v=-rtVWja_vJI).

#### What is `defer` and why is it useful?

In the Go programming language, there is the `defer` keyword. It allows you to defer the execution of a function inside another function such the inner function will only execute after all of the code after the inner function finishes execution. 

```go
package main

import "fmt"

func main() {
	defer fmt.Println("inner function executed")
	fmt.Println("all code after inner function executed")
}
```
The output to stdout would be:

```
all code after inner function executed
inner function executed
```

Thus, `defer` is incredibly useful when you need to close the database connection after execution of a statement or need to run a function for cleanup.

Note that when you have multiple `defer` statements, the last `defer` statement will execute first, i.e, it executes in a Last In First Out order (like a stack). Thus, for example, in the code snippet below:

```go
package main

import "fmt"

func main() {
	defer fmt.Println("function 1")
	defer fmt.Println("function 2")
	defer fmt.Println("function 3")
	fmt.Println("main code executed")
}
```
The output to stdout would be:

```
main code executed
function 3
function 2
function 1
```

#### Defining the `defer` function in Gleam

Issac Harris Holt basically shows this code snippet of the `defer` function you can implement yourself in Gleam:
```gleam
fn defer(cleanup: fn() -> Nil, body: fn() -> a) -> a {
  let res = body()
  cleanup()
  res
}
```

#### Using `use` with the `defer` function

Without the `use` keyword, the `defer` function is a bit clunky to use (no pun intended, hehe). Building from the online store backend API example, let's say we need to implement some function for some API add users to the backend database when users sign up, and we need to make sure we close the database connection after the function finishes execution.

Usage of the function would look something like this:
```gleam
pub fn add_user(user: User, db_conn: sqlight.Connection) -> Result(Nil, Error) {
  defer(
    close_db_connection, // NOTE: We are referring to the function, hence no parenthesis. Adding parenthesis will execute the function instead
    fn() {
      // ...code to add user to backend database
    }
  )
}
```
Currently doesn't seem that bad, but if we have multiple functions to defer, it would be a mess:
```gleam
pub fn add_user(user: User, db_conn: sqlight.Connection) -> Result(Nil, Error) {
  defer(
    close_db_connection,
    defer(
      some_other_function,
      defer(
        another_function,
        fn() {
          // ...code to add user to backend database
        }
      )
    )
  )
}
```
But with the `use` keyword, say no more to the callback hell:
```gleam
pub fn add_user(user: User, db_conn: sqlight.Connection) -> Result(Nil, Error) {
  use <- defer(close_db_connection)
  use <- defer(some_other_function)
  use <- defer(another_function)
  // ...code to add user to backend database
}
```
I hope this article has made it clearer on how to handle errors in Gleam. Hope you had fun with Gleam!
