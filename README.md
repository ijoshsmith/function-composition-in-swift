# Function Composition in Swift

This exploration of function composition in Swift is also available in an Xcode playground file, which allows you to experiment with the presented code and immediately see the result. The playground can be downloaded from this repository.

Suppose you need to process comma-separated values. You receive some CSV text and need to only keep the rows that contain exactly three values. Here is the CSV text to process.
```
A,B,C
D,E,F
G,H
```
The last row (G,H) is invalid because it has only two values.

The following functions would do the trick.
```swift
import Foundation

func splitLines(ofText text: String) -> [String] {
    return text.components(separatedBy: .newlines)
}

func createRows(fromLines lines: [String]) -> [[String]] {
    return lines.map { line in
        line.components(separatedBy: ",")
    }
}

func removeInvalidRows(fromRows rows: [[String]]) -> [[String]] {
    return rows.filter { row in
        row.count == 3
    }
}
```
Weaving these functions together might look something like this.
```swift
let lines = splitLines(ofText: csv)
let rows = createRows(fromLines: lines)
let validRows = removeInvalidRows(fromRows: rows)
```
It can be tempting to remove temporary variables that store each return value, but that often makes the code harder to understand.
```swift
let huh = removeInvalidRows(fromRows: createRows(fromLines: splitLines(ofText: csv)))
```
The code above is somewhat confusing because the functions appear in the opposite order from which they run. It's like reading a story backwards. It'd be nice to avoid extra variables without adding extra weirdness.

What's a Swift developer to do?
 
Let's take a page from the functional programmer's book and use function composition! Composing two functions yields a new function, which wraps them both up. It's simple. The following code creates a new operator that we can put between two functions and it composes a new function. Ignore the `AdditionPrecedence` bit for now.
```swift
infix operator --> :AdditionPrecedence

func --> <A, B, C> (
    aToB: @escaping (A) -> B,
    bToC: @escaping (B) -> C)
    -> (A) -> C
{
    return { a in
        let b = aToB(a)
        let c = bToC(b)
        return c
    }
}
```
If you're new to Swift this might seem complicated, but it's not, I promise. The operator function name is `-->` and it has three type parameters: `A`, `B`, and `C`. It also has two parameters which happen to be closures (i.e. functions). The first closure turns an `A` into a `B`, and the second closure turns a `B` into a `C`. The `-->` operator returns a new function that transforms an `A` into a `C` and returns it.
 
Now let's see the CSV example from before rewritten to use function composition.
```swift
let processCSV = splitLines(ofText:) --> createRows(fromLines:) --> removeInvalidRows(fromRows:)
let validRows = processCSV(csv)
```
That's pretty nice! The `processCSV` function is the result of composing three functions. The composed functions run in the same order they appear when defining `processCSV`. Note that the `-->` operator is left associative, meaning the functions compose in left-to-right order. This is why I used `AdditionPrecedence` when declaring the operator. It means `-->` has the same precedence and associativity as the standard `+` operator.
 
This all seems fine and dandy, but does it make debugging tricky? How can you inspect a function's return value when using function composition? One approach is to add logging statements to the functions being composed, but that only works if you can edit those functions, which isn't always the case. There's a better way, check it out.
```swift
func --> <A, B> (
    aToB: @escaping (A) -> B,
    sideEffect: @escaping (B) -> Void)
    -> (A) -> B
{
    return { a in
        let b = aToB(a)
        sideEffect(b)
        return b
    }
}
```
This overload of the `-->` operator function gives special treatment to a special case. When the function on the righthand side returns `Void`, it is assumed to exist only to produce a side effect, such as logging to the console or saving data to a file. In the world of functional programming this is considered an "impure" function, because it does not operate only on its inputs, and therefore can have unpredictable side effects. 

Here's how we can use this new version of the `-->` function to perform logging.
```swift
let processCSVWithLogging = splitLines(ofText:)
    --> { print("lines: \($0)") }
    --> createRows(fromLines:)
    --> { print("rows: \($0)") }
    --> removeInvalidRows(fromRows:)
    --> { print("valid rows: \($0)") }

let validRows = processCSVWithLogging(csv)
```
The function composition operator works fine with optional values, since `Optional<T>` is a Swift enum and can be used in a generic function just like any other type. But it would be nice if there was extra support for working with optional values when using function composition, similar to the optional chaining feature in Swift. For example, `account.emergencyContact?.sendAlert()` will only send an alert to an emergency contact if one exists.

Here is a new variant of the function composition operator that supports optional chaining. The function on the left returns an optional value, and the function on the right will only be called if that value is non-nil.
```swift
infix operator -->? :AdditionPrecedence

func -->? <A, B, C> (
    aToB: @escaping (A) -> B?,
    bToC: @escaping (B) -> C?)
    -> (A) -> C?
{
    return { a in
        guard let b = aToB(a) else { return nil }
        let c = bToC(b)
        return c
    }
}
```
Let's see this new `-->?` operator in action…
```swift
import UIKit

func url(forCompany stockSymbol: String) -> URL? {
    let companyMap = ["AAPL":  "http://apple.com",
                      "GOOGL": "http://google.com",
                      "MSFT":  "http://microsoft.com"]
    guard let path = companyMap[stockSymbol] else { return nil }
    return URL(string: path)
}

func data(fromURL url: URL) -> Data? {
    return try? Data(contentsOf: url)
}

func attributedHTML(withData data: Data) -> NSAttributedString? {
    return try? NSAttributedString(data: data,
                                   options: [NSDocumentTypeDocumentAttribute: NSHTMLTextDocumentType],
                                   documentAttributes: nil)
}

let attributedHTMLForCompany = url(forCompany:) -->? data(fromURL:) -->? attributedHTML(withData:)

let html = attributedHTMLForCompany("AAPL")
```
If any of the functions return `nil`, none of the subsequent functions will be called. This is a nice Swifty addition to our  function composition toolbox. Remember, the `-->` and `-->?` operators can be used together.

That's all for this quick excursion into the world of function composition. Perhaps this style of coding feels right to you. If so, give it a try!
