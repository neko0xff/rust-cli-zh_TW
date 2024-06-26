# 測試

經過數十年的軟體開發，人們發現了一個真理：未經測試的軟體很少能運作。
（很多人甚至會說：“大多數經過測試的軟體也無法運行。”但我們都是樂觀主義者，對吧？）

因此，為了確保您的程式執行您期望的操作，測試它是明智的。

一種簡單的方法是撰寫 `README` 文件，它描述了你的程式應該做什麼。
當您準備好發布新版本時，仔細閱讀 `README` 並確保行為仍然如預期。
你可以讓它成為一個更嚴格的練習也寫下您的程式應如何對錯誤輸入做出反應。

這是另一個獨特的想法：
在撰寫程式碼之前先寫下 `README`。

<aside>

**說明:**
如果您還沒聽說過[test-driven development] (TDD,測試驅動開發)，
請查看一下。

[test-driven development]: https://en.wikipedia.org/wiki/Test-driven_development


</aside>

## 自動化測試

現在，這一切都很好，但是手動完成這些工作？
但這會耗費很多時間。
與此同時，
很多人都喜歡讓電腦代勞這些工作。
讓我們來談談如何實現這些測試的自動化。

Rust 有內建的測試框架，
讓我們試著寫出第一個測試吧：

```rust,ignore
# fn answer() -> i32 {
#   42
# }
#
#[test]
fn check_answer_validity() {
    assert_eq!(answer(), 42);
}
```

您可以在幾乎任何文件中加入這段程式碼並且 `cargo test` 會找到並運行它。
這裡的關鍵字是 `#[test]` 屬性。
它允許建構系統發現這些函數並將其作為測試運行、驗證它們是否不會導致恐慌(panic)。

<aside class="exercise">

**供讀者練習:**
讓這個測試能正常運作。

最終的輸出結果應該如下所示：

```text
running 1 test
test check_answer_validity ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

</aside>

既然現在我們已經了解瞭 *如何(how)* 撰寫測試，
那麼還需要弄清楚要測試 *什麼(what)* 。 
正如你所看到的，為函數撰寫斷言相當容易。 
但 CLI 應用程式通常不只一個函數！ 
更糟糕的是，它經常要處理用戶輸入、讀取檔案和寫入輸出。

## 撰寫可測試的程式碼

有兩種互補的功能測試方法：
測試小單元，根據這些小單元建立完整的應用程式、
這些稱為 "單元測試(unit tests)"。
還有一種是 "從外部(from the outside)" 測試最終應用程序，
稱為 "黑盒測試(black box tests)" 或 "整合測試(integration tests)"。

讓我們從第一種開始。

To figure out what we should test,
let's see what our program features are.
Mainly, `grrs` is supposed to print out the lines that match a given pattern.
So, let's write unit tests for _exactly this_:
We want to ensure that our most important piece of logic works,
and we want to do it in a way that is not dependent
on any of the setup code we have around it
(that deals with CLI arguments, for example).

Going back to our [first implementation](impl-draft.md) of `grrs`,
we added this block of code to the `main` function:

```rust,ignore
// ...
for line in content.lines() {
    if line.contains(&args.pattern) {
        println!("{}", line);
    }
}
```

Sadly, this is not very easy to test.
First of all, it's in the main function, so we can't easily call it.
This is easily fixed by moving this piece of code into a function:

```rust,no_run
fn find_matches(content: &str, pattern: &str) {
    for line in content.lines() {
        if line.contains(pattern) {
            println!("{}", line);
        }
    }
}
```

Now we can call this function in our test,
and see what its output is:

```rust,ignore
#[test]
fn find_a_match() {
    find_matches("lorem ipsum\ndolor sit amet", "lorem");
    assert_eq!( // uhhhh
```

Or… can we?
Right now, `find_matches` prints directly to `stdout`, i.e., the terminal.
We can't easily capture this in a test!
This is a problem that often comes up
when writing tests after the implementation:
We have written a function that is firmly integrated
in the context it is used in.

<aside class="筆記">

**筆記:**
This is totally fine when writing small CLI applications.
There's no need to make everything testable!
It is important to think about
which parts of your code you might want to write unit tests for, however.
While we'll see that it's easy to change this function to be testable,
this is not always the case.

</aside>

Alright, how can we make this testable?
We'll need to capture the output somehow.
Rust's standard library has some neat abstractions
for dealing with I/O (input/output)
and we'll make use of one called [`std::io::Write`].
This is a [trait][trpl-traits] that abstracts over things we can write to,
which includes strings but also `stdout`.

[trpl-traits]: https://doc.rust-lang.org/book/ch10-02-traits.html
[`std::io::Write`]: https://doc.rust-lang.org/1.39.0/std/io/trait.Write.html

If this is the first time you've heard "trait"
in the context of Rust,
you are in for a treat.
Traits are one of the most powerful features of Rust.
You can think of them like interfaces in Java,
or type classes in Haskell
(whatever you are more familiar with).
They allow you to abstract over behavior
that can be shared by different types.
Code that uses traits can
express ideas in very generic and flexible ways.
This means it can also get difficult to read, though.
Don't let that intimidate you:
Even people who have used Rust for years
don't always get what generic code does immediately.
In that case,
it helps to think of concrete uses.
For example,
in our case,
the behavior that we abstract over is "write to it".
Examples for the types that implement ("impl") it
include:
The terminal's standard output,
files,
a buffer in memory,
or TCP network connections.
(Scroll down in the [documentation for `std::io::Write`][`std::io::Write`]
to see a list of "Implementors".)

With that knowledge,
let's change our function to accept a third parameter.
It should be of any type that implements `Write`.
This way,
we can then supply a simple string
in our tests
and make assertions on it.
Here is how we can write this version of `find_matches`:

```rust,ignore
{{#include testing/src/main.rs:23:29}}
```

The new parameter is `mut writer`,
i.e., a mutable thing we call "writer".
Its type is `impl std::io::Write`,
which you can read as
"a placeholder for any type that implements the `Write` trait".
Also 筆記 how we
replaced the `println!(…)`
we used earlier
with `writeln!(writer, …)`.
`println!` works the same as `writeln!`
but always uses standard output.

Now we can test for the output:

```rust,ignore
{{#include testing/src/main.rs:31:36}}
```

To now use this in our application code,
we have to change the call to `find_matches` in `main`
by adding [`&mut std::io::stdout()`][stdout] as the third parameter.
Here's an example of a main function
that builds on what we've seen in the previous chapters
and uses our extracted `find_matches` function:

```rust,ignore
{{#include testing/src/main.rs:13:21}}
```

[stdout]: https://doc.rust-lang.org/1.39.0/std/io/fn.stdout.html

<aside class="筆記">

**筆記:**
Since `stdout` expects bytes (not strings),
we use `std::io::Write` instead of `std::fmt::Write`.
As a result,
we give an empty vector as "writer" in our tests
(its type will be inferred to `Vec<u8>`),
in the `assert_eq!` we use a `b"foo"`.
(The `b` prefix makes this a _byte string literal_
so its type is going to be `&[u8]` instead of `&str`).

</aside>

<aside class="筆記">

**筆記:**
We could also make this function return a `String`,
but that would change its behavior.
Instead of writing to the terminal directly,
it would then collect everything into a string,
and dump all the results in one go at the end.

</aside>

<aside class="exercise">

**供讀者練習:**
[`writeln!`] returns an [`io::Result`]
because writing can fail,
for example when the buffer is full and cannot be expanded.
Add error handling to `find_matches`.

[`writeln!`]: https://doc.rust-lang.org/1.39.0/std/macro.writeln.html
[`io::Result`]: https://doc.rust-lang.org/1.39.0/std/io/type.Result.html

</aside>

We've just seen how to make this piece of code easily testable.
We have

1. identified one of the core pieces of our application,
2. put it into its own function,
3. and made it more flexible.

Even though the goal was to make it testable,
the result we ended up with
is actually a very idiomatic and reusable piece of Rust code.
That's awesome!

## Splitting your code into library and binary targets

We can do one more thing here.
So far we've put everything we wrote into the `src/main.rs` file.
This means our current project produces a single binary.
But we can also make our code available as a library, like this:

1. Put the `find_matches` function into a new `src/lib.rs`.
2. Add a `pub` in front of the `fn` (so it's `pub fn find_matches`)
   to make it something that users of our library can access.
3. Remove `find_matches` from `src/main.rs`.
4. In the `fn main`, prepend the call to `find_matches` with `grrs::`,
   so it's now `grrs::find_matches(…)`.
   This means it uses the function from the library we just wrote!

The way Rust deals with projects is quite flexible
and it's a good idea to think about
what to put into the library part of your crate early on.
You can for example think about writing a library
for your application-specific logic first
and then use it in your CLI just like any other library.
Or, if your project has multiple binaries,
you can put the common functionality into the library part of that crate.

<aside class="筆記">

**筆記:**
Speaking of putting everything into a `src/main.rs`:
If we continue to do that,
it'll become difficult to read.
The [module system] can help you structure and organize your code.

[module system]: https://doc.rust-lang.org/1.39.0/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html

</aside>


## Testing CLI applications by running them

Thus far, we've gone out of our way
to test the _business logic_ of our application,
which turned out to be the `find_matches` function.
This is very valuable
and is a great first step
towards a well-tested code base.
(Usually, these kinds of tests are called "unit tests".)

There is a lot of code we aren't testing, though:
Everything that we wrote to deal with the outside world!
Imagine you wrote the main function,
but accidentally left in a hard-coded string
instead of using the argument of the user-supplied path.
We should write tests for that, too!
(This level of testing is often called
"integration testing", or "system testing".)

At its core,
we are still writing functions
and annotating them with `#[test]`.
It's just a matter of what we do inside these functions.
For example, we'll want to use the main binary of our project,
and run it like a regular program.
We will also put these tests into a new file in a new directory:
`tests/cli.rs`.

<aside>

**筆記:**
By convention,
`cargo` will look for integration tests in the `tests/` directory.
Similarly,
it will look for benchmarks in `benches/`,
and examples in `examples/`.
These conventions also extend to your main source code:
libraries have a `src/lib.rs` file,
the main binary is `src/main.rs`,
or, if there are multiple binaries,
cargo expects them to be in `src/bin/<name>.rs`.
Following these conventions will make your code base more discoverable
by people used to reading Rust code.

</aside>

To recall,
`grrs` is a small tool that searches for a string in a file.
We have previously tested that we can find a match.
Let's think about what other functionality we can test.

Here is what I came up with.

- What happens when the file doesn't exist?
- What is the output when there is no match?
- Does our program exit with an error when we forget one (or both) arguments?

These are all valid test cases.
Additionally,
we should also include one test case
for the "happy path",
i.e., we found at least one match
and we print it.

To make these kinds of tests easier,
we're going to use the [`assert_cmd`] crate.
It has a bunch of neat helpers
that allow us to run our main binary
and see how it behaves.
Further,
we'll also add the [`predicates`] crate
which helps us write assertions
that `assert_cmd` can test against
(and that have great error messages).
We'll add those dependencies not to the main list,
but to a "dev dependencies" section in our `Cargo.toml`.
They are only required when developing the crate,
not when using it.

```toml
{{#include testing/Cargo.toml:16:18}}
```

[`assert_cmd`]: https://docs.rs/assert_cmd
[`predicates`]: https://docs.rs/predicates

This sounds like a lot of setup.
Nevertheless –
let's dive right in
and create our `tests/cli.rs` file:

```rust,ignore
{{#include testing/tests/cli.rs:1:15}}
```

You can run this test with
`cargo test`,
just like the tests we wrote above.
It might take a little longer the first time,
as `Command::cargo_bin("grrs")` needs to compile your main binary.

## Generating test files

The test we've just seen only checks that our program writes an error message
when the input file doesn't exist.
That's an important test to have,
but maybe not the most important one:
Let's now test that we will actually print the matches we found in a file!

We'll need to have a file whose content we know,
so that we can know what our program _should_ return
and check this expectation in our code.
One idea might be to add a file to the project with custom content
and use that in our tests.
Another would be to create temporary files in our tests.
For this tutorial,
we'll have a look at the latter approach.
Mainly, because it is more flexible and will also work in other cases;
for example, when you are testing programs that change the files.

To create these temporary files,
we'll be using the [`assert_fs`] crate.
Let's add it to the `dev-dependencies` in our `Cargo.toml`:

```toml
{{#include testing/Cargo.toml:19}}
```

[`assert_fs`]: https://docs.rs/assert_fs

Here is a new test case
(that you can write below the other one)
that first creates a temp file
(a "named" one so we can get its path),
fills it with some text,
and then runs our program
to see if we get the correct output.
When the `file` goes out of scope
(at the end of the function),
the actual temporary file will automatically get deleted.

```rust,ignore
{{#include testing/tests/cli.rs:17:32}}
```

<aside class="exercise">

**供讀者練習:**
Add integration tests for passing an empty string as pattern.
Adjust the program as needed.

</aside>

## What to test?

While it can certainly be fun to write integration tests,
it will also take some time to write them,
as well as to update them when your application's behavior changes.
To make sure you use your time wisely,
you should ask yourself what you should test.

In general it's a good idea to write integration tests
for all types of behavior that a user can observe.
That means that you don't need to cover all edge cases:
It usually suffices to have examples for the different types
and rely on unit tests to cover the edge cases.

It is also a good idea not to focus your tests on things you can't actively control.
It would be a bad idea to test the exact layout of `--help`
as it is generated for you.
Instead, you might just want to check that certain elements are present.

Depending on the nature of your program,
you can also try to add more testing techniques.
For example,
if you have extracted parts of your program
and find yourself writing a lot of example cases as unit tests
while trying to come up with all the edge cases,
you should look into [`proptest`].
If you have a program which consumes arbitrary files and parses them,
try to write a [fuzzer] to find bugs in edge cases.

[`proptest`]: https://docs.rs/proptest
[fuzzer]: https://rust-fuzz.github.io/book/introduction.html

<aside>

**筆記:**
You can find the full, runnable source code used in this chapter
[in this book's repository][src].

[src]: https://github.com/rust-cli/book/tree/master/src/tutorial/testing

</aside>
