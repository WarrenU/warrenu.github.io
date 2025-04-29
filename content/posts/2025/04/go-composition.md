---
title: "Go Favors Composition Over Inheritance"
date: 2025-04-29
tags: ["golang", "composition", "inheritance", "interfaces"]
description: "A quick example of how Go replaces inheritance with composition and interfaces."
---

Go doesn’t use traditional inheritance like many OOP languages AND that’s by design.

Instead, Go encourages **composition** and **interfaces**, which often lead to simpler, more modular code. Here's a quick example:

```go
package main

import "fmt"

// Define a behavior
type Notifier interface {
    Notify()
}

// Provide the behavior via a concrete type
type EmailNotifier struct {
    Email string
}

func (e EmailNotifier) Notify() {
    fmt.Println("Sending email to:", e.Email)
}

// Compose the behavior into another struct
type User struct {
    Name string
    EmailNotifier // embedded
}

func main() {
    user := User{
        Name:          "Warren",
        EmailNotifier: EmailNotifier{Email: "warren@example.com"},
    }

    fmt.Println("User:", user.Name)
    user.Notify() // User "inherits" Notify through embedding
}
```

Instead of subclassing, we **embed** behavior directly and rely on interfaces to model capabilities. This pattern helps avoid deep inheritance trees and keeps your codebase flexible.

---

Do you prefer this over traditional inheritance?
