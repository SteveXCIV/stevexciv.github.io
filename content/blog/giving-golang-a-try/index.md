+++
title = "Giving Go a Try"
author = "Steve Troetti"
date = 2025-09-03
updated = 2025-09-03
description = "A constructive, honest review of my first real Go project: a CLI todo app."
[taxonomies]
tags = ["go", "golang", "learning", "cli"]
+++

# Giving Go a Try

In this post, I'll document my experience building an app in Go, which is a new-to-me language.
Keep reading if you want to know more about how I built this, and my first impressions of Go.

If you just want to see the code, hop on over to [the repository](https://github.com/SteveXCIV/golang-todo-cli).

## Background

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

### What to Build

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
