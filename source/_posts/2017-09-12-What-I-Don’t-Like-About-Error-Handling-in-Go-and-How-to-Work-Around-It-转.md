---
title: 'What I Don’t Like About Error Handling in Go, and How to Work Around It[转]'
date: 2017-09-12 22:33:08
tags:
 - golang
 - 搬运工
categories:
 - 学习
 - 技术
---

More often than not, people who write Go have some sort of opinion on its error handling model. Depending on your experience with other languages, you may be used to different approaches. That’s why I’ve decided to write this article, as despite being relatively opinionated, I think drawing on my experiences can be useful in the debate. The main issues I wanted to cover are that it is difficult to force good error handling practice, that errors don’t have stack traces, and that error handling itself is too verbose. However I’ve looked at some potential workarounds for these problems which could help negate the issues somewhat.

## Quick Comparison to Other Languages

[In Go, all errors are values.](https://blog.golang.org/errors-are-values) Because of this, a fair amount of functions end up returning an `error`, looking something like this:

```golang
func (s *SomeStruct) Function() (string, error)
```

As a result of this, the calling code will regularly have `if` statements to check for them:

```golang
bytes, err := someStruct.Function()
if err != nil {
  // Process error
}
```

Another approach is the `try-catch` model that is used in other languages such as Java, C#, Javascript, Objective C, Python etc. You could see the following Java code as synonymous to the previous Go examples, declaring `throws` instead of returning an `error`:

```java
public String function() throws Exception
```

And then doing `try-catch` instead of `if err != nil`:

```java
try {
  String result = someObject.function()
  // continue logic
}
catch (Exception e) {
  // process exception
}
```

Of course, there are more differences than this. For example, an `error` can’t crash your program, whereas an `Exception` can. There are others as well, and I want to focus on them in this article.

## Implementing Centralised Error Handling

Taking a step back, let’s look at why and how we might want to have a centralised place for handling errors.

An example most people would be familiar with is a web service – if some unexpected server-side error were to happen, we would generate a 5xx error. At a first pass in Go you might implement this:

```golang
func init() {
    http.HandleFunc("/users", viewUsers)
    http.HandleFunc("/companies", viewCompanies)
}
 
func viewUsers(w http.ResponseWriter, r *http.Request) {
    user // some code
    if err := userTemplate.Execute(w, user); err != nil {
        http.Error(w, err.Error(), 500)
    }
}
 
func viewCompanies(w http.ResponseWriter, r *http.Request) {
    companies = // some code
    if err := companiesTemplate.Execute(w, companies); err != nil {
        http.Error(w, err.Error(), 500)
    }
}
```

This isn’t a good solution, as we would have to repeat the same error handling across all of our handler functions. It would be much better to do it all in one place for maintainability purposes. Fortunately, there is [an alternative by Andrew Gerrand on the Go blog](https://blog.golang.org/error-handling-and-go) which works quite nicely. We can create a Type which does http error handling: 

```golang
type appHandler func(http.ResponseWriter, *http.Request) error
 
func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := fn(w, r); err != nil {
        http.Error(w, err.Error(), 500)
    }
}
```

And then it can be used as a wrapper to decorate our handlers:

```golang
func init() {
    http.Handle("/users", appHandler(viewUsers))
    http.Handle("/companies", appHandler(viewCompanies))
}
```

Then all we need to do is change the signature of handler functions so they return `errors`. This works nicely, as we have been able to apply the [dry](https://en.wikipedia.org/wiki/Don't_repeat_yourself) principle and not re-use code unnecessarily – now we return default errors in a single place.

## Error Context

In the previous example, there are many potential errors which we could receive, all of which could be generated in many parts of the call stack. This is when things start to get tricky.

To demonstrate this, we can expand on our handler. It’s more likely to look like this as the template execution is not the only place where an error could occur:

```golang
func viewUsers(w http.ResponseWriter, r *http.Request) error {
    user, err := findUser(r.formValue("id")) 
    if err != nil {
      return err;
    }
    return userTemplate.Execute(w, user);
}
```

The call chain could get quite deep, and throughout it, all sorts of errors could be instantiated in different places. This post by [Russ Cox](https://research.swtch.com/go2017) explains the best practice to prevent this from being too much of a problem:

    Part of the intended contract for error reporting in Go is that functions include relevant available
    context, including the operation being attempted (such as the function name and its arguments)
    
The example given is a call to the OS package:

```golang
err := os.Remove("/tmp/nonexist")
fmt.Println(err)
```

Which prints the output:

```golang
remove /tmp/nonexist: no such file or directory
```

To summarise, you are outputting the method called, the arguments given, and the specific thing that went wrong, immediately after doing it. When creating an `Exception` message in another language you would also follow this practice. So if we stuck to these rules in our `viewUsers` handler, it could almost always be clear what the cause of an error is.

The problem comes from people not following this best practice, and quite often in third party Go libraries you will see messages like:
```
Oh no I broke
```

Which is just not helpful – you don’t know anything about the context which makes it really hard to debug. Even worse is when these sorts of errors are ignored or returned really far back up the stack until they are handled:

```golang
if err != nil {
  return err
}
```

This means that when they happened isn’t communicated.

It should be noted that all these mistakes can be made in an Exception driven model – poor error messages, swallowing exceptions etc. So why do I think that model is more helpful?

If we’re dealing with a poor exception message, we are still able to know where it occurred in the call stack. This is because of stack traces, which raises something I don’t get about Go – you have the concept of panic in Go which contains a stack trace, but an error which does not. I think the reasoning is that a panic can crash your program, so requires a stack trace, whereas a handled error does not as you are supposed to do something about it where it occurs.

So let’s go back to our previous example – a third party library with a poor error message, which just gets propagated all the way up the call chain. Do you think debugging would be easier if you had this?

```golang
panic: Oh no I broke
[signal 0xb code=0x1 addr=0x0 pc=0xfc90f]
 
goroutine 1103 [running]:
panic(0x4bed00, 0xc82000c0b0)
/usr/local/go/src/runtime/panic.go:481 +0x3e6
github.com/Org/app/core.(_app).captureRequest(0xc820163340, 0x0, 0x55bd50, 0x0, 0x0)
/home/ubuntu/.go_workspace/src/github.com/Org/App/core/main.go:313 +0x12cf
github.com/Org/app/core.(_app).processRequest(0xc820163340, 0xc82064e1c0, 0xc82002aab8, 0x1)
/home/ubuntu/.go_workspace/src/github.com/Org/App/core/main.go:203 +0xb6
github.com/Org/app/core.NewProxy.func2(0xc82064e1c0, 0xc820bb2000, 0xc820bb2000, 0x1)
/home/ubuntu/.go_workspace/src/github.com/Org/App/core/proxy.go:51 +0x2a
github.com/Org/app/core/vendor/github.com/rusenask/goproxy.FuncReqHandler.Handle(0xc820da36e0, 0xc82064e1c0, 0xc820bb2000, 0xc5001, 0xc820b4a0a0)
/home/ubuntu/.go_workspace/src/github.com/Org/app/core/vendor/github.com/rusenask/goproxy/actions.go:19 +0x30
```

I think this might be something which has been overlooked in the design of Go – not that things aren’t overlooked in all languages.

If we use Java as an arbitrary example, one of the silliest mistakes people make is not logging the stack trace:

```golang
LOGGER.error(ex.getMessage()) // Doesn't log stack trace
LOGGER.error(ex.getMessage(), ex) // Does log stack trace
```

But Go seems to not have this information by design.

In terms of getting context information – Russ also mentions the community are talking about some potential interfaces for stripping out error contexts. It would be interesting to hear more about this.

## Solution to the Stack Trace Problem
Fortunately, after doing some searching, I found this excellent [Go Errors](https://github.com/go-errors/errors) library which helps solves the problem, by adding stack traces to errors:

```golang
if errors.Is(err, crashy.Crashed) {
  fmt.Println(err.(*errors.Error).ErrorStack())
}
```

However, I’d think it would be an improvement for this feature to have first class citizenship in the language, so you wouldn’t have to fiddle around with types. Also, if we are working with a third party library like in the previous example then it is probably not using `crashy` – we still have the same problem.

## What Should We Do with an Error?

We also have to think about what should happen when an error occurs. [It’s definitely useful that they can’t crash your program](https://davidnix.io/post/error-handling-in-go/), and it’s also idiomatic to handle them immediately:

```golang
err := method()
if err != nil {
  // some logic that I must do now in the event of an error!
}
```

But what happens if we want to call lots of methods which return errors, and then handle them all in the same place? Something like this:

```golang
err := doSomething()
if err != nil {
  // handle the error here
}
 
func doSomething() error {
  err := someMethod()
  if err != nil {
    return err
  }
  err = someOther()
  if err != nil {
    return err
  }
  someOtherMethod()
}
```

It feels a little verbose, whereas in other languages you can treat multiple statements that fail as a block:

```java
try {
  someMethod()
  someOther()
  someOtherMethod()
}
catch (Exception e) {
  // process exception
}
```

Or just propagate failure in the method signature:

```java
public void doSomething() throws SomeErrorToPropogate {
  someMethod()
  someOther()
  someOtherMethod()
}
```

Personally I think both of these example achieve the same thing, only the `Exception` model is less verbose and more flexible. If anything, I find the `if err != nil` to feel like boilerplate. Maybe there is a way that it could be cleaned up?

## Treating Multiple Statements That Fail as a Block
To begin with, I did some more reading and found a relatively pragmatic solution on the by [Rob Pike on the Go Blog](https://blog.golang.org/errors-are-values).

He defines a struct with a method which wraps errors:

```golang
type errWriter struct {
    w   io.Writer
    err error
}
 
func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

This let’s us do:

```golang
ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
// and so on
if ew.err != nil {
    return ew.err
}
```

This is also a good solution, but I still feel like something is missing – as we can’t re-use this pattern. If we wanted a method which took a string as an argument, then we’d have to change the function signature. Or what if we didn’t want to perform a write? We could try and make it more generic:

```golang
type errWrapper struct {
    err error
}
```

```golang
func (ew *errWrapper) do(f func() error) {
    if ew.err != nil {
        return
    }
    ew.err = f();
}
```

But we have the same problem that it won’t compile if we want to call functions which have different arguments. However you simply wrap those function calls:

```golang
w := &errWrapper{}
 
w.do(func() error {
    return someFunction(1, 2);
})
 
w.do(func() error {
    return otherFunction("foo");
})
 
err := w.err
 
if err != nil {
// process error here
}
```

This works, but doesn’t help too much as it ends up being more verbose than the standard `if err != nil` checks. I would be interested to hear if anyone can offer any other solutions. Maybe the language itself needs some sort of way to propagate or group errors in a less bloated fashion – but it feels like it’s been specifically designed to not do that.

## Conclusion

After reading this, you might think that by picking on `errors` I’m opposed to Go. But that’s not the case, I’m just describing how it compares to my experience with the `try catch` model. It’s a great language for systems programming, and some outstanding tools have been produced by it. To name a few there is [Kubernetes](https://kubernetes.io/), [Docker](https://www.docker.com/), [Terraform](https://www.terraform.io/), [Hoverfly](http://hoverfly.io/en/latest/) and others. There’s also the advantage of your tiny, highly performant, native binary. But errors have been difficult to adjust to. I hope my reasoning makes sense, and also that some of the solutions and workarounds could be of help.

[原文链接](https://opencredo.com/why-i-dont-like-error-handling-in-go/)
