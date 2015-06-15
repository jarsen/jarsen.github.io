---
layout: post
title: Result vs. Throws
feature-img: "img/pipe.jpg"
---

Swift 2.0 is causing a lot of well deserved excitement. Chris and the folks over at
apple have been doing an *amazing* job. Sending tons of love and kudos their way.
I have been loving Swift the past year and am really excited about all the new
features—and, of course, the open source announcement.

When the `do`/`catch` model for error handling was announced, I observed a bit of frustration on
twitter that `Result` was not the adopted model, but since then several developers have come out
with [articles](http://www.sunsetlakesoftware.com/2015/06/12/swift-2-error-handling-practice)
detailing what they [like better](http://owensd.io/2015/06/12/swift-throws-get-it.html) about it
that have got [quite a bit of attention](https://twitter.com/clattner_llvm/status/610154161552756736).

While I am excited about the prospect of standardized error handling, I still am not convinced that
throwing errors is a better approach than `Result`. The following post is my attempt to detail why
I feel this way. I'm sure there will be errors and counter-points I have yet to consider, in
which case please leave a comment at the bottom or ping me on twitter ([@jarsen](http://twitter.com/jarsen)).

## What is Result?

In case you aren't familiar, `Result` is an enum very similar to `Optional`: both have
cases that hold generic values, but where we have `Optional.None` to represent `nil` values,
we have instead `Result.Failure` with an associated `ErrorType`. Basically, think of `Result`
as `Optional` with that can carry an error instead of being nil.

## Deal With It

One of the purported advantages of `do`/`catch` error handling is that it forces the programmer
to handle errors.

But `Result` enforces error handling as well. Since `Result` is an `enum`, in order to
get at your potentially successful value you must deconstruct it using pattern matching. If you do so using
a switch statement, the switch statement will require you do be exhaustive in your case
handling, meaning that you must necessarily handle the `Failure` case.

Now, the astute observer may say, "But Jason, in Swift 2.0 I can deconstruct enums with the new
`if-case` statements, and thus can avoid handling failures."

{% highlight swift %}
if case let .Success(value) = result {
    doSomething(value)
}
{% endhighlight %}

"Or, you might run something and never handle the result at all. Throwing an error forces you
to write a catch"

{% highlight swift %}
let _ = someResultFunction() // ignore result entirely
{% endhighlight %}

Both these arguments have a good point. All I counter is with, "Look what I can do with try:"

{% highlight swift %}
try! functionThatThrows() // forces execution without a catch
{% endhighlight %}

Sure, `try!` will make your program crash if something throws, but that's besides the point.
The point is you can't really say the compiler forces you to handle errors if there is a
backdoor (albeit treacherous).

Finally, the cynic in me says many will end up writing empty `catch` statements, or else simply:

{% highlight swift %}
do {
    // stuff
}
catch {
    print("Error while doing stuff")
}
{% endhighlight %}

I don't know if this really qualifies as "handling" anything. And in many cases not
doing anything with an error might be the right thing to do.

## Exhaustive

I mentioned that the `switch` statement would require you to be *exhaustive* in your error handling.
That brings me to what I consider to be a weakness of `throws`: it doesn't know anything about types.

This means, whenever you catch, you must always have a catch-all statement, because you never know what
the callee might throw at you.

{% highlight swift %}
enum Fail : ErrorType {
    case Normal
    case Epic
}

func foo() throws {
    throw Fail.Epic
}

do {
    try foo()
}
catch Fail.Normal {
    print("Normal fail")
}
catch Fail.Epic {
    print("Epic fail!")
}
catch {
    print("this statement will never be executed")
}
{% endhighlight %}

Compare this to the same code using `Result`:

{% highlight swift %}

switch someResult {
case let .Success(value):
    print(value)
case Fail.Normal:
    print("Normal fail")
case Fail.Epic:
    print("Epic fail!")
}

{% endhighlight %}

I'm unsire if Swift will gain the ability to `throw` specific error types in the future.
Several people I have talked to think it's a for-sure enhancement. Others think lack of
types is actually a feature, claiming that specifying error types makes the API *fragile*.

If that's the case, why even have pattern matching in `catch` statements? If your API could
at any minute change what it throws, why bother handling specific errors at all?

## Contracts

For practice, I started writing a little JSON parsing library using throwing instead
of results. It was really frustrating to deal with Cocoa APIs because they don't detail
what errors might be thrown—I was left writing generic catch-all cases.

I believe error types belong in the function contract. `Result` forces the API to
communicate to its caller what in can expect in both the successful case, and in the case
of failure.

Failing to communicate about the error path in the contract implies that for some reason
the "happy path" is more important. How can you expect to build robust error handling
with a lackadaisical approach to error handling contracts?

## Hijacking Control Flow

Consider the following code:

{% highlight swift %}
do {
    try thing1()
    try thing2()
}
catch {
    print(error)
}
{% endhighlight %}

If `thing1()` fails, `thing2()` will never be executed. In many cases that might be fine.
However, what if you actually want to do something different in the case that both fail,
or in the case that only one fails? Using `throws` makes this difficult to impossible, as
`throws` will hijack your code flow and immediately terminate the `do` block.

{% highlight swift %}
let result1 = thing1()
let result2 = thing2()

switch (result1, result2) {
case (.Success, .Success):
    print("Everything is awesome!")
case (.Failure, .Failure):
    print("Everything is not awesome")
default:
    print("Everything is half awesome")
}

{% endhighlight %}

Since `Result` does not hijack your control flow, you are able to handle errors
in a more robust manner. This also makes `Result` a great candidate for asynchronous code.

## Async Error Handling


{% highlight swift %}

func asyncThing(callback: Result<String, Fail> -> Void) {
    // ...
}

asyncThing() { result in
    switch result {
    case .Success:
        print("success")
    case .Failure:
        print("failure")
    }
}

{% endhighlight %}



## No Asserts

Try writing your own XCTAssert for Results. Then try writing your own XCTAsserts for throwing functions.
In my opinion, the simplicity of doing such a thing for `Result` and the complexity of the solution for throws
indicates a code smell.

## throws is "easier"

maybe "more familiar"

No more complicated than `Optional`, which we see fit to force on everyone.

Having to learn something new, if it's a better way of doing things, is not bad

Having a well defined contract is easy.

## Result<(), ErrorType>

## Advanced Operators
