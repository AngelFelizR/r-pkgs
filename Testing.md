# (PART) Testing {-}

# Testing {#tests}



Testing is a vital part of package development.
It ensures that your code does what you want it to do.
Testing, however, adds an additional step to your development workflow.
The goal of this chapter is to show you how to make this task easier and more effective by doing formal automated testing using the testthat package.

The first stage of your testing journey is to become convinced that testing has enough benefits to justify the work.
For some of us, this is easy to accept.
Others must learn the hard way.

Once you've decided to embrace automated testing, it's time to learn some mechanics and figure out where testing fits into your development workflow.

As you and your R packages evolve, you'll start to encounter testing situations where it's fruitful to use techniques that are somewhat specific to testing and differ from what we do below `R/`.

## Why is formal testing worth the trouble?

Up until now, your workflow probably looks like this:

1. Write a function.
1. Load it with `devtools::load_all()`, maybe via Ctrl/Cmd + Shift + L.
1. Experiment with it in the console to see if it works.
1. Rinse and repeat.

While you _are_ testing your code in this workflow, you're only doing it informally.
The problem with this approach is that when you come back to this code in 3 months time to add a new feature, you've probably forgotten some of the informal tests you ran the first time around.
This makes it very easy to break code that used to work. 

Many of us embrace automated testing when we realize we're re-fixing a bug for the second or fifth time.
While writing code or fixing bugs, we might perform some interactive tests to make sure the code we're working on does what we want.
But it's easy to forget all the different use cases you need to check, if you don't have a system for storing and re-running the tests.
This is a common practice among R programmers.
The problem is not that you don't test your code, it's that you don't automate your tests.

In this chapter you'll learn how to transition from informal *ad hoc* testing, done interactively in the console, to automated testing (also known as unit testing).
While turning casual interactive tests into formal tests requires a little more work up front, it pays off in four ways:

* Fewer bugs.
  Because you're explicit about how your code should behave, you will have fewer
  bugs.
  The reason why is a bit like the reason double entry book-keeping works:
  because you describe the behaviour of your code in two places, both in your
  code and in your tests, you are able to check one against the other.
  
  With informal testing, it's tempting to just explore typical and authentic
  usage, similar to writing examples.
  However, when writing formal tests, it's natural to adopt a more adversarial
  mindset and to anticipate how unexpected inputs could break your code.
  
  If you always introduce new tests when you add a new feature or function,
  you'll prevent many bugs from being created in the first place,
  because you will proactively address pesky edge cases.
  Tests also keep you from (re-)breaking one feature, when you're tinkering with
  another.

* Better code structure.
  Code that is well designed tends to be easy to test and you can turn this to
  your advantage.
  If you are struggling to write tests, consider if the problem is
  actually the design of your function(s).
  The process of writing tests is a great way to get free, private, and
  personalized feedback on how well-factored your code is.
  If you integrate testing into your development workflow (versus planning to
  slap tests on "later"), you'll subject yourself to constant pressure to break
  complicated operations into separate functions that work in isolation.
  Functions that are easier to test are usually easier to understand and
  re-combine in new ways.

* Call to action.
  When we start to fix a bug, we first like to convert it into a (failing) test.
  This is wonderfully effective at making your goal very concrete:
  make this test pass.
  This is basically a special case of a general methodology known as test driven
  development.
  
* Robust code.
  If you know that all the major functionality of your package is well covered
  by the tests, you can confidently make big changes without worrying about
  accidentally breaking something.
  This provides a great reality check when you think you've discovered some
  brilliant new way to simplify your package.
  Sometimes such "simplifications" fail to account for some important use case
  and your tests will save you from yourself.

## Introducing testthat

This chapter describes how to test your R package using the testthat package:
<https://testthat.r-lib.org>

If you're familiar with frameworks for unit testing in other languages, you should note that there are some fundamental differences with testthat.
This is because R is, at heart, more a functional programming language than an object-oriented programming language.
For instance, because R's main object-oriented systems (S3 and S4) are based on generic functions (i.e., methods belong to functions not classes), testing approaches built around objects and methods don't make much sense.

testthat 3.0.0 (released 2020-10-31) introduced the idea of an **edition** of testthat, specifically the third edition of testthat, which we refer to as testthat 3e.
An edition is a bundle of behaviors that you have to explicitly choose to use, allowing us to make otherwise backward incompatible changes.
This is particularly important for testthat since it has a very large number of packages that use it (almost 5,000 at last count).
To use testthat 3e, you must have a version of testthat >= 3.0.0 **and** explicitly opt-in to the third edition behaviors.
This allows testthat to continue to evolve and improve without breaking historical packages that are in a rather passive maintenance phase.
You can learn more in the [testthat 3e article](https://testthat.r-lib.org/articles/third-edition.html) and the blog post [Upgrading to testthat edition 3](https://www.tidyverse.org/blog/2022/02/upkeep-testthat-3/).

We recommend testthat 3e for all new packages and we recommend updating existing, actively maintained packages to use testthat 3e.
Unless we say otherwise, this chapter describes testthat 3e.

## Test mechanics and workflow  {#tests-mechanics-workflow}

### Initial setup

To setup your package to use testthat, run:


```r
usethis::use_testthat(3)
```

This will:

1.  Create a `tests/testthat/` directory.

1.  Add testthat to the `Suggests` field in the `DESCRIPTION`.
    Specify testthat 3e in the `Config/testthat/edition` field.
    The affected `DESCRIPTION` fields might look like:
    
        Suggests: testthat (>= 3.0.0)
        Config/testthat/edition: 3

1.  Create a file `tests/testthat.R` that runs all your tests when
    `R CMD check` runs. (You'll learn more about that in 
    [automated checking](#r-cmd-check).)
    The contents of this file will be something like:
    
    
    ```r
    library(testthat)
    library(abcde)
    
    test_check("abcde")
    ```
    
This initial setup is usually something you do once per package.
However, even in a package that already uses testthat, it is safe to run `use_testthat(3)`, when you're ready to opt-in to testthat 3e.

Do not edit `tests/testthat.R`!
It is run during `R CMD check` (and, therefore, `devtools::check()`), but is not used in most other test-running scenarios (such as `devtools::test()` or `devtools::test_active_file()`).
If you want to do something that affects all of your tests, there is almost always a better way than modifying the boilerplate `tests/testthat.R` script.
This chapter details many different ways to make objects and logic available during testing.

### Create a test

As you define functions in your package, in the files below `R/`, you add the corresponding tests to `.R` files in `tests/testthat/`.
We strongly recommend that the organisation of test files match the organisation of `R/` files, discussed in section \@ref(code-organising):
The `foofy()` function (and its friends and helpers) should be defined in `R/foofy.R` and their tests should live in `tests/testthat/test-foofy.R`.

```
R                                     tests/testthat
└── foofy.R                           └── test-foofy.R
    foofy <- function(...) {...}          test_that("foofy does this", {...})
                                          test_that("foofy does that", {...})
```

Even if you have different conventions for file organisation and naming, note that testthat tests **must** live in files below `tests/testthat/` and these file names **must** begin with `test`.
The test file name is displayed in testthat output, which provides helpful context[^bye-bye-context].

[^bye-bye-context]: The legacy function `testthat::context()` is superseded now and its use in new or actively maintained code is discouraged.
In testthat 3e, `context()` is formally deprecated; you should just remove it.
Once you adopt an intentional, synchronized approach to the organisation of files below `R/` and `tests/testthat/`, the necessary contextual information is right there in the file name, rendering the legacy `context()` superfluous.

<!-- Hadley thinks this is too much detail about use_r()/use_test(). I will likely agree when I revisit this later. Leaving it for now. -->

usethis offers a helpful pair of functions for creating or toggling between files:

* `usethis::use_r()`
* `usethis::use_test()`

Either one can be called with a file (base) name, in order to create a file *de novo* and open it for editing:


```r
use_r("foofy")    # creates and opens R/foofy.R
use_test("blarg") # creates and opens tests/testthat/test-blarg.R
```

The `use_r()` / `use_test()` duo has some convenience features that make them "just work" in many common situations:

* When determining the target file, they can deal with the presence or absence
  of the `.R` extension and the `test-` prefix.
  - Equivalent: `use_r("foofy.R")`, `use_r("foofy")`
  - Equivalent: `use_test("test-blarg.R")`, `use_test("blarg.R")`, `use_test("blarg")`
* If the target file already exists, it is opened for editing. Otherwise, the
  target is created and then opened for editing.

:::rstudio-tip
If `R/foofy.R` is the active file in your source editor, you can even call `use_test()` with no arguments!
The target test file can be inferred: if you're editing `R/foofy.R`, you probably want to work on the companion test file, `tests/testthat/test-foofy.R`.
If it doesn't exist yet, it is created and, either way, the test file is opened for editing.
This all works the other way around also.
If you're editing `tests/testthat/test-foofy.R`, a call to `use_r()` (optionally, creates and) opens `R/foofy.R`.
:::

Bottom line: `use_r()` / `use_test()` are handy for initially creating these file pairs and, later, for shifting your attention from one to the other.

When `use_test()` creates a new test file, it inserts an example test:


```r
test_that("multiplication works", {
  expect_equal(2 * 2, 4)
})
```

You will replace this with your own logic, but it's a nice reminder of the basic form:

* A test file holds one or more `test_that()` tests.
* Each test describes what it's testing: e.g. "multiplication works".
* Each test has one or more expectations: e.g. `expect_equal(2 * 2, 4)`.

Below we go into much more detail about how to test your own functions.

### Run tests

Depending on where you are in the development cycle, you'll run your tests at various scales.
When you are rapidly iterating on a function, you might work at the level of individual tests.
As the code settles down, you'll run entire test files and eventually the entire test suite.

**Micro-iteration**: This is the interactive phase where you initiate and refine a function and its tests in tandem.
Here you will run `devtools::load_all()` often, and then execute individual expectations or whole tests interactively in the console.
Note that `load_all()` attaches testthat, so it puts you in the perfect position to test drive your functions and to execute individual tests and expectations.


```r
# tweak the foofy() function and re-load it
devtools::load_all()

# interactively explore and refine expectations and tests
expect_equal(foofy(...), EXPECTED_FOOFY_OUTPUT)

testthat("foofy does good things", {...})
```

**Mezzo-iteration**: As one file's-worth of functions and their associated tests start to shape up, you will want to execute the entire file of associated tests, perhaps with `testthat::test_file()`:

<!-- `devtools::test_file()` exists, but is deprecated, because of the collision.

Consider marking as defunct / removing before the book is published. -->


```r
testthat::test_file("tests/testthat/test-foofy.R")
```

:::rstudio-tip
In RStudio, you have a couple shortcuts for running a single test file.

If the target test file is the active file, you can use the "Run Tests" button in the upper right corner of the source editor.

There is also a useful function, `devtools::test_active_file()`.
It infers the target test file from the active file and, similar to how `use_r()` and `use_test()` work, it works regardless of whether the active file is a test file or a companion `R/*.R` file.
You can invoke this via "Run a test file" in the Addins menu.
However, for heavy users (like us!), we recommend [binding this to a keyboard shortcut](https://support.rstudio.com/hc/en-us/articles/206382178-Customizing-Keyboard-Shortcuts-in-the-RStudio-IDE); we use Ctrl/Cmd + T.
:::

**Macro-iteration**: As you near the completion of a new feature or bug fix, you will want to run the entire test suite.

Most frequently, you'll do this with `devtools::test()`:


```r
devtools::test()
```

Then eventually, as part of `R CMD check` with `devtools::check()`:


```r
devtools::check()
```
:::rstudio-tip
`devtools::test()` is mapped to Ctrl/Cmd + Shift + T.
`devtools::check()` is mapped to  Ctrl/Cmd + Shift + E.
:::

<!-- We'll probably want to replace this example eventually, but it's a decent placeholder.
The test failure is something highly artificial I created very quickly. 
It would be better to use an example that actually makes sense, if someone elects to really read and think about it.-->

The output of `devtools::test()` looks like this:

    devtools::test()
    ℹ Loading usethis
    ℹ Testing usethis
    ✓ | F W S  OK | Context
    ✓ |         1 | addin [0.1s]
    ✓ |         6 | badge [0.5s]
       ...
    ✓ |        27 | github-actions [4.9s]
       ...
    ✓ |        44 | write [0.6s]
    
    ══ Results ═════════════════════════════════════════════════════════════════
    Duration: 31.3 s
    
    ── Skipped tests  ──────────────────────────────────────────────────────────
    • Not on GitHub Actions, Travis, or Appveyor (3)
    
    [ FAIL 1 | WARN 0 | SKIP 3 | PASS 728 ]

Test failure is reported like this:

    Failure (test-release.R:108:3): get_release_data() works if no file found
    res$Version (`actual`) not equal to "0.0.0.9000" (`expected`).
    
    `actual`:   "0.0.0.1234"
    `expected`: "0.0.0.9000"

Each failure gives a description of the test (e.g., "get_release_data() works if no file found"), its location (e.g., "test-release.R:108:3"), and the reason for the failure (e.g., "res$Version (`actual`) not equal to "0.0.0.9000" (`expected`)").

The idea is that you'll modify your code (either the functions defined below `R/` or the tests in `tests/testthat/`) until all tests are passing.

## Test organisation

A test file lives in `tests/testthat/`.
Its name must start with `test`.
We will inspect and execute a test file from the stringr package.

<!-- https://github.com/hadley/r-pkgs/issues/778 -->

But first, for the purposes of rendering this book, we must attach stringr and testthat.
Note that in real-life test-running situations, this is taken care of by your package development tooling:

* During interactive development, `devtools::load_all()` makes testthat and the
  package-under-development available (both its exported and unexported
  functions).
* During arms-length test execution, this is taken care of by
  `devtools::test_active_file()`, `devtools::test()`, and `tests/testthat.R`.
  
**Your test files should not include these `library()` calls.
We also explicitly request testthat edition 3, but in a real package this will be declared in DESCRIPTION.**


```r
library(testthat)
library(stringr)
local_edition(3)
```

<!-- TODO: check if stringr has released and, if so, remove this footnote and edit DESCRIPTION. -->

Here are the contents of `tests/testthat/test-dup.r` from stringr[^dev-stringr]:

[^dev-stringr]: Note that we are building the book against a dev version of stringr.


```r
test_that("basic duplication works", {
  expect_equal(str_dup("a", 3), "aaa")
  expect_equal(str_dup("abc", 2), "abcabc")
  expect_equal(str_dup(c("a", "b"), 2), c("aa", "bb"))
  expect_equal(str_dup(c("a", "b"), c(2, 3)), c("aa", "bbb"))
})
#> [32mTest passed[39m 😸

test_that("0 duplicates equals empty string", {
  expect_equal(str_dup("a", 0), "")
  expect_equal(str_dup(c("a", "b"), 0), rep("", 2))
})
#> [32mTest passed[39m 🥳

test_that("uses tidyverse recycling rules", {
  expect_error(str_dup(1:2, 1:3), class = "vctrs_error_incompatible_size")
})
#> [32mTest passed[39m 😀
```

This file shows a typical mix of tests:

* "basic duplication works" tests typical usage of `str_dup()`.
* "0 duplicates equals empty string" probes a specific edge case.
* "uses tidyverse recycling rules" checks that malformed input results in a
  specific kind of error.

Tests are organised hierarchically:
__expectations__ are grouped into __tests__ which are organised in __files__:

* A __file__ holds multiple related tests.
  In this example, the file `tests/testthat/test-dup.r` has all of the tests
  for the code in `R/dup.r`.

* A __test__ groups together multiple expectations to test the output from a
  simple function, a range of possibilities for a single parameter from a more
  complicated function, or tightly related functionality from across multiple
  functions.
  This is why they are sometimes called __unit__ tests.
  Each test should cover a single unit of functionality.
  A test is created with `test_that(desc, code)`.
  
  It's common to write the description (`desc`) to create something that reads
  naturally, e.g. `test_that("basic duplication works", { ... })`.
  A test failure report includes this description, which is why you want a
  concise statement of the test's purpose, e.g. a specific behaviour.

* An __expectation__ is the atom of testing.
  It describes the expected result of a computation:
  Does it have the right value and right class?
  Does it produce an error when it should?
  An expectation automates visual checking of results in the console.
  Expectations are functions that start with `expect_`.

You want to arrange things such that, when a test fails, you'll know what's wrong and where in your code to look for the problem.
This motivates all our recommendations regarding file organisation, file naming, and the test description.
Finally, try to avoid putting too many expectations in one test - it's better to have more smaller tests than fewer larger tests.

## Expectations

An expectation is the finest level of testing.
It makes a binary assertion about whether or not an object has the properties you expect.
This object is usually the return value from a function in your package.

All expectations have a similar structure:

* They start with `expect_`.

* They have two main arguments:
  the first is the actual result, the second is what you expect.
  
* If the actual and expected results don't agree, testthat throws an error.

* Some expectations have additional arguments that control the finer points of
  comparing an actual and expected result.

While you'll normally put expectations inside tests inside files, you can also run them directly.
This makes it easy to explore expectations interactively.
There are more than 40 expectations in the testthat package, which can be explored in testthat's [reference index](https://testthat.r-lib.org/reference/index.html).
We're only going to cover the most important expectations here.

### Testing for equality

`expect_equal()` checks for equality, with some reasonable amount of numeric tolerance:


```r
expect_equal(10, 10)
expect_equal(10, 10L)
expect_equal(10, 10 + 1e-7)
expect_equal(10, 11)
#> Error: 10 (`actual`) not equal to 11 (`expected`).
#> 
#>   `actual`: [32m10[39m
#> `expected`: [32m11[39m
```

If you want to test for exact equivalence, use `expect_identical()`.


```r
expect_equal(10, 10 + 1e-7)
expect_identical(10, 10 + 1e-7)
#> Error: 10 (`actual`) not identical to 10 + 1e-07 (`expected`).
#> 
#>   `actual`: [32m10.0000000[39m
#> `expected`: [32m10.0000001[39m

expect_equal(2, 2L)
expect_identical(2, 2L)
#> Error: 2 (`actual`) not identical to 2L (`expected`).
#> 
#> `actual` is [32ma double vector[39m (2)
#> `expected` is [32man integer vector[39m (2)
```

### Testing errors

Use `expect_error()` to check whether an expression throws an error.
It's the most important expectation in a trio that also includes `expect_warning()` and `expect_message()`.
We're going to emphasize errors here, but most of this also applies to warnings and messages.

Usually you care about two things when testing an error:

* Does the code fail? Specifically, does it fail for the right reason?
* Does the accompanying message make sense to the human who needs to deal with
  the error?

The entry-level solution is to expect a specific type of condition:


```r
1 / "a"
#> Error in 1/"a": non-numeric argument to binary operator
expect_error(1 / "a") 

log(-1)
#> Warning in log(-1): NaNs produced
#> [1] NaN
expect_warning(log(-1))
```

This is a bit dangerous, though, especially when testing an error.
There are lots of ways for code to fail!
Consider the following test:


```r
expect_error(str_duq(1:2, 1:3))
```

This expectation is intended to test the recycling behaviour of `str_dup()`.
But, due to a typo, it tests behaviour of a non-existent function, `str_duq()`.
The code throws an error and, therefore, the test above passes, but for the *wrong reason*.
Due to the typo, the actual error thrown is about not being able to find the `str_duq()` function: 


```r
str_duq(1:2, 1:3)
#> Error in str_duq(1:2, 1:3): could not find function "str_duq"
```

Historically, the best defense against this was to assert that the condition message matches a certain regular expression, via the second argument, `regexp`.
    

```r
expect_error(1 / "a", "non-numeric argument")
expect_warning(log(-1), "NaNs produced")
```

This does, in fact, force our typo problem to the surface:


```r
expect_error(str_duq(1:2, 1:3), "recycle")
#> Error in str_duq(1:2, 1:3): could not find function "str_duq"
```

Recent developments in both base R and rlang make it increasingly likely that conditions are signaled with a *class*, which provides a better basis for creating precise expectations.
That is exactly what you've already seen in this stringr example.
This is what the `class` argument is for:
    

```r
# fails, error has wrong class
expect_error(str_duq(1:2, 1:3), class = "vctrs_error_incompatible_size")
#> Error in str_duq(1:2, 1:3): could not find function "str_duq"

# passes, error has expected class
expect_error(str_dup(1:2, 1:3), class = "vctrs_error_incompatible_size")
```

<!-- This advice feels somewhat at odds with Hadley's ambivalence about classed errors.
I.e. I think he recommends using a classed condition only when there's a specific reason to.
Then again, maybe the desire to test it is a legitimate reason? -->

If you have the choice, express your expectation in terms of the condition's class, instead of its message.
Often this is under your control, i.e. if your package signals the condition.
If the condition originates from base R or another package, proceed with caution.
This is often a good reminder to re-consider the wisdom of testing a condition that is not fully under your control in the first place.

To check for the *absence* of an error, warning, or message, pass `NA` to the `regexp` argument:
  

```r
expect_error(1 / 2, NA)
```

Of course, this is functionally equivalent to simply executing `1 / 2` inside a test, but some developers find the explicit expectation expressive.

If you genuinely care about the condition's message, testthat 3e's snapshot tests are the best approach, which we describe next.

### Snapshot tests

Sometimes it's difficult or awkward to describe an expected result with code.
Snapshot tests are a great solution to this problem and this is one of the main innovations in testthat 3e.
The basic idea is that you record the expected result in a separate, human-readable file.
Going forward, testthat alerts you when a newly computed result differs from the previously recorded snapshot.
Snapshot tests are particularly suited to monitoring your package's user interface, such as its informational messages and errors.
Other use cases include testing images or other complicated objects.

We'll illustrate snapshot tests using the waldo package.
Under the hood, testthat 3e uses waldo to do the heavy lifting of "actual vs. expected" comparisons, so it's good for you to know a bit about waldo anyway.
One of waldo's main design goals is to present differences in a clear and actionable manner, as opposed to a frustrating declaration that "this differs from that and I know exactly how, but I won't tell you".
Therefore, the formatting of output from `waldo::compare()` is very intentional and is well-suited to a snapshot test.
The binary outcome of `TRUE` (actual == expected) vs. `FALSE` (actual != expected) is fairly easy to check and could get its own test.
Here we're concerned with writing a test to ensure that differences are reported to the user in the intended way.

waldo uses a few different layouts for showing diffs, depending on various conditions.
Here we deliberately constrain the width, in order to trigger a side-by-side layout.[^actual-waldo-test].
(We'll talk more about the withr package below.)

[^actual-waldo-test]: The actual waldo test that inspires this example targets an unexported helper function that produces the desired layout.
But this example uses an exported waldo function for simplicity.


```r
withr::with_options(
  list(width = 20),
  waldo::compare(c("X", letters), c(letters, "X"))
)
#>     old | new    
#> [1] [33m"X"[39m -        
#> [2] [90m"a"[39m | [90m"a"[39m [1]
#> [3] [90m"b"[39m | [90m"b"[39m [2]
#> [4] [90m"c"[39m | [90m"c"[39m [3]
#> 
#>      old | new     
#> [25] [90m"x"[39m | [90m"x"[39m [24]
#> [26] [90m"y"[39m | [90m"y"[39m [25]
#> [27] [90m"z"[39m | [90m"z"[39m [26]
#>          - [34m"X"[39m [27]
```

The two primary inputs differ at two locations:
once at the start and once at the end.
This layout presents both of these, with some surrounding context, which helps the reader orient themselves.

Here's how this would look as a snapshot test:

<!-- Actually using snapshot test technology here is hard.
I can sort of see how it might be done, by looking at the source of testthat's vignette about snapshotting.
For the moment, I'm just faking it. -->


```r
test_that("side-by-side diffs work", {
  withr::local_options(width = 20)
  expect_snapshot(
    waldo::compare(c("X", letters), c(letters, "X"))
  )
})
```

If you execute `expect_snapshot()` or a test containing `expect_snapshot()` interactively, you'll see this:

```
Can't compare snapshot to reference when testing interactively
ℹ Run `devtools::test()` or `testthat::test_file()` to see changes
```

followed by a preview of the snapshot output.

This reminds you that snapshot tests only function when executed non-interactively, i.e. while running an entire test file or the entire test suite.
This applies both to recording snapshots and to checking them.

The first time this test is executed via `devtools::test()` or similar, you'll see something like this (assume the test is in `tests/testthat/test-diff.R`):

```
── Warning (test-diff.R:63:3): side-by-side diffs work ─────────────────────
Adding new snapshot:
Code
  waldo::compare(c(
    "X", letters), c(
    letters, "X"))
Output
      old | new    
  [1] "X" -        
  [2] "a" | "a" [1]
  [3] "b" | "b" [2]
  [4] "c" | "c" [3]
  
       old | new     
  [25] "x" | "x" [24]
  [26] "y" | "y" [25]
  [27] "z" | "z" [26]
           - "X" [27]
```

There is always a warning upon initial snapshot creation.
The snapshot is added to `tests/testthat/_snaps/diff.md`, under the heading "side-by-side diffs work", which comes from the test's description.
The snapshot looks exactly like what a user sees interactively in the console, which is the experience we want to check for.
The snapshot file is *also* very readable, which is pleasant for the package developer.
This readability extends to snapshot changes, i.e. when examining Git diffs and reviewing pull requests on GitHub, which helps you keep tabs on your user interface.
Going forward, as long as your package continues to re-capitulate the expected snapshot, this test will pass.

If you've written a lot of conventional unit tests, you can appreciate how well-suited snapshot tests are for this use case.
If we were forced to inline the expected output in the test file, there would be a great deal of quoting, escaping, and newline management.
Ironically, with conventional expectations, the output you expect your user to see tends to get obscured by a heavy layer of syntactical noise.

What about when a snapshot test fails?
Let's imagine a hypothetical internal change where the default labels switch from "old" and "new" to "OLD" and "NEW".
Here's how this snapshot test would react:

```
── Failure (test-diff.R:63:3): side-by-side diffs work──────────────────────────
Snapshot of code has changed:
old[3:15] vs new[3:15]
  "    \"X\", letters), c("
  "    letters, \"X\"))"
  "Output"
- "      old | new    "
+ "      OLD | NEW    "
  "  [1] \"X\" -        "
  "  [2] \"a\" | \"a\" [1]"
  "  [3] \"b\" | \"b\" [2]"
  "  [4] \"c\" | \"c\" [3]"
  "  "
- "       old | new     "
+ "       OLD | NEW     "
and 3 more ...

* Run `snapshot_accept('diff')` to accept the change
* Run `snapshot_review('diff')` to interactively review the change
```

This diff is presented more effectively in most real-world usage, e.g. in the console, by a Git client, or via a Shiny app (see below).
But even this plain text version highlights the changes quite clearly.
Each of the two loci of change is indicated with a pair of lines marked with `-` and `+`, showing how the snapshot has changed.

You can call `testthat::snapshot_review('diff')` to review changes locally in a Shiny app, which lets you skip or accept individual snapshots.
Or, if all changes are intentional and expected, you can go straight to `testthat::snapshot_accept('diff')`.
Once you've re-synchronized your actual output and the snapshots on file, your tests will pass once again.
In real life, snapshot tests are a great way to stay informed about changes to your package's user interface, due to your own internal changes or due to changes in your dependencies or even R itself.

`expect_snapshot()` has a few arguments worth knowing about:

* `cran = FALSE`: By default, snapshot tests are skipped if it looks like the
  tests are running on CRAN's servers.
  This reflects the typical intent of snapshot tests, which is to proactively
  monitor user interface, but not to check for correctness, which presumably
  is the job of other unit tests which are not skipped.
  In typical usage, a snapshot change is something the developer will want to
  know about, but it does not signal an actual defect.
* `error = FALSE`: By default, snapshot code is *not* allowed to throw an error.
  See `expect_error()`, described above, for one approach to testing errors.
  But sometimes you want to assess "Does this error message make sense to a
  human?" and having it laid out in context in a snapshot is a great way to see
  it with fresh eyes.
  Specify `error = TRUE` in this case:
  
    
    ```r
    expect_snapshot(error = TRUE,
      str_dup(1:2, 1:3)
    )
    ```
  
* `transform`: Sometimes a snapshot contains volatile, insignificant elements,
  such as a temporary filepath or a timestamp.
  The `transform` argument accepts a function, presumably written by you, to
  remove or replace such changeable text.
  Another use of `transform` is to scrub sensitive information from the
  snapshot.
* `variant`: Sometimes snapshots reflect the ambient conditions, such as the
  operating system or the version of R or one of your dependencies, and you need
  a different snapshot for each variant. This is an experimental and somewhat
  advanced feature, so if you can arrange things to use a single snapshot, you
  probably should.
  
In typical usage, testthat will take care of managing the snapshot files below `tests/testthat/_snaps/`.
This happens in the normal course of you running your tests and, perhaps, calling `testthat::snapshot_accept()`.

### Shortcuts for other common patterns

We conclude this section with a few more expectations that come up frequently.
But remember that testthat has [many more pre-built expectations](https://testthat.r-lib.org/reference/index.html) than we can demonstrate here.

Several expectations can be described as "shortcuts", i.e. they streamline a pattern that comes up often enough to deserve its own wrapper.

* `expect_match(object, regexp, ...)` is a shortcut that wraps
  `grepl(pattern = regexp, x = object, ...)`.
  It matches a character vector input against a regular expression `regexp`.
  The optional `all` argument controls whether all elements or just one element
  needs to match.
  Read the `expect_match()` documentation to see how additional arguments, like
  `ignore.case = FALSE` or `fixed = TRUE`, can be passed down to `grepl()`.
   
    
    ```r
    string <- "Testing is fun!"
      
    expect_match(string, "Testing") 
     
    # Fails, match is case-sensitive
    expect_match(string, "testing")
    #> Error: `string` does not match "testing".
    #> Actual value: "Testing is fun!"
      
    # Passes because additional arguments are passed to grepl():
    expect_match(string, "testing", ignore.case = TRUE)
    ```
* `expect_length(object, n)` is a shortcut for
  `expect_equal(length(object), n)`.
* `expect_setequal(x, y)` tests that every element of `x` occurs in `y`, and
  that every element of `y` occurs in `x`.
  But it won't fail if `x` and `y` happen to have their elements in a different
  order.
* `expect_s3_class()` and `expect_s4_class()` check that an object `inherit()`s
  from a specified class. `expect_type()`checks the `typeof()` an object.

    
    ```r
    model <- lm(mpg ~ wt, data = mtcars)
    expect_s3_class(model, "lm")
    expect_s3_class(model, "glm")
    #> Error: `model` inherits from 'lm' not 'glm'.
    ```

`expect_true()` and `expect_false()` are useful catchalls if none of the other expectations does what you need.

## What to test

> Whenever you are tempted to type something into a print statement or a 
> debugger expression, write it as a test instead.
> --- Martin Fowler

There is a fine balance to writing tests.
Each test that you write makes your code less likely to change inadvertently;
but it also can make it harder to change your code on purpose.
It's hard to give good general advice about writing tests, but you might find these points helpful:

* Focus on testing the external interface to your functions - if you test the 
  internal interface, then it's harder to change the implementation in the 
  future because as well as modifying the code, you'll also need to update all 
  the tests.

* Strive to test each behaviour in one and only one test.
  Then if that behaviour later changes you only need to update a single test.

* Avoid testing simple code that you're confident will work.
  Instead focus your time on code that you're not sure about, is fragile, or has
  complicated interdependencies.
  That said, I often find I make the most mistakes when I falsely assume that
  the problem is simple and doesn't need any tests.

* Always write a test when you discover a bug.
  You may find it helpful to adopt the test-first philosophy.
  There you always start by writing the tests, and then write the code that
  makes them pass.
  This reflects an important problem solving strategy:
  start by establishing your success criteria, how you know if you've solved the
  problem.
  
### Test coverage

Another concrete way to direct your test writing efforts is to examine your test coverage.
The covr package (<https://covr.r-lib.org) can be used to determine which lines of your package's source code are (or are not!) executed when the test suite is run.
This is most often presented as a percentage.
Generally speaking, the higher the better.

In some technical sense, 100% test coverage is the goal, however, this is rarely achieved in practice and that's often OK.
Going from 90% or 99% coverage to 100% is not always the best use of your development time and energy.
In many cases, that last 10% or 1% often requires some awkward gymnastics to cover.
Sometimes this forces you to introduce mocking or some other new complexity.
Don't sacrifice the maintainability of your test suite in the name of covering some weird edge case that hasn't yet proven to be a problem.
Also remember that not every line of code or every function is equally likely to harbor bugs.
Focus your testing energy on code that is tricky, based on your expert opinion and any empirical evidence you've accumulated about bug hot spots.

We use covr regularly, in two different ways:

* Local, interactive use.
  We mostly use `devtools::test_coverage_active_file()` and
  `devtools::test_coverage()`, for exploring the coverage of an individual file
  or the whole package, respectively.
* Automatic, remote use via GitHub Actions (GHA).
  We cover continuous integration and GHA more thoroughly elsewhere, but we
  will at least mention here that
  `usethis::use_github_action("test-coverage")` configures a GHA workflow that
  constantly monitors your test coverage.
  Test coverage can be an especially helpful metric when evaluating a pull
  request (either your own or from an external contributor).
  A proposed change that is well-covered by tests is less risky to merge.

## High-level principles for testing

In later sections, we offer concrete strategies for how to handle common testing dilemmas in R.
Here we lay out the high-level principles that underpin these recommendations:

* A test should ideally be self-sufficient and self-contained.
* The interactive workflow is important, because you will mostly interact with
  your tests when they are failing.
* It's more important that test code be obvious than, e.g., as DRY as possible.
* However, the interactive workflow shouldn't "leak" into and undermine the test
  suite.

Writing good tests for a code base often feels more challenging than writing the code in the first place.
This can come as a bit of a shock when you're new to package development and you might be concerned that you're doing it wrong.
Don't worry, you're not!
Testing presents many unique challenges and maneuvers, which tend to get much less air time in programming communities than strategies for writing the "main code", i.e. the stuff below `R/`.
As a result, it requires more deliberate effort to develop your skills and taste around testing.

Many of the packages maintained by our team violate some of the advice you'll find here.
There are (at least) two reasons for that:

* testthat has been evolving for more than twelve years and this chapter
  reflects the cumulative lessons learned from that experience.
  The tests in many packages have been in place for a long time and reflect
  typical practices from different eras and different maintainers.
* These aren't hard and fast rules, but are, rather, guidelines. There will
  always be specific situations where it makes sense to bend the rule.
  
This chapter can't address all possible testing situations, but hopefully these guidelines will help your future decision-making.

### Self-sufficient tests

> All tests should strive to be hermetic:
> a test should contain all of the information necessary to set up, execute, and tear down its environment.
> Tests should assume as little as possible about the outside environment ....
> 
> From the book Software Engineering at Google, [Chapter 11](https://abseil.io/resources/swe-book/html/ch11.html)

Recall this advice found in section \@ref(code-r-landscape), which covers your package's "main code", i.e. everything below `R/`:

> The `.R` files below `R/` should consist almost entirely of function definitions.
> Any other top-level code is suspicious and should be carefully reviewed for possible conversion into a function.

We have analogous advice for your test files:

> The `test-*.R` files below `tests/testthat/` should consist almost entirely of calls to `test_that()`.
> Any other top-level code is suspicious and should be carefully considered for relocation into calls to `test_that()` or to other files that get special treatment inside an R package or from the testthat.

Eliminating (or at least minimizing) top-level code outside of `test_that()` will have the beneficial effect of making your tests more hermetic.
This is basically the testing analogue of the general programming advice that it's wise to avoid unstructured sharing of state.

Logic at the top-level of a test file has an awkward scope:
Objects or functions defined here have what you might call "test file scope", if the definitions appear before the first call to `test_that()`.
If top-level code is interleaved between `test_that()` calls, you can even create "partial test file scope".

While writing tests, it can feel convenient to rely on these file-scoped objects, especially early in the life of a test suite, e.g. when each test file fits on one screen.
But we find that implicitly relying on objects in a test's parent environment tends to make a test suite harder to understand and maintain over time.

Consider a test file with top-level code sprinkled around it, outside of `test_that()`:


```r
dat <- data.frame(x = c("a", "b", "c"), y = c(1, 2, 3))

skip_if(today_is_a_monday())

test_that("foofy() does this", {
  expect_equal(foofy(dat), ...)
})

dat2 <- data.frame(x = c("x", "y", "z"), y = c(4, 5, 6))

skip_on_os("windows")

test_that("foofy2() does that", {
  expect_snapshot(foofy2(dat, dat2)
})
```

We recommend relocating file-scoped logic to either a narrower scope or to a broader scope.
Here's what it would look like to use a narrow scope, i.e. to inline everything inside `test_that()` calls:


```r
test_that("foofy() does this", {
  skip_if(today_is_a_monday())
  
  dat <- data.frame(x = c("a", "b", "c"), y = c(1, 2, 3))
  
  expect_equal(foofy(dat), ...)
})

test_that("foofy() does that", {
  skip_if(today_is_a_monday())
  skip_on_os("windows")
  
  dat <- data.frame(x = c("a", "b", "c"), y = c(1, 2, 3))
  dat2 <- data.frame(x = c("x", "y", "z"), y = c(4, 5, 6))
  
  expect_snapshot(foofy(dat, dat2)
})
```

Below we will discuss techniques for moving file-scoped logic to a broader scope.

### Self-contained tests

Each `test_that()` test has its own execution environment, which makes it somewhat self-contained.
For example, an R object you create inside a test does not exist after the test exits:


```r
exists("thingy")
#> [1] FALSE

test_that("thingy exists", {
  thingy <- "thingy"
  expect_true(exists(thingy))
})
#> [32mTest passed[39m 🎊

exists("thingy")
#> [1] FALSE
```

The `thingy` object lives and dies entirely within the confines of `test_that()`.
However, testthat doesn't know how to cleanup after actions that affect other aspects of the R landscape:

* The filesystem: creating and deleting files, changing the working directory,
  etc.
* The search path: `library()`, `attach()`.
* Global options, like `options()` and `par()`, and environment variables.

Watch how calls like `library()`, `options()`, and `Sys.setenv()` have a persistent effect *after* a test, even when they are executed inside `test_that()`:


```r
grep("jsonlite", search(), value = TRUE)
#> character(0)
getOption("opt_whatever")
#> NULL
Sys.getenv("envvar_whatever")
#> [1] ""

test_that("landscape changes leak outside the test", {
  library(jsonlite)
  options(opt_whatever = "whatever")
  Sys.setenv(envvar_whatever = "whatever")
  
  expect_match(search(), "jsonlite", all = FALSE)
  expect_equal(getOption("opt_whatever"), "whatever")
  expect_equal(Sys.getenv("envvar_whatever"), "whatever")
})
#> [32mTest passed[39m 🥳

grep("jsonlite", search(), value = TRUE)
#> [1] "package:jsonlite"
getOption("opt_whatever")
#> [1] "whatever"
Sys.getenv("envvar_whatever")
#> [1] "whatever"
```

These changes to the landscape even persist beyond the current test file, i.e. they carry over into all subsequent test files.

If it's easy to avoid making such changes in your test code, that is the best strategy!
But if it's unavoidable, then you have to make sure that you clean up after yourself.
This mindset is very similar to one we advocated for in section \@ref(code-r-landscape), when discussing how to design well-mannered functions.



We like to use the withr package (<https://withr.r-lib.org>) to make temporary changes in global state, because it automatically captures the initial state and arranges the eventual restoration.
You've already seen an example of its usage, when we explored snapshot tests:


```r
test_that("side-by-side diffs work", {
  withr::local_options(width = 20)             # <-- (°_°) look here!
  expect_snapshot(
    waldo::compare(c("X", letters), c(letters, "X"))
  )
})
```

This test requires the display width to be set at 20 columns, which is considerably less than the default width.
`withr::local_options(width = 20)` sets the `width` option to 20 and, at the end of the test, restores the option to its original value.
withr is also pleasant to use during interactive development:
deferred actions are still captured on the global environment and can be executed explicitly via `withr::deferred_run()` or implicitly by restarting R.

We recommend including withr in `Suggests`, if you're only going to use it in your tests, or in `Imports`, if you also use it below `R/`.
Call withr functions as we do above, e.g. like `withr::local_whatever()`, in either case.
See section \@ref(suggested-packages-and-tests) for a full discussion.

::: tip
The easiest way to add a package to DESCRIPTION is with, e.g., `usethis::use_package("withr", type = "Suggests")`.
For tidyverse packages, withr is considered a "free dependency", i.e. the tidyverse uses withr so 
extensively that we don't hesitate to use it whenever it would be useful.
:::

withr has a large set of pre-implemented `local_*()` / `with_*()` functions that should handle most of your testing needs, so check there before you write your own.
If nothing exists that meets your need, `withr::defer()` is the general way to schedule some action at the end of a test.[^on-exit]

[^on-exit]: Base R's `on.exit()` is another alternative, but it requires more from you.
You need to capture the original state and write the restoration code yourself.
Also remember to do `on.exit(..., add = TRUE)` if there's *any* chance a second `on.exit()` call could be added in the test.
You probably also want to default to `after = FALSE`.

Here's how we would fix the problems in the previous example using withr:
*Behind the scenes, we reversed the landscape changes, so we can try this again.*


```r
grep("jsonlite", search(), value = TRUE)
#> character(0)
getOption("opt_whatever")
#> NULL
Sys.getenv("envvar_whatever")
#> [1] ""

test_that("withr makes landscape changes local to a test", {
  withr::local_package("jsonlite")
  withr::local_options(opt_whatever = "whatever")
  withr::local_envvar(envvar_whatever = "whatever")
  
  expect_match(search(), "jsonlite", all = FALSE)
  expect_equal(getOption("opt_whatever"), "whatever")
  expect_equal(Sys.getenv("envvar_whatever"), "whatever")
})
#> [32mTest passed[39m 🎊

grep("jsonlite", search(), value = TRUE)
#> character(0)
getOption("opt_whatever")
#> NULL
Sys.getenv("envvar_whatever")
#> [1] ""
```

testthat leans heavily on withr to make test execution environments as reproducible and self-contained as possible.
In testthat 3e, `testthat::local_reproducible_output()` is implicitly part of each `test_that()` test.


```r
test_that("something specific happens", {
  local_reproducible_output()     # <-- this happens implicitly
  
  # your test code, which might be sensitive to ambient conditions, such as
  # display width or the number of supported colors
})
```

`local_reproducible_output()` temporarily sets various options and environment variables to values favorable for testing, e.g. it suppresses colored output, turns off fancy quotes, sets the console width, and sets `LC_COLLATE = "C"`.
Usually, you can just passively enjoy the benefits of `local_reproducible_output()`.
But you may want to call it explicitly when replicating test results interactively or if you want to override the default settings in a specific test.

### Plan for test failure

We regret to inform you that most of the quality time you spend with your tests will be when they are inexplicably failing.

> In its purest form, automating testing consists of three activities: writing tests, running tests, and **reacting to test failures**....
> 
> Remember that tests are often revisited only when something breaks.
> When you are called to fix a broken test that you have never seen before, you will be thankful someone took the time to make it easy to understand.
> Code is read far more than it is written, so make sure you write the test you’d like to read!
> 
> From the book Software Engineering at Google, [Chapter 11](https://abseil.io/resources/swe-book/html/ch11.html)

Most of us don't work on a code base the size of Google.
But even in a team of one, tests that you wrote six months ago might as well have been written by someone else.
Especially when they are failing.

When we do reverse dependency checks, often involving hundreds or thousands of CRAN packages, we have to inspect test failures to determine if changes in our packages are to blame.
As a result, we regularly engage with failing tests in other people's packages, which leaves us with lots of opinions about practices that create unnecessary testing pain.

Test troubleshooting nirvana looks like this:
In a fresh R session, you can do `devtools::load_all()` and immediately run an individual test or walk through it line-by-line.
There is no need to hunt around for setup code that has to be run manually first, that is found elsewhere in the test file or perhaps in a different file altogether.
Test-related code that lives in an unconventional location causes extra self-inflicted pain when you least need it.

Consider this extreme and abstract example of a test that is difficult to troubleshoot due to implicit dependencies on free-range code:


```r
# dozens or hundreds of lines of top-level code, interspersed with other tests,
# which you must read and selectively execute

test_that("f() works", {
  x <- function_from_some_dependency(object_with_unknown_origin)
  expect_equal(f(x), 2.5)
})
```

This test is much easier to drop in on if dependencies are invoked in the normal way, i.e. via `::`, and test objects are created inline:


```r
# dozens or hundreds of lines of self-sufficient, self-contained tests,
# all of which you can safely ignore!

test_that("f() works", {
  useful_thing <- ...
  x <- somePkg::someFunction(useful_thing)
  expect_equal(f(x), 2.5)
})
```

This test is self-sufficient.
The code inside `{ ... }` explicitly creates any necessary objects or conditions and makes explicit calls to any helper functions.
This test doesn't rely on objects or dependencies that happen to be be ambiently available.

Self-sufficient, self-contained tests are a win-win:
It is literally safer to design tests this way and it also makes tests much easier for humans to troubleshoot later.

### Repetition is OK

One obvious consequence of our suggestion to minimize code with "file scope" is that your tests will probably have some repetition.
And that's OK!
We're going to make the controversial recommendation that you tolerate a fair amount of duplication in test code, i.e. you can relax some of your DRY ("don't repeat yourself") tendencies.

> Keep the reader in your test function.
> Good production code is well-factored; good test code is obvious.
> ... think about what will make the problem obvious when a test fails.
>
> From the blog post [Why Good Developers Write Bad Unit Tests](https://mtlynch.io/good-developers-bad-tests/)

Here's a toy example to make things concrete.


```r
test_that("multiplication works", {
  useful_thing <- 3
  expect_equal(2 * useful_thing, 6)
})
#> [32mTest passed[39m 🌈

test_that("subtraction works", {
  useful_thing <- 3
  expect_equal(5 - useful_thing, 2)
})
#> [32mTest passed[39m 🎊
```

In real life, `useful_thing` is usually a more complicated object that somehow feels burdensome to instantiate.
Notice how `useful_thing <- 3` appears in more than once place.
Conventional wisdom says we should DRY this code out.
It's tempting to just move `useful_thing`'s definition outside of the tests:


```r
useful_thing <- 3

test_that("multiplication works", {
  expect_equal(2 * useful_thing, 6)
})
#> [32mTest passed[39m 😸

test_that("subtraction works", {
  expect_equal(5 - useful_thing, 2)
})
#> [32mTest passed[39m 🥳
```

But we really do think the first form, with the repetition, if often the better choice.

At this point, many readers might be thinking "but the code I might have to repeat is much longer than 1 line!".
Below we describe the use of test fixtures.
This can often reduce complicated situations back to something that resembles this simple example.

### Remove tension between interactive and automated testing

Your test code will be executed in two different settings:

* Interactive test development and maintenance, which includes tasks like:
  - Initial test creation
  - Modifying tests to adapt to change
  - Debugging test failure
* Automated test runs, which is accomplished with functions such as:
  - Single file: `devtools::test_active_file()`, `testthat::test_file()`
  - Whole package: `devtools::test()`, `devtools::check()`
  
Automated testing of your whole package is what takes priority.
This is ultimately the whole point of your tests.
However, the interactive experience is clearly important for the humans doing this work.
Therefore it's important to find a pleasant workflow, but also to ensure that you don't rig anything for interactive convenience that actually compromises the health of the test suite.

These two modes of test-running should not be in conflict with each other.
If you perceive tension between these two modes, this can indicate that you're not taking full advantage of some of testthat's features and the way it's designed to work with `devtools::load_all()`.

When working on your tests, use `load_all()`, just like you do when working below `R/`.
By default, `load_all()` does all of these things:

* Simulates re-building, re-installing, and re-loading your package.
* Makes everything in your package's namespace available, including unexported
  functions and objects and anything you've imported from another package.
* Attaches testthat, i.e. does `library(testthat)`.
* Runs test helper files, i.e. executes `test/testthat/helper.R` (more on that
  below).

This eliminates the need for any `library()` calls below `tests/testthat/`, for the vast majority of R packages.
Any instance of `library(testthat)` is clearly no longer necessary.
Likewise, any instance of attaching one of your dependencies via `library(somePkg)` is unnecessary.
In your tests, if you need to call functions from somePkg, do it just as you do below `R/`.
If you have imported the function into your namespace, use `fun()`.
If you have not, use `somePkg::fun()`.
It's fair to say that `library(somePkg)` in the tests should be about as rare as taking a dependency via `Depends`, i.e. there is almost always a better alternative.

Unnecessary calls to `library(somePkg)` in test files have a real downside, because they actually change the R landscape.
`library()` alters the search path.
This means the circumstances under which you are testing may not necessarily reflect the circumstances under which your package will be used.
This makes it easier to create subtle test bugs, which you will have to unravel in the future.

One other function that should almost never appear below `tests/testhat/` is `source()`.
There are several special files with an official role in testthat workflows (see below), not to mention the entire R package machinery, that provide better ways to make functions, objects, and other logic available in your tests.

## Files relevant to testing  {#tests-files-overview}

Here we review which package files are especially relevant to testing and, more generally, best practices for interacting with the file system from your tests.

### Hiding in plain sight: files below `R/`

The most important functions you'll need to access from your tests are clearly those in your package!
Here we're talking about everything that's defined below `R/`.
The functions and other objects defined by your package are always available when testing, regardless of whether they are exported or not.
For interactive work, `devtools::load_all()` takes care of this.
During automated testing, this is taken care of internally by testthat.

This implies that test helpers can absolutely be defined below `R/` and used freely in your tests.
It might make sense to gather such helpers in a clearly marked file, such as one of these:

```
.                              
├── ...
└── R
    ├── ...
    ├── test-helpers.R
    ├── test-utils.R
    ├── utils-testing.R
    └── ...
```

### `tests/testhat.R`

Recall the initial testthat setup described in section \@ref(tests-mechanics-workflow):
The standard `tests/testhat.R` file looks like this:


```r
library(testthat)
library(abcde)

test_check("abcde")
```
    
We repeat the advice to not edit `tests/testthat.R`.
It is run during `R CMD check` (and, therefore, `devtools::check()`), but is not used in most other test-running scenarios (such as `devtools::test()` or `devtools::test_active_file()` or during interactive development).
Do not attach your dependencies here with `library()`.
Call them in your tests in the same manner as you do below `R/`.

### Testthat helper files

Another type of file that is always executed by `load_all()` and at the beginning of automated testing is a helper file, defined as any file below `tests/testthat/` that begins with `helper`.
Helper files are a mighty weapon in the battle to eliminate code floating around at the top-level of test files.
Helper files are a prime example of what we mean when we recommend moving such code into a broader scope.
Objects or functions defined in a helper file are available to all of your tests.

If you have just one such file, you should probably name it `helper.R`.
If you organize your helpers into multiple files, you could include a suffix with additional info.
Here are examples of how such files might look:

```
.                              
├── ...
└── tests
    ├── testthat
    │   ├── helper.R
    │   ├── helper-blah.R
    │   ├── helper-foo.R    
    │   ├── test-foofy.R
    │   └── (more test files)
    └── testthat.R
```

Many developers use helper files to define custom test helper functions, which we describe in detail below.
Compared to defining helpers below `R/`, some people find that `tests/testthat/helper.R` makes it more clear that these utilities are specifically for testing the package.
This location also feels more natural if your helpers rely on testthat functions.

A helper file is also a good location for setup code that is needed for its side effects.
This is a case where `tests/testthat/helper.R` is clearly more appropriate than a file below `R/`.
For example, in an API-wrapping package, `helper.R` is a good place to (attempt to) authenticate with the testing credentials.

### Testthat setup files

Testthat has one more special file type: setup files, defined as any file below `test/testthat/` that begins with `setup`.
Here's an example of how that might look:

```
.                              
├── ...
└── tests
    ├── testthat
    │   ├── helper.R
    │   ├── setup.R
    │   ├── test-foofy.R
    │   └── (more test files)
    └── testthat.R
```

A setup file is handled almost exactly like a helper file, but with two big differences:

* Setup files are not executed by `devtools::load_all()`.
* Setup files often contain the corresponding teardown code.

Setup files are good for global test setup that is tailored for test execution in non-interactive or remote environments.
For example, you might turn off behaviour that's aimed at an interactive user, such as messaging or writing to the clipboard.

If any of your setup should be reversed after test execution, you should also include the necessary teardown code in `setup.R`[^legacy-teardown].
We recommend maintaining teardown code alongside the setup code, in `setup.R`, because this makes it easier to ensure they stay in sync.
The artificial environment `teardown_env()` exists as a magical handle to use in `withr::defer()` and `withr::local_*()` / `withr::with_*()`.

[^legacy-teardown]: A legacy approach (which still works, but is no longer recommended) is to put teardown code in `tests/testthat/teardown.R`.

Here's a `setup.R` example from the reprex package, where we turn off clipboard and HTML preview functionality during testing:


```r
op <- options(reprex.clipboard = FALSE, reprex.html_preview = FALSE)

withr::defer(options(op), teardown_env())
```

Since we are just modifying options here, we can be even more concise and use the pre-built function `withr::local_options()` and pass `teardown_env()` as the `.local_envir`:


```r
withr::local_options(
  list(reprex.clipboard = FALSE, reprex.html_preview = FALSE),
  .local_envir = teardown_env()
)
```

### Files ignored by testthat

testthat only automatically executes files where these are both true:

* File is a direct child of `tests/testthat/`
* File name starts with one of the specific strings:
  - `helper`
  - `setup`
  - `test`

It is fine to have other files or directories in `tests/testthat/`, but testthat won't automatically do anything with them (other than the `_snaps` directory, which holds snapshots).

### Storing test data

Many packages contain files that hold test data.
Where should these be stored?
The best location is somewhere below `tests/testthat/`, often in a subdirectory, to keep things neat.
Below is an example, where `useful_thing1.rds` and `useful_thing2.rds` hold objects used in the test files.

```
.
├── ...
└── tests
    ├── testthat
    │   ├── fixtures
    │   │   ├── make-useful-things.R
    │   │   ├── useful_thing1.rds
    │   │   └── useful_thing2.rds
    │   ├── helper.R
    │   ├── setup.R
    │   └── (all the test files)
    └── testthat.R
```

Then, in your tests, use `testthat::test_path()` to build a robust filepath to such files.


```r
test_that("foofy() does this", {
  useful_thing <- readRDS(test_path("fixtures", "useful_thing1.rds"))
  # ...
})
```

`testthat::test_path()` is extremely handy, because it produces the correct path in the two important modes of test execution:

* Interactive test development and maintenance, where working directory is
  presumably set to the top-level of the package.
* Automated testing, where working directory is usually set to something
  below `tests/`.

### Where to write files during testing {#tests-files-where-write}

If it's easy to avoid writing files from you tests, that is definitely the best plan.
But there are many times when you really must write files.

**You should only write files inside the session temp directory.**
Do not write into your package's `tests/` directory.
Do not write into the current working directory.
Do not write into the user's home directory.
Even though you are writing into the session temp directory, you should still clean up after yourself, i.e. delete any files you've written.

Most package developers don't want to hear this, because it sounds like a hassle.
But it's not that burdensome once you get familiar with a few techniques and build some new habits.
A high level of file system discipline also eliminates various testing bugs and will absolutely make your CRAN life run more smoothly.

This test is from roxygen2 and demonstrates everything we recommend:


```r
test_that("can read from file name with utf-8 path", {
  path <- withr::local_tempfile(
    pattern = "Universit\u00e0-",
    lines = c("#' @include foo.R", NULL)
  )
  expect_equal(find_includes(path), "foo.R")
})
```

`withr::local_tempfile()` creates a file within the session temp directory whose lifetime is tied to the "local" environment -- in this case, the execution environment of an individual test.
It is a wrapper around `base::tempfile()` and passes, e.g., the `pattern` argument through, so you have some control over the file name.
You can optionally provide `lines` to populate the file with at creation time or you can write to the file in all the usual ways in subsequent steps.
Finally, with no special effort on your part, the temporary file will automatically be deleted at the end of the test.

Sometimes you need even more control over the file name.
In that case, you can use `withr::local_tempdir()` to create a self-deleting temporary directory and write intentionally-named files inside this directory.
  
## Test fixtures

When it's not practical to make your test entirely self-sufficient, prefer making the necessary object, logic, or conditions available in a structured, explicit way.
There's a pre-existing term for this in software engineering: a *test fixture*.

> A test fixture is something used to consistently test some item, device, or
> piece of software.
> --- Wikipedia

The main idea is that we need to make it as easy and obvious as possible to arrange the world into a state that is conducive for testing.
We describe several specific solutions to this problem:

* Put repeated code in a constructor-type helper function. Memoise it, if
  construction is demonstrably slow.
* If the repeated code has side effects, write a custom `local_*()` function to
  do what's needed and clean up afterwards.
* If the above approaches are too slow or awkward and the thing you need is
  fairly stable, save it as a static file and load it.
  
<!--
I have not found a good example of memoising a test helper in the wild.

Here's a clean little example of low-tech memoisation, taken from pillar, in
case I come back to this.

# Only check if we have color support once per session
num_colors <- local({
  num_colors <- NULL
  function(forget = FALSE) {
    if (is.null(num_colors) || forget) {
      num_colors <<- cli::num_ansi_colors()
    }
    num_colors
  }
})
-->

### Create `useful_thing`s with a helper function

Is it fiddly to create a `useful_thing`?
Does it take several lines of code, but not much time or memory?
In that case, write a helper function to create a `useful_thing` on-demand:


```r
new_useful_thing <- function() {
  # your fiddly code to create a useful_thing goes here
}
```

and call that helper in the affected tests:


```r
test_that("foofy() does this", {
  useful_thing1 <- new_useful_thing()
  expect_equal(foofy(useful_thing1, x = "this"), EXPECTED_FOOFY_OUTPUT)
})

test_that("foofy() does that", {
  useful_thing2 <- new_useful_thing()
  expect_equal(foofy(useful_thing2, x = "that"), EXPECTED_FOOFY_OUTPUT)
})
```

Where should the `new_useful_thing()` helper be defined?
This comes back to what we outlined in section \@ref(tests-files-overview).
Test helpers can be defined below `R/`, just like any other internal utility in your package.
Another popular location is in a test helper file, e.g. `tests/testthat/helper.R`.
A key feature of both options is that the helpers are made available to you during interactive maintenance via `devtools::load_all()`.

If it's fiddly AND costly to create a `useful_thing`, your helper function could even use memoisation to avoid unnecessary re-computation.
Once you have a helper like `new_useful_thing()`, you often discover that it has uses beyond testing, e.g. behind-the-scenes in a vignette.
Sometimes you even realize you should just define it below `R/` and export and document it, so you can use it freely in documentation and tests.

### Create (and destroy) a "local" `useful_thing`

So far, our example of a `useful_thing` was a regular R object, which is cleaned-up automatically at the end of each test.
What if the creation of a `useful_thing` has a side effect on the local file system, on a remote resource, R session options, environment variables, or the like?
Then your helper function should create a `useful_thing` **and clean up afterwards**.
Instead of a simple `new_useful_thing()` constructor, you'll write a customized function in the style of withr's `local_*()` functions:


```r
local_useful_thing <- function(..., env = parent.frame()) {
  # your fiddly code to create a useful_thing goes here
  withr::defer(
    # your fiddly code to clean up after a useful_thing goes here
    envir = env
  )
}
```

Use it in your tests like this:


```r
test_that("foofy() does this", {
  useful_thing1 <- local_useful_thing()
  expect_equal(foofy(useful_thing1, x = "this"), EXPECTED_FOOFY_OUTPUT)
})

test_that("foofy() does that", {
  useful_thing2 <- local_useful_thing()
  expect_equal(foofy(useful_thing2, x = "that"), EXPECTED_FOOFY_OUTPUT)
})
```

Where should the `local_useful_thing()` helper be defined?
All the advice given above for `new_useful_thing()` applies:
define it below `R/` or in a test helper file.

To learn more about writing custom helpers like `local_useful_thing()`, see the [testthat vignette on test fixtures](https://testthat.r-lib.org/articles/test-fixtures.html).

### Store a concrete `useful_thing` persistently

If a `useful_thing` is costly to create, in terms of time or memory, maybe you don't actually need to re-create it for each test run.
You could make the `useful_thing` once, store it as a static test fixture, and load it in the tests that need it.
Here's a sketch of how this could look:


```r
test_that("foofy() does this", {
  useful_thing1 <- readRDS(test_path("fixtures", "useful_thing.rds"))
  expect_equal(foofy(useful_thing1, x = "this"), EXPECTED_FOOFY_OUTPUT)
})

test_that("foofy() does that", {
  useful_thing2 <- readRDS(test_path("fixtures", "useful_thing.rds"))
  expect_equal(foofy(useful_thing2, x = "that"), EXPECTED_FOOFY_OUTPUT)
})
```

Now we can revisit a file listing from earlier, which addressed exactly this scenario:

```
.
├── ...
└── tests
    ├── testthat
    │   ├── fixtures
    │   │   ├── make-useful-things.R
    │   │   ├── useful_thing1.rds
    │   │   └── useful_thing2.rds
    │   ├── helper.R
    │   ├── setup.R
    │   └── (all the test files)
    └── testthat.R
```

This shows static test files stored in `tests/testthat/fixtures/`, but also notice the companion R script, `make-useful-things.R`.
From data analysis, we all know there is no such things as a script that is run only once.
Refinement and iteration is inevitable.
This also holds true for test objects like `useful_thing1.rds`.
We highly recommend saving the R code used to create your test objects, so that they can be re-created as needed.

## Building your own testing tools

Let's return to the topic of duplication in your test code.
We've encouraged you to have a higher tolerance for repetition in test code, in the name of making your tests obvious.
But there's still a limit to how much repetition to tolerate.
We've covered techniques such as loading static objects with `test_path()`, writing a constructor like `new_useful_thing()`, or implementing a test fixture like `local_useful_thing()`.
There are even more types of test helpers that can be useful in certain situations.

### Helper defined inside a test

Consider this test for the `str_trunc()` function in stringr:


```r
# from stringr (hypothetically)
test_that("truncations work for all sides", {
  expect_equal(
    str_trunc("This string is moderately long", width = 20, side = "right"),
    "This string is mo..."
  )
  expect_equal(
    str_trunc("This string is moderately long", width = 20, side = "left"),
    "...s moderately long"
  )
  expect_equal(
    str_trunc("This string is moderately long", width = 20, side = "center"),
    "This stri...ely long"
  )
})
```

There's a lot of repetition here, which increases the chance of copy / paste errors and generally makes your eyes glaze over.
Sometimes it's nice to create a hyper-local helper, *inside the test*.
Here's how the test actually looks in stringr


```r
# from stringr (actually)
test_that("truncations work for all sides", {

  trunc <- function(direction) str_trunc(
    "This string is moderately long",
    direction,
    width = 20
  )

  expect_equal(trunc("right"),   "This string is mo...")
  expect_equal(trunc("left"),    "...s moderately long")
  expect_equal(trunc("center"),  "This stri...ely long")
})
```

A hyper-local helper like `trunc()` is particularly useful when it allows you to fit all the important business for each expectation on one line.
Then your expectations can be read almost like a table of actual vs. expected, for a set of related use cases.
Above, it's very easy to watch the result change as we truncate the input from the right, left, and in the center.

Note that this technique should be used in extreme moderation.
A helper like `trunc()` is yet another place where you can introduce a bug, so it's best to keep such helpers extremely short and simple.

### Custom expectatations

If a more complicated helper feels necessary, it's a good time to reflect on why that is.
If it's fussy to get into position to *test* a function, that could be a sign that it's also fussy to *use* that function.
Do you need to refactor it?
If the function seems sound, then you probably need to use a more formal helper, defined outside of any individual test, as described earlier.

One specific type of helper you might want to create is a custom expectation.
Here are two very simple ones from usethis:


```r
expect_usethis_error <- function(...) {
  expect_error(..., class = "usethis_error")
}

expect_proj_file <- function(...) {
  expect_true(file_exists(proj_path(...)))
}
```

`expect_usethis_error()` checks that an error has the `"usethis_error"` class.
`expect_proj_file()` is a simple wrapper around `file_exists()` that searches for the file in the current project.
These are very simple functions, but the sheer amount of repetition and the expressiveness of their names makes them feel justified.

It is somewhat involved to make a proper custom expectation, i.e. one that behaves like the expectations built into testthat.
We refer you to the [Custom expectations](https://testthat.r-lib.org/articles/custom-expectation.html) vignette if you wish to learn more about that.

Finally, it can be handy to know that testthat makes specific information available when it's running:

* The environment variable `TESTTHAT` is set to `"true"`.
  `testthat::is_testing()` is a shortcut:
    
    ```r
    is_testing <- function() {
      Sys.getenv("TESTTHAT_PKG")
    }
    ```
* The package-under-test is available as the environment variable `TESTTHAT_PKG`
  and `testthat::testing_package()` is a shortcut:
    
    ```r
    testing_package <- function() {
      Sys.getenv("TESTTHAT_PKG")
    }
    ```

In some situations, you may want to exploit this information without taking a run-time dependency on testthat.
In that case, just inline the source of these functions directly into your package.

## When testing gets hard

Despite all the techniques we've covered so far, there remain situations where it still feels very difficult to write tests.
In this section, we review more ways to deal with challenging situations:

* Skipping a test in certain situations
* Mocking an external service
* Dealing with secrets

### Skipping a test {#tests-skipping}

Sometimes it's impossible to perform a test - you may not have an internet connection or you may not have access to the necessary credentials.
Unfortunately, another likely reason follows from this simple rule:
the more platforms you use to test your code, the more likely it is that you won't be able to run all of your tests, all of the time.
In short, there are times when, instead of getting a failure, you just want to skip a test.

#### `testthat::skip()`

Here we use `testthat::skip()` to write a hypothetical custom skipper, `skip_if_no_api()`:


```r
skip_if_no_api() <- function() {
  if (api_unavailable()) {
    skip("API not available")
  }
}

test_that("foo api returns bar when given baz", {
  skip_if_no_api()
  ...
})
```

`skip_if_no_api()` is a yet another example of a test helper and the advice already given about where to define it applies here too.

`skip()`s and the associated reasons are reported inline as tests are executed and are also indicated clearly in the summary:


```r
devtools::test()
#> ℹ Loading abcde
#> ℹ Testing abcde
#> ✔ | F W S  OK | Context
#> ✔ |         2 | blarg
#> ✔ |     1   2 | foofy
#> ────────────────────────────────────────────────────────────────────────────────
#> Skip (test-foofy.R:6:3): foo api returns bar when given baz
#> Reason: API not available
#> ────────────────────────────────────────────────────────────────────────────────
#> ✔ |         0 | yo                                                              
#> ══ Results ═════════════════════════════════════════════════════════════════════
#> ── Skipped tests  ──────────────────────────────────────────────────────────────
#> • API not available (1)
#> 
#> [ FAIL 0 | WARN 0 | SKIP 1 | PASS 4 ]
#> 
#> 🥳
```

Something like `skip_if_no_api()` is likely to appear many times in your test suite.
This is another occasion where it is tempting to DRY things out, by hoisting the `skip()` to the top-level of the file.
However, we still lean towards calling `skip_if_no_api()` in each test where it's needed. 


```r
# we prefer this:
test_that("foo api returns bar when given baz", {
  skip_if_no_api()
  ...
})

test_that("foo api returns an errors when given qux", {
  skip_if_no_api()
  ...
})

# over this:
skip_if_no_api()

test_that("foo api returns bar when given baz", {...})

test_that("foo api returns an errors when given qux", {...})
```

Within the realm of top-level code in test files, having a `skip()` at the very beginning of a test file is one of the more benign situations.
But once a test file does not fit entirely on your screen, it creates an implicit yet easy-to-miss connection between the `skip()` and individual tests.

#### Built-in `skip()` functions

Similar to testthat's built-in expectations, there is a family of `skip()` functions that anticipate some common situations.
These functions often relieve you of the need to write a custom skipper.
Here are some examples of the most useful `skip()` functions:


```r
test_that("foo api returns bar when given baz", {
  skip_if(api_unavailable(), "API not available")
  ...
})
test_that("foo api returns bar when given baz", {
  skip_if_not(api_available(), "API not available")
  ...
})

skip_if_not_installed("sp")
skip_if_not_installed("stringi", "1.2.2")

skip_if_offline()
skip_on_cran()
skip_on_os("windows")
```

#### Dangers of skipping

One challenge with skips is that they are currently completely invisible in CI — if you automatically skip too many tests, it's easy to fool yourself that all your tests are passing when in fact they're just being skipped!
In an ideal world, your CI/CD would make it easy to see how many tests are being skipped and how that changes over time.

*2022-06-01: Recent changes to GitHub Actions mean that we will likely have better test reporting before the second edition of this book is published. Stay tuned!*

It's a good practice to regularly dig into the `R CMD check` results, especially on CI, and make sure the skips are as you expect.
But this tends to be something you have to learn through experience.

### Mocking

The practice known as mocking is when we replace something that's complicated or unreliable or out of our control with something simpler, that's fully within our control.
Usually we are mocking an external service, such as a REST API, or a function that reports something about session state, such as whether the session is interactive.

The classic application of mocking is in the context of a package that wraps an external API.
In order to test your functions, technically you need to make a live call to that API to get a response, which you then process.
But what if that API requires authentication or what if it's somewhat flaky and has occasional downtime?
It can be more productive to just *pretend* to call the API but, instead, to test the code under your control by processing a pre-recorded response from the actual API.

Our main advice about mocking is to avoid it if you can.
This is not an indictment of mocking, but just a realistic assessment that mocking introduces new complexity that is not always justified by the payoffs.

Since most R packages do not need full-fledged mocking, we do not cover it here.
Instead we'll point you to the packages that represent the state-of-the-art for mocking in R today:

* mockery: <https://github.com/r-lib/mockery>
* mockr: <https://krlmlr.github.io/mockr/>
* httptest: <https://enpiar.com/r/httptest/>
* httptest2: <https://enpiar.com/httptest2/>
* webfakes: <https://webfakes.r-lib.org>

### Secrets

Another common challenge for packages that wrap an external service is the need to manage credentials.
Specifically, it is likely that you will need to provide a set of test credentials to fully test your package.

Our main advice here is to design your package so that large parts of it can be tested without live, authenticated assess to the external service.

Of course, you will still want to be able to test your package against the actual service that it wraps, in environments that support secure environment variables.
Since this is also a very specialized topic, we won't go into more detail here.
Instead we refer you to the [Wrapping APIs](https://httr2.r-lib.org/articles/wrapping-apis.html#secret-management) vignette in the httr2 package, which offers substantial support for secret management.

## Special considerations for CRAN packages

### CRAN check flavors and related services {#tests-cran-flavors-services}

*This section will likely move to a different location, once we revise and expand the content elsewhere in the book on `R CMD check` and package release. But it can gestate here.*

CRAN runs `R CMD check` on all contributed packages on a regular basis, on multiple platforms or what they call "flavors".
This check includes, but is not limited to, your testthat tests.
CRAN's check flavors almost certainly include platforms other than your preferred development environment(s), so you must proactively plan ahead if you want your tests to pass there.

You can see CRAN's current check flavors here: <https://cran.r-project.org/web/checks/check_flavors.html>.
There are various combinations of:

* Operating system and CPU: Windows, macOS (x86_64, arm64), Linux (various distributions)
* R version: r-devel, r-release, r-oldrel
* C, C++, FORTRAN compilers
* Locale, in the sense of the `LC_CTYPE` environment variable (this is about
  which human language is in use and character encoding)

It would be impractical for individual package developers to personally maintain all of these testing platforms.
Instead, we turn to various community- and CRAN-maintained resources to test our packages.
In order of how often we use them:

* GitHub Actions (GHA).
  Many R package developers host their source code on GitHub and use GHA to
  check their package, e.g., every time they push.
  
  The usethis package offers several functions to help you configure GHA
  workflows for checking your package.
  The most appropriate level of checking depends on the nature of your user base
  and how likely it is that your package could behave differently across the
  flavors (e.g. does it contain compiled code?)
  - `usethis::use_github_action_check_release()`: an entry-level, bare-minimum 
    workflow that checks with the latest release of R on Linux.
  - `usethis::use_github_action_check_standard()`: Covers the three major
    operating systems and both the released and development versions of R. This
    is a good choice for a package that is (or aspires to be) on CRAN or
    Bioconductor.
  - The tidyverse/r-lib team uses an even more extensive check matrix, which
    would be overkill for most other packages. It's necessary in this case in
    order to meet our commitment to support the current version, the development
    version, and four previous versions of R.
* R-hub builder (R-hub).
  This is a service supported by the R Consortium where package developers can
  submit their package for checks that replicate various CRAN check flavors.
  This is useful when you're doing the due diligence leading up to a CRAN
  submission.
  
  You can use R-hub via a web interface (<https://builder.r-hub.io>) or, as we
  recommend, through the [rhub R package](<https://r-hub.github.io/rhub/>).
  
  The `rhub::check_for_cran()` function is morally similar to the GHA workflow
  configured by `usethis::use_github_action_check_standard()`, i.e. it's a good
  solution for a typical package heading to CRAN.
  rhub has many other functions for accessing individual check flavors.
* Win-Builder is a service maintained by the CRAN personnel who build the
  Windows binaries for CRAN packages.
  You use it in a similar way as R-hub, i.e. it's a good check to run when
  preparing a CRAN submission.
  (Win-Builder is basically the inspiration for R-hub, i.e. Win-builder is such
  a convenient service that it makes sense to extend it for more flavors.)
  
  The Win-Builder homepage (<https://win-builder.r-project.org>) explains how to
  upload a package via ftp, but we recommend using the convenience functions
  `devtools::check_win_release()` and friends.
* macOS builder is a service maintained by the CRAN personnel who build the
  macOS binaries for CRAN packages.
  This is a relatively new addition to the list and checks packages with "the
  same setup and available packages as the CRAN M1 build machine".
  
  You can submit your package using the web interface
  (<https://mac.r-project.org/macbuilder/submit.html>) or with
  `devtools::check_mac_release()`.
  
### Testing on CRAN

The need to pass tests on all of CRAN's flavors is not the only thing you need to think about.
There are other considerations that will influence how you write your tests and how (or whether) they run on CRAN.
When a package runs afoul of the CRAN Repository Policy (<https://cran.r-project.org/web/packages/policies.html>), the test suite is very often the culprit (although not always).

If a specific test simply isn't appropriate to be run by CRAN, include `skip_on_cran()` at the very start.


```r
test_that("some long-running thing works", {
  skip_on_cran()
  # test code that can potentially take "a while" to run  
})
```

Under the hood, `skip_on_cran()` consults the `NOT_CRAN` environment variable.
Such tests will only run when `NOT_CRAN` has been explicitly defined as `"true"`.
This variable is set by devtools and testthat, allowing those tests to run in environments where you expect success (and where you can tolerate and troubleshoot occasional failure).

In particular, the GitHub Actions workflows that we recommend elsewhere **will** run tests with `NOT_CRAN = "true"` call.
For certain types of functionality, there is no practical way to test it on CRAN and your own checks, on GHA or an equivalent continuous integration service, are your best method of quality assurance.

There are even rare cases where it makes sense to maintain tests outside of your package altogether.
The tidymodels team uses this strategy for integration-type tests of their whole ecosystem that would be impossible to host inside an individual CRAN package.

The following subsections enumerate other thing to keep in mind for maximum success when testing on CRAN.

#### Speed

Your tests need to run relatively quickly - ideally, less than a minute, in total.
Use `skip_on_cran()` in a test that is unavoidably long-running.

#### Reproducibility

Be careful about testing things that are likely to be variable on CRAN machines.
It's risky to test how long something takes (because CRAN machines are often heavily loaded) or to test parallel code (because CRAN runs multiple package tests in parallel, multiple cores will not always be available).
Numerical precision can also vary across platforms, so use `expect_equal()` unless you have a specific reason for using `expect_identical()`.

#### Flaky tests

Due to the scale at which CRAN checks packages, there is basically no latitude for a test that's "just flaky", i.e. sometimes fails for incidental reasons.
CRAN does not process your package's test results the way you do, where you can inspect each failure and exercise some human judgment about how concerning it is.
  
It's probably a good idea to eliminate flaky tests, just for your own sake!
But if you have valuable, well-written tests that are prone to occasional nuisance failure, definitely put `skip_on_cran()` at the start.

The classic example is any test that accesses a website or web API.
Given that any web resource in the world will experience occasional downtime, it's best to not let such tests run on CRAN.
The CRAN Repository Policy says:

> Packages which use Internet resources should fail gracefully with an informative message if the resource is not available or has changed (and not give a check warning nor error).

Often making such a failure "graceful" would run counter to the behaviour you actually want in practice, i.e. you would want your user to get an error if their request fails.
This is why it is usually more practical to test such functionality elsewhere.
  
Recall that snapshot tests, by default, are also skipped on CRAN.
You typically use such tests to monitor, e.g., how various informational messages look.
Slight changes in message formatting are something you want to be alerted to, but do not indicate a major defect in your package.
This is the motivation for the default `skip_on_cran()` behaviour of snapshot tests.
  
Finally, flaky tests cause problems for the maintainers of your dependencies.
When the packages you depend on are updated, CRAN runs `R CMD check` on all reverse dependencies, including your package.
If your package has flaky tests, your package can be the reason another package does not clear CRAN's incoming checks and can delay its release.

#### Process and file system hygiene

In section \@ref(tests-files-where-write), we urged you to only write into the session temp directory and to clean up after yourself.
This practice makes your test suite much more maintainable and predictable.
For packages that are (or aspire to be) on CRAN, this is absolutely required per the CRAN repository policy:

> Packages should not write in the user's home filespace (including clipboards), nor anywhere else on the file system apart from the R session's temporary directory (or during installation in the location pointed to by TMPDIR: and such usage should be cleaned up)....
Limited exceptions may be allowed in interactive sessions if the package obtains confirmation from the user.

Similarly, you should make an effort to be hygienic with respect to any processes you launch:

> Packages should not start external software (such as PDF viewers or browsers) during examples or tests unless that specific instance of the software is explicitly closed afterwards.

Accessing the clipboard is the perfect storm that potentially runs afoul of both of these guidelines, as the clipboard is considered part of the user's home filespace and, on Linux, can launch an external process (e.g. xsel or xclip).
Therefore it is best to turn off any clipboard functionality in your tests (and to ensure that, during authentic usage, your user is clearly opting-in to that).

<!--
Creating and maintaining a healthy test suite takes real effort. As a codebase grows, so too will the test suite. It will begin to face challenges like instability and slowness. A failure to address these problems will cripple a test suite. Keep in mind that tests derive their value from the trust engineers place in them. If testing becomes a productivity sink, constantly inducing toil and uncertainty, engineers will lose trust and begin to find workarounds. A bad test suite can be worse than no test suite at all.

Remember that tests are often revisited only when something breaks. When you are called to fix a broken test that you have never seen before, you will be thankful someone took the time to make it easy to understand. Code is read far more than it is written, so make sure you write the test you’d like to read!

https://abseil.io/resources/swe-book/html/ch11.html

Because they make up such a big part of engineers’ lives, Google puts a lot of focus on test maintainability. Maintainable tests  are ones that "just work": after writing them, engineers don’t need to think about them again until they fail, and those failures indicate real bugs with clear causes. The bulk of this chapter focuses on exploring the idea of maintainability and techniques for achieving it.

https://abseil.io/resources/swe-book/html/ch12.html
-->

<!--
Another important path-building function to know about is `fs::path_package()`.
It is essentially `base::system.file()` with one very significant added feature:
it produces the correct path for both an in-development or an installed package.

```
during development               after installation                             

/path/to/local/package           /path/to/some/installed/package
├── DESCRIPTION                  ├── DESCRIPTION
├── ...                          ├── ...
├── inst                         └── some-installed-file.txt
│   └── some-installed-file.txt  
└── ...
```

`fs::path_package("some-installed_file.txt")` builds the correct path in both cases.

A common theme you've now encountered in multiple places is that devtools and related packages try to eliminate hard choices between having a smooth interactive development experience and arranging things correctly in your package.
-->