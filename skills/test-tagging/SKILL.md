---
name: test-tagging
description: >-
  Tag ScalaTest tests and run or exclude specific subsets (e.g. slow or heavy
  tests) in an sbt project, including defining custom tags and sbt task
  aliases. Triggers on requests like "run only the slow tests", "tag these
  tests", "skip heavy tests in CI", "filter tests by tag",
  "テストにタグを付けて", "特定のタグのテストだけ実行して",
  "重いテストを除外して".
---

# Tagging ScalaTest Tests and Running Selected Subsets

Attach tags to ScalaTest tests so that specific subsets (slow tests, heavy
tests, integration tests, ...) can be included or excluded at run time.

Reference: https://blog.3qe.us/entry/2026/03/01/023443

## Prerequisites

1. The target is an sbt project using ScalaTest. Check
   `libraryDependencies` for `org.scalatest`. If ScalaTest is missing, add
   it:

   ```scala
   libraryDependencies += "org.scalactic" %% "scalactic" % "3.2.19"
   libraryDependencies += "org.scalatest" %% "scalatest" % "3.2.19" % Test
   ```

2. Other frameworks (MUnit, specs2, etc.) have their own tagging
   mechanisms and are out of scope for this skill; say so explicitly.

## Concepts

A tag is identified by its fully qualified name. For each custom tag,
define **both** of the following (convention: `tags` package for Java
annotations, `tagobjects` package for Scala objects — mirroring ScalaTest's
own `org.scalatest.tags` / `org.scalatest.tagobjects`):

- a **Java annotation** — required for tagging a whole suite (class)
- a **Scala tag object** — used for tagging individual tests

Both must share the same fully qualified name: the tag object's constructor
argument is the annotation's fully qualified name.

ScalaTest also ships predefined tags such as `org.scalatest.tagobjects.Slow`
(annotation: `org.scalatest.tags.Slow`); prefer them when they fit.

## Steps

### 1. Define the tag

Java annotation, e.g. `src/test/java/com/example/tags/Heavy.java`
(the file name must match the annotation name):

```java
package com.example.tags;

import java.lang.annotation.*;
import org.scalatest.TagAnnotation;

@TagAnnotation
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.METHOD, ElementType.TYPE })
public @interface Heavy {}
```

Scala tag object, e.g.
`src/test/scala/com/example/tagobjects/Tags.scala`:

```scala
package com.example.tagobjects

import org.scalatest.Tag

object Heavy extends Tag("com.example.tags.Heavy")
```

Note that the tag object lives in `tagobjects` but references the
annotation's fully qualified name in the `tags` package.

### 2. Tag the tests

How tags are attached depends on the test style. For `AnyFunSuite`, pass
tag objects as extra arguments to `test`:

```scala
import org.scalatest.funsuite.AnyFunSuite
import org.scalatest.tagobjects.Slow
import com.example.tagobjects.Heavy

class MySuite extends AnyFunSuite {
  test("heavy test", Heavy, Slow) {
    assert(3 + 3 == 6)
  }
}
```

For spec styles (`AnyFlatSpec` etc.), use `taggedAs`:

```scala
"heavy computation" should "finish" taggedAs (Heavy, Slow) in {
  assert(3 + 3 == 6)
}
```

To tag an entire suite, annotate the class with the Java annotation:

```scala
import org.scalatest.tags.Slow

@Slow
class SlowSuite extends AnyFunSuite {
  test("very slow test") {
    assert(1 + 1 == 2)
  }
}
```

When unsure about a given style, consult the "Tagging tests" section of
that style trait's Scaladoc.

### 3. Run selected tests

Pass ScalaTest runner arguments after `--`, using the tag's fully
qualified name:

```
sbt> testOnly -- -n "com.example.tags.Heavy"     # run only tests WITH the tag
sbt> testOnly -- -l "org.scalatest.tags.Slow"    # run tests WITHOUT the tag
```

- `-n` (include) and `-l` (exclude) can be combined and each accepts a
  space-separated list of tag names in one quoted argument
- Always use the annotation-side fully qualified name (`...tags.Heavy`,
  not `...tagobjects.Heavy`)

### 4. Optional: define sbt task aliases

For frequently used selections, define custom tasks in `build.sbt`:

```scala
lazy val testOnlyHeavy = taskKey[Unit]("Run only the heavy tests")
testOnlyHeavy := (Test / testOnly)
  .toTask(" -- -n com.example.tags.Heavy")
  .value

lazy val testWithoutSlow = taskKey[Unit]("Run tests excluding slow ones")
testWithoutSlow := (Test / testOnly)
  .toTask(" -- -l org.scalatest.tags.Slow")
  .value
```

The leading space inside `toTask(" -- ...")` is required.

### 5. Verify

Run the selection and confirm the executed test count matches the tagged
tests (e.g. `sbt testOnlyHeavy` runs only the `Heavy`-tagged tests, and
`sbt "testOnly -- -l org.scalatest.tags.Slow"` skips the slow ones).

## Caveats

- **Fully qualified names matter**: `-n` / `-l` match on the annotation's
  fully qualified name. A typo silently selects nothing instead of failing
- **Java annotation file name**: the `.java` file name must match the
  annotation name, or compilation fails
- **Suite-level tagging requires the Java annotation**: a Scala tag object
  alone cannot tag a whole class
- **Style-dependent syntax**: `test(name, tags*)` is FunSuite-specific;
  spec styles use `taggedAs`
