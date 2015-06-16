---
layout: post
title: Result vs. Throws
feature-img: "img/pipe.jpg"
---

Swift 2.0 is causing a lot of well deserved excitement. Chris and the folks over at
apple have been doing an *amazing* job.
I have been loving Swift the past year and am really excited about all the new
features—and, of course, the open source announcement.

When the `do`/`catch` model for error handling was announced, there was a bit of frustration
that `Result` was not the adopted model. Since then, several developers have come out
with [articles](http://www.sunsetlakesoftware.com/2015/06/12/swift-2-error-handling-practice)
detailing what they [like better](http://owensd.io/2015/06/12/swift-throws-get-it.html) about it
that have got [quite a bit of attention](https://twitter.com/clattner_llvm/status/610154161552756736).

While I am excited about the prospect of standardized error handling, I still am not convinced that
throwing errors is a better approach than `Result`. I have a lot of mixed feelings.
I'm sure there will be errors and counter-points I have yet to consider, in
which case please leave a comment at the bottom or ping me on twitter ([@jarsen](http://twitter.com/jarsen)), but here are my thoughts so far.

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

Both these arguments have good points. Although in the latter case, the only reason
someone would do this is to run a function that performs side effects, in which case
I would argue that the number of functions performing side effects and returning
_synchronous_ errors where you don't care about the successful value of the operation
is pretty small. If you have a lot of these, I think you're using `Result` wrong.

And finally, look what I can do:

{% highlight swift %}
try! functionThatThrows() // forces execution without a catch
{% endhighlight %}

Both approaches can ignore errors. Sure, `try!` will make your program crash if something throws,
and you might not think that counts as ignoring errors, but I can see the lazy programmer putting
this in because it works in dev every time, or because it rarely executes, or because for some reason
`functionThatThrows()` only throws a fraction of the time. It's an open invitation to shoot yourself in the foot.
It is a more dangerous way to ignore errors, but it ignores them nonetheless.

Finally, the cynic in me says many will end up writing empty `catch` statements, or else simply:

{% highlight swift %}
do {
    try stuff()
}
catch {
    print("Error while doing stuff")
}
{% endhighlight %}

I don't know if this really qualifies as "handling" anything.

And in many cases not doing anything with an error might be the right thing to do.

Ultimately I don't think that making it impossible to ignore errors is the goal here; I think
the goal should be to make it difficult to ignore errors.

## Exhaustive

**Handling errors appropriately is just as important as not ignoring errors; ignorance
about their types is just ignoring them in a different way.** It is important to know
what types of errors you deal with, and for the compiler to help you handle them exhaustively.

I mentioned that the `switch` statement would require you to be exhaustive in your error handling.
In contrast, `throws` doesn't know anything about the error types at all. "Exhaustive" in Swift's `try`/`catch`
world simply means: no matter what, you must always have a catch-all statement because you have no idea
what `ErrorType`s to expect from a function contract.

{% highlight swift %}
enum Fail : ErrorType {
    case Normal
    case Epic
}

func foo() throws {
    // some code here, may throw any `Fail` case
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
    print("this statement will NEVER be executed")
}
{% endhighlight %}

Compare this to the same code using `Result`:

{% highlight swift %}

switch someResult {
case let .Success(value):
    print(value)
case .Failure(.Normal):
    print("Normal fail")
case .Failure(.Epic):
    print("Epic fail!")
}

{% endhighlight %}

I'm unsure if Swift will gain the ability to `throw` specific error types in the future.
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

**Failing to communicate about the error path in the function contract implies that for some reason
the "happy path" is more important.** How can you expect to build robust error handling
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

If `thing1()` fails, `thing2()` will never be executed. In many cases that might be the desired
behavior. However, what if you actually want to do something different in the case that both fail,
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

One of the main arguments I've seen against `throws` is that is has no way of handling asynchronous
errors. Async error handling is a big deal—most of my error handling code revolves around
async networking errors. That said, I actually think that this point has been a little over-exaggerated by
those contending for `Result`—async errors are just handled with a different pattern altogether.
However, I still believe `Result` is better than said solution. Take a look at this async networking function.

{% highlight swift %}

func fetchJSON(endpoint: String, completionHandler: (JSONDictionary?, ErrorType?) -> Void) {
    //...
}

fetchJSON("/api/v3/users") { json, error in
    if let error = error {
        switch error {
        case ErrorType.Type1:
            // type 1
        case ErrorType.Type2:
            // type 2
        }
        return
    }

    guard let json = json else {
        return
    }

    // handle JSON
}

{% endhighlight %}

This simple function takes an API endpoint, goes and does the networking and JSON parsing,
and then returns the result or an error to the user. This code is not horrible. However, it
does mean the developer needs to understand the conventions being used: namely that you need
to check if the error is nil first.


Notice I said it "returns the result or an error." Gee, that sounds an awful like a `Result`.

{% highlight swift %}

func fetchJSON(endpoint: String, completionHandler: Result<JSONDictionary, ErrorType> -> Void) {
    //...
}

fetchJSON("/api/v3/users") { result in
    switch result {
    case let .Success(json):
        // handle JSON
    case let .Failure(.Type1):
        // handle error type 1
    case let .Failure(.Type2):
        // handle error type 2
    }
}

{% endhighlight %}

I think the equivalent code using `Result` is much better: I have a single switch that handles
_all_ cases.

## No Asserts

Currently there are no functions available in the `XCTAssert` family for asserting whether or not
a function or closure throw an error.

Please join me in filing a [radar](http://bugreport.apple.com).

Here's how you might write one in the meantime:

{% highlight swift %}
func XCTAssertThrows<A>(file: String = __FILE__, line: Int = __LINE__, test: () throws -> A) {
    do {
        try test()
        XCTFail("Whoops", file: file, line: UInt(line))
    }
    catch {

    }
}
{% endhighlight %}

## throws is "simpler"

A lot of people seem to think that `throws` is easier. I'm not sure if I agree, but even so
I find it irrelevant. What does "simpler" mean? Does it mean more familiar
for those who have never touched Haskell or Rust or the likes?

`Result` is no less simple than `Optional`, which in his great wisdom Papa Swift found
fit to force upon everyone. It may be foreign at first, but if it's a better way of doing
things: learn it. Learning "hard" things can make you a better programmer, and can make
your software better.

Finally, the new error handling introduces 5 keywords: `throws`, `rethrows`, `do`, `try`, and `catch`.
I don't know that I would say the introduction of 5 new keywords is "simple".

## Result<(), ErrorType>

One gripe with `Result` I see is that sometimes people end up having to type something as `Result<(), ErrorType>`
because all they really want is a pure Failure. I agree, it can feel a little weird. However, it's not that
hard, it's rare, and you really only do it when you're going to `map` it to something else later.

If you find yourself doing this often, I would reexamine your code. Are you really using `Result` in the
correct places? If all you're doing is returning whether an operation fails or not, and you don't care
what the error is, return a `Bool` instead, or perhaps an `ErrorType?`.

## Advanced Operators

`Result` brings with it a lot of powerful functional programming ideas and higher order functions. Talking
about `apply`, `map`, and `flatMap` is really beyond the scope of this post (plus I don't want to say the
words: Applicative, Functor, or Monad so you keep reading). Perhaps this is where some of the fear of complexity
comes from. But they're really not complex—just new ways of thinking.

Such higher order functions enable powerful abstractions like JSON parsing a la [Argo](http://github.com/thoughtbot/argo), or
my [Pipes](http://github.com/jarsen/pipes) library I [blogged about the other week](http://jasonlarsen.me/2015/05/23/pipes.html). The forward pipe operator makes working with
`Result` awesome. **The forward pipe operator allows you to compose a data pipeline completely agnostic towards optionals or results.** And it reads so nice:

{% highlight swift %}
let processedText = inputFileName
                    |> escapeInput
                    |> readFile
                    |> processText
{% endhighlight %}

But I'm biased there. :)

## But, Standards...

All this said, I think standards are important. And to me, that's one of the biggest selling points `throws`
has. The Swift team has spoken, the Cocoa APIs have been rewritten, and let's face it: they're not going back.

Fortunately, using functions like [antitypical/Result](http://github.com/antitypical/Result)'s
`materialize` and `dematerialize` (once a few compiler bugs are worked out) it is possible to switch
between thrown errors and and results. So, if you too like `Result`, then you're not up the river
with no paddle when working with other APIs.

That said, there are lots of advantages to sticking close to the herd. That is worth some weight in your judgements.

## Conclusion

I'm new to this, just like everyone else; I'm still playing with it, and seeing how it feels.
Sometimes I find myself liking it, others I find myself frustrated. These
are just my impressions after a week—I'm sure I have plenty of errors and there are better ways that
I missed. Again, comments below or on [twitter](http://twitter.com/jarsen) are appreciated.

Papa Swift and everyone working on the Swift team are doing a _tremendous_ job.
Just because I don't happen to like the new error handling model as much as I like `Result` yet
doesn't mean they made the wrong decision: there are lots of factors to consider, and I guarantee you
they thought about it long and hard. I also hope that some more error handling features are on the
horizon, like some sort of `await` for handling async errors.

I have yet to shake my greater love for `Result`. As such, I think I will be sticking to it for most of my
failable code. That said, I'm still playing with `throws` and only time will tell.
