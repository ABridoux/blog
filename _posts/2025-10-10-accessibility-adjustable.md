---
layout: post
title:  Using Accessibility Adjustable
date:  2025-10-10 10:00:00 +0200
categories: [UI, Customization]
tags: [iOS, accessibility, UIKit]
description: Exploring how to implement the adjustable accessibility trait in uncommon places.
---

A few months ago, when improving Voice Over support for the search in KaraFun iOS, I was pleasantly surprised by the solution Apple Music implemented to change the filtering of the search results. Of course I stole the idea but I wanted to write about it since it might be useful in some other place.

## UIAccessibilityAction

When performing a search in Music with the "Apple Music" section selected (not "Library"), search results are displayed under a list of filters like "Top Results" or "Artists" that allow to display only specific elements. We replicated the same component in KaraFun iOS because it works well in the app too (it would be better if it was provided with the UI frameworks but that's another topic). Thus, when making this component accessible, I looked at how Apple Music handled the filter selection with Voice Over.

It's smart though quite simple as the overall bar of filters is a single accessibility element - it has no children - and has the [`adjustable` accessibility trait](https://developer.apple.com/documentation/uikit/uiaccessibilitytraits/adjustable). This trait can usually be found on controls such as sliders or steppers to change their value with the up and down swipe gestures thus when implementing similar controls, it's something you would want to support. On a slider the value will jump by a certain amount whereas for a stepper it will simply increment or decrement the value. With the Music and KaraFun filters, it will move to the next or previous filter.

## Implementation

It's trivial in both UIKit and SwiftUI as it requires to implement functions to increment (swipe up) and decrement (swipe down) a value. With UIKit it's part of the `UIAccessibilityAction` informal protocol and requires to implement those two functions.

```swift
override func accessibilityIncrement() {
    // select next filter
}

override func accessibilityDecrement() {
    // select previous filter
}
```

Regarding SwiftUI, it's the [`accessibilityAdjustableAction`](https://developer.apple.com/documentation/swiftui/view/accessibilityadjustableaction(_:)) modifier that should be used.

```swift
.accessibilityAdjustableAction { direction in
    switch direction {
    case .increment:
        // select next filter
    case .decrement:
        // select previous filter
    }
}
```

## Usage
With those functions implemented, Voice Over will allow the user to filter the search when the bar filters is selected by announcing it is adjustable and then reacting to the swipe up and down gestures - the gestures for actions - and then the user can navigate to the search results with a swipe to the trailing edge.

So even the adjustable trait is mostly used on sliders and steppers, we saw that it could be useful for other controls offering a single selection. Something to keep in mind!