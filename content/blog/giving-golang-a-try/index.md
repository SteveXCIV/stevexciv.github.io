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

## Creating the task structure

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

- Looking at project plan doc, I need to support:
  - Add (aka Create op)
  - List (aka Read op)
  - Complete (aka Update op)
  - Delete
  - Search (another kind of Read op)

---
