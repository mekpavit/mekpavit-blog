+++
title = "The Story of Call Stack in Go"
date = 2021-01-09T00:00:00+07:00
tags = ["go"]
draft = false
+++

If an error is returned to your Go's main function, how do you figure out where does it come from? If, now, you only rely on the error messge, you must read this!


## Go's call stack VS other language's call stack when an error happens {#go-s-call-stack-vs-other-language-s-call-stack-when-an-error-happens}

So, what's wrong with the Go's call stack? Why I have to bother writing the story about it? Before answer this question, I want you to know what other language does with the call stack when it catches an error. Since Python is a popular one, I will use it as an example.

Ok, suppose that you have a `main` which calls an errorable function, `functionA`. Somehow, this `functionA` also calls another errorable function called `functionB`. If there is an error occured when you're in `main`, what is it looks like in Python?

```python
import math
import traceback
import sys

def main():
    try:
        functionA()
    except:
        traceback.print_exc(file=sys.stdout)

def functionA():
    functionB()

def functionB():
    math.isfinite("not number")

main()
```

```python
Traceback (most recent call last):
  File "<stdin>", line 7, in main
  File "<stdin>", line 12, in functionA
  File "<stdin>", line 15, in functionB
AttributeError: 'module' object has no attribute 'isfinite'
```

You can see that the Python's call stack contains all calls since `main` to `functionB`. This is because Python stops the program and records the call stack exactly at the point the error happens.

What's great about this is that, you can just use `try-except` block on the outer-most function and still obtains the whole call stack that include the call where the error occurs. Just like in this example, you put `try-except` block in `main` and when you print the call stack with `traceback.print_exc`, you still have the `functionB` call in the stack.

But what's about Go? If there is an error happens inside `functionB` just like the previous one, what is the call stack looks like when I'm inside `main`?

```go
package main

import (
	"errors"
	"fmt"
	"runtime"
)

func main() {
	err := functionA()
	if err != nil {
		// Print the call stack in Python style if there is an error...
		fmt.Println("Traceback (most recent call last):")
        programCounters := make([]uintptr, 10)
        n := runtime.Callers(0, programCounters)
        frames := runtime.CallersFrames(programCounters[:n])
        for frame, hasNext := frames.Next(); hasNext; frame, hasNext = frames.Next() {
            fmt.Printf("  File '%s' line %d, in %s\n", frame.File, frame.Line, frame.Function)
        }
		fmt.Println(err.Error())
	}
}

func functionA() error {
	return functionB()
}

func functionB() error {
	return errors.New("error!")
}
```

```go
Traceback (most recent call last):
  File '/usr/local/go/src/runtime/extern.go' line 216, in runtime.Callers
  File '/var/folders/s2/b3qkm_7n6s9d7y7xlvx7yfq40000gn/T/babel-Ssc4iS/go-src-9U0LdE.go' line 15, in main.main
  File '/usr/local/go/src/runtime/proc.go' line 204, in runtime.main
error!
```

WTF, nor `functionA` or `functionB` is mentioned in the call stack! This is because Go doesn't record the call that causes the error, it just simply pop any calls out of the stack when they return. When you're inside `main` and print the call stack, you won't see anything since `functionA` and `functionB` calls already returned.

So, if you want to see the full call stack that contains the error call, you must print the stack inside `functionB` like this:

```go
package main

import (
	"errors"
	"fmt"
	"runtime"
)

func main() {
	err := functionA()
	if err != nil {
		fmt.Println(err.Error())
	}
}

func functionA() error {
	return functionB()
}

func functionB() error {
    // Print the call stack in Python style if there is an error...
    fmt.Println("Traceback (most recent call last):")
    programCounters := make([]uintptr, 10)
    n := runtime.Callers(0, programCounters)
    frames := runtime.CallersFrames(programCounters[:n])
    for frame, hasNext := frames.Next(); hasNext; frame, hasNext = frames.Next() {
        fmt.Printf("  File '%s' line %d, in %s\n", frame.File, frame.Line, frame.Function)
    }
	return errors.New("error!")
}
```

```go
Traceback (most recent call last):
  File '/usr/local/go/src/runtime/extern.go' line 216, in runtime.Callers
  File '/var/folders/s2/b3qkm_7n6s9d7y7xlvx7yfq40000gn/T/babel-Bp8Xas/go-src-U2OZYD.go' line 25, in main.functionB
  File '/var/folders/s2/b3qkm_7n6s9d7y7xlvx7yfq40000gn/T/babel-Bp8Xas/go-src-U2OZYD.go' line 18, in main.functionA
  File '/var/folders/s2/b3qkm_7n6s9d7y7xlvx7yfq40000gn/T/babel-Bp8Xas/go-src-U2OZYD.go' line 11, in main.main
  File '/usr/local/go/src/runtime/proc.go' line 204, in runtime.main
error!
```

As you can see, the `functionB` and `functionA` calls are now in the call stack! So, does this mean that we can't simply get the full stack call at `main` or any outer function? The answer is NO! Luckily, with the introduce of wrappable error in Go 1.13, we have a elegant way to solve this problem!


## ErrorWithCallStack now comes to rescue you!!! {#errorwithcallstack-now-comes-to-rescue-you}

In Go 1.13, they introduce the concept of [wrappable error](https://blog.golang.org/go1.13-errors). If you don't have time to read the full article, there are 3 main points that help us solve the stack call problem:

-   Error can wrap other error by exposing `Unwrap` method that return the wrapped error
-   You can check whether or not an particular error is wrapped inside the other error using `errors.As`
-   `errors.As` checks through the whole wrapping chain of the given error not just the only one level

When `errorA` happens inside inner function; If you capture and store `errorA` with the full call stack inside the special error type `errorB`, `main` will now be able to check whether the returned error is `errorB` and extract the call stack and `errorA` from it.

The special error `errorB` is exactly the `ErrorWithCallStack` I mentioned in the sectoin name. I believe you get the idea now! Let's see how we can implement `ErrorWithCallStack` by ourselves!

**First**, you need to create `ErrorWithCallStack` with the following properties:

-   Contain `CallStack` field that will be used to store the call stack and `WrappedError` field that stores wrapped error

<!--listend-->

```go
type ErrorWithCallStack struct {
	CallStack []Call
	WrappedError error
}
```

-   Implement `Error` and `Unwrap` methods

<!--listend-->

```go
func (err *ErrorWithCallStack) Error() string {
	return err.WrappedError.Error()
}

func (err *ErrorWithCallStack) Unwrap() error {
	return err.WrappedError
}
```

-   Have `NewErrorWithCallStack` constructor function that capture and store the call stack and warpped error inside `CallStack` field and `WrapperError` field respectively
-   Mechanism to not construct new `ErrorWithCallStack` if the given `error` is already the `ErrorWithCallStack`

<!--listend-->

```go
func NewErrorWithCallStack(wrappedError error) *ErrorWithCallStack {
	// if the wrappedError is already the ErrorWithCallStack, do nothing
	var errorWithCallStackInsideWrapperError *ErrorWithCallStack
	if errors.As(wrappedError, &errorWithCallStackInsideWrapperError) {
		return errorWithCallStackInsideWrapperError
	}

	// get the call stack and store it
	callStack := make([]Call, 0)
	programCounters := make([]uintptr, MaxCallsInStack)
	n := runtime.Callers(0, programCounters)
	frames := runtime.CallersFrames(programCounters[:n])
	for frame, hasNext := frames.Next(); hasNext; frame, hasNext = frames.Next() {
		callStack = append(callStack, Call{File: frame.File, Line: frame.Line, Function: frame.Function})
	}
	return &ErrorWithCallStack{CallStack: callStack, WrappedError: wrappedError}
}
```

**Second**, when there is an error happens, you need to wrap the error with `NewErrorWithCallStack` function.

You can also wrap the returned error in the other outer function. This will ensure that if someone forget to you `NewErrorWithCallStack` inside the deepest function, there is still a call stack to the next one.

```go
func functionA() error {
	if err := functionB(); err != nil {
		// if you forget to use NewErrorWithCallStack inside functionB, you still at least has this
		return NewErrorWithCallStack(err)
	}
	return nil
}

// always wrap =ErrorWithCallStack= to the error when an error happens
func functionB() error {
	return NewErrorWithCallStack(errors.New("error!"))
}
```

**Third**, in `main`, use `errors.As` to check if the returned error contains `*ErrorWithCallStack` type. If yes, print/log the stack call.

```go
func main() {
	err := functionA()
	if err != nil {
		// print the call stack if the error contains =ErrorWithCallStack=, else print only error message
		var errorWithCallStack *ErrorWithCallStack
		if errors.As(err, &errorWithCallStack) {
            fmt.Println("Traceback (most recent call last):")
            for _, call := range errorWithCallStack.CallStack {
                fmt.Printf("  File '%s' line %d, in %s\n", call.File, call.Line, call.Function)
            }
            fmt.Println(err.Error())
		} else {
			fmt.Println(err.Error())
		}
	}
}
```

You can see the full implementation and the usage of `ErrorWithCallStack` here:

```go
package main

import (
	"errors"
	"fmt"
	"runtime"
)

const MaxCallsInStack = 10

// First, create the ErrorWithCallStack type
type ErrorWithCallStack struct {
	CallStack []Call
	WrappedError error
}

func NewErrorWithCallStack(wrappedError error) *ErrorWithCallStack {
	// If the wrappedError is already the ErrorWithCallStack, do nothing
	var errorWithCallStackInsideWrapperError *ErrorWithCallStack
	if errors.As(wrappedError, &errorWithCallStackInsideWrapperError) {
		return errorWithCallStackInsideWrapperError
	}

	// Get the call stack and store it
	callStack := make([]Call, 0)
	programCounters := make([]uintptr, MaxCallsInStack)
	n := runtime.Callers(0, programCounters)
	frames := runtime.CallersFrames(programCounters[:n])
	for frame, hasNext := frames.Next(); hasNext; frame, hasNext = frames.Next() {
		callStack = append(callStack, Call{File: frame.File, Line: frame.Line, Function: frame.Function})
	}
	return &ErrorWithCallStack{CallStack: callStack, WrappedError: wrappedError}
}

func (err *ErrorWithCallStack) Error() string {
	return err.WrappedError.Error()
}

func (err *ErrorWithCallStack) Unwrap() error {
	return err.WrappedError
}

type Call struct {
	File string
	Line int
	Function string
}

func functionA() error {
	if err := functionB(); err != nil {
		// If you forget to use NewErrorWithCallStack inside functionB, you still at least has this
		return NewErrorWithCallStack(err)
	}
	return nil
}

// Second, always wrap =ErrorWithCallStack= to the error when an error happens
func functionB() error {
	return NewErrorWithCallStack(errors.New("error!"))
}

func main() {
	err := functionA()
	if err != nil {
		// Print the call stack if the error contains =ErrorWithCallStack=, else print only error message
		var errorWithCallStack *ErrorWithCallStack
		if errors.As(err, &errorWithCallStack) {
            fmt.Println("Traceback (most recent call last):")
            for _, call := range errorWithCallStack.CallStack {
                fmt.Printf("  File '%s' line %d, in %s\n", call.File, call.Line, call.Function)
            }
            fmt.Println(err.Error())
		} else {
			fmt.Println(err.Error())
		}
	}
}
```

```go
Traceback (most recent call last):
  File '/usr/local/go/src/runtime/extern.go' line 216, in runtime.Callers
  File '/var/folders/s2/b3qkm_7n6s9d7y7xlvx7yfq40000gn/T/babel-Bp8Xas/go-src-xpZ715.go' line 28, in main.NewErrorWithCallStack
  File '/var/folders/s2/b3qkm_7n6s9d7y7xlvx7yfq40000gn/T/babel-Bp8Xas/go-src-xpZ715.go' line 60, in main.functionB
  File '/var/folders/s2/b3qkm_7n6s9d7y7xlvx7yfq40000gn/T/babel-Bp8Xas/go-src-xpZ715.go' line 51, in main.functionA
  File '/var/folders/s2/b3qkm_7n6s9d7y7xlvx7yfq40000gn/T/babel-Bp8Xas/go-src-xpZ715.go' line 64, in main.main
  File '/usr/local/go/src/runtime/proc.go' line 204, in runtime.main
error!
```

You can see the whole call stack is captured even if you're in `main`! Now, with `ErrorWithCallStack`, you can have the call stack anywhere you want just like Python!


## Summary {#summary}

Because Go treats an error as a value, Go doesn't stop the call stack from being poped when an error occurs. This causes the important information on the stack call such as where is the error happens to be disappeared from the stack.

But with the introduce of wrappable error and `errors.As` function, we can develop a special error type `ErrorWithCallStack` that helps us storing the call stack at the moment an error occurs.
