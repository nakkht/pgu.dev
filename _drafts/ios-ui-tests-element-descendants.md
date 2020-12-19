---
layout: post
title: "iOS UI testing: element descendants"
author: Paulius Gudonis
---

## The case

To avoid manually making screenshots for each iPhone and iPad device (including different screen sizes) for [onout](https://onout.com) project, I decided to integrate [fastlane snapshot](https://docs.fastlane.tools/actions/snapshot/) early on during development. That way I already setup project for the future: creating localized screenshots for App Store, verifying that UI components look as expected on different screens, are accessible, etc.

After recording test case for [onout](https://onout.com) to display `ColorPickerView` (written in SwiftUI) on iPhone, Xcode generated the following test procedure:

```swift
let app = XCUIApplication()
app.windows.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)	
    .element.children(matching: .other)
    .element.children(matching: .button)
    .matching(identifier: "tile-button").element(boundBy: 0).press(forDuration: 1.6)
app.collectionViews.buttons["Set color"].tap()
```

Seems pretty straightforward, running the test case on iPhone yields expected flow:
![iphone-12-onout-ui-test](/assets/post/iphone-12-onout-ui-test.mov){:.width-40}

Unfortunately, running the same test case on iPad fails. Specifically, on line:
```swift
.matching(identifier: "tile-button").element(boundBy: 0).press(forDuration: 1.6);
```

With an error message:
> Failed to get matching snapshot: No matches found for Children matching type Button from input {(
    Other
)}

Which means the last `.element.children(matching: .other)` does not contain any button element with identifier `tile-button`. Let's try to generate test case on iPad instead and see if there is any difference. The following was generated while recording the same procedure using iPad:

```swift
let app = XCUIApplication()
app.windows.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .other)
    .element.children(matching: .button)
    .matching(identifier: "tile-button").element(boundBy: 0).press(forDuration: 2.2)
app.collectionViews.buttons["Set color"].tap()
```

Beside having different `forDuration` values its nearly identical and running the test case on iPad works flawlessly. But running on iPhone it fails with the same error message. Comparing both code snippets it becomes clear that the amount of `.element.children(matching: .other)` differs. In the first snippet, there are 8 `.element.children(matching: .other)` element queries (depth of 8) whereas in the second snippet there are 9 `.element.children(matching: .other)` element queries (depth of 9). This means that UI hierarchy is generated differently on iPhone and iPad.

## The solution

Immediate idea for solution was to have 2 duplicate test cases: 1 for iPad and 1 for iPhone by using `UIDevice.current.userInterfaceIdiom` to check whether we are running on iPad or iPhone. It solved the issue, however, it felt like a cheap and dirty workaround. There had to be a better and more elegant solution.

Since I was not fully familiar with `XCUIElementQuery` I started searching for more information and looking through documentation. After some time of digging, the answer was right in front of me all along. Looking at [children(matching:)](https://developer.apple.com/documentation/xctest/xcuielementquery/1501006-children) discussion notes, it becomes really clear:

> Use [descendants(matching:)](https://developer.apple.com/documentation/xctest/xcuielementquery/1500659-descendants) if you need to match descendants at any depth beneath the queried element, and not just direct child elements.

Looking at [descendants(matching:)](https://developer.apple.com/documentation/xctest/xcuielementquery/1500659-descendants) documentation, it appears it is possible to simplify even further: from `descendants(matching: .button)` to just using `buttons` property. This results in the following code:

```swift
let app = XCUIApplication()
app.windows.element.buttons["tile-button"].firstMatch.press(forDuration: 2.0)
app.collectionViews.buttons["Set color"].tap()
```

The code snippet not only got significantly simplified, it also works on both iPhones and iPad.
![ipad-onout-ui-test](/assets/post/ipad-onout-ui-test.mov)

## Final thoughts

Initially, my taken approach was pretty naive. Expecting Xcode to generate device agnostic UI test cases by simply recording interaction with the app is not practical. Xcode's generated UI test case code should be used only as a baseline for writing test cases yourself. To fully take advantage of UI testing there are key points:
- Understand `XCUIElementQuery` - it is a really powerful API.
- Understand accessibility API - UI tests are heavily based on finding/interacting with UI elements by using accessibility API, e.g. `accessibilityIdentifier` to find correct elements on the screen.
