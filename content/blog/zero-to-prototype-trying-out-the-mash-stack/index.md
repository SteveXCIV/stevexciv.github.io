+++
title = "Zero to Prototype: Trying Out the MASH Stack"
 author = "Steve Troetti"
 date = 2025-04-27
 updated = 2025-04-27
 description = "My experiences giving the MASH Stack a try."

[taxonomies]
tags = ["rust", "fullstack", "mash", "maud", "axum", "sqlx", "htmx"]
+++

# Zero to Prototype: Trying Out the MASH Stack

> Want to jump right into the code? Check out [this tag][code].

## Introduction

As a backend engineer, I spend most of my days thinking about servers, databases, and APIs.
Most of the web apps I'm looking at are dashboards to see how my data pipelines are performing.
You could say I don't make it to the frontend side of things much, and that's part of why the MASH stack caught my eye.
I first learned of this stack when I read this [blog post][eschwartz_blog] by Evan Schwartz.
He talks about organically arriving at this stack and also discovering that [Shantanu Mishra][smishra_home] had already given it a [name and a homepage][mash_home].

MASH, in short is:

- [Maud][maud]: a Rust-based templating engine
- [Axum][axum]: an HTTP server framework
- [SQLx][sqlx]: a Rust-based async SQL client without the weight of an ORM
- [HTMX][htmx]: a near-zero JS solution for dynamic frontends

Suffice to say I was intrigued.
If multiple other software engineers were successfully making things with this combination of components, there must be something to it.
Also, again, I don't do a lot on the frontend - the prospect of simple, fast, _and_ almost fully Rust-based web apps is something I couldn't resist giving a second look.
I decided on building the classic todo list app; the requirements are simple enough that I figured it shouldn't take long to stand up a prototype.
This is a short writeup of my experiences, thoughts, and some things I want to explore more in the future.

## Project Setup

I started off like any Rust project and fired up `cargo new`.
I decided right from the start that if I was really going to test-drive the MASH stack, I wanted to treat this like a real application.
That means I want a little more polish than standard "demo" code, so I have a couple of extra dependencies; here's what I ended up with:

```toml
[dependencies]
anyhow = "1.0.98"
axum = { version = "0.8.3", features = ["tracing"] }
clap = { version = "4.5.37", features = ["derive", "env"] }
dotenvy = "0.15.7"
maud = { version = "0.27.0", features = ["axum"] }
serde = { version = "1.0.219", features = ["derive"] }
sqlx = { version = "0.8.5", features = ["runtime-tokio", "sqlite"] }
tokio = { version = "1.44.2", features = ["full"] }
tower-http = { version = "0.6.2", features = ["fs", "trace"] }
tracing = "0.1.41"
tracing-subscriber = { version = "0.3.19", features = ["env-filter"] }
```

As for the stack components, we have:

- `axum` and the `tracing` feature for request logging
- `maud` and the `axum` feature, which I'll talk more about in the next section
- `sqlx` with the Tokio runtime and `sqlite` feature for a minimalistic database

For everything else:

- `anyhow` to be a bit hand-wavey about error handling; this is something I want to revisit
- `clap` and `dotenvy` for moving config out to the environment like a real app would
- `serde` to support form-encoded data the frontend will be sending
- `tokio`, `tower-http`, and the `tracing*` crates for general web server "stuff" and communicating to stdout via traces

When all that was out of the way, I stood up the most basic hello world server as a starting point.
The `main` method did some basic setup before calling `routes::create_router()`, and my `routes.rs` file looked like:

```rust
use axum::{Router, routing::get};

async fn home() -> &'static str {
    "Hello from Home!"
}

pub fn create_router() -> Router {
    Router::new().route("/", get(home))
}
```

Just enough of a starting point that I could call `cargo run` and fire off a request via `curl`.
Now it was time to actually use MASH.

## Serving Maud Markup from Axum Handlers

One word that stuck with me from Evan Schwartz's blog post was "synergy".
He does a deep dive on some of the ergonomic benefits of the MASH stack and I recommend giving it a read.
I will however echo his sentiment: the components of this stack fit together almost effortlessly.

Let's take a look at a simple example with Maud's `Markup` type and the Axum integration feature.
At the risk of oversimplifying, Axum connects _routes_ to _handlers_ which are async functions that return some `Response`.
Anything implementing Axum's `IntoResponse` trait can also be returned by a handler.
Maud's `axum` feature provides an `IntoResponse` for `Markup`, which is the output of Maud's `html!` macro.
In short, it means that this is a valid handler:

```rust
use maud::{DOCTYPE, Markup, html};

pub async fn home() -> Markup {
    html! {
        (DOCTYPE)
        head {
            title { "Home" }
        }
        body {
            p { "Hello world!" }
        }
    }
}
```

This makes for a pretty seamless fit between Maud and Axum.
Handlers are able to follow the general pattern of:

- optionally extract some data from the request/context
- optionally transform the data in some interesting way
- inject data into the template
- return rendered markup as a response

It's also worth noting that for extremely simple tools or UIs, this is already a ton of functionality with very little fanfare.
That being said, we still have half the stack's letters to go over, so let's keep going.

## Installing HTMX and Vendoring Dependencies

HTMX is the one part of this project that doesn't have an entry in `Cargo.toml`.
That's because it's a lightweight Javascript library, and incidentally, the only `<script>` tag actually needed.
HTMX is available via unpkg CDN.
That being said, the HTMX docs link to [this blog post][wesleyac_blog] by Wesley Aptekar-Cassels, about not using a CDN in production.
Given my general lack of expertise with the Javascript ecosystem, I would need to do more research to get into the details of Wesley's argument.
That being said, there _are_ some compelling points there, and I thought that this would be a good chance to try something I hadn't seen other MASH projects do.

So, I decided to forego the CDN and vendor the HTMX source into my application.
This is where the `tower_http` direct dependency in `Cargo.toml` comes in.
It provides `ServeDir` ([docs][servedir]) to serve requests for files in a directory, which Axum can use directly as middleware.
`ServeDir` also comes with some nice features such as handling invalid requests gracefully and returning `404 Not Found` for missing files.

Setting it up was easy:

```rust
use tower_http::services::ServeDir;

pub fn create_router() -> Router {
    Router::new()
        .nest_service("/public", ServeDir::new("public"))
        // ... rest of router ...
}
```

Then, I went ahead and made a `public/js` directory in my project root, and downloaded `htmx.min.js` as well as its LICENSE file.
One thing I found cool about this pattern is it greatly simplifies compliance with licenses which require distributing them _with_ the code.
Finally, all that was left to do was link `htmx.min.js` into my Maud markup:

```rust
pub async fn home() -> Markup {
    html! {
        (DOCTYPE)
        head {
            title { "Home" }
            script src="/public/js/htmx_2.0.4/htmx.min.js" type="text/javascript" {}
        }
        // ... rest of document ...
    }
}
```

> **NOTE:** I use `script ... {}` here (closing brackets at the end) self-closing script tags are generally unsupported by browsers; there's a [rabbit hole][script_tags] to go down, if you so desire.

With that done, the application frontend became ✨ _dynamic_ ✨

The only problem is that there was no data to display at this point.
It was time to add persistent state and APIs for modifying the data.

## SQLx: A not-ORM for SQL in Rust

SQLx says right in the README that it is **not** an ORM.
Having only tried Diesel in the past, this was a new and exciting experience for me.
SQLx's approach to queries is a no-DSL, "just write SQL" approach - the minimalism is a big win for me.
It also offers really great out-of-the-box features such as migrations and compile-time query verification _without_ mandating their use.
I really appreciated this because it let me prototype without the burden of having to form my code around a specific toolchain or DSL.

For example, here's the `Todo` struct and the function that fetches all todos from the database:

```rust
#[derive(sqlx::FromRow)]
pub struct Todo {
    pub id: i64,
    pub description: String,
    pub completed_at: Option<i64>,
}

pub async fn get_all_todos(pool: &SqlitePool) -> anyhow::Result<Vec<Todo>> {
    let todos = query_as::<_, Todo>("SELECT * FROM todos ORDER BY id")
        .fetch_all(pool)
        .await?;
    Ok(todos)
}
```

That's it.
One derive, and a plain-old SQL query.
Minimalism at its finest.

### Embedding migrations and creating a pool

I decided to forego the compile-time query verification for right now, but I did take advantage of SQLx's migration support.
This does mean using the SQLx CLI tool, so there is one extra `cargo install sqlx-cli` step, but I think it was worth it.

I created a reversible migration with the CLI; the "up" portion which creates the `todos` table looks like this:

```sql
CREATE TABLE IF NOT EXISTS todos (
  -- this is an alias for the row's unique ID (see: https://www.sqlite.org/autoinc.html)
  id INTEGER PRIMARY KEY NOT NULL,
  description TEXT NOT NULL,
  completed_at BIGINT
);
```

SQLx gives you options when it comes to actually applying migrations.
You are able to use the CLI to prepare the database outside of your application code, but you can just as easily embed them in the application code.
This is really convenient; with a larger application you might want to run migrations one time as a pre-deploy step,
but in a smaller app or one where you're using in-memory SQLite, the same migration files just work.

Embedding migrations happens with the `sqlx::migrate!()` macro; here's the code I wrote for handling the database access:

```rust
// Embeds all ./migrations into the application binary
static MIGRATOR: Migrator = sqlx::migrate!();

pub async fn create_pool() -> Result<SqlitePool, Error> {
    let pool = SqlitePool::connect_with(
        SqliteConnectOptions::from_str("sqlite://db/app.db")?
            .create_if_missing(true),
    )
    .await?;
    MIGRATOR.run(&pool).await?;
    Ok(pool)
}
```

Forgive the hardcoded path to the SQLite database file, please.
Aside from that, this code is simple and easy to work with.
I just call `create_pool().await?` and I get back a connection pool ready-to-use, always on the latest table schema.

### Transactions

SQLx also makes transactions pretty painless.
The only thing to remember is to manually commit the transaction, otherwise when it `drop`s, it will roll back.
Here's the function that toggles the completion state of a `Todo`:

```rust
pub async fn toggle_todo(pool: &SqlitePool, id: i64) -> anyhow::Result<Todo> {
    // open a new transaction
    let mut tx = pool.begin().await?;

    // fetch existing todo
    let mut todo: Todo = query_as("SELECT * FROM todos WHERE id = (?1)")
        .bind(id)
        .fetch_one(&mut *tx)
        .await?;

    if todo.is_completed() {
        // uncomplete the todo
        query("UPDATE todos SET completed_at = NULL WHERE id = (?1)")
            .bind(id)
            .execute(&mut *tx)
            .await?;
        todo.completed_at = None;
    } else {
        let completed_at =
            SystemTime::now().duration_since(UNIX_EPOCH)?.as_millis() as i64;

        // update the database row
        query("UPDATE todos SET completed_at = (?1) WHERE id = (?2)")
            .bind(completed_at)
            .bind(id)
            .execute(&mut *tx)
            .await?;
        todo.completed_at = Some(completed_at);
    }

    // close the transaction (important!)
    tx.commit().await?;

    Ok(todo)
}
```

Since a read-then-write is necessary here, a transaction is required to ensure that the database is updated atomically.
SQLx makes it very easy to do this.

## Putting it All Together

Let's take a look at the `Todo` completion toggle end-to-end.
It's probably the most complex thing this app does, and it touches every part of the stack, so I think it's a good example of the stack in action.

Let's zoom in on how individual todos are rendered for a moment:

```rust
fn render_todo(todo: &Todo) -> Markup {
    let id = get_todo_id(todo);
    html! {
        li #(&id) {
            label .checkbox {
                input .big-checkbox .mr-4
                    hx-put={"/api/v1/todos/" (todo.id) "/toggle"}
                    hx-target={"#" (&id)}
                    hx-swap="outerHTML"
                    type="checkbox"
                    checked[todo.is_completed()];
                @if todo.is_completed() {
                    s { (todo.description) }
                } @else {
                    (todo.description)
                }
            }
        }
    }
}
```

There's a couple of things going on here, so let's break it down:

- This function renders an `<li>` with a unique ID derived from the `Todo`
- Inside the `<li>` are a `<label>` and a checkbox `<input>`
- The `<label>` wraps the checkbox, making them behave as one unit in terms of clickability
- The `<label>` also applies a strikethrough for completed `Todo`s only
- The `<input>` triggers a `PUT /api/v1/todos/{todo.id}/toggle` request when it is toggled (i.e. clicked on)
- The results of this request swap the `outerHTML` (the whole element) of the parent `<li>` with the `PUT` request's response

With that all in mind, here's the actual handler code for toggling a `Todo`:

```rust
pub async fn toggle_todo(
    State(AppState { pool }): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Markup> {
    match todos::toggle_todo(&pool, id).await {
        Ok(t) => Ok(render_todo(&t)),
        Err(e) => Err(internal_server_error(e)),
    }
}
```

`todos::toggle_todo` is the function we saw earlier which changes the todo state within a transaction.

Finally, the handler is wired up to the router like so:

```rust
pub fn create_router(state: AppState) -> Router {
    Router::new()
        .route("/api/v1/todos/{id}/toggle", put(views::toggle_todo))
    // ... rest of router omitted ...
}
```

This makes for a remarkably simple application structure that still provides a good level of interactivity.
Simple, fast, and easy.
There's a lot to like here.

## Closing Thoughts

The MASH stack proved to be an excellent fit for projects with smaller scopes due to its convenience and seamless component integration.
This experience has left me feeling generally positive about the stack's potential, owing to its simplicity and flexibility.
I recommend giving this stack a try if you're in need of a simple webapp and want to write it in Rust.
I'm certainly going to be doing more investigation on my own to explore the potential of this stack.

However, that being said, there are a couple of open questions that I'd like to explore more in the more immediate future.
One big one that's been on my mind throughout the entire project is testability.
Part of this is admittedly my fault; the code I built could use a bit of a clean-up pass and better separation of concerns.
Right now, `views` does a ton of heavy lifting, it renders data as markup and calls into the DAO functions of `todos`.
A better design might be to add another layer where the data modifications take place, and then `views` just handles rendering.
End-to-end testing would still be a concern though, given that the backend is sending out markup with functional meaning for the HTMX frontend.
I've been doing some reading on testing HTMX applications, and I have some ideas I intend to try out.

Stay tuned for more updates as I delve deeper into these areas!

[code]: https://github.com/SteveXCIV/mash_todo/tree/prototype
[eschwartz_blog]: https://emschwartz.me/building-a-fast-website-with-the-mash-stack-in-rust/
[smishra_home]: https://8hantanu.net/
[mash_home]: https://yree.io/mash/
[maud]: https://github.com/lambda-fairy/maud
[axum]: https://github.com/tokio-rs/axum
[sqlx]: https://github.com/launchbadge/sqlx
[htmx]: https://github.com/bigskysoftware/htmx
[wesleyac_blog]: https://blog.wesleyac.com/posts/why-not-javascript-cdn
[servedir]: https://docs.rs/tower-http/latest/tower_http/services/struct.ServeDir.html
[script_tags]: https://stackoverflow.com/q/69913
