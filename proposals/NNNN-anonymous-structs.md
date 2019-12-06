# Anonymous Structs

* Proposal: [SE-NNNN](NNNN-closures-as-structs.md)
* Authors: [Nickolas Pokhylets](https://github.com/nickolas-pohilets), [Matthew Johnson](https://github.com/anandabits)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

This proposal introduces anonymous structs using closure-inspired syntactic sugar as an alternative to a more verbose local struct declaration.  As with closures, trailing syntax is supported.

Initial Swift-evolution discussion thread: [Equality Of Functions](https://forums.swift.org/t/equality-of-functions/30295)

## Motivation

While Swift has often been called a protocol-oriented language it still lacks some features necessary to facilitate protocol-oriented library designs in practice.  One missing feature is syntactic support for ad-hoc, single-use conformances on par with the expressivity that closures provide for ad-hoc, single-use functions.

Local type declarations involve a lot of syntactic ceremony that is unnecessary for singe-use types.  This ceremony includes a name, explicit protocol conformance declarations, and fully written declarations for all members.  In addition to the ceremony of the local type declaration itself, use of the type requires explicit instantiation, which may itself be verbose if there are several stored properties to initialize.

For example, a SwiftUI `View` might want to declare a single-use `Destination` for a `NavigationLink`:

```swift
struct MyView: View {
    var body: some View {
        struct Destination: View {
            var body: some View {
                Text("You've made it to the destination")
            }
        }
        return NavigationLink("Take me there", destination: Destination())
    }
}
```

Notice that in addition to introducing a lot of boilerplate, the local type declaration also breaks the use of `ViewBuilder`, therefore requiring an explicit `return` statement.

With trailing anonymous struct syntax, the above example becomes far more clear and concise:

```swift
struct MyView: View {
    var body: some View {
        NavigationLink("Take me there") {
            Text("You've made it to the destination")
        }
    }
}
```

This syntax gives protocol-oriented designs the same usage-site clarity and convenience that is currently only possible with closure-based designs.

## Proposed solution

We propose to introduce an enhanced closure literal syntax for use in type contexts expecting a constrained generic type or existential.  This syntax will be de-sugared by the compiler to an anonymous struct that provides conformance to the necessary protocols, as well as an instantiation of that type which is used as the value of the literal expression.

### Stored properties

The capture list is used to declare and initialize the stored properties of an anonymous struct.  By default, properties are constant, but the `var` keyword may be used to make a stored property mutable.

```swift
let x = 0
let eq: some Equatable = { [x, var y = x + 1] }

// desugars to:
struct _Anonymous: Equatable { 
    let x: Int 
    var y: Int
}
let eq: some Equatable = _Anonymous(x: x, y: x + 1)
```

Note: Because structs are value types and properties are initialized by copy it is not possible for an anonymous struct to implicitly capture a variable by reference in the way that a closure can.

Attributes, including property wrappers, are also supported:

```swift
let x = 0
let eq: some Equatable = { [x, @Clamping(min: 0, max: 255) var y = x + 1] }
```

Access modifiers are not supported.  An instance of an anonymous type is only ever accessed through witness tables so access modifiers would carry no relevant meaning.

### Bodyless anonymous structs

Astute readers will notice that the example above omits the `in` keyword that is required in all closures which use a capture list.  This keyword separates a closure signature (which includes the capture list) from statements in the body of the closure and is required even in `Void`-returning closures with no statements in the body.

Anonymous structs support a bodyless shorthand when the stored properties themselves, default implementations and compiler-synthesized conformances and members are sufficient to meet the requirements of the type context.  When a body is unnecessary, the `in` separator may be omitted.  There is a subtle but important difference between the omitting the body and the statement-less body we sometimes see in `Void`-returning closures.

For example, given a protocol which only has property requirements the capture list may be sufficient:

```swift
protocol Foo {
    var x: Int { get }
    var y: Int { get set }
}
let x = 0
let eq: some Equatable & Foo = { [x, var = x + 1] }
```

The same is also true when there are requirements with default implementations or which may be satisfied by a synthesized initializer:

```swift
protocol Bar: Foo {
    init(x: Int, y: Int)
    func total() -> Int
}
extension Bar {
    func total() -> Int {
        return x + y
    }
}
let x = 0
let eq: some Equatable & Bar = { [x, var = x + 1] }
```

### Single-requirement bodies

The similarity of anonymous structs with function closures is most apparent in the case where the type context includes a singe function, read-only property or read-only subscript requirement which is not fulfilled by stored properties, synthesized code, or default implementations.  In this case, the closure parameters and body are used to fulfill the single requirement which must be fulfilled explicitly.  The SwiftUI example from the motivation is a good parameter-less example of this:

```swift
NavigationLink("Take me there") {
    Text("You've made it to the destination")
}
```

#### Implicit capture

As with function closures, variables may be captured directly, rather than explicitly in the capture list.  As with capture list these become stored properties initialized by value:

```swift
let destinationName = "The Promised Land"
NavigationLink("Take me there") {
    Text("You've made it to the \(destinationName)")
}
```

In the above example, `destinationName` becomes a stored property of the anonymous struct.  Explicit capture is also supported when a body is present:

```swift
let destinationName = "The Promised Land"
NavigationLink("Take me there") { [destinationName] in
    Text("You've made it to the \(destinationName)")
}
```

#### Parameters

Parameters also work the same as they do with function closures:

```swift
protocol Monoid {
    associatedtype Value
    var empty: Value { get }
    func combine(_ lhs: Value, _ rhs: Value) -> Value
}
extension Sequence {
    func fold<M: Monoid>(_ monoid: M) -> Element
        where M.Value == Element
}
[1, 2, 3].fold { [empty = 0] lhs, rhs in lhs + rhs }
```

The `$` shorthand is also available:

```swift
[1, 2, 3].fold { [empty = 0] in $0 + $1 }
```

#### `mutating` requirements

When the requirement fulfilled by the body is `mutating`, any `var` property captures may be mutated:

```swift
protocol Foo {
    associatedtype Bar
    mutating func update(with value: Bar)
}
func bar<F: Foo>(_ foo: F) { ... }

// The body fulfills the `update` requirement
bar { [var count = 0] (value: Int) in 
    count += value
}
```

#### callAsFunction` requirements

Anonymous structs enable new library designs which use protocols with `callAsFunction` requirements instead of using function types.

```swift
protocol Function {
    associatedtype Input
    associatedtype Output
    func callAsFunction(_ input: Input) -> Output
}
extension Optional {
    // A hypothetical alternative implementation of `map`
    func map<F: Function>(_ function: F) -> F.Output? where F.Input == Wrapped { 
        switch self {
        case .some(let wrapped): return function(wrapped)
        case .none: return nil
        }
    }
}
let x = 42 as Int?
x.map { "\($0)" } 
```

This technique can allow libraries to avoid allocations that would be necessary with closures.  It also supports additional constraints (such as `Equatable` and `Hashable` ) in addition to the primary `Function` requirement.

### Mutable properties and subscripts

When the single requirement that must be explicitly fulfilled is a mutable property or subscript, `get` / `set` blocks may be used directly in the body:

```swift
protocol KeyValueMap {
    associatedtype Key: Hashable
    associatedtype Value

    subscript(key: Key) -> Value { get set }
}
let map: some KeyValueMap = { [var storage = [Int: Int]()] (key: Int) -> Int in
    get { storage[key] }
    set { storage[key] = newVaue }
}
```

### Multi-declaration bodies

In some cases, it may be necessary to explicitly fulfill more than one requirement.  This is possible by using the `in struct` body separator.  When this separator is used, any declaration that is valid in the body of a struct _other than stored property declarations_ will be valid in the body of the anonymous struct:

```swift
protocol Performer {
    associatedtype Input
    func perform(with input: Input)
}
let a = 0
let b = "hello"
let c = true
let d = 42.0

let performer: some Performer = { [a, b, c, d] in struct
    typealias Input = Handler // defined elsewhere
    func perform(with handler: Handler) {
        handler.handle(a, b, c, d)
    }    
}
```

This may be especially useful when it is necessary to explicitly specify associated type bindings.  This syntax eliminates the indirection introduced by the full local struct syntax as well as the need to introduce a name that is often of dubious value:

```swift
let a = 0
let b = "hello"
let c = true
let d = 42.0

struct Arbitrary {
    let a: Int
    let b: String
    let c: Bool
    let d: Double
    typealias Input = Handler // defined elsewhere
    func perform(with handler: Handler) {
        handler.handle(a, b, c, d)
    }    
}
let performer: some Performer = Arbitrary(a: a, b: b, c: c, d: d)
```

Multi-declaration anonymous structs also retain support for the capture list and avoid the need to explicitly initialize an instance, which reduces clarity, especially when there are several stored properties that are explicitly initialized.  The above example requires double the lines of code when a full local struct declaration is written.

### Static requirements and metatype values

Multi-declaration bodies support all declarations that may be provided in a struct, including static declarations.  However, in some cases this syntax may be overkill.  When the type context expects a _metatype_ rather than an instance and there is a single static requirement, the single-requirement body syntax may be used to fulfill the static requirement.  But in this context, a capture list is not allowed as there is no storage available in which to place captured values:

```swift
protocol Worker {
    static func doSomething()
}
func work<W: Worker>(with worker: W.Type) {
    worker.doSomething()
}
work { print("I'm working") }
```

This enables library designs that rely on passing stateless code around using the type system.  Combined with a hypothetical static `callAsFunction` feature, we would have support something similar to `@convention(thin)` functions at the type level.

## Detailed design

### Grammar changes

The grammar changes below are based on [Summary of the Grammar](https://docs.swift.org/swift-book/ReferenceManual/zzSummaryOfTheGrammar.html) from The Swift Programming Language.  In order to accommodate the differences between function closures and anonymous type literals we introduce several new grammar rules that are slightly modified versions of the rules used by ordinary closures.  The `closure-parameter-*` rules are re-used as-is.

```
type-closure-expression → { type-capture-list }
type-closure-expression → { statements? }
type-closure-expression → { type-closure-signature in statements? }
type-closure-expression → { type-closure-signature in getter-setter-block }
type-closure-expression → { type-closure-signature in struct declarations? }
type-closure-signature → type-capture-list? closure-parameter-clause throws? function-result?
type-closure-signature → type-capture-list

closure-parameter-clause → ( ) | ( closure-parameter-list ) | identifier-list
closure-parameter-list → closure-parameter | closure-parameter , closure-parameter-list
closure-parameter → closure-parameter-name type-annotation
closure-parameter-name → identifier

type-capture-list → [ type-capture-list-items ]
type-capture-list-items → type-capture-list-item | type-capture-list-item , type-capture-list-items
type-capture-list-item → attributes? type-capture-specifier? expression
type-capture-specifier → var | weak var? | unowned var? | unowned(safe) var? | unowned(unsafe) var?

```

With the `type-closure-expression` rule in hand, the `primary-expression` rule is updated to include the an additional case:

```
primary-expression → type-closure-expression
```

Trailing syntax is supported by introducing `trailing-type-closure` and adding a new case to `function-call-expression`:

```
trailing-type-closure → type-closure-expression
function-call-expression → postfix-expression function-call-argument-clause? trailing-type-closure
```

Both of these change are analogous to the existing support for function closures.

## Source compatibility

This change is additive, however there may be rare cases involving overload sets where it introduces ambiguity where no ambiguity was previously present.

## Effect on ABI stability

There is no impact on ABI stability

## Effect on API resilience

There is no impact on API resilience

## Alternatives considered

### Use syntax distinct from closure syntax

Some programmers might find it confusing to have closures and anonymous structs share the same syntax.  We could distinguish anonymous structs somehow, perhaps with double-brace syntax:

```swift
[1, 2, 3].fold {{ [empty = 0] in $0 + $1 }}
```

This approach was used in [an early post](https://forums.swift.org/t/equality-of-functions/30295/56?u=anandabits) about the topic.  Subsequent discussion has indicated that this is unnecessary.

## Future enhancements

### Anonymous classes

This proposal focuses exclusively on anonymous structs.  Support for anonymous classes could be added in the future.  This could be introduced using an `in class` body separator, or some other syntax could be introduced to specify a class should be synthesized.  This would allow anonymous type syntax to be used in contexts which include an `AnyObject` constraint.

### Ad-hoc `callAsFunction` constraints

In the section discussing `callAsFunction` requirements, the following example was given:

```swift
extension Optional {
    func map<F: Function>(_ function: F) -> F.Output? where F.Input == Wrapped { ... }
}
```

We could introduce ad-hoc `callAsFunction` requirements which would be a structural constraint using function syntax:

```swift
extension Optional {
    func map<T>(_ function: some (Wrapped) -> T) -> T? { ... }
}
```

This approach (when combined with opaque type syntax) is able to deliver the benefits of unboxed "closures" without a syntactic burden.  In fact, some languages, such as C++, use a similar design for closures rather than the boxed "existential function" approach used by Swift.