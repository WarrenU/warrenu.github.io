---
title: "Goroutines and Channels Patterns in Go"
date: 2025-09-05
tags: ["golang", "concurrency", "goroutines", "channels", "patterns"]
description: "Exploring concurrency patterns with goroutines, channels, and proper cleanup in Go."
---

Go's concurrency model is one of its most powerful features. While goroutines and channels seem simple at first, mastering their patterns is crucial for building robust concurrent applications.

Let's explore some patterns by building up a real-world example step by step.

## The Challenge: Producer-Consumer with Graceful Shutdown

Imagine you're building a system where multiple workers produce data and one consumer processes it. You need to handle:
- Multiple producers running concurrently
- Graceful shutdown when interrupted
- Timeout handling to prevent deadlocks
- Proper cleanup of resources

Let's start with the basics and build up to the complete solution.

## Pattern 1: Basic Producer-Consumer

First, let's see a simple producer-consumer pattern:

```go
func main() {
    ch := make(chan string)
    
    // Simple producer
    go func() {
        for i := 0; i < 5; i++ {
            ch <- fmt.Sprintf("message-%d", i)
        }
        close(ch)
    }()
    
    // Simple consumer
    for msg := range ch {
        fmt.Println("Received:", msg)
    }
}
```

This works, but what happens if we need multiple producers? Or if we want to stop gracefully?

## Pattern 2: Multiple Producers with WaitGroup

Let's add multiple producers and use `sync.WaitGroup` to coordinate them:

```go
func main() {
    ch := make(chan string)
    var wg sync.WaitGroup
    
    // Start multiple producers
    for i := 1; i < 4; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 3; j++ {
                ch <- fmt.Sprintf("Producer-%d: message-%d", id, j)
            }
        }(i)
    }
    
    // Close channel when all producers are done
    go func() {
        wg.Wait()
        close(ch)
    }()
    
    // Consumer
    for msg := range ch {
        fmt.Println("Received:", msg)
    }
}
```

Better! But we still can't stop the system gracefully or handle timeouts.

## Pattern 3: Adding Graceful Shutdown

Now let's add an exit channel for graceful shutdown:

```go
func main() {
    ch := make(chan string)
    exit := make(chan struct{})
    var wg sync.WaitGroup
    
    // Producers with exit signal
    for i := 1; i < 4; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for {
                select {
                case <-exit:
                    return // Exit gracefully
                default:
                    ch <- fmt.Sprintf("Producer-%d: working", id)
                    time.Sleep(200 * time.Millisecond)
                }
            }
        }(i)
    }
    
    // Consumer with exit signal
    wg.Add(1)
    go func() {
        defer wg.Done()
        for {
            select {
            case <-exit:
                return
            case msg := <-ch:
                fmt.Println("Received:", msg)
            }
        }
    }()
    
    // Simulate shutdown after 2 seconds
    time.AfterFunc(2*time.Second, func() {
        close(exit)
    })
    
    wg.Wait()
}
```

Great! Now we can stop the system, but what about timeouts and signal handling?

## The Complete Solution

Here's the full implementation with all the advanced patterns:

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

func main() {
	fmt.Println("main routine started")
	ch := make(chan string)
	exit := make(chan struct{})
	var wg sync.WaitGroup

	// Multiple producers with timeout handling
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func(id int) {
			fmt.Printf("Producer %d started\n", id)
			defer func() {
				fmt.Printf("Producer %d exiting\n", id)
				wg.Done()
			}()
			count := 1
			for {
				select {
				case <-exit:
					return
				case <-time.After(5 * time.Second):
					fmt.Printf("Producer %d: timeout occurred\n", id)
					return
				default:
					// Channel send with timeout error handling
					select {
					case ch <- fmt.Sprintf("Producer-%d: message-%d", id, count):
						count++
					case <-exit:
						return
					case <-time.After(1 * time.Second):
						fmt.Printf("Producer %d: channel send timeout\n", id)
						continue
					}
				}
				time.Sleep(200 * time.Millisecond)
			}
		}(i)
	}

	// Consumer with timeout and graceful shutdown
	wg.Add(1)
	go func() {
		fmt.Println("Consumer started")
		defer func() {
			fmt.Println("Consumer exiting")
			wg.Done()
		}()
		count := 0
		for {
			select {
			case <-exit:
				return
			case <-time.After(3 * time.Second):
				fmt.Println("Consumer: timeout waiting for messages")
				continue
			case msg, ok := <-ch:
				if !ok {
					fmt.Println("Consumer: channel closed")
					return
				}
				fmt.Printf("Consumer read: %s\n", msg)
				count++
				if count >= 10 {
					close(exit)
					return
				}
			}
		}
	}()

	// Handle system signals (Ctrl+C, etc.)
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
	go func() {
		<-sigChan
		fmt.Println("interrupt signal received")
		close(exit)
	}()

	wg.Wait()
	fmt.Print("main routine exiting\n")
}
```

## Key Patterns Explained

Now let's break down the advanced patterns used in the complete solution:

### 1. **Graceful Shutdown with Exit Channel**

```go
exit := make(chan struct{})
// Later: close(exit) broadcasts to all listeners
```

The `exit` channel acts as a broadcast mechanism. When closed, all goroutines listening to it will receive a zero value and can clean up gracefully.

### 2. **Nested Select for Timeout Handling**

```go
select {
case <-exit:
    return
case <-time.After(5 * time.Second):
    return // Timeout
default:
    select {
    case ch <- data:
        // Success
    case <-time.After(1 * time.Second):
        // Send timeout
    }
}
```

The outer `select` handles the main loop, while the inner `select` handles channel operations with their own timeouts.

### 3. **WaitGroup for Coordinated Cleanup**

```go
var wg sync.WaitGroup
wg.Add(1) // Before starting goroutine
defer wg.Done() // Inside goroutine
wg.Wait() // Wait for all to finish
```

Ensures the main goroutine waits for all workers to complete cleanup before exiting.

### 4. **Signal Handling**

```go
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
go func() {
    <-sigChan
    close(exit) // Trigger graceful shutdown
}()
```

Allows your application to respond to system signals like Ctrl+C.

### 5. **Channel State Checking**

```go
case msg, ok := <-ch:
    if !ok {
        return // Channel closed
    }
    // Process msg
```

Always check the `ok` value to detect closed channels.

## Common Pitfalls to Avoid

- **Goroutine Leaks**: Always ensure goroutines have a way to exit
- **Blocking Operations**: Use timeouts or `default` cases in `select` statements
- **Closing Channels Twice**: Only close a channel once, and only from the sender
- **Race Conditions**: Use proper synchronization primitives like `sync.WaitGroup`

## When to Use These Patterns

- **Producer-Consumer**: Decouple data production from consumption
- **Graceful Shutdown**: Essential for servers and long-running applications  
- **Timeout Handling**: Prevent resource leaks and improve responsiveness
- **Signal Handling**: Respond to system signals (Ctrl+C, etc.)

## Key Takeaways

1. **Start Simple**: Begin with basic producer-consumer patterns
2. **Add Complexity Gradually**: Introduce timeouts, graceful shutdown, and signal handling step by step
3. **Always Clean Up**: Use `defer` statements and `WaitGroup` for proper resource cleanup
4. **Handle Edge Cases**: Timeouts, channel closures, and signal interruptions

Go's concurrency model is about **communication** rather than **sharing memory**. Channels are the primary communication mechanism, and goroutines are the workers that communicate through them.

---

*Want to see more Go patterns? Check out my other posts on [Go composition]({{< ref "go-composition" >}}) and [system design]({{< ref "posts/2025/04/system-design" >}}).*
