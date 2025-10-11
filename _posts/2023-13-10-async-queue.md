---
layout: post
title: AsyncQueue
date: 2023-10-13 10:00:00 +0200
categories: [Swift, Concurrency]
tags: [Swift, concurrency, data-model]
description: Implementation of an asynchronous queue that can be observed.
---

Thanks to [Naernon](https://mastodon.design/@Naernon@mastodon.social) and [Tom Baranes](https://www.linkedin.com/in/tom-baranes-59263447/) for reviewing this article.

When structured concurrency was released with Swift 5.6, many tried to start using `AsyncSequence`. And many were disappointed - myself included.

## Observation

When it comes to observation especially, asynchronous sequences are quite harder to use than Combine if the latter is available. There are two main benefits using Combine for Observation.

### Easy Stream Creation

With both `PassthroughSubject` and `CurrentValueSubject` or `@Published` it’s quite straightforward to make a property observable or to send a stream of values. Yet only `AsyncChannel` from the open source repository [Swift Async Algorithms](https://github.com/apple/swift-async-algorithms) tries to compensate for that. And it’s still harder to use for the other Combine’s benefit.

### Stream Sharing

Most `AsyncSequence` implementation will not duplicate their values when consumed by several tasks concurrently. So all tasks will receive only some values of the sequence rather than the overall stream. Regarding this topic, it seems that people at Apple working on the concurrency [do not agree](https://github.com/apple/swift-async-algorithms/issues/110) on the best approach.

Both those `AsyncSequence` drawbacks might be resolved with the new `Observable` protocol, although it’s unfortunate it’s only available from iOS 17, macOS 14 and so on.

That said, there are places where `AsyncSequence` really shines and I wanted to share such a case in this article. In the KaraFun application, users can download karaokes for an offline usage. Such download requests can happen from many places: the user wants to download a specific karaoke, or the top 500 songs from the catalog, one of their playlists, the list goes on. Meanwhile, only one karaoke at a time for one user is allowed to be downloaded to prevent overuse of KaraFun’s servers. And downloading should be performed in background. Thus this a good use case for a queue which isolates its state while being observable to download karaokes when the user requests to download them. In this article we’ll implement such an `AsyncQueue` - a lighter version of KaraFun implementation.

> The overall implementation can be found on [this gist](https://gist.github.com/ABridoux/43a4c1dab95f80cd1cdbb0905d628de3).
{: .prompt-info }

## Model
For the queue data model, we will use an array as the queue. It’s not efficient but this article deals with `AsyncSequence` so let’s focus on that. If you are interested in implementing a more optimized queue, there’s a [last section](#a-more-efficient-queue) at the end of this post. Once the data model is ready, it will be isolated with an actor.

```swift
struct Queue<Element> {
  private var elements: [Element] = []
}
```

The queue is generic to be reusable. When it comes to downloading karaokes, the `Element` type could be the id of the karaoke to be downloaded.

Now for the queue methods.

```swift
extension Queue {
  mutating func enqueue(_ element: Element) {
    elements.append(element)
  }

  mutating func dequeue() -> Element? {
    guard !elements.isEmpty else { return nil }
    return elements.removeFirst()
  }
}
```

Since we want to able to flush all downloads easily, we implement a last method.

```swift
extension Queue {
  mutating func removeAll() {
    elements.removeAll()
  }
}
```

We could be happy with this queue and have a shared `DownloadCoordinator` that owns one to manage the downloads. But of course it would not be isolated and would crash if - for instance - the user adds a download from the UI on the main actor while a background task consumes the queue to get the next item to be downloaded.

## Isolating the Queue

Isolating the queue behind an actor is fairly straightforward.

```swift
actor AsyncQueue<Element> {
  private var queue = Queue<Element>()
}
```

Similarly, it’s easy to enqueue an element and remove all of them.

```swift
extension AsyncQueue {
  func enqueue(_ element: Element) {
    queue.enqueue(element)
  }

  func removeAll() {
    queue.removeAll()
  }
}
```

When it comes to the `dequeue()` function, let’s have a break and think about what we have here. This function will either return an `Element` if it has one, or nil if it’s empty. This certainly looks like a `next()` function from the `￼AsyncIteratorProtocol`￼ so it would be interesting to find a way to make an `AsyncSequence` out of the elements in the queue. Let’s try that.

### AsyncIteratorProtocol

We want the `next()` function to return an element if its queue has one, but what if the queue is actually empty? We could return `nil` but it would stop the iteration over the asynchronous sequence. So we should rather await for the next element to be enqueued to return it. It seems to be a good job for a checked continuation! We’ll store an optional `CheckedContinuation<Element, Never>` that will be instantiated if the `next()` function is invoked while the queue is empty, and resumed when a new element is enqueued.

So let’s add that to the `AsyncQueue`.

```swift
private var nextContinuation: CheckedContinuation<Element, Never>?
```
We are now ready to write the `next()` function. We’ll take this opportunity to already conform to `AsyncIteratorProtocol`.

```swift
extension AsyncQueue: AsyncIteratorProtocol {
  func next() async -> Element? {
	// 1
    guard nextContinuation == nil else { return nil }
    
    if let next = queue.dequeue() {
      // 2
      return next
    } else {
      // 3
      return await withCheckedContinuation { nextContinuation = $0 }
    }
}
```

Here are some comments:

1. If the continuation is not `nil`, it means that a task is already awaiting for the next element to be enqueued, so the `next()` function is called from another task. In such a scenario, the easiest choice is to return nil so that the other task ends the iteration immediately.
2. If the queue is not empty, we dequeue the next element.
3. If the queue is empty, we setup the continuation and create a suspension point. Actor Reentrancy can be a bit tricky as explained in [this article](https://swiftrocks.com/how-async-await-works-internally-in-swift) but it should be of no concern in this scenario.

> Concurrency warning for Swift 6 might indicate that the Element type is not Sendable. So feel free to add the Sendable constraint to Element if that’s something you need.
{: .prompt-info }

We’re almost done with the `AsyncIteratorProtocol`. Although there’s a little detail we forgot about. Can you guess what it is? The continuation resuming will never be called! It would be the role of the `enqueue(_:)` function so let’s add that. We have to change it to:

```swift
func enqueue(_ element: Element) {
  if let nextContinuation {
    nextContinuation.resume(returning: element)
    self.nextContinuation = nil
  } else {
     queue.enqueue(element)
   }
}
```

If a continuation exists, it means that a task is already awaiting for the next element, so we resume the continuation with this element instead of enqueuing it. Also it’s required to nullify the continuation once it has been resumed to prevent reusing it (which is an error) and to set the asynchronous queue in a correct state.

### AsyncSequence

Now that the `AsyncIteratorProtocol` is ready, how should we offer an `AsyncSequence`? A simple solution is to implement a small structure that will implement this protocol while keeping a reference to the `AsyncQueue` for the async iterator.

```swift
extension AsyncQueue {
  struct Elements: AsyncSequence {
    
    typealias AsyncIterator = AsyncQueue
    
    private let queue: AsyncQueue

    func makeAsyncIterator() -> AsyncQueue {
      queue
    }
  }
}
```

The last step is then to add a computed property on `AsyncSequence` to instantiate this structure.

```swift
extension AsyncQueue {
  nonisolated var elements: Elements {
    Elements(queue: self)
  }
}
```

The `nonisolated` keyword is used to avoid having to make the call to `AsyncQueue.element` isolated which would require an await keyword. It’s ok to use it here since it’s a computed property that doesn’t mutate the queue.

## Usage

The `AsyncQueue` is now ready to be used! It could look like that.

```swift
let queue = AsyncQueue<Karaoke.Id>()

// from UI
Task {
  await queue.enqueue(karaokeId)
}

// from a background context
Task {
  for await karaokeId in queue.elements {
    await downloadKaraoke(withId: karaokeId)
  }
}
```

Something nice about this queue is that if the user decides to cancel downloads, it’s easy to call `removeAll()`. The task consuming the download ids would simply wait for the next one if the user decides to download other karaokes.

So what do you think? Could it be useful to you? Do you know a better solution? Please let me know on Mastodon or by email.

## A More Efficient Queue

There are several ways to implement a queue but one of my favorite is a double stack. I learnt about it in a Kodeco’s book [Data Structures and Algorithms (chapter 8)](https://www.kodeco.com/books/data-structures-algorithms-in-swift/v4.0/chapters/8-queues#toc-chapter-014-anchor-006). The idea is to have a `O(1)` complexity to enqueue and dequeue elements. The `O(n)` cost will be payed only when the array to enqueue elements will be reversed in the array to dequeue elements. This is called an amortized cost since it’s payed only when needed and not often. It’s similar to the way Swift `Array` do not require a fixed size but rather augment their size and copy all of their elements when they reach their current max size.

But enough of theory, let’s put that in practice!

```swift
struct Queue<Element> {
  private var enqueueArray: [Element] = []
  private var dequeueArray: [Element] = []

  var isEmpty: Bool {
    enqueueArray.isEmpty && dequeueArray.isEmpty
  }
}
```

Now for the queue methods.

```swift
extension Queue {
  mutating func enqueue(_ element: Element) {
    enqueueArray.append(element) // O(1)
  }

  mutating func dequeue() -> Element? {
    if dequeueArray.isEmpty {
      dequeueArray = enqueueArray.reversed() // O(n)
      enqueueArray.removeAll() // O(n)
    }
    return dequeueArray.popLast() // O(1)
  }
}
```

And for the removal.

```swift
extension Queue {
  mutating func removeAll() {
    enqueueArray.removeAll()
    dequeueArray.removeAll()
  }   
}
```