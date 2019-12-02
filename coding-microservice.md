# Coding microservice with Rust

---

Let's build a microservice that will have 2 endpoints:

* `POST /users` - user registration (username + password)
* `POST /users/auth` - authenticating existing users (username + password), will
  return JWT token
* `GET /protected ` - "protected" route, authentication with JWT token

---

# What we will need

* Stable Rust version
* PostgreSQL

---

# Install stable Rust via `rustup`

```
# Install rustup - Rust toolchain installer
curl https://sh.rustup.rs -sSf | sh
```


Installation via `homebrew` (`brew install rust`) will for too for now.

---

# Install diesel.rs

```sh
$ cargo install diesel_cli
```


---

# Project setup


```bash
# Create new project:
cargo new --bin users-rs && cd users-rs

# Setup DATABASE_URL
echo DATABASE_URL=postgres://you@localhost/usersrs > .env

# Setup Diesel
diesel setup

# Add following dependencies to Cargo.toml
[dependencies]
diesel = { version = "1.0.0", features = ["postgres", "chrono"] }
dotenv = "0.9.0"
chrono = "0.4.6"
bcrypt = "0.3"
iron = "0.6.0"
router = "0.6.0"
bodyparser = "0.8.0"
serde = "1"
serde_json = "1"
serde_derive = "1"
jsonwebtoken = "5"

# Generate the first migration
diesel migration generate create_users

# Fetch dependencies & build
cargo build
```
---

# Write migrations for users table

```sql
// up.sql:
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR NOT NULL,
  password_hash VARCHAR NOT NULL,
  created_at TIMESTAMP NOT NULL,
  UNIQUE(username)
);

// down.sql:
DROP TABLE users;
```

---

# Run migrations

`diesel migration run`

You can re-run latest migration with:

`diesel migration redo`

---

# Check src/schema.rs

```rust
table! {
    users (id) {
        id -> Integer,
        username -> Text,
        password_hash -> Text,
        created_at -> Timestamp,
    }
}
```

---

# Check that tooling works - Hello world!

```sh
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
     Running `target/debug/auth-microservice-rs`
Hello, world!
```

---

# Let's write first endpoint

```rust
extern crate iron;
extern crate router;

use iron::{status, Iron, IronResult, Request, Response};
use router::Router;

fn create_handler(req: &mut Request) -> IronResult<Response> {
    Ok(Response::with((status::Ok, "Hello world!")))
}

fn main() {
    let mut router = Router::new();
    router.post("/users", create_handler, "create_handler");
    Iron::new(router).http("localhost:3000").unwrap();
}
```

---

# Test it out

```sh
$ cargo run
   Compiling auth-microservice-rs v0.1.0 (/Users/jz/Code/auth-microservice-rs)
warning: unused variable: `req`
 --> src/main.rs:7:19
  |
7 | fn create_handler(req: &mut Request) -> IronResult<Response> {
  |                   ^^^ help: consider using `_req` instead
  |
  = note: #[warn(unused_variables)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 1.73s
     Running `target/debug/auth-microservice-rs`
```

Then in another terminal

```sh
$ curl -X POST localhost:3000/users
Hello world!⏎
```

---

# Create src/lib.rs

```rust
#[macro_use]
extern crate diesel;
extern crate dotenv;

use diesel::prelude::*;
use diesel::pg::PgConnection;
use dotenv::dotenv;
use std::env;

pub fn establish_connection() -> PgConnection {
    dotenv().ok();

    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    PgConnection::establish(&database_url)
        .expect(&format!("Error connecting to {}", database_url))
}
```

---

# Add test-case for `establish_connection` into src/lib.rs

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_establish_connection_works() {
        let conn = establish_connection();
        conn.test_transaction::<_, diesel::result::Error, _>(|| Ok(()));
    }
}
```

---

# Run the test

```sh
$ cargo test
    Finished dev [unoptimized + debuginfo] target(s) in 0.15s
     Running target/debug/deps/auth_microservice_rs-0f013f52a97a20dc

running 1 test
test tests::test_establish_connection_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

     Running target/debug/deps/auth_microservice_rs-3698d7dc4655879e

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests auth-microservice-rs

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

---

# Create src/models.rs

```rust
#[derive(Queryable)]
pub struct User {
    pub id: i32,
    pub username: String,
    pub password_hash: String,
    pub created_at: diesel::data_types::PgTimestamp,
}
```

---

# Add `NewUser` struct to src/models.rs

```rust
use super::schema::users;

#[derive(Insertable)]
#[table_name = "users"]
pub struct NewUser<'a> {
    pub username: &'a str,
    pub password_hash: &'a str,
    pub created_at: chrono::NaiveDateTime,
}
```

---

# Declare schema and models modules in src/lib.rs

```rust
pub mod schema;
pub mod models;
```

---

# Define custom error enum in src/lib.rs

```rust
#[derive(Debug)]
pub enum AuthenticationError {
    InvalidUsername,
    IncorrectPassword,
    InvalidLogin,
    DatabaseError(diesel::result::Error),
    BcryptError(bcrypt::BcryptError),
}

impl From<bcrypt::BcryptError> for AuthenticationError {
    fn from(e: bcrypt::BcryptError) -> Self {
        AuthenticationError::BcryptError(e)
    }
}
```

---

# Write a function to create user in src/lib.rs

```rust
use chrono::prelude::Utc;
use bcrypt::{hash, DEFAULT_COST};
use chrono::prelude::Utc;
use models::{NewUser, User};


pub fn create_user(conn: &PgConnection, username: &str, password: &str) -> Result<User, AuthenticationError> {
    use schema::users;

    let now = Utc::now().naive_utc();
    let password_hash = hash(password, DEFAULT_COST).unwrap();
    let new_user = NewUser {
        username: username,
        password_hash: &password_hash,
        created_at: now,
    };

    diesel::insert_into(users::table)
        .values(&new_user)
        .get_result(conn)
        .map_err(AuthenticationError::DatabaseError)
}
```

---

# Write Iron handler in src/main.rs

```rust
extern crate iron;
extern crate router;
extern crate users_rs;

use users_rs::*;
use iron::{Iron, IronResult, Request, Response, status};
use router::Router;
use std::io::Read;

fn create_handler(req: &mut Request) -> IronResult<Response> {
    let body = req.get::<bodyparser::Json>();
    match body {
        Ok(Some(body)) => {
            let connection = establish_connection();
            let username = body.get("username").unwrap().as_str().unwrap();
            let password=  body.get("password").unwrap().as_str().unwrap();
            let user = create_user(&connection, &username, &password);
            if let Ok(user) = user {
                let response = format!("{{\"id\":{}}}", user.id);
                Ok(Response::with((status::Created, response)))
            } else {
                Ok(Response::with(status::UnprocessableEntity))
            }
        }
        Ok(None) => Ok(Response::with(status::UnprocessableEntity)),
        Err(err) => Ok(Response::with(status::BadRequest)),
    }
}


fn main() {
    let mut router = Router::new();
    router.post("/users", create_handler, "create_handler");
    Iron::new(router).http("localhost:3000");
}
```

---

# Test it out

```
$ curl -v -d '{"username":"jz","password":"heslo123"}' localhost:3000/users
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 3000 (#0)
> POST /users HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/7.54.0
> Accept: */*
> Content-Length: 40
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 40 out of 40 bytes
< HTTP/1.1 201 Created
< Content-Length: 8
< Content-Type: text/plain
< Date: Tue, 19 Feb 2019 12:03:19 GMT
<
* Connection #0 to host localhost left intact$
{"id":1}

```

---

# Refactor - introduce Login struct in src/lib.rs

```rust
pub struct Login<'a> {
    pub username: &'a str,
    pub password: &'a str,
}

impl<'a> Login<'a> {
    fn is_valid(&self) -> bool {
        !self.username.is_empty() && !self.password.is_empty()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn login_is_valid_fails_with_empty() {
        let login = login {
            username: "",
            password: "",
        };

        assert!(!login.is_valid());
    }

    #[test]
    fn login_is_valid_succeeds() {
        let login = login {
            username: "joe",
            password: "doe",
        };

        assert!(login.is_valid());
    }
}
```

---

# Refactor - continue in src/main.rs

```rust
fn create_handler(req: &mut Request) -> IronResult<Response> {
    let body = req.get::<bodyparser::Struct<Login>>();

    match body {
        Ok(Some(login)) => {
            let connection = establish_connection();
            let user = create_user(&connection, &login);
            if let Ok(user) = user {
                let response = format!("{{\"id\":{}}}", user.id);
                Ok(Response::with((status::Created, response)))
            } else {
                Ok(Response::with(status::UnprocessableEntity))
            }
        }
        Ok(None) => Ok(Response::with(status::UnprocessableEntity)),
        Err(err) => Ok(Response::with(status::BadRequest)),
    }
}
```


---

# JWT token src/auth.rs

```rust
extern crate jsonwebtoken as jwt;

use crate::User;
use chrono::prelude::Utc;
use jwt::errors::ErrorKind;
use jwt::{decode, encode, Algorithm, Header, Validation};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,
    company: String,
    exp: usize,
}

pub fn issue_token(user: &User) -> String {
    let my_claims = Claims {
        sub: user.username.to_owned(),
        company: "Blueberry".to_owned(),
        exp: 10000000000,
    };
    let key = "secret";
    match encode(&Header::default(), &my_claims, "secret".as_ref()) {
        Ok(t) => t,
        Err(_) => panic!(), // in practice you would return the error
    }
}

```

---

# Add auth handler in src/main.rs

```rust
use crate::auth::issue_token;

fn auth_handler(req: &mut Request) -> IronResult<Response> {
    let body = req.get::<bodyparser::Struct<Login>>();
    match body {
        Ok(Some(login)) => {
            let connection = establish_connection();
            let user = auth_user(&connection, &login);
            if let Ok(user) = user {
                let token = issue_token(&user);
                Ok(Response::with((status::Created, token)))
            } else {
                Ok(Response::with(status::UnprocessableEntity))
            }
        }
        Ok(None) => Ok(Response::with(status::UnprocessableEntity)),
        Err(err) => Ok(Response::with(status::BadRequest)),
    }
}
```

---

pub fn validate_token(token: &str) -> Result<bool, jwt::errors::Error> {
    let token = decode::<Claims>(&token, "secret".as_ref(), &Validation::default());
    match token {
        Ok(token) => {
            println!("{:?}", token);
            Ok(true)
        }
        Err(e) => {
            println!("{:?}", e);
            Err(e)
        }
    }
}


---

# Things missing

* Invalid request handling - the code na panics when not supplied JSON object with username and password
* Diesel connection pooling - we create a connection in each handler
* Validations - duplicate username will be handled by DB, but other validations are not present

