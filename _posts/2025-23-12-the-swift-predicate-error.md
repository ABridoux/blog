---
layout: post
title:  The Swift Predicate Error
date:  2025-12-23 12:00:00 +0200
categories: [Swift, API]
tags: [data, SwiftData, CoreData]
description: Why the Predicate macro is a dead end for SwiftData, and why I developed SafeFetching.
---
## Overview
A few months ago, [I talked at a CocoaHeads event](https://www.youtube.com/watch?v=KO0pRr22CKc) about my concerns regarding the `#Predicate` macro. I explained why I believe it’s a poor decision for SwiftData, and how it reassured in the development and maintenance of [SafeFetching](https://github.com/ABridoux/SafeFetching). Since the points I exposed are still true as of today, here is the textual version of the talk with additional details.

{% include embed/youtube.html id='KO0pRr22CKc' %}
> Jump to 15'10'' to see me heroically saving my Mac 😬
{: .prompt-info }

## SafeFetching
A few years ago, I started developing the SafeFetching library. The name doesn't reflect what Swift and other languages call "safe", but I essentially wanted to avoid runtime errors with `NSPredicate` in CoreData. Too often had I written typos when initializing a `NSPredicate` and I wanted the compiler to warn me when I was comparing attribute with mismatching types, or when writing "HASPREFIX" rather than "BEGINSWITH".

After several months of iterations, this is what the API looked like, given that `User` is a subclass of `NSManagedObject`, an entity defined in a `xcdatamodel` file.

```swift
User.request()
  .all
  .where(\.score == 10)
  .fetch(in: context)
```
The predicate API lies in the `where(_:)`  function.

This was mostly operators overloading[^operators-overloading], and intensive use of `KeyPath`.  When it came to more advanced operators like "BEGINSWITH", I had to get creative and offer yet another operator.
```swift
User.request()
  .all
  .where(\.name *. .hasPrefix("Ri"))
  .fetch(in: context)
```
It was not perfect, but at least we could use it at KaraFun and be sure we would get a compiler error when miswriting an operator or an attribute name.

*That said, there was a huge caveat.*

In Swift, it's not possible to retrieve a property name from a key path. I had to rely on transforming the key path to `NSExpression` and get the `keyPath` string property from it. It was ok as long as the key path was targeting a property exposed to Objective-C. That's the case with `@NSManaged` annotated properties, but other ones like `isDeleted` would also become available in the predicate... and cause a runtime error.

It seemed at this point that CoreData predicate was inherently prone to runtime errors. And that's where I stopped working on this library, only maintaining what would eventually break.

## SwiftData and the Predicate macro
When SwiftData was initially released with the [`#Predicate`](https://developer.apple.com/documentation/Foundation/Predicate) macro in 2023, I was really excited to see Apple moving forward to bring the safety of Swift to one of my favorite frameworks.
Unfortunately, migrating to new frameworks takes times, and many features from CoreData were missing in SwiftData, so we decided not to use the new framework at KaraFun.

A few months after that I experimented with the `#Predicate` macro to understand how it could be used, especially since it was not available for CoreData. I had the idea to replace SafeFetching by an implementation of `#Predicate` that would work with CoreData, since we were still to use it for several years.

I quickly realized that the `#Predicate` macro was very limited when compared to the `NSPredicate` API, but also that *it could never replace it*, no matter how much effort Apple would put in this new API.

### Comparing NSPredicate and #Predicate
Generally speaking, the `#Predicate` macro feels more modern and safer. Especially with basic operators like an equality comparison.
```swift
let predicate = #Predicate<User> { $0.score == 10.0 }
// 
let predicate = NSPredicate(format: "score == 10.0")
```

As previously mentioned, `NSPredicate` requires to use String, which is very error prone and prevent any type-checking.
A good point for `#Predicate`, now let's try something a bit more advanced: checking a prefix equality.

#### Advanced Operators
This is how it looks like with `NSPredicate`:
```swift
let predicate = NSPredicate(format: #"name BEGINSWITH "Ma""#)
```
> Enclosing a `String` with `#` symbols is a support for [raw text](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0200-raw-string-escaping.md), like quotes `"`.
{: .prompt-info }

Surprisingly, SwiftData doesn't support the [`hasPrefix` ](https://developer.apple.com/documentation/swift/substring/hasprefix(_:)) method on `String` (from `NSString`). The method that is supported is [`starts(with:)`](https://developer.apple.com/documentation/swift/sequence/starts(with:)) from the `Sequence` protocol.
```swift
let predicate = #Predicate<User> { $0.name.starts(with: "Ma") }
```
It's no big deal, at least not as big as the fact that there is no counterpart to the "ENDSWITH" operator available with `NSPredicate`.

#### Valid Swift Code
In Swift, there is currently no methods defined on `Sequence` to check the equality of the end of two sequences. This means that there is as of today no equivalent to this `NSPredicate`.
```swift
let predicate = NSPredicate(format: #"name ENDSWITH "ma""#)
```

You can try to write the following:
```swift
let predicate = #Predicate<User> { $0.name.ends(with: "Ma") } // 🛑 doesn't compile
let predicate = #Predicate<User> { $0.name.hasSuffix("Ma") } // 🛑 doesn't compile
```
You'll get different errors. In both cases the compiler will refuse to build your code.

The rule is really clear here: to write a valid predicate with the `#Predicate` macro, the code inside it has to be valid Swift code, but should also be supported in the macro.
It's kind of logic when you think about it: not every piece of code in Swift can be translated to a SQL predicate. The consequence is that you cannot know at the call site what can be written, or what *should* be written. Should you use the `hasPrefix(:)` or the `starts(with:)` method? You cannot possibly know before trying to compile the code.

And what are the available methods? There is not list in the documentation stating what methods are supported in the `#Predicate` macro, you have to try to compile the code to find out. For instance, there is an equivalent of "CONTAINS[cd]" with `NSPredicate` to check that a String attribute contains another String with case and diacritics insensitive comparison. The `#Predicate` macro supports [`localizedStandardContains(_:)`](https://developer.apple.com/documentation/swift/stringprotocol/localizedstandardcontains(_:)). But if you try to use the case insensitive version only - which exists in Swift as [`localizedCaseInsensitiveCompare(_:)`](https://developer.apple.com/documentation/swift/stringprotocol/localizedcaseinsensitivecompare(_:)) - you'll get a compiler error stating that this function is not supported in the `#Predicate` macro ("CONTAINS[c]" with `NSPredicate`). Why? SQL support both functions.

One might argue that we're still at the very beginning of this new API and that those issues would be fixed in a future release. If that's true, I believe the problem is deeper: there are other SQL functions that have no Swift counterparts, and that *might never get one*.

As a proof, here's a last example, with the normalized search contains operator, denoted with "CONTAINS[n]" in a `NSPredicate`. Normalized equality check compares Unicode String bytes per bytes rather than with the Unicode norm where two characters might for example be encoded in two different ways. There is currently no method on String in Swift to do that, and I don't see how there could possibly be one as it's really related to database logic. Thus I fear the `#Predicate` macro would never support that.
> Normalized search is truly powerful and might speed up search in a CoreData SQLite store by a 10 factor. If you want to learn more about this topic, I recommend the book [CoreData](https://www.objc.io/books/core-data/) from Objc.io
{: .prompt-tip }

###  #Predicate and Complex Types
Ok, so SwiftData has not yet (maybe never?) all the features that Core Data offers. But at least on other topics which are related to Swift, it has to be better, right?
Huh, I don't think so.

I was expecting the `#Predicate` macro to be well-suited to handle more complex types. Especially enums as I find them quite useful a an attribute type. How disappointed I was.

Let's work with an example, a `Plan` enum to specify the user's subscription.
```swift
public enum Plan {
  case freemium
  case premium
}
```
To be able to store this type with SwiftData, all we need to do is to make it conform to `Codable`.  That's the official way to store a struct or a enum.
Now let's try to use that. But I'll cut to the chase: none of the following will compile.
```swift
let predicate = #Predicate<User> { $0.plan == .freemium }
let predicate = #Predicate<User> { $0.plan == Plan.freemium }
```
Trying to reason about it, there is no way the macro can know that a value of type `Plan` should be replaced by its encoded value to be compared to the ones stored in the database. This would also be true for a `RawRepresentable` type for instance as the macro would need to know that the enum case should be replaced with its associated raw value. For that, the macro needs more context, and it's not yet possible in Swift.

And if you try to "outsmart" the compiler like so,
```swift
let plan = Plan.freemium
let predicate = #Predicate<User> { $0.plan == plan }
```
it *does compile* but you'll get a runtime error when executing the request (or query with SwiftUI). A plain, good old-fashioned runtime error.

And just before we wrap up, it's quite simple to store enum values with CoreData. Rather than encoding and decoding them, we'll make it conform to `RawRepresentable` with a String raw type. Then there's a bunch of code to write when declaring such an attribute for the `User` type to get all CoreData tracking and proper type casting.

```swift
class User: NSManagedObject {
  var plan: Plan {
    get {
      willAccessValue(forKey: "plan")
      let rawValue = primitiveValue(forKey: "plan") as! String
      return Plan(rawValue: rawValue) ?? .freemium
    }
    set {
      willChangeValue(forKey: "plan")
      setPrimitiveValue(newValue.rawValue, forKey: "plan")
      didChangeValue(forKey: "plan")
    }
  }
}
```

Of course it's a perfect use case for a macro, and this is what we use at KaraFun.

```swift
class User: NSManagedObject {
  @Primitive(.rawRepresentable) var plan: Plan
}
```
And if you want to actually use that in a predicate, you can. It might crash at runtime if you mistyped something, it doesn't check that the enum test value is valid, but it works.
```swift
let predicate = NSPredicate(format: #"plan == "freemium""#)
```

### Conclusion
After experimenting with the `#Predicate` macro, it feels like a dead end. The macro limitations requiring valid Swift code doesn't match the needs to write predicates for a SQL database. Maybe we'll get Swift macros that can parse something else than Swift code, like Rust macros offers. Meanwhile, I don't see how SwiftData can pretend replacing CoreData one day.

On the other hand, even where SwiftData and the `#Predicate` macro should outclass CoreData with Swift safety, like handling non-scalar types at a higher level, they fail. And I don't see how they can success, as explained with the evaluation of a `Plan` case.

To me, it feels some folks at Apple decided to use Swift macros to experiment with predicates in SwiftData because both were under development. It started to connect, felt cool (and it is for simple predicates), and when they realized it would not work and prevent many features from CoreData to be brought to SwiftData, it was too late. The `#Predicate` macro was already showcased in the WWDC videos.

That's when I decided to give SafeFetching another shot, this time taking a completely different approach. With a macro, yes, but one that keeps CoreData features, and this time without any runtime error... Promise!

## SafeFetching 1.0.0
The new approach with SafeFetching was about working with a specific type, `FetchableMember` rather than relying on key paths. Such `FetchableMember`s each correspond to an attribute or a relationship of the `NSManagedObject` to be used in a predicate.

If we were to work again with a `User` class, here's what it might look like to make it available for fetching with the SafeFetching API.
```swift
class User: NSManagedObject {
  @NSManaged var score: Double
  @NSManaged var name: String
}

extension User: Fetchable {
  static var fetchableMembers: FetchableMembers {
    FetchableMembers()
  }

  struct FetchableMembers {
    let score = FetchableMember<User, Double>(identifier: "score")
    let name = FetchableMember<User, String>(identifier: "name")
  }
}
```
The `score` and `name` properties of `User` each have their `FetchableMember` counterpart in a `FetchableMembers` struct,  which is then provided in the `fetchableMembers` static property. That is how SafeFetching will instantiate the members struct to be used in a predicate.
Of course, writing all of that for each managed object in a codebase would be cumbersome. This new version of the library would not exist without the macros feature in Swift. Something I did not have at hand when I stopped working on SafeFetching.\
Thus, most of the time, only the macro annotation would be required.
```swift
@FetchableManagedObject
final class User: NSManagedObject {

  @NSManaged var score: Double
  @NSManaged var name: String
}
```

### Writing Predicates
When it comes to writing predicate, I hope you will find the syntax very similar to the `#Predicate` one, and quite natural in Swift.
```swift
User.request()
  .all()
  .where { $0.score > 10.0 }
  .fetch(in: context)
```
The predicate part lies in the `where(:)` function again, but this time it takes a closure. The parameter (`$0`) is actually an instance of the `User.FetchableMembers`  struct, the one available through `fetchableMembers`. The `>` operator is still an overload, but on the `FetchableMember` type.

Working with a specific type brings a tremendous amount of operators to SafeFetching. Here are a few.
```swift
.where { $0.name.hasPrefix("Ma") }
.where { $0.name.contains("Ma", options: .normalized) }

.where { $0.score.isIn(10...20) }
// or with an 'import @_spi(SafeFetching) SafeFetching' statement:
.where { (10...20).contains($0.score) }
```
> All available operators are listed in the [documentation](https://abridoux.github.io/SafeFetching/documentation/safefetching/build-predicates).
{: .prompt-tip }

### Benefits
There are several benefits to this approach.

First, writing a predicate in SafeFetching is type-safe, as much as with the `#Predicate` macro. It's not possible to compare unrelated types, or to call a method that doesn't exit on a type.

Then, by working on a specific type like `FetchableMember`, many features from CoreData fetching become available, either by operator overloading, methods defined on the type, or generics trickery - it's even possible to [compare](https://abridoux.github.io/SafeFetching/documentation/safefetching/build-predicates#NSManagedObject) a `NSManagedObjectID` and `NSManagedObject` like `NSPredicate` supports.

Finally, it's the same Swift syntax we're used to. You don't have to guess what is the Swift equivalent to a "BEGINSWITH" operator. It's easy to discover `hasPrefix` and other operators or methods, since everything is constrained to the `FetchableMember` type.

Should you have any doubts whether you can use SafeFetching in your CoreData codebase, or how to adopt it, feel free to [reach out to me](mailto:alexis@woodys-findings.com) or to ask a question in the [repository forums](https://github.com/ABridoux/blog/discussions/new/choose).

## Conclusion
After being disappointed by the `#Predicate` macro and its limitations, and developed a version of SafeFetching that I think is a better alternative, I wonder if Apple would ever circle back and try something else to specify predicates in their next generation framework to manage a local database on their products.

Also, I am considering to maybe port SafeFetching to SwiftData. The `#Predicate` macro convinced me that static checking is not a viable solution for predicates to fetch a SQL database.

## Notes
[^operators-overloading]: Writing predicates with operators overloading is an idea I shamelessly stole from Vapor.