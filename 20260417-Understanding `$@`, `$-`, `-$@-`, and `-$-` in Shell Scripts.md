---
title: "Understanding `$@`, `$*`, `\"$@\"`, and `\"$*\"` in Shell Scripts"
description: "When working with shell scripts, handling command-line arguments correctly is critical. Variables..."
pubDatetime: 2026-04-17T10:20:55.000Z
---

When working with shell scripts, handling command-line arguments correctly is critical. Variables like `$@` and `$*` are commonly used to access all positional parameters—but their behavior changes significantly depending on whether they are quoted.

This subtle distinction is a frequent source of bugs. In this article, we’ll break down exactly how each form behaves, with concrete examples.

---

## Positional Parameters: A Quick Refresher

Consider the following script invocation:

```bash
./script.sh foo bar "baz qux"
```

The positional parameters are:

* `$1` → `foo`
* `$2` → `bar`
* `$3` → `baz qux`

To refer to *all* arguments, we use `$@` or `$*`.

---

## Summary Table

| Expression | Meaning       | Behavior                                   |
| ---------- | ------------- | ------------------------------------------ |
| `$@`       | All arguments | Word-split into separate tokens            |
| `$*`       | All arguments | Word-split into separate tokens            |
| `"$@"`     | All arguments | Preserves each argument as a separate item |
| `"$*"`     | All arguments | Joins all arguments into a single string   |

---

## `$@` and `$*` (Unquoted)

```bash
for arg in $@; do
  echo "$arg"
done
```

```bash
for arg in $*; do
  echo "$arg"
done
```

### Behavior

Without quotes, both `$@` and `$*` behave almost identically:

* They expand to all arguments
* Then undergo **word splitting** based on whitespace

### Problem

```bash
./script.sh "hello world"
```

This will be treated as:

* `hello`
* `world`

The original argument structure is lost, which can lead to incorrect behavior.

👉 In practice, unquoted `$@` and `$*` are rarely appropriate.

---

## `"$@"`: The Safe Default

```bash
for arg in "$@"; do
  echo "$arg"
done
```

### Behavior

* Each argument is preserved as a **distinct element**
* Whitespace inside arguments is retained

### Example

```bash
./script.sh foo "bar baz"
```

Output:

```plaintext
foo
bar baz
```

### Key Properties

* Maintains the original number of arguments
* Preserves boundaries between arguments
* Safe for filenames and arbitrary input

👉 **This is the form you should use in almost all cases.**

---

## `"$*"`: Single String Expansion

```bash
echo "$*"
```

### Behavior

* All arguments are concatenated into **one string**
* Separated by the first character of `IFS` (space by default)

### Example

```bash
./script.sh foo "bar baz"
```

Output:

```plaintext
foo bar baz
```

### Caveats

* Argument boundaries are lost
* Not suitable for iteration or per-argument processing

👉 Useful primarily for logging or display purposes.

---

## Visual Comparison

Given:

```bash
./script.sh A "B C" D
```

| Expression | Result        |
| ---------- | ------------- |
| `$@`       | A B C D       |
| `$*`       | A B C D       |
| `"$@"`     | "A" "B C" "D" |
| `"$*"`     | "A B C D"     |

---

## Common Pitfall

### ❌ Incorrect

```bash
for arg in $@; do
  cp "$arg" /dest/
done
```

If an argument contains spaces, it will be split incorrectly.

---

### ✅ Correct

```bash
for arg in "$@"; do
  cp "$arg" /dest/
done
```

This preserves each argument exactly as passed.

---

## Practical Guidelines

* Use `"$@"` when passing or iterating over arguments
* Avoid unquoted `$@` and `$*` unless you explicitly want word splitting
* Use `"$*"` only when you intentionally need a single combined string

---

## Why This Difference Exists

The distinction arises from how the shell performs:

* **Parameter expansion**
* **Word splitting**
* **Quote removal**

Notably, `"$@"` is a special case: it expands to multiple quoted strings, preserving each positional parameter individually. This behavior is unique and not shared by other expansions.

---

## Conclusion

* `"$@"` is the safest and most predictable way to handle arguments
* `$@` and `$*` (unquoted) can break argument boundaries
* `"$*"` collapses everything into a single string

If you follow one rule, make it this:

> **Always use `"$@"` unless you have a specific reason not to.**
