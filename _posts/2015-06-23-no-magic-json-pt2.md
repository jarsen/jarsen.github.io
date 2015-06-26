---
layout: post
title: No-Magic JSON Parsing with Swift Part 2
feature-img: "img/wizards_only_fools.png"
---

Last time we talked about how we could [simplify our JSON parsing](http://jasonlarsen.me/2015/06/23/no-magic-json.html) with Swift 2 error handling and get rid of all the magic. Today we're going to use the power of type inference to take our old `JSONObject` struct from having 12 methods (or more, if you want to add methods for values like CGFloat, etc.) to only 4!

Let's look at our `valueForKey` function.

{% highlight swift %}
func valueForKey<A>(key: String, type: A.Type) throws -> A {
    let pathComponents = key.componentsSeparatedByString(".")

    var accumulator: Any = dictionary

    for component in pathComponents {
        if let dict = accumulator as? [String:AnyObject] {
            if let value = dict[component] {
                accumulator = value
                continue
            }
        }
        throw JSONError.NoValueForKey(key)
    }

    if let value = accumulator as? A {
        return value
    }
    else {
        throw JSONError.TypeMismatch
    }
}
{% endhighlight %}

With this function we tell our struct what type we are looking to get out and and what key. But the Swift compiler is smart. Real smart. Given just the minimum amount of information, it can _infer_ what types we're talking about.

It turns out we can actually remove the `type` argument completely, and as long as we give the compiler a little help
at the function's callsite by specifying the type that the return value will be assigned to, it can infer `A` for us.

{% highlight swift %}
func valueForKey<A>(key: String) throws -> A {
    let pathComponents = key.componentsSeparatedByString(".")

    var accumulator: Any = dictionary

    for component in pathComponents {
        if let dict = accumulator as? [String:AnyObject] {
            if let value = dict[component] {
                accumulator = value
                continue
            }
        }
        throw JSONError.NoValueForKey(key)
    }

    if let value = accumulator as? A {
        return value
    }
    else {
        throw JSONError.TypeMismatch
    }
}
{% endhighlight %}

And we can use it like so:

{% highlight swift %}
let street: String = try json.valueForKey("address.street")
{% endhighlight %}

Et voila.

This means that we can remove all of our `intForKey`, `stringForKey`, etc. And, if we revamp our `optionalForKey` in the same way by also removing the `type` argument, we can do the same for all our calls to `optionalForKey`. The only functions I didn't remove in my implementation were `objectForKey` and `optionalObjectForKey` because since the values there are really `[String:AnyObject]` you can't get a `JSONObject` out without the initializer.

Here's what our updated `Person` model might look like:

{% highlight swift %}
struct Person {
    var name: String
    var age: Int
    var height: Double
    var street: String?

    init(json: JSONObject) throws {
        try name = json.valueForKey("name")
        try age = json.valueForKey("age")
        try height = json.valueForKey("height")
        try street = json.optionalForKey("address.street")
    }
}
{% endhighlight %}

I hope you've enjoyed this. As always, comments are welcome—either down below or on [twitter](http://twitter.com/jarsen) ([@jarsen](http://twitter.com/jarsen))

You can get the [full code here](https://gist.github.com/jarsen/b27de2ed9e84b84086ae)
