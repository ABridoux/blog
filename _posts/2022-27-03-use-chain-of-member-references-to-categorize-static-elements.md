---
layout: post
title: Use chain of member references to categorize static elements
date: 2022-03-27 10:00:00 +0200
categories: [Swift, Architecture]
tags: [Swift, architecture]
description: An idea to use the new chain of member references syntax to improve the declaration of static properties when working with a large number of elements.
---

## State of the Art

Declaring static properties in Swift is very common because it’s convenient and expressive. For instance, when using the red component on `UIColor`, it’s possible to omit the type:

```swift
view.backgroundColor = .red
```

Meanwhile, before Swift 5.4 was released, this was not possible to take things further with this syntax.
For example, if we wanted to set the alpha component on a static element of `UIColor`, this was not possible:

```swift
view.backgroundColor = .red.withAlphaComponent(0.5) // doesn't compile
```

Since Swift 5.4, thanks to the [proposal SE–0287](https://github.com/apple/swift-evolution/blob/main/proposals/0287-implicit-member-chains.md), the above syntax is now valid which is awesome. Moreover, this syntax led me to a solution that helped better manage a static properties collection, and I wanted to share that here.

## Iterating

Let’s take an example to expose the problem I had. In an application, it’s very common to log or send events to a distant server. So let’s introduce a LogEvent in the app as well as a LogService that will send those events.

```swift
struct LogEvent {
    let category: String
    let name: String
}

enum LogService {

    static func send(event: LogEvent) {
        // here the event is sent
    }
}
```

Now, we might want to avoid exposing the `LogEvent` initializer in our app because we want to restrict the possible events to be sent. To do so, it’s natural in Swift to add those events as static properties in an extension.
If we have a small number of events, it’s fine to declare all of them in an extension. Here we declare events to track when the user signed in and out, when they played a video, and when they jumped to the next one.

```swift
extension LogEvent {
  static let userSignIn = LogEvent(category: "user", name: "sign-in")
  static let userSignOut = LogEvent(category: "user", name: "sign-out")
  static let videoPause = LogEvent(category: "video", name: "pause")
  static let videoNext = LogEvent(category: "video", name: "next")
}
```

It’s great because the rest of the code in our app will have a list of events to choose from and nothing else. This improves discoverability and ensures all events are declared in a single place which simplifies maintenance.

Of course, a programmer’s real life is never that simple and we might end up with dozens of events because the manager wants to know as precisely as possible how the users interacts with our app.


So we might have to add events to know when the user tried to make an in-app purchase, when they tried this new awesome feature that just got released, when they changed a preference in the app… Quickly enough, the number of log events in the extension grows and the discoverability is diminished since there are too much options.
Several solutions already exist to reduce those negative effects though. Like splitting events in different extensions per topic.

```swift
// user
extension LogEvent {
  static let userSignIn = LogEvent(category: "user", name: "sign-in")
  static let userSignOut = LogEvent(category: "user", name: "sign-out")
}

// video
extension LogEvent {
  static let videoPause = LogEvent(category: "video", name: "pause")
  static let videoNext = LogEvent(category: "video", name: "next")
}
```

But it doesn’t improve discoverability. Especially if the caller has to first browse all the user events to reach the video events or if the word identifying the topic is used elsewhere (like `userBoughtVideo`).

We might then be tempted to trade expressivity for discoverability by requiring to use the type again and declare types in an `Events` namespace.

```swift
enum Events {
  enum User {
    static let signIn = LogEvent(category: "user", name: "sign-in")
    static let signOut = LogEvent(category: "user", name: "sign-out")
  }

  enum Video {
    static let pause = LogEvent(category: "video", name: "pause")
    static let next = LogEvent(category: "video", name: "next")
  }
}
```

At the call site, this would give:

```swift
LogService.send(Events.User.signIn)
```

Which is ok but a bit disappointing and less expressive. Anyway that’s the best solution I had found previously to Swift 5.4.
Of course, now is the moment I introduce the new solution I found.

## Using the New Proposal

Using the new feature mentioned in the beginning of this post, we can get best of both solutions: keep the call site expressive, and keep the same discoverability and maintenance level.

Fo each log event topic, we’ll declare a dedicated struct to hold the events and add a static property on `LogEvent` that uses it.

So for the user this gives:

```swift
extension LogEvent {

  struct User {
    let signIn = LogEvent(category: "user", name: "sign-in")
    let signOut = LogEvent(category: "user", name: "sign-out")
  }
}

extension LogEvent {
  public static let user = User()
}
```

And for the video topic:

```swift
extension LogEvent {

  struct Video {
    let pause = LogEvent(category: "video", name: "pause")
    let next = LogEvent(category: "video", name: "next")
  }
}

extension LogEvent {
  public static let video = Video()
}
```

Now here is how the call site looks:

```swift
LogService.send(event: .user.signIn)
LogService.send(event: .video.next)
```

I find this approach to have several pros:
- The caller is not overwhelmed with all the available options. Only the topics are exposed, then the topic’s events. So the discoverability is improved.
- The maintenance remains simple since events are declared in one place (even if split into separate files) and ordered per topic.
- It’s still clear that we are using pre-defined events, even though they are now categorized.

And for the cons I could think of:

- It can be strange for the caller to first access a property that is not of the type `LogEvent` to then call a property of this type.
- The caller has to know a bit more about how events are categorized: is event A in the user events or in the video ones? Although this can be improved by documenting.

