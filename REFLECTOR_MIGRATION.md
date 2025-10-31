# Reflector Migration - Java to Clojure

This document describes the migration of `sci.impl.Reflector.java` to `sci.impl.reflector.cljc`.

## Background

SCI previously used a Java-based reflector (based on `clojure.lang.Reflector`) that was compiled as a separate Maven artifact `borkdude/sci.impl.reflector`. This was necessary to make a few methods public and add support for type hints in the method matching algorithm.

## Changes Made

### 1. Created `src/sci/impl/reflector.cljc`

A new Clojure namespace that implements the minimal set of reflector functions needed by SCI:

- `get-methods` - Delegates to `clojure.lang.Reflector/getMethods`
- `get-static-field` - Delegates to `clojure.lang.Reflector/getStaticField`
- `invoke-constructor` - Delegates to `clojure.lang.Reflector/invokeConstructor`
- `invoke-static-method` - Delegates to `clojure.lang.Reflector/invokeStaticMethod`
- `invoke-matching-method` - Custom implementation supporting type hints via `arg-types` parameter

### 2. FISupport Integration

The FISupport (Functional Interface Support) functionality from `FISupport.java` was ported into the reflector namespace. This enables IFn -> FunctionalInterface adaptation, allowing Clojure functions to be used where Java functional interfaces are expected.

### 3. Updated References

Updated all references to the Java reflector:
- `src/sci/impl/interop.cljc` - Changed to require and use `sci.impl.reflector`
- `src/sci/impl/analyzer.cljc` - Changed to require and use `sci.impl.reflector`
- `src/sci/impl/test.cljc` - Removed unused import

### 4. Removed Dependencies

- Removed `borkdude/sci.impl.reflector` dependency from `deps.edn`
- Removed `borkdude/sci.impl.reflector` dependency from `project.clj`
- Removed the entire `reflector/` subproject directory

### 5. Cleaned up Build Configuration

- Removed `reflector/src-java11` from the `:dev` alias in `deps.edn`

## Method Usage Analysis

The following methods from the original Reflector.java were used by SCI:

| Method | Usage Count | Location | Delegates to clojure.lang.Reflector? |
|--------|-------------|----------|-------------------------------------|
| `getMethods` | 3 | analyzer.cljc (1), interop.cljc (2) | Yes |
| `getStaticField` | 3 | analyzer.cljc (2), interop.cljc (1) | Yes |
| `invokeMatchingMethod` | 3 | analyzer.cljc (1), interop.cljc (2) | No - Custom implementation |
| `invokeStaticMethod` | 1 | analyzer.cljc (1) | Yes |
| `invokeConstructor` | 1 | interop.cljc (1) | Yes |

## Key Differences from Java Implementation

### `invoke-matching-method`

The main custom functionality is `invoke-matching-method`, which:
1. Supports an optional `arg-types` parameter for type hints
2. Includes the complete type matching algorithm from the Java version
3. Handles boxed argument widening (Integer -> Long, Float -> Double)
4. Integrates functional interface adaptation for IFn instances

### Performance Considerations

The Clojure implementation uses the same algorithms as the Java version and delegates to `clojure.lang.Reflector` where possible, maintaining equivalent performance characteristics.

## Testing

All existing tests pass, including:
- Instance method invocation tests
- Static method invocation tests
- Constructor invocation tests
- Static field access tests
- Functional interface adaptation tests (Runnable, Callable, Comparator)

## Future Work

This implementation is JVM-only (`:clj` reader conditional). To support CLR and ClojureDart in the future:
- Add `:clr` and `:cljd` branches to the reader conditionals
- Implement platform-specific reflection logic for each platform
- The functional interface adaptation may need platform-specific handling
