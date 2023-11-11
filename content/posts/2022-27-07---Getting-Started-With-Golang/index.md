---
title: "Getting started with Golang."
date: "2022-07-27T23:46:37.121Z"
template: "post"
draft: false
slug: "/posts/getting-started-with-golang"
category: "Software Engineering"
tags:
  - "Web Development"
description: "Go is an open-source programming language that was first created at Google. It is statically typed and comes with memory allocation, garbage collection, and with concurrency built-in. Go’s design gives the language high performance and speed."
---

**Golang**, also known as **Go** is an open-source programming language that was first created at Google by Robert Griesemer, Rob Pike, and Ken Thompson in 2007. It was created with these things in mind:

- efficient execution
- efficient compilation
- ease of programming

It is statically typed and comes with memory allocation, garbage collection, and with concurrency built-in. Go’s design gives the language high performance and speed.

### Getting Started: Install Go
To get started with Go, install it in your development environment. For that, visit [the official Go download page](https://go.dev/dl/), and download for your specific machine.

Confirm if the installation was successful by running the command  `go version`. The output should be similar as shown below. ⬇️

```
go version go1.18 darwin/amd64
``` 

### Writing Our First Go Program
Now we have installed Go on our machine, let's write our first Go program. In your code Open your code editor, and create a file called `hello-world.go` in , and add the following code shown below. ⬇️


```
package main
import "fmt"

func main() {
    fmt.Println("Hello World!")
}
``` 

Now to run our code above, we run the command `go run hello-world.go`. The output is shown below. ⬇️

```
Hello World!
``` 

Let's look at the different parts of our program

1. **package main**: Go programs start running in the main package. It is a special package that is used with programs that are meant to be executable.

2. **import fmt**: `fmt` is a core library package that contains functionalities related to formatting and printing output or reading input from various I/O sources. 

3. **func main() {}**: The `main()` function is a special function that is the entry point of an executable program. 


### Links to learning resources.

- [Go Documentation](https://go.dev/doc/)

- [Effective Go](https://go.dev/doc/effective_go)

- [A Tour of Go](https://go.dev/tour/welcome/1)


### Conclusion
In this article, we've seen how to set Go on our machine and write a program that prints `Hello World!` to the terminal. With the resources shared above, you're on the way to becoming a Go Pro 💪🏽. Alright, see you soon. ✌🏽