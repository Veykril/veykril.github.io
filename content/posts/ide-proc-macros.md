+++
title = "IDEs and proc-macros"
description = "proc-macro woes"
date = 2022-02-25
[taxonomies]
tags=["macros", "proc-macros", "ide", "completions"]
+++

> Note that this is written with rust-analyzer in mind, as that is where I come from, nevertheless it should apply to IDEs in general.

[`rust-analyzer`](https://github.com/rust-analyzer/rust-analyzer/) enabled `#[attribute]` expansion by default on [Sep 27, 2021](https://github.com/rust-analyzer/rust-analyzer/pull/10366), and since then we've seen several issues pop up about user experience degrading when it comes to completions inside of attributed items.
This is a pretty big issue for most users, especially those who write async or webserver code, as attributes are prominently used there.
Yet we haven't really started addressing the issue properly [until recently](https://github.com/rust-analyzer/rust-analyzer/pull/11444).

We briefly talked about it in the [2021 rust-analyzer recap](https://rust-analyzer.github.io/blog/2021/12/30/2021-recap.html#proc-macros-and-attributes), but I figured a separate post about the more general problem as well as possible solutions might be of interest to some people.

This post will expand on the issue by talking about not just attributes, but also about function-like proc-macros.
I will however not touch on derive attributes specifically, as they do not really suffer from the problem as they do not replace their annotated item.

<!-- more -->

# The Problem

So what exactly is the problem?
To understand that we need to do a shallow dive into how proc-macros behave and the ecosystem around them as well as about how rust-analyzer computes completions.

If you are already familiar with how proc-macros work as well as how they are usually implemented then you can skip the following section.

> Note: If you have the time, consider reading [The Little Book of Rust Macros](https://veykril.github.io/tlborm/introduction.html) which explains rust's declarative and procedural macro systems more in-depth.

## Proc-macros in a Nutshell

There are three different kinds of proc-macros: attribute proc-macros, derive proc-macros and function-like proc-macros.
Their definition mostly works the same, at their core, proc-macros are really just rust functions that receive a stream of tokens (two for attributes) as their input, and emit a new stream of tokens as their output.
The input for attributes are the item they annotate and the tokens between the parentheses in the attribute invocation.
From here on, when I talk about the input of an attribute, I generally mean the annotated item part, as that is of more importance here.
When these macros are invoked, rust will turn the input into a so called [`TokenStream`](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html) (a fancy collection of tokens).
Rust then invokes the proc-macro function with this `TokenStream` as the input, and in case of attributes and function-like proc-macros, replaces the invocation with the output `TokenStream`.
For derives, it emits the `TokenStream` output after the annotated item instead.
If the proc-macro panics, the panic will be propagated as a compile error at the macro invocation site.
Panicking is one of two ways for proc-macros to signal failure, the other being the macro expanding into a [`compile_err!(...)`](https://doc.rust-lang.org/std/macro.compile_error.html) invocation.

```rust
use proc_macro::TokenStream;

#[proc_macro_attribute]
pub fn attribute(_attr_input: TokenStream, annotated_item: TokenStream) -> TokenStream {
    annotated_item
}
```
{{ annotated(text="The simplest attribute definition, it just outputs the annotated item again.") }}


Now to do something useful in a proc-macro, you have to parse the input `TokenStream` into something that can be better worked with.
This can of course be done by just picking off tokens and parsing them with your favorite parsing techniques, or even just working with the tokens directly.
However, either of these usually require a lot of boilerplate code, even more so for attributes and derives where you want to parse rust syntax.
Thankfully, a rust crate for parsing purposes emerged called [`syn`](https://docs.rs/syn/latest/syn/), which offers an API for parsing `TokenStream`s into structured data as well as offering definitions for all currently existing rust syntaxes.
I won't go into details of how to use `syn` or how it works except for one crucial aspect: its failure mode.
The way failure is modelled is the following: When encountering unexpected syntax, return a `TokenStream` that consists solely out of a `compile_err!(...)` invocation with an appropriate message and span.
This isn't really unique to `syn` though, but the more general approach of how error handling is usually done in procedural macros.
This failure mode detail is crucial to the problem at hand and we will come back to that in a bit.

This is really all that's relevant for us here (I did say *shallow* dive after all), but for completions sake, once parsing is done, the output `TokenStream` has to be constructed.
This is usually done by transforming the `syn` data structures and then turning them into a `TokenStream` again, usually via another crate called [`quote`](https://docs.rs/quote/latest/quote/).


## `rust-analyzer` Completions in a Nutshell

To put it brief, when rust-analyzer calculates completions for the current cursor position, it duplicates the current file, inserts a dummy identifier (think of it as looking at a future syntax tree with something the user has typed) and then tries to expand all macro invocations at the cursor position in both files until one of the expansions fails, at which point it stops.
After that, it just looks at all the surrounding syntax to calculate a completion context, which can then be used to drive the actual completion computations.

## Proc-macro Failures

So with proc-macro basics out of the way, what is it that makes rust-analyzer stop showing completions in attribute annotated items in certain situations?
In short: proc-macro failure.

Usually when a user types code, they are bound to introduce invalid rust syntax at some point sooner or later.
This is irrelevant when this invalid syntax occurs outside of macros, as rust-analyzer can easily recover from most syntax errors.
It gets more complicated within attributed items though.
When no syntax errors are introduced while typing in such an item, everything works just fine.
The proc-macro expands and rust-analyzer can do its completion calculations on the macro expanded output.

When a syntax error does get introduced though, what will usually happen is that the proc-macro either panics (the opposite of being graceful), in which case rust-analyzer just discards the item, or it emits a `compile_err!(...)` invocation (and nothing else) in which case rust-analyzer also discards the item and replaces it with this practically empty expansion.
And here lies the problem: when calculating completions for the current cursor position, rust-analyzer descends into all macro invocations at the current position first.
It then calculates completions, but at this point there is nothing to calculate them for, as the proc-macro basically erased everything.

Now this was only speaking under the assumption of attributes, but this actually also affects function-like proc-macros.
While function-like proc-macros don't take rust syntax as input, rust-analyzer can still at least do identifier completions for them, as long as they expand properly[^1].
So for these as well, if the proc-macro panics or expands to just a `compile_err!(...)`, we once again lose the ability to do completions.

[^1] In the far future we might able to use some [tricks](https://github.com/rust-analyzer/rust-analyzer/issues/7402#issuecomment-770196608) to improve on possible completions there.

# Possible Solutions

Now, there are few possible solutions to this, each with their own caveats.
So let's look at some possible ideas that either rust-analyzer or proc-macro crates can employ to make this situation work out better.

## Don't descend into attribute expansions

Perhaps the simplest approach at fixing this is to not descend the completion context into the expansion of attributes.
This way, we won't have any problems with invalid syntax as rust-analyzer's general parsing recovery strategies will handle syntax errors as usual, by building proper but incomplete syntax trees.

This comes at the expense of not being able to show names (with call-site hygiene) for completions introduced by the attribute into the direct surrounding scope.
It may also show incorrect completions or even not show some completions at all, as the original item is not actually recorded in our semantic layers due to the item in actuality just being seen as an attribute macro call.

This also skips over the problem for function-like proc-macros completely as keeping the macro-call there doesn't really do anything for completions either.

## Check the expansion for `compile_err!`

Another approach that somewhat builds on the previous one would be to check if the proc-macro expands to one or more `compile_err!(...)` invocations or if it panics, and not descend into the expansion for completions in that case.

This might not be too feasible to do in rust-analyzer, as `compile_err!(...)` invocations expand into nothingness (with its diagnostic as a side effect), so finding those can turn out to be tricky, especially if the proc-macro manages to expand into more macro-calls that all expand to a `compile_err!(...)` on different expansion levels.
It also suffers from the problem that the completion context for the cursor position now changes based on whether the user typed something valid or not, as we may be looking at the successful expansion or the unexpanded attributed item.

Those new problems aside, we also carry over the problems from the previous section, including ignoring the problem entirely for function-like proc-macros again.

## Fix up invalid syntax nodes

The following is an approach suggested by [@dtolnay](https://github.com/dtolnay/), based on rustc's behavior[^2]: Fix up or snip out invalid syntax nodes before passing them to the proc-macro.

The reason rustc does this is to prevent the proc-macro from emitting the same diagnostic as a `compile_err!(...)` that rustc will already emit.
This also allows proc-macro authors to assume they will always receive `TokenStream`s that encode valid rust syntax in the case of attribute and derive proc-macros.
Note that this only applies to attributes and derives, since the syntax there is known to be rust syntax, making it irrelevant to function-like proc-macros (which we do care about as well).

Now snipping out invalid syntax is a no-go for us, as we can't just snip out the identifier the user is typing, so we are left with fixing up syntax.
This is the approach we use as of this writing, implemented in [#11444](https://github.com/rust-analyzer/rust-analyzer/pull/11444).
Roughly speaking, we inspect the syntax nodes we pass to attributes, fix up errors, and save these fix ups to then undo them after the expansion if possible.
Now in theory this works well enough (assuming some improvements, as this is currently a proof of concept), but this actually still has its own problems.

For one we ignore function-like proc-macros again, as we do not know the syntax they expect so we can't do any fix ups.
But there is another problem regarding attributes here: while they require proper rust syntax as their input, they can still make more strict assumptions about what they expect and decide to fail if these are not upheld.
An example of that is the ability to configure the attribute invocation via special expected keyword usage or pseudo-attributes, attributes that do not exist in reality and which are stripped before expansion by the invoked attribute.

Some crates, like `salsa` for example, make use of these pseudo-helper attributes(not to be confused with [derive-helper attributes](https://veykril.github.io/tlborm/proc-macros/methodical/derive.html#helper-attributes)):

```rust
#[salsa::query_group(DatabaseStorage)]
trait Database {
    #[salsa::input] // <- this is a virtual attribute that salsa strips
    fn input_string(&self, key: ()) -> Arc<String>;
}
```

The `#[salsa::input]` attribute here tells the [`#[salsa::query_group]`](https://docs.rs/salsa-macros/0.16.0/salsa_macros/attr.query_group.html) attribute, that the `input_string` function is an `input` query, opposed to a default query which then has an effect on the actual expansion of the `#[salsa::query_group]` attribute.

So what happens if we pass a helper attribute salsa doesn't expect?
You guessed right, it bails out with a `compile_err!(...)` expansion.
Now in this case it won't be too problematic as this requires a `#[salsa::<cursor here>]` attribute specifically, where we don't even have anything to complete since helper attributes don't actually exist (a shortcoming of the proc-macro api in my eyes and something I played around with in [salsa#286](https://github.com/salsa-rs/salsa/pull/286)), but I hope it gets the point across.

So even this approach doesn't fully fix just the problem for attributes (although it get's really close).

[^2] rustc regressed in its current behavior [rust#76360#issuecomment-951145592](https://github.com/rust-lang/rust/issues/76360#issuecomment-951145592)

## "fix" the proc-macro ecosystem

Now we've seen a few approaches at how rust-analyzer could fix this problem, but there is another side from which we can look at tackling this problem: the proc-macros themselves.

This is actually the approach we first envisioned, but eagerly trying to move the ecosystem to do something we aren't even 100% on board with just asks for trouble.

So what do I mean when I say fixing proc-macros?
Well, the gist of it is to make them more IDE(rust-analyzer) friendly.

As stated earlier, the current ecosystem is centered around parsing without recovery and bailing on the first unexpected token found.
So we clearly would want to move the ecosystem towards recoverable parsing (no small feat by any means) so that even if faced with unexpected input, the proc-macro would still produce its expansion on a best effort basis *together with the `compile_err!(...)` invocations*.
This change would also allow more easily yielding multiple errors at once as a side effect, as no more immediate error returning would occur.

An example of this is [tokio#4612](https://github.com/tokio-rs/tokio/pull/4162), which makes the tokio attributes more failure resistant.
It does so by falling back to dummy values when something is going wrong, this allows it to produce a useful expansion while emitting an additional `compile_err!(...)` telling what failed.

Why is this such a huge thing?
Parser recovery is a difficult problem in general, and while there probably won't be too much trouble with a library appearing that will do recoverable rust-syntax parsing, function-like proc-macro authors will have to handwrite this recovery themselves almost all the time as their input is usually a domain-specific syntax.
Although maybe there are libraries out there that allow recoverable parsing to be written easily without much fuss (I have not done my homework on this matter yet).

This pushes the burden from one team (rust-analyzer) to basically every rust developer instead (this does sound quite terrifying), but with the proper language and/or library support it might turn out just fine.

Now here we still have a diagnostics problem for attributes though: if rust-analyzer doesn't fix up the nodes before passing them to attributes, it will report the syntax error itself and then basically the same error again from macro's parsing failure as well.

# Conclusion

So, what is the ideal solution?
Well, there isn't really just one.
All of them have their problems, and for what it is worth we may have yet to discover one solution to rule them all.
As I see it, the latter two are currently the most promising, and if put together, they cover almost all issues.
The fix-up approach takes care of most attribute related problems already, and the problems that we are left with afterwards can be taken care of by the proc-macros themselves.

For attributes, this means it is fine to assume valid rust syntax, but with having more strict requirements on the syntax, the proc-macro has to handle these requirements with recoverable strategies.
As for function-like proc-macros, the IDE really isn't able to do much at all unfortunately, so here the proc-macro is on its own and should try to recover from unexpected input as much as it can if completions are feasible for it in the first place that is.

With that all said, the best thing rust-analyzer can (and probably should) do here is to now improve on the syntax fix-up system and make it work as well as possibly.
I also hope that this post gave some better insights into the problem, and to what degree IDEs can accommodate for these.

> Think you have another possibly nice solution to the problem? Propose it in the [rust-analyzer issue](https://github.com/rust-analyzer/rust-analyzer/issues/11014)!
