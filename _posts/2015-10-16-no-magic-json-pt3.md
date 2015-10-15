---
layout: post
title: Protocol-Oriented JSON in Swift
feature-img: "img/crusty.png"
---

Previously we [explored](http://jasonlarsen.me/2015/06/23/no-magic-json-pt2.html) how to use Swift's amazing type system to simplify our JSON code. We came up with a solution that was concise, without building a complicated tree of JSON enums, using crazy operators, or getting deep into reflection. And we did it all in 100-some-odd lines of code.

Today we're going to use _protocol-oriented programming_ to take it even further.

Last time we created our own class, `JSONObject`, to encapsulate a JSON dictionary and to hold our `valueForKey` family of helpers. This encapsulation had several benefits, however, it also came with the baggage of having to constantly wrap our basic `[String:AnyObject]` JSON dictionaries. This can become especially tedious when we have nested JSON objects we want to extract.

This time let's use the power of protocols in Swift to extend `Dictionary`. Of course, we don't want to extend all instances of `Dictionary`, just the instances that look like `[String:AnyObject]`.

## JSON Keys

Although Swift allows us to restrict extensions on generic types, it only allows us to restrict the types based on protocols. Thus, we need a protocol to describe our `Key` type.

{% highlight swift %}

public protocol JSONKeyType: Hashable {
    var stringValue: String { get }
}

extension String: JSONKeyType {
    public var stringValue: String {
        return self
    }
}

{% endhighlight %}

We'll never make anything other than `String` conform to `JSONKeyType`. It's a little bit of a work around, but it will allow us do this:

{% highlight swift %}

extension Dictionary where Key: JSONKeyType {
    // functions available only on instances where Key is a String
}

{% endhighlight %}

## JSON Values

We defined a protocol for our key type, so let's define a protocol for our value type.

{% highlight swift %}

public protocol JSONValueType {
    typealias _JSONValue = Self

    static func JSONValue(object: Any) throws -> _JSONValue
}

{% endhighlight %}

In order for a type to conform to `JSONValueType`, it must implement a function that constructs itself given an `Any`, or throw an error if it can't. The `typealias` is there to help the compiler out with some of the things we'll do later.

Many of the types we'll be dealing with are directly bridgeable from their `Any` counterpart. We can extend the `JSONValueType` protocol to provide a default implementation for these cases, then simply add conformance to said types.

{% highlight swift %}

extension JSONValueType {
    public static func JSONValue(object: Any) throws -> _JSONValue {
        guard let objectValue = object as? _JSONValue else {
            throw JSONError.TypeMismatch(expected: JSONValue.self,
                                         actual: object.dynamicType)
        }
        return objectValue
    }
}

extension String: JSONValueType {}
extension Int: JSONValueType {}
extension UInt: JSONValueType {}
extension Float: JSONValueType {}
extension Double: JSONValueType {}
extension Bool: JSONValueType {}

{% endhighlight %}

Et voila!

If there are other types we wish to extract from our JSON, we can add conformance to `JSONValueType` for them as well.

{% highlight swift %}

extension NSURL: JSONValueType {
    public static func JSONValue(object: Any) throws -> NSURL {
        guard let urlString = object as? String,
              objectValue = NSURL(string: urlString) else {
            throw JSONError.TypeMismatch(expected: self,
                                         actual: object.dynamicType)
        }
        return objectValue
    }
}

{% endhighlight %}

This extension allows us to extract `NSURL`s when the `Any` is a valid URL string. We can write similar extension for `NSDate`, and even `Array` and `Dictionary`. (You can find implementations for the last two in the [gist](https://gist.github.com/jarsen/0f0919d3f43b2a2268e6). I have not included an `NSDate` implementation because of the plethora of date-time representations, but writing your own for your specific should be straightforward.)

## Extending Dictionary

Now to the heart of the matter: actually extending `Dictionary`.

{% highlight swift %}

extension Dictionary where Key: JSONKeyType {
    private func anyForKey(key: Key) throws -> Any {
        let pathComponents = key.stringValue.characters
                                .split(".").map(String.init)
        var accumulator: Any = self

        for component in pathComponents {
            if let componentData = accumulator as? [Key: Value],
                   value = componentData[component as! Key] {
                accumulator = value
                continue
            }

            throw JSONError.KeyNotFound(key: key)
        }

        return accumulator
    }

    public func JSONValueForKey<A: JSONValueType>(key: Key) throws -> A {
        let any = try anyForKey(key)
        guard let result = try A.JSONValue(any) as? A else {
            throw JSONError.TypeMismatchWithKey(key: key,
                                                expected: A.self,
                                                actual: any.dynamicType)
        }

        if let _ = result as? NSNull {
            throw JSONError.NullValue(key: key)
        }

        return result
    }

    public func JSONValueForKey<A: JSONValueType>(key: Key) throws -> [A] {
        let any = try anyForKey(key)
        return try Array<A>.JSONValue(any)
    }

    public func JSONValueForKey<A: JSONValueType>(key: Key) throws -> A? {
        do {
            return try self.JSONValueForKey(key) as A
        }
        catch JSONError.KeyNotFound {
            return nil
        }
        catch JSONError.NullValue {
            return nil
        }
        catch {
            throw error
        }
    }
}

{% endhighlight %}

This should look mostly familiar if you followed along with [Part 2](http://jasonlarsen.me/2015/06/23/no-magic-json-pt2.html). What's new is us restricting `A` to conform to `JSONValueType`. Now it is impossible for us to try and extract a value from our JSON and shove it into a non-JSONValueType conforming box. For example:

{% highlight swift %}

// the old way let this compile
let firstName: UILabel = json.valueForKey("first_name")

{% endhighlight %}

With our code in Part 2, this code would compile, even though we would likely figure it out down the road when we tried to use `firstName`. The fact that `A` must be a `JSONValueType` gives us a little bit more safety here.

Some more changes that have been made are to consolidate the public API to just one function signature: `JSONValueForKey`. The compiler will figure out which version of the function to call based off of the type it is being assigned to—be it a plain ol' `A`, optional `A?`, or even an array of `[A]`! (The latter requires `Array` to conform to `JSONValueType`. You can find this in the [gist](https://gist.github.com/jarsen/0f0919d3f43b2a2268e6)).

Finally, use the power of `typealias` to make it more clear what we're talking about when dealing with our JSON dictionaries.

{% highlight swift %}

public typealias JSONObject = [String: AnyObject]

{% endhighlight %}

And now your JSON code can look like this:

{% highlight swift %}

var json: JSONObject = ["url": "http://apple.com", "foo": (2 as NSNumber), "str": "Hello, World!", "array": [1,2,3,4,7], "object": ["foo": (3 as NSNumber), "str": "Hello, World!"], "bool": (true as NSNumber)]
do {
    var str: String = try json.JSONValueForKey("str")
    var foo2: Int = try json.JSONValueForKey("foo")
    var foo3: Int? = try json.JSONValueForKey("foo")
    var foo4: Int? = try json.JSONValueForKey("bar")
    var arr: [Int] = try json.JSONValueForKey("array")
    var obj: JSONObject? = try json.JSONValueForKey("object")
    let innerfoo: Int = try obj!.JSONValueForKey("foo")
    let innerfoo2: Int = try json.JSONValueForKey("object.foo")
    let bool: Bool = try json.JSONValueForKey("bool")
    let url: NSURL = try json.JSONValueForKey("url")
}
catch {
    // ...
}

{% endhighlight %}

You can find the full code in this [gist](https://gist.github.com/jarsen/0f0919d3f43b2a2268e6), and most likely in a more permanent Github repo home soon.

Please share this if you like it and leave me feedback below in the comments or on the twitters [@jarsen](https://twitter.com/jarsen)!

I hope this makes Crusty proud.

A special thanks to a bunch of local Utah iOS developers who have been working on this and giving lots of great ideas, especially: Bart Whiteley, Brian Mullen, BJ Homer, Derrick Hathaway, Dave DeLong, Mark Schultz and Tim Shadel.
