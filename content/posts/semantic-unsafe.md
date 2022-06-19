+++
title = "Unsafe code highlighting with rust-analyzer"
date = 2022-06-19
[taxonomies]
tags=["syntax-highlighting", "ide", "unsafe", "semantic-highlighting"]
+++

> This is basically a small appreciation post for semantic highlighting ;)

As Rust devs new and seasoned alike should know, unsafe code is a fickle thing and should be avoided whenever possible.
But when the unavoidable need arises, Rust at least requires the explicit use of an `unsafe { ... }` block expression.
This is a nice way of designing this unsafe access, not just because it requires the developer to make a conscious decision about reaching out to this superset of Rust, but also because it allows a reader to quickly skim code for potentially problematic regions when something goes very much wrong.

The thing with that last point though is that an unsafe block only marks a region of code.
It doesn't really show the exact operations that are being done in there that are in fact unsafe!
This wouldn't necessarily be relevant if unsafe operations in Rust were special in the way they look, but that is not the case at all.
Every unsafe operation that exists in Rust is actually one that looks just like a safe one: dereferencing (a raw pointer), reading a (union) field, calling a(n unsafe) function or accessing a (mutable) static.

```rs
unsafe {
    let value = *totally_not_a_raw_ptr;
    let value = very_safe_function();
    let value = A_STATIC_BUT_NOT_MUTABLE_I_SWEAR;
    let value = this.is_not_a_union;
}
```
{{ annotated(text="Wouldn't it be nice to make these possibly totally not safe operations stand out?") }}

All of these operations are generally safe, except for when they occur on certain constructs, so knowing whether they are safe or not in an unsafe block requires looking at the target of the operation (or requiring the unsafe blocks to always only surround the unsafe operation itself).

This is where an IDE like `rust-analyzer` can come into play to help out visually!
Most people should be acquainted with the concept of syntax highlighting, so what if we could make use of that and highlight unsafe operations differently?
This idea sounds great, but in practice the classic approaches to syntax highlighting can't help us here (they're usually defined via a bunch of [regular expressions](https://en.wikipedia.org/wiki/Regular_expression)).
After all, we just established that these unsafe operations don't look any different from safe ones.
Fortunately an IDE knows a lot more about code than just the syntax, it knows its *semantics* and can therefore do the highlighting part on a semantic level.


### Enabling unsafe highlighting

With that long and boring introduction out of the way let me show you how to visualize this separation of safe and unsafe operations with `rust-analyzer`!
Of note is that this requires support from your editor.
After all, the editor is still the responsible entity for rendering the text, as long your editor supports the `textDocument/semanticTokens` request of the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/overviews/lsp/overview/) you should be good to go.

The exact procedure for enabling this will differ based on your editor of choice, but in VSCode this would be the addition of the following lines to your `settings.json`:

```json
"editor.semanticTokenColorCustomizations": {
    "rules": {
        "*.unsafe:rust": "#eb5046"
    }
}
```
{{ annotated(text=`That's it! The curious may want to read the <a href="https://code.visualstudio.com/api/language-extensions/semantic-highlight-guide#theming">VSCode theming section</a>.`) }}

With that set, `rust-analyzer` will now reveal to us the dark truth of the snippet from earlier!

<!-- Oh dear, zola doesn't add classes to all spans so we need to copy and manually construct the code blocks for overwriting...-->
<pre data-lang="rs" class="language-rs z-code"><code class="language-rs" data-lang="rs"><span class="z-source z-rust"><span class="z-storage z-modifier z-rust" style="color:rgb(235, 80, 70);">unsafe</span> <span class="z-meta z-block z-rust"><span class="z-punctuation z-section z-block z-begin z-rust">{</span>
    <span class="z-storage z-type z-rust">let</span> value <span class="z-keyword z-operator z-assignment z-rust">=</span> <span class="z-keyword z-operator z-arithmetic z-rust" style="color:rgb(235, 80, 70);">*</span>totally_not_a_raw_ptr<span class="z-punctuation z-terminator z-rust">;</span>
    <span class="z-storage z-type z-rust">let</span> value <span class="z-keyword z-operator z-assignment z-rust">=</span> <span class="z-support z-function z-rust" style="color:rgb(235, 80, 70);">very_safe_function</span><span class="z-meta z-group z-rust"><span class="z-punctuation z-section z-group z-begin z-rust">(</span></span><span class="z-meta z-group z-rust"><span class="z-punctuation z-section z-group z-end z-rust">)</span></span><span class="z-punctuation z-terminator z-rust">;</span>
    <span class="z-storage z-type z-rust">let</span> value <span class="z-keyword z-operator z-assignment z-rust">=</span> <span class="z-constant z-other z-rust" style="color:rgb(235, 80, 70);">A_STATIC_BUT_NOT_MUTABLE_I_SWEAR</span><span class="z-punctuation z-terminator z-rust">;</span>
    <span class="z-storage z-type z-rust">let</span> value <span class="z-keyword z-operator z-assignment z-rust">=</span> this<span class="z-punctuation z-accessor z-dot z-rust">.</span><span class="z-other z-rust" style="color:rgb(235, 80, 70);">is_not_a_union</span><span class="z-punctuation z-terminator z-rust">;</span>
</span><span class="z-meta z-block z-rust"><span class="z-punctuation z-section z-block z-end z-rust">}</span>
</span></span></code></pre>
{{ annotated(text="Who would've thought, all of it is unsafe!") }}

Would you look at that!
We can now clearly see which parts of our unsafe block are actually performing unsafe operations.
As a bonus, this also marks the `unsafe` token itself and the names of unsafe declarations, such as unsafe functions and unsafe traits.

### "Unsafe" macros

But there is something we forgot about, something that allows one to easily hide things from the naked eye ... macros!
Pesky macros, they can just expand to unsafe operations as they please in a varying kind of ways.
Worry not though, as `rust-analyzer` can see right through their tricks!

<!-- Oh dear, zola doesn't add classes to all spans so we need to copy and manually construct the code blocks for overwriting...-->
<pre data-lang="rs" class="language-rs z-code"><code class="language-rs" data-lang="rs"><span class="z-source z-rust"><span class="z-meta z-macro z-rust"><span class="z-support z-function z-rust">macro_rules!</span> <span class="z-entity z-name z-macro z-rust">i_am_potentially_unsafe</span> <span class="z-meta z-block z-rust"><span class="z-punctuation z-section z-block z-begin z-rust">{</span></span></span><span class="z-meta z-macro z-rust"><span class="z-meta z-block z-rust">
    <span class="z-meta z-group z-macro-matcher z-rust"><span class="z-punctuation z-section z-group z-begin z-rust">(</span>$expr<span class="z-punctuation z-separator z-rust">:</span><span class="z-storage z-type z-rust">expr</span><span class="z-punctuation z-section z-group z-end z-rust">)</span></span> <span class="z-keyword z-operator z-rust">=&gt;</span> <span class="z-meta z-block z-macro-body z-rust"><span class="z-punctuation z-section z-block z-begin z-rust">{</span>
        <span class="z-keyword z-operator z-arithmetic z-rust">*</span>$expr
    <span class="z-punctuation z-section z-block z-end z-rust">}</span></span>
</span></span><span class="z-meta z-macro z-rust"><span class="z-meta z-block z-rust"><span class="z-punctuation z-section z-block z-end z-rust">}</span></span></span>
<span class="z-storage z-type z-rust">let</span> reference <span class="z-keyword z-operator z-assignment z-rust">=</span> <span class="z-keyword z-operator z-bitwise z-rust">&amp;</span><span class="z-constant z-numeric z-integer z-decimal z-rust">0</span><span class="z-punctuation z-terminator z-rust">;</span>
<span class="z-storage z-modifier z-rust" style="color:rgb(235, 80, 70);">unsafe</span> <span class="z-meta z-block z-rust"><span class="z-punctuation z-section z-block z-begin z-rust">{</span>
    <span class="z-support z-macro z-rust">i_am_potentially_unsafe!</span> <span class="z-meta z-block z-rust"><span class="z-punctuation z-section z-block z-begin z-rust">{</span> reference </span><span class="z-meta z-block z-rust"><span class="z-punctuation z-section z-block z-end z-rust">}</span></span><span class="z-punctuation z-terminator z-rust">;</span>
    <span class="z-support z-macro z-rust" style="color:rgb(235, 80, 70);">i_am_potentially_unsafe</span>! <span class="z-meta z-block z-rust"><span class="z-punctuation z-section z-block z-begin z-rust">{</span> reference <span class="z-keyword z-operator z-rust">as</span> <span class="z-storage z-type z-rust">*const</span> <span class="z-storage z-type z-rust">u32</span> </span><span class="z-meta z-block z-rust"><span class="z-punctuation z-section z-block z-end z-rust">}</span></span><span class="z-punctuation z-terminator z-rust">;</span>
</span><span class="z-meta z-block z-rust"><span class="z-punctuation z-section z-block z-end z-rust">}</span>
</span></span></code></pre>
{{ annotated(text="Look at that, the language does have unsafe macros! Well, kind of ...") }}

`rust-analyzer` actually inspects macro invocations for whether they emit unsafe operations and marks the identifier of the macro-call appropriately.
So while Rust itself doesn't really have the concept of an unsafe macro, `rust-analyzer` kind of imagines it.
There is one case where `rust-analyzer` deliberately marks a macro expanding to unsafe operations not as unsafe though:

```rs
macro_rules! teardown {
    () => {
        unsafe { *std::ptr::null() };
    }
}
teardown!();
```
{{ annotated(text="This is clearly doing bad things, but yet it is not marked as unsafe?") }}

Why is that?
Well, while a macro like the one shown here technically does unsafe things under the hood (even worse, it straight up invokes undefined behavior), invoking the macro does not require an unsafe block, so for consistency we aren't marking it as unsafe.
The reasoning here is that usually a macro that wraps its unsafety in an unsafe block does so because it keeps the required invariants up itself, making it safe overall.
If it does not, I'd like to argue that the macro is incorrectly written as one could invoke undefined behavior by invoking the macro without actually noticeably writing unsafe code.

Note, semantic highlighting doesn't stop here!
`rust-analyzer` has a lot more tags to offer, one of which most of you should be familiar with: the <span style="text-decoration: underline">underlining</span> of mutable variables and function calls!
You can see a full list of all the current tags in the [manual](https://rust-analyzer.github.io/manual.html#semantic-syntax-highlighting), but exploring those is for another day.
