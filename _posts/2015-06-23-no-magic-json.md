---
layout: post
title: No-Magic JSON Parsing with Swift
feature-img: "img/wizards_only_fools.png"
---

Since the beginning, JSON parsing has been a common pain point for Swift many developers. This is largely  due to difficulties in dealing with a strict type system (which JSON does not have) and optionals, as well as a lack of a consistent error handling approach.

Several great JSON parsing libraries have surfaced which avoid the mess of tangled if-lets (a.k.a. the infamous "pyramid of doom"). I really like the approach taken by thoughtbot in their [JSON](https://robots.thoughtbot.com/efficient-json-in-swift-with-functional-concepts-and-generics) [parsing](https://robots.thoughtbot.com/real-world-json-parsing-with-swift) articles, which ultimately led to their [Argo](https://github.com/thoughtbot/Argo) library on github (although I don't like everything about Argo).

However, most solutions tend to rely on a lot of magic: the magic of applicatives, functors, and monads; the magic of unfamiliar custom operators; the magic of a complex nested enum structure.

Now that swift 2.0 has brought error handling to us, we can have much less magical JSON parsing. Here is an example of what your JSON parsing code might look like with just a thin error-throwing layer over `[String:AnyObject]`.

## Let's Get Started

{% highlight swift %}
enum JSONError : ErrorType {
    case NoValueForKey(String)
    case TypeMismatch
}

class JSONObject {
    private let dictionary: [String:AnyObject]

    init(dictionary: [String:AnyObject]) {
        self.dictionary = dictionary
    }

    func valueForKey<A>(key: String, type: A.Type) throws -> A {
        if let value = self[key] {
            if let _ = value as? NSNull {
                throw JSONError.NoValueForKey(key)
            }
            if let value = value as? A {
                return value
            }
            throw JSONError.TypeMismatch
        }
        throw JSONError.NoValueForKey(key)
    }

    func stringForKey(key: String) throws -> String {
        return try valueForKey(key, type: String.self)
    }

    func intForKey(key: String) throws -> Int {
        return try valueForKey(key, type: Int.self)
    }

    func doubleForKey(key: String) throws -> Double {
        return try valueForKey(key, type: Double.self)
    }

    func arrayForKey<A>(key: String, type: A.Type) throws -> [A] {
        return try valueForKey(key, type: [A].self)
    }

    func objectForKey(key: String) throws -> JSONObject {
        let dict = try valueForKey(key, type: [String:AnyObject].self)
        return JSONObject(dictionary: dict)
    }
}
{% endhighlight %}

Here we've added a bunch of functions that let us get basic things like `String`, `Int`, and `Double` out of our `[String:AnyObject]` Pandora's Box. If we either find a nil or NSNull value, we know the key is bad. And if we find the wrong type object hidden behind the `AnyObject` we throw a `TypeMismatch` error.

How does this look in practice? Let's try it on a simple `Person` model.

{% highlight swift %}
struct Person {
    var name: String
    var age: Int
    var height: Double

    static func fromJSON(json: JSONObject) throws -> Person {
        let name = try json.stringForKey("name")
        let age = try json.intForKey("age")
        let height = try json.doubleForKey("height")
        return Person(name: name, age: age, height: height)
    }
}

do {
    let jsonObject: JSONObject = ["name": "Bob", "age": 26, "height": 5.75]
    let person = try Person.fromJSON(jsonObject)
}
catch let JSONError.NoValueForKey(key) {
    print("Didn't find value for key: \(key)")
}
catch {
    print("Wrong type")
}
{% endhighlight %}

Not bad! *No pyramid of doom, no external dependencies, you didn't have to learn a single new operator, and no worrying about the scary words like functor or monad!*

## Optional JSON Keys

Of course, we also want to be able to handle optional values in some cases.

{% highlight swift %}
extension JSONObject {
    private func optionalForKey<A>(key: String, type: A.Type) throws -> A? {
        do {
            let value = try valueForKey(key, type: type)
            return value
        }
        catch JSONError.NoValueForKey {
            return nil
        }
        catch {
            throw JSONError.TypeMismatch
        }
    }

    func stringOptionalForKey(key: String) throws -> String? {
        return try optionalForKey(key, type: String.self)
    }

    func intOptionalForKey(key: String) throws -> Int? {
        return try optionalForKey(key, type: Int.self)
    }

    func doubleOptionalForKey(key: String) throws -> Double? {
        return try optionalForKey(key, type: Double.self)
    }

    func arrayOptionalForKey<A>(key: String, type: A.Type) throws -> [A]? {
        return try optionalForKey(key, type: [A].self)
    }

    func objectOptionalForKey(json: JSONObject, key: String) throws -> JSONObject? {
        return try optionalForKey(key, type: JSONObject.self)
    }
}
{% endhighlight %}

Awesome. Now we have some pretty basic JSON handling in place, but honestly it's probably most of what you will ever need.

## Key Paths

There's one more thing that might be nice though: handling key paths. So, let's rewrite our `valueForKey`.

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

Now we can do this:

{% highlight swift %}
let street = try json.stringForKey("address.street")
{% endhighlight %}

And, because we implemented our `optionalForKey` on top of `valueForKey`, everything else will just keep working.

## Conclusion

The Swift 2.0 beta2 brings _extensions to generic types_. Generic types extensions allow us to add functionality to things like `Array` and `Dictionary` in specific cases. Unfortunately for the moment it seems that you can't give it specific types for Key/Value, only protocol conformance. If we could specify specific types, we could extend `[String:AnyObject]` and typealias it to `JSONObject` instead of having a wrapper class.

I think this JSON code is quite nice, and easy for anyone to understand without having their mind blown. Even though [I went a little caremad the other week about Result](http://jasonlarsen.me/2015/06/14/result-vs-throws.html), I like how this code turned out better than my old `Result` based JSON parsing: it's easier to explain to others, and it's much easier on the compiler, which saves you time when trying to figure out what is wrong. It is also much more idiomatic.

I'm sure there are some cool ways to improve this code, I'd love to hear your thoughts! Please leave a comment or give me a shout on [twitter](http://twitter.com/jarsen) ([@jarsen](http://twitter.com/jarsen))

And here's [the final code](https://gist.github.com/jarsen/2b0913111c0427642c41) all in one gist.
