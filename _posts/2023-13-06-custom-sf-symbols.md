---
layout: post
title: Custom SF Symbols
date: 2023-06-13 10:00:00 +0200
categories: [UI, Design]
tags: [UI, design, SF-symbols]
description: Quick recap about SF Symbols and how to make custom ones.
---

Lastly, we decided at KaraFun to try using only SF Symbols for the icons in the Apple’s platforms applications. The main goal was to keep the iconography of our design system while having the power of SF Symbols. When a symbol was not offered by Apple, we would have to create it. But not always from scratch. Here’s how it went.

## Recap

SF Symbols is an icons library offered by Apple from iOS 13, macOS 10.15, tvOS 13 and watchOS 6. Today, there are more than 4,000 symbols although many of them are variations of one symbol. A symbol will often have a filled version, as well as a circled or squared one. That’s great because when you need the symbol, it’s very likely that one variant will be a match.

SF Symbols are also very easy to retrieve through the UI APIs:

```swift
// UIKit
let image = UIImage(systemName: "envelope")
// AppKit
let image = NSImage(symbolName: "envelope")
// SwiftUI
Image(systemName: "envelope")
```

The [SF Symbols app](https://developer.apple.com/sf-symbols/) offered by Apple is useful to navigate in the symbols, but also preview them in different configurations and to fine-tune custom ones as we’ll see.

One main difference with icons though, is that **SF symbols are typography, not images** (even vectorial ones). They are similar to symbols like ©, % or ☮︎.

Thus, they are configured like text with a point size, a weight, etc… while behaving like a letter with a baseline, a spacing, etc… Consequently, they perfectly match an associated text which is similarly configured. This is a main benefit compared to vectorial images like SVG or PDF. Because the last ones can only grow in scale, it requires a few try and experiment to export a vectorial images matching a piece to a text. And changing the text font would require to also update the icon.

But that’s not all and SF symbols go beyond as it’s also possible to configure the color of a SF Symbol in several ways. How the symbol will be colored depends on its level. It’s possible to have up to three levels of layers that will each have a different color when configured so.

But that’s not all and SF symbols go beyond as it’s also possible to configure the color of a SF Symbol in several ways. How the symbol will be colored depends on its level. It’s possible to have up to three levels of layers that will each have a different color when configured so.

There are currently four possible configurations:
- monochrome which renders a single color for all layers (and no color for the primary layer in fill variations)
- hierarchical which use a color for the primary layer and less prominent variations of the color for the other layers
- palette to give different color to layers
- multicolor to use default colors for layers (like green for a plus symbol to add, red for a minus to delete…)

Finally, since SF Symbols are known from the UI frameworks, they can be configured automatically in several places. This way it’s guaranteed to have the proper rendering for all symbols. It’s the case in a toolbar on macOS for instance or in a tab bar on iOS.

## Custom SF Symbols
All of that is great but quite often you might end up as we did searching for a symbol that does not exist in the official library. So you decide to make a custom symbol. As you might expect with what is said above, it will be more complicated than exporting a SVG or PDF image and import it inside the SF Symbol app.

After taking a look at the video Create custom symbols - WWDC21 - Videos - Apple Developer, I decided we could make custom SF Symbols from icons in our design system at KaraFun.

Under the hood, to make a SF symbols is to provide a collection of variations of a SVG image. Those variations are first categorized in scales: small, medium and large. Each scale have 9 weights variations: ultralight, thin, light, regular, medium, semibold, bold, heavy and black (🥵). So a SF symbol can de described with 3 x 9 = 27 variations.

To provide those 27 variations, Apple offers two templates to work with: static and dynamic.

### Static Template
Working with the static template means providing each 27 variations. That’s a lot and it seems tedious. That said, it does work perfectly. When doing so, I would advise to export a template with a symbol that looks like to the custom symbol you want to produce. This way you could keep the margins in place, and measure metrics like weight of the exported symbols to adapt your own.

But surely, the dynamic approach should be easier then? Not that much.

### Dynamic Template
It’s also possible to export a dynamic template that will require to provide "only" three variants in the small scale: ultralight, regular and black. The OS and the SF Symbols app will take care of generating the remaining 24 variants using interpolation. To get the idea, you can imagine how it’s possible to draw a line which connects two dots to find all dots that this line connects.

Anyway, the good news is that there is less work to do. The bad news is that this work requires much, much more skills. Let’s see why.

The first thing to do is to export a symbol as a dynamic template through the SF Symbols app (select a symbol then File ->Export the model).

Then the exported SVG should be imported in a vector editor software like Sketch, Figma, Adobe Illustrator, or Affinity Designer. I personally use Sketch so I’ll use it to mention the software.

Once imported in Sketch, you can start by adapting your custom symbol to the regular weight. Things like size, stroke width… When you get the result you want, it’s required to make it an outlined form (in Sketch: Layer->Convert to Outline). That’s because SF symbols needs more information than a stroke to colorise and for interpolation to work properly. Full paths are needed. So far so good.

But then you have to use the outlined shape to match the two other variant (ultralight and black). That’s because you absolutely have to preserve all points and paths in the shape for the interpolation to work. If you don’t miss, you should be able to export the modified SVG from Sketch and import it back into the SF Symbols app. Personally, after a few hours of trials that always ended up in the SF Symbols app telling me interpolation was not possible, I decided I did not have the skills neither the tools to make custom symbols this way.

I was almost ready to abandon this technique. But when looking back the WWDC video and talking about it with a friend, I realized that another solution was possible by composing Apple’s provided symbols.

### Composing Symbols
To compose symbols, we’ll import two symbols which have part we are interested in into Sketch. Let’s say we want a rectangle with a gear shape as a badge. We’ll take the rectangle.badge.plus and doc.badge.gearshape and export them as variable templates. Once imported in Sketch, we can duplicate the rectangle.badge.plus template and replace each plus layers by the corresponding gear shape layers copied from doc.badge.gearshape. Once we are satisfied with the results, we can export the SVG and import it back into the SF Symbols app. Then start annotating to have proper rendering mode, variable value and so on.

{% 
    include embed/video.html src='/assets/img/post-resources/custom-sf-symbols/demo.mp4' 
    title="Composing Symbols"
%}

If you do not have a designer in your team that knows how to make symbols from scratch in three different weights, that’s the only method which works I was able to find to make custom symbols in a simple way without buying a font editor app.

> The new Symbol components that allow to easily combine custom symbols with components like background circle, slash… require to use the dynamic template. So that’s something to think about before falling back to the static template.
{: .prompt-info }