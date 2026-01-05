# Swift Compiler Bug: Extension on Deeply Nested Generic ~Copyable Type

## Summary

An empty extension on a deeply nested generic `~Copyable` type fails to compile with type resolution errors when declared in the **same file** as the type definition. The bug does NOT occur if the extension is moved to a separate file.

## Reproduction

The bug was discovered in `swift-standards` but could not be minimally reproduced in a standalone package. To reproduce:

1. Clone https://github.com/swift-standards/swift-standards
2. In `Sources/Binary/Binary.Cursor.swift`, find the commented line:
   ```swift
   //extension Binary.Cursor.Set.Reader {}
   ```
3. Uncomment it:
   ```swift
   extension Binary.Cursor.Set.Reader {}
   ```
4. Build:
   ```bash
   swift build --target Binary
   ```

## Error

```
error: 'Mutable' is not a member type of enum 'Binary.Binary'
   public struct Cursor<Storage: Binary.Mutable>: ~Copyable {
                                         `- error: 'Mutable' is not a member type of enum 'Binary.Binary'
```

Note the erroneous `Binary.Binary` - the compiler incorrectly doubles the namespace.

## Key Observations

1. **Same-file specific**: The bug only occurs when the extension is in the same file as the struct definition
2. **Empty extension fails**: Even `extension Binary.Cursor.Set.Reader {}` with no body fails
3. **Sibling types work**: `extension Binary.Cursor.Move.Reader { ... }` compiles fine
4. **Inline works**: Defining methods directly in the struct compiles

## Workarounds

1. **Move extension to separate file** - works but defeats code organization
2. **Inline methods in struct definition** - chosen workaround

## Environment

- Swift 6.2.3 (swiftlang-6.2.3.3.21 clang-1700.6.3.2)
- macOS 26.0 (arm64-apple-macosx26.0)

## Type Structure

```
Binary (enum namespace)
├── Contiguous (protocol with Space, Scalar associated types)
├── Mutable (protocol refining Contiguous)
├── Position<Scalar, Space> (typealias to Coordinate.X<Space>.Value<Scalar>)
├── Cursor<Storage: Binary.Mutable> (~Copyable struct)
│   ├── Move (~Copyable struct)
│   │   ├── Reader (~Copyable struct) ← extension WORKS
│   │   └── Writer (~Copyable struct) ← extension WORKS
│   └── Set (~Copyable struct)
│       ├── Reader (~Copyable struct) ← extension FAILS
│       └── Writer (~Copyable struct) ← extension FAILS
```

## Related Issues

- SR-631 / [#43248](https://github.com/swiftlang/swift/issues/43248) - Different: file-ordering dependent, closed
- [#63866](https://github.com/swiftlang/swift/issues/63866) - Different: about generic argument syntax
