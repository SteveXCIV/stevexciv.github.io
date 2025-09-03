+++
title = "Giving Golang a Try"
author = "Steve Troetti"
date = 2025-08-27
updated = 2025-08-27
description = "My expericnes learning Golang with a small project."

[taxonomies]
tags = ["golang"]
+++

---

# NOTES:

## Before implementing

- read through/interacted w/ tour of go: https://go.dev/tour/welcome/1
- used go by example for concrete examples: https://gobyexample.com/
-

## Creating the task structure and manager struct

- `iota` is convenient but I did find a lot of its "magic" somewhat intimidating for a beginner
- JSON only includes fields that are **public** so they need to be `TitleCase`
- Can use `json:"name"` aliases

Sample:

```go
type task struct {
	Id       int       `json:"id"`
	Title    string    `json:"title"`
	Priority Priority  `json:"priority"`
	DueDate  time.Time `json:"dueDate"`
	Category string    `json:"category"`
}

func main() {
	var testTask = task{
		Id:       1,
		Title:    "Schedule dentist appointment",
		Priority: High,
		DueDate:  time.Now().AddDate(0, 0, 7),
		Category: "Health",
	}
	jsonData, _ := json.Marshal(testTask)
	fmt.Println(string(jsonData))
}
```

```json
{
  "id": 1,
  "title": "Schedule dentist appointment",
  "priority": 2,
  "dueDate": "2025-09-03T22:14:09.447822-04:00",
  "category": "Health"
}
```

- commit `86bcf02c482732ebdc8d5746f2c0b9a7b26d000e`
- didn't like the verbose date rendering, needed to figure out how JSON serde actually works

### Tangent 1: JSON

- learned about JSON marshalling with custom `Marshaler` implementation
  - this is a pretty convenient way to handle custom serialization behaviors
  - https://pkg.go.dev/encoding/json#Marshal
  - https://pkg.go.dev/encoding/json/v2#Marshaler
- Created a type def for `DueDate` - convenient wrapper type for `time.Time`
- Time formatting is "example based"
  - I kinda don't like this - give me back format strings please
  - Why is `2006-01-02` some kind of sentinel value?
    - Found this answer: https://stackoverflow.com/a/52966197
    - Cool Easter egg. I don't want Easter eggs in a programming language.
  - https://pkg.go.dev/time#Layout
  - https://gobyexample.com/time-formatting-parsing

Implemented `DueDate` in `e0affaf1f888461ae04484b7eea619060d0ad495`.

- Moved on to handling `Priority`
- Discovered that `MarshalJSON` seems to not work properly if implemeneted on pointers to the type
- Got `LOW, MEDIUM, HIGH` working

Finished `Status` and had all desired custom marshal/unmarshal working in commit `1f9deddde089cbbda418b05d6b91d401e10ddba9`.

**Finished with JSON, back to task structure.**

- Made `Task` exported (aka "public") in anticipation of a future split to separate package
- Decided it was time to just go with the package split
- Used Ask mode here to understand how to structure the project into multiple files
- Renamed `todo.go` to `main.go`

### Tangent 2: Hacky one file implementation to multiple files

`Task`, `Priority`, `Status`, and `DueDate` were all living in my main file and I didn't like that.

Decided on a simple program structure using a new `tasks` package:

```
├── go.mod
├── LICENSE
├── main.go
├── PROJECT_PLAN.md
└── tasks
    ├── task_test.go
    └── task.go
```

Also decided to add tests because I've implemented a decent chunk of functionality.

- Learned that golang package paths are **case-sensitive** (don't like this):

```go
"github.com/stevexciv/golang-todo-cli/tasks"
```

is distinct from:

```go
"github.com/SteveXCIV/golang-todo-cli/tasks"
```

Another thing I **do not like**; the test harness `go test` requires the full module path to run tests:

```
go test -v github.com/stevexciv/golang-todo-cli/tasks
```

This is clunky and inconvenient, but ultimately solvable using scripts or a dedicated command runner like `just`.
Still, I don't like having to wrap the official CLI tool for such a trivial thing.

**ADDENDUM:** I found that you can actually run all test with `go test ./...`. In my humble opinion, this **should probably be the default**. (source: https://stackoverflow.com/a/16353449)

Something **I did like**, testing was fairly straightforward once I got the runner working.

I also managed to find a bug! My initial implementation for `UnmarshalJSON` was not properly handling the fact that the `b []byte` value is JSON, e.g. the string was wrapped in quotes.
This meant that my original code was trying to unmarshal the literal `"pending"` (including the quote chars) as `Pending` instead of unmarshalling `pending` (no quote chars).

Another thing **I really like** is out-of-the-box support for table-driven tests.
IMO table-driven tests are tragically underutilized.

I got to write a really cool test for JSON marshal/unmarshal and it was **easy and fast**, and as a bonus I got to learn about **string literals** (link: https://stackoverflow.com/a/46917369)

```go
func TestStatusJSONRoundTrip(t *testing.T) {
	var tests = []struct {
		status       Status
		expectedJSON string
	}{
		{Pending, `"pending"`},
		{Completed, `"completed"`},
	}

	for _, test := range tests {
		marshaled, err := json.Marshal(test.status)
		if err != nil {
			t.Errorf("failed to marshal status %v: %v", test.status, err)
		}

		if string(marshaled) != test.expectedJSON {
			t.Errorf("status %v: expected %s, got %s", test.status, test.expectedJSON, string(marshaled))
		}

		var unmarshaled Status
		err = json.Unmarshal(marshaled, &unmarshaled)

		if err != nil {
			t.Errorf("failed to unmarshal status %v: %v", test.status, err)
		}

		if test.status != unmarshaled {
			t.Errorf("status %v != unmarshaled %v", test.status, unmarshaled)
		}
	}
}
```

**Next up, creating the manager struct.**

**NOTE:** I deviated pretty far from the structure of the plan here while still sticking to the requirements. I opted to fully test the manager API in one shot rather than implementing it piecemeal. Even though I didn't follow the plan precisely, I still feel like this experience gave me a pretty good handle on developing a simple project in Go.

- Looking at project plan doc, I need to support:
  - Add (aka Create op)
  - List (aka Read op)
  - Search (another kind of Read op)
  - Complete (aka Update op)
  - Delete

Since this was business logic, I decided to switch it up a bit and go TDD.

I also discovered that Golang doesn't support optional params.
This was something I was used to from Rust, and to an extent Java (but Java makes it a bit easier to fake it with overloads).

This helpful SO answer suggested using a param `struct`: https://stackoverflow.com/a/13603885

I liked this a lot because it mirrors some of the design patterns I'm used to in Rust.

I stubbed out a basic `Manager` struct and a rough outline of what I thought I'd need for the public API in commit: `df9e5831d74fb7e8c210b6bd76abf59630bd5699`

**Interesting thing I noted:** there aren't a lot of helper functions like `IsEmpty` for thing like strings or slices, instead you just compare the `len` to zero or compare to the zero value. I can appreciate that this means there's usually only one right way to do something in Go.

**Issue I ran into:** since enums are zero valued integers by default, expressing filters on them is a bit tricky since the default filter struct (in my case `ListTasksRequest`) would filter on the enum zero values. I did some research (Ask mode) plus this article: https://medium.com/@persona.piotr/golang-optional-type-d52f5498e300 - I ended up deciding to use pointers which default to `nil`. I made all fields except `OverdueOnly` pointers, partially to provide the `nil` defualt, partially to avoid copying (in the case of the `Category string`), and left the `OverdueOnly bool` as a regular value because it's very cheap to copy.

**Another issue related to this:** taking pointers to constants directly is not allowed.
This kind of sucks. The compiler could probably easily sub this out for a random/unused variable name, but tons of projects actually do this:

- Kubernetes: https://github.com/kubernetes/utils/tree/master/ptr
- Docker
- AWS: https://github.com/aws/aws-sdk-go-v2/blob/main/aws/to_ptr.go

**More Go quirks:** there's this thing called "re-slicing" in Go that is/was an idiomatic approach for removing elements from a slice.
This seemed (a) weird because it's just verbose enough to need to read it carefully every time you encounter it, and (b) controversial: https://stackoverflow.com/q/46917331
Luckily, Go now supports a `slices.Delete` function: https://go.dev/play/p/J--7kzTNvjE

## Creating the CLI interface

After some thought, I decided on roughly this design for my first attempt at making the CLI interface:

- a `Command` interface with a common `Execute(m *tasks.Manager)` function
- a `Parse` function that parses `Command`s from a slice of args
- a `cli` package to store it all

At this point, I needed to do some reading on how CLI flags, args, and parsing in general works.
Luckily, Go by Example had my back: https://gobyexample.com/command-line-arguments

I quickly discovered, however, that the built in `flag` package doesn't allow you to pass in your own string for supplying the arguments.
This is kind of a hindrance for testability purposes, but definitely not an impossible problem to solve.
In fact, Go by Example has a recipe for this exact case: https://gobyexample.com/command-line-subcommands

In my exploration I did also find this third party library called Cobra: https://pkg.go.dev/github.com/spf13/cobra#section-readme
It looks like something I definitely want to explore in the future, but for right now, I decided to stick with just the standard library.

**Another thing I found that I wasn't thrilled about:** deep equality.

Go doesn't have any built-in way to customize `==` and `!=`.
This seems to be fairly commonly discussed:

- https://stackoverflow.com/q/15311969
- https://old.reddit.com/r/golang/comments/1ff40jy/recommended_way_to_implement_custom_struct/

Since I'm just trying to get my unit tests working, I went ahead and used the built-in reflection technique:

```go
			if !reflect.DeepEqual(&tt.addCmd, addCmd) {
				t.Fatalf("unexpected command: wanted=%v, got=%v", tt.addCmd, addCmd)
			}
```

### Tangent 1: Rethinking the design

As I was working through the CLI code, I thought about my current design.
I originally had liked the idea of a request/response style API for the task `Manager`, but I started to see it as over-engineered when looking at the CLI `Command`/`Execute` pattern.
I decided to rework the `Manager` to get rid of this extra layer of indirection.

Starting with the `Add` command (just because that's what I had implemented parsing for first), I got rid of `AddTaskRequest` and instead just made the manager accept arguments directly in `AddTask`.

Another thing I changed was accepting a `func` for computing `DueDate` from `now time.Time`, so the caller doesn't need to know the absolute time upfront.
This makes relative task scheduling much simpler.

While I was at it, I also just made the internal constructor function for `Manager` a package-level (i.e. not exported) function, since we only use it for testing.

Also, I went ahead and got rid of **all** `*Request` structs, now the `Command` implementations would just operate on the `Manager` directly.

Finally, to make testing the `Command` implementations easier, I went ahead and made `Manager` an interface.

### Tangent 2: Mock manager

I also went ahead and made a quick little `mockManager` struct to implement the `Manager` interface.
Unfortunately this ended up in a bit of a less-than ergonomic pattern for accessing the manager because I had to make a pointer to a pointer.
This happened because the `Execute` function of `Command` takes a `*Manager` as a parameter, and `Manager` is only implemented for `*mockManager`.
I decided to just make a TODO comment for this and revisit it later, since it really only impacts tests.

And as soon as I started implementing the tests, I realized that the double pointer approach was going to cause me to fight the type system, so I got rid of it.

**And finally, with a simple wiring up of commands in main, I had all core functionality done in:** `e01b49a8e15886b41013e0de7dd077e3c5f5f0b2`.

My very last step was to create a rendering helper function to print out `Task` tables, I used the built-in `text/tabwriter` for this.
I wasn't completely satisfied with the fact that there didn't seem to be an easy way to make a horizontal rule without looping over all the tasks, but I'll live.
Finally, with that last change, the TODO CLI was done.

---
