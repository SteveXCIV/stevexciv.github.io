+++
title = "Giving Go a Try"
author = "Steve Troetti"
date = 2025-09-03
updated = 2025-09-03
description = "A constructive, honest review of my first real Go project: a CLI todo app."
[taxonomies]
tags = ["go", "golang", "learning", "cli"]
+++

In this post, I'll document my experience building an app in Go, which is a new-to-me language.
Keep reading if you want to know more about how I built this, and my first impressions of Go.

If you just want to see the code, hop on over to [the repository](https://github.com/SteveXCIV/golang-todo-cli).

# Background

If you know me, or you've read my writing before, you'll know that I'm a Java developer by day, and a Rustacean by night.
I've also done some Python here and there as the situation called for, but that's been about all I've worked with in the past 5 years or so.
While I don't think Java will ever go away, and I still love Rust, I was itching to try something _new_.

For a moment, let's rewind back to late 2019, I was working with a small group on a project that I was pretty excited about.
It was a Python app that, at the time, we felt had a lot of potential.
Unfortunately, however, that project ended up fizzling out, but I still remember one thing from back then.
The software we had been building correlated near real-time data from multiple feeds, and we were quickly outgrowing Python.
We'd been talking for some time about a Go port, because one of our contributors was familiar with Go.

I never got a chance to explore Go back then, and I don't really like leaving things unfinished.
Now that I was looking for something new, I decided to just <em title="(sorry)">Go</em> for it.

## What to Build

I remember I had a professor back in undergrad that loved to use the "Lazy developers are good developers" line.
That professor would be proud because I got a bit lazy when picking a project.

I fired up a chat session with an LLM and asked for a project plan to learn about Go.
Specifically, I requested approximately 8 hours of work, basic but flexible requirements, and some focus areas for my learnings.
My plan was to treat this as I would any LLM output: a starting point that I would then workshop with my own human intuition.

The plan that I received was remarkably mundane: a simple TODO app with a CLI.
I could have come up with this on my own, sure, but I was still thankful to have some notes to go off of.
So there I had my plan, I knew what I was going to build.

I was going to make a simple TODO CLI that stored tasks as a JSON document and had basic CRUD operations - not bad.

The rest of this post details my experience building the application.

# Getting Started

I began my Go journey at the docs.
There happens to be a conveniently-placed "Get Started" button which links to [learning resources](https://go.dev/learn/).

Two that I found particularly helpful were:

- [A Tour of Go](https://go.dev/tour/welcome/1)
- [Go by Example](https://gobyexample.com/)

I've always been an _experiential_ learner, so I was **thrilled** to see that there's two example-based resources to learn from.
After a brisk pass through the tour, and a couple of cherry-picked examples, I decided it was time to write some code.

# Implementation Experience

**NOTE:** I've kept the [plan docuement](https://github.com/SteveXCIV/golang-todo-cli/PROJECT_PLAN.md) that I AI-generated in the repository as a _reference_.
I've deviated from it in my actual implementation, but it still served as a decent scaffold.

## Creating the Task struct

The first step of the plan is to define the `Task` struct for storing our TODOs.
This sounds perfectly reasonable because this is going to be the primary data type we pass around.
I settled on the following fields:

- `Id`: a unique integer ID for tasks
- `Title`: the name of the task
- `Priority`: an enum denoting the low/medium/high priority
- `DueDate`: when this task is due
- `Category`: an optional user-provided category
- `Status`: an enum denoting pending/complete state

This was my first encounter with Go-specific behaviors.

### Enums

Go has no built-in "enum" type.
Instead, enums are just constants with an opaque `type` alias.
They also typically use a built-in called `iota` which is a language feature more-or-less purpose-built for enums.
[Go by Example](https://gobyexample.com/enums) does a good job of explaining the enum pattern.

#### Iota

If you click through links from the "Go by Example" link above, you'll undoubtedly end up at the [Go blog post](https://blog.learngoprogramming.com/golang-const-type-enums-iota-bc4befd096d3) on `iota`.

The `iota` build-in is essentially an incrementing counter that resets inside a new `const` declaration block.
The idea is that it's easy to assign incrementing values to enums:

```go
type Priority int

const (
  Low     Priority = iota // iota -> 0
  Medium                  // iota -> 1
  High                    // iota -> 2
)
```

It works well for this purpose, but it also has **tons** of other uses, and my first impression is that it may be too many.

##### Iota Iteratively Evaluates Expressions

The initial value can be an _expression_ using `iota`.
This expression gets re-evaluated on each line, with the incrmented `iota` value:

```go
type DoubleX int

const (
	Zero DoubleX = 2 * iota  // iota -> 2 * 0 = 0
	One                      // iota -> 2 * 1 = 2
	Two                      // iota -> 2 * 2 = 4
)
```

This is convenient, but I could easily see it as a pain point for working with large enums.

### Field Naming

My first inclination with the `Task` struct was to use **lower-case** names for the fields.
I understood that Go treats these as **package-private** and though it would be fine.
I was operating under the impression that I'd start with private fields and make them public (i.e. use a `TitleCase` name) as needed.
One thing I didn't immediately realize, however, is that Go's built-in `json` module would ignore them.

In retrospect, this is a reasonable design decision.
You probably want your data objects to make all their meaningful fields public anyway; it also makes sense with how the Go implementation works (more on that later).

### JSON Field Names

One potential issue with this approach is that it couples the JSON field name with Go's convention-based access modifier system.
Luckily, there's a way to decouple these.
The `json` module also uses something called ["struct tags"](https://go.dev/wiki/Well-known-struct-tags).
These are string literals that can appear next to `struct` fields.
Other modules can access these values via reflection.
The `json` module has [a DSL](https://pkg.go.dev/encoding/json#Marshal) for controlling the conversion to JSON.

### Customizing Output Formats

---

## 3. Implementation Experience

Overview

- Short narrative of how implementation proceeded (from single file → packages → tests → CLI).

  3.1 Data modeling & JSON

- Design of Task struct and exported fields for JSON.
- Problems encountered: verbose time format, custom marshal/unmarshal, pointer vs non-pointer receiver behavior.
- How DueDate and Priority/Status types were implemented and why.

Include a minimal code example (replace with your final excerpt):

```go
type Task struct {
    ID       int       `json:"id"`
    Title    string    `json:"title"`
    Priority Priority  `json:"priority"`
    DueDate  DueDate   `json:"dueDate"`
    Category string    `json:"category"`
}
```

3.2 Testing

- Why I chose TDD for manager logic.
- Table-driven tests were a win — examples and what they validated.
- Test runner awkwardness (module path vs ./...); commands used.

Example test command:

- go test ./...

  3.3 Project structure & packages

- Single-file start → tasks package split.
- Module path case-sensitivity and lessons learned.
- Final layout (show small tree snippet).

  3.4 Manager API & optional parameters

- Challenge: no optional params in Go.
- Solution: param structs and pointer fields to signal "unset".
- Trade-offs (pointer ergonomics, taking pointers to constants).

  3.5 CLI design & wiring

- Command interface and Execute pattern.
- Parsing approach (stdlib flag vs future Cobra).
- Testing commands and deep-equality issues (reflect.DeepEqual use).

  3.6 Small pain points and pleasant surprises

- Pain: time formatting, module path quirks, no custom == for structs, pointer-to-const inconvenience.
- Pleasant: fast compile, built-in testing, idiomatic table-driven tests, simple stdlib for many tasks.

---

## 4. Conclusion

What I built

- Brief recap of the finished CLI todo app and core features.

Overall assessment

- Honest pros/cons of Go from this experience.
  - Pros: simplicity, tooling, tests, performance.
  - Cons: quirks around time/layout, ergonomics for optional values, case-sensitive module paths.

Would I use Go again?

- Scenarios where Go makes sense (small CLIs, services, simple concurrent programs).
- Where I'd prefer other languages (complex data modeling where richer type ergonomics help).

---

## 5. Key takeaways (TL;DR)

- 3–6 bullets summarizing practical lessons and recommendations for new Go learners.

---

## 6. Next steps / Future work

- Try Cobra for CLI.
- Add concurrency or file-locking for persistence.
- Explore Go modules and versioning in larger projects.
- Port to a small web UI or REST API.

---

## 7. Appendix

A. Useful links & references

- Tour of Go — https://go.dev/tour
- Go by Example — https://gobyexample.com
- time.Layout docs — https://pkg.go.dev/time#Layout
- encoding/json Marshaler — https://pkg.go.dev/encoding/json#Marshaler

B. Notable commits

- List the commit hashes and short notes (pull from your draft).

C. Commands

- go test ./...
- go run ./main.go add ...
- any build/test helper commands you used (just, Makefile, etc.)

---

Notes for polishing

- Add 2–3 small, well-commented code excerpts to illustrate core ideas (Task struct, Marshal/Unmarshal, a representative test).
- Keep tone constructive and honest — readers value practical trade-offs.
- Link to the repository and include a short "How to run" section if you publish the
