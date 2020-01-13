# Closures As Structs

* Proposal: [SE-NNNN](NNNN-closures-as-structs.md)
* Authors: [Nickolas Pokhylets](https://github.com/nickolas-pohilets), [Matthew Johnson](https://github.com/anandabits)
* Review Manager: TBD
* Status: **Awaiting implementation**

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

As a special case, enable closure literals to be treated as a syntax sugar for instances of anonymous structs implementing a protocol with a single requirement.

Swift-evolution thread: [Equality Of Functions](https://forums.swift.org/t/equality-of-functions/30295)

## Motivation

Swift declares that functions are first-class citizens, but currently they have some limitations compared to other values. In particular closures cannot be compared, which limits algorithms operating on closures as data.

One can think of closures as instances of anonymous local struct types with a single method. And of function type as an existential type wrapping such structs. And some languages (C++) actually implement closures exactly this way.

Everything that can be implemented using closures can be achieved using hand-written structs with a single method. But not everything implementable with a struct, can be implemented with a closure.

In contrast to closures, structs can conform to additional protocols. If protocol conformance can be synthesised by the compiler, struct declaration ends up consisting of the single method body and some boilerplate code.

Callable structs can be handled as generic types, rather then being packed into existential containers. This eliminates need for heap allocation in performance-critical code.

Callable structs can be used to implement generic function type (`<T> (T) -> T`), which currently cannot be expressed for regular closures.

Also, structs can be easily mirrored, while currently Swift does not provide mirroring capabilities for closures, and existing design of closure metadata make some cases of closures not inspectible even by SwiftRemoteMirror.

But implementing closure-like structs by hand is a tedious work, which requires a lot of boilerplate code to capture the context, and introduces a name that is often of dubious value.

Having convenience of the closure syntax and full power of structs do not need to be a a conflicting requirements. This proposal introduces a solution to have both. 

## Proposed solution

When closure literal is type-checked against a generic type variable contstrainted by a single-method protocol, or an existential type for a signle-method protocol, convert closure literal into expression creating an instance of the anonymous struct.

```swift
protocol Predicate: Hashable {
    associatedtype ValueType
    func evaluate(_ x: ValueType) -> Bool
}

func f<P: Predicate>(_ p: Predicate) where P.ValueType == String {
    let value: P.ValueType = ...
    if p.evaluate(value) {
        ...
    }
}

let expected = "abc"

// This:
f { x in x == expected }

// Is converted into:
struct _closure_0001: Predicate {
    let expected: String
    func evaluate(_ x: String) -> Bool {
        return x == expected
    }
}
f(_closure_0001(expected: expected))
```

## Detailed design

### Applicability

Protocol or combination of protocols is considered to be a single-method protocol, if:

* either it contains exactly one non-static function-like requirement - it does not matter if that requirement can be synthesized by the compiler or if it has a default implementation in the extension.
* or there is exactly one non-static function-like requirement, which cannot be synthesized by the compiler and has no default implementation provided in the extension.

If closure literal is type-checked against a protocol which is not a singe-method protocol, compilation error is produced.

The following entities are considered to be a function-like requirement:

* non-mutating method
* read-only property
* read-only subscript

### Capture by reference

Closures as structs are allowed to capture variables only by value. Attempt to capture value by reference should produce a compilation error.

Values captured by reference would need to boxed. Boxing variables affects how compiler synthesises protocol conformance. Automatic boxing may silently produce a behaviour different from what developer expects. If needed, developers still can box captured values manually, being explicit about their intentions. From the other side, if developers want to capture mutable variable by value, they can use capture list to express this intention.

### Mutating methods

Invoking a closure does not change value of the closure. Closures-as-structs are expected to behave like regular closures, so using closure body to implement mutating function requirement is not allowed. But resulting struct may still have  other mutating methods, if they were synthesized by the compiler or provided by the protocol extension.

### Access control

Currently code in nested structs is allowed to access private members of the enclosing scope. So nested structs and closures are consistent in this behaviour, no changes are required.

Properties synthesized for storing captured values have access level `private`.

### Disambiguating `self` and `Self`.

Whithin the body of the closure-as-struct, `self` and `Self` always refers to the `self` and `Self` from the parent context, and never to the anonymous struct itself. `self` from the outer context is captured in the anonymous struct as property named `self`. When writing equivalent struct manually, one would need to escape name of that property with backticks.

If closure-as-struct adopts any methods from protocol extensions, inside protocol extensions code `self` and `Self` work the same way as they would do for any other struct conforming to this protocol - meaning that they refer to the anonymous struct itself.

### Calling closure-as-struct

For the consuming code, closure-as-struct is seen as a regular struct conforming to a protocol. So, to invoke closure body, one need to invoke corresponding method by name, unless that method is `callAsFunction`, as covered by [SE-0253](https://github.com/apple/swift-evolution/blob/master/proposals/0253-callable.md).

### Syntax changes

Proposal does not introduce any changes to the closure syntax.

## Source compatibility

This change is additive. It changes semantics of the expressions what previously would not type-check.

## Effect on ABI stability

No ABI changes. That's a purely syntactic change.

## Effect on API resilience

None

## Alternatives considered

### 1. Add new attributes for function types indicating conformance to common protocols - `@hashable (Int) -> Bool`.

This works only for limited set of common protocols. It still does not allow closures to be used in generic code. It changes ABI of the function types, but the change is additive and backwards-compatible.

### 2. Treat function types as protocol constraints.

This solution introduces a lot of backward-incompatible changes:

* Current ABI of functions is completely different from the ABI of existential containers. Trying to make existential containers for functions fit into current ABI would introduce extra complexity into impementation of the existential containers.
* It changes behavior of the `type(of:)` applied to functions - now it is returning function type, but would be returning underlying anonymous struct type.
* Function types behave like generic types - types of the arguments and result are passed from the outside and produce distinct types. Protocols with associated values behave differently.
* Function types have additional information that cannot be modelled even by protocols with associated types - `throws`, `inout` parameters.
* Function types as protocols would create precendent of a structural protocol - currently all protocols are nominal. 

Also, for function types as protocols it is natural to expect to be able to manually write a struct conforming to that protocol. But this rises questions about requirements for argument labels, which don't have obvious answer.

### 3. Allow capturing mutable variables from the context.

As described above, this may lead to hard-to-notice bugs.
