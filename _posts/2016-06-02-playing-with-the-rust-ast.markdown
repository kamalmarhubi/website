---
title: Things you could do with the Rust AST
date: 2016-06-02T12:47:58+02:00
---

Lately I've been contributing to [rustfmt], which is a tool for formatting Rust code. Right now, it formats entire files, which is great if you keep your entire project formatted by rustfmt. If you're adding code to an existing unrustfmted project, using rustfmt leads to big diffs where most of the changes are just formatting. These can be really annoying to review, so the feature I'm working on is to allow it to format just parts of files. Then you could just reformat the parts you actually changed, making things much nicer for whoever is reviewing the change.

[rustfmt]: https://github.com/rust-lang-nursery/rustfmt

Working on rustfmt has meant getting a bit of familiarity with Rust's abstract syntax tree (AST), which is how the rustc compiler represents source code. The AST has nodes to represent each expression, statement, function, module, and all the other bits of syntax that make up a Rust program.
It's been interesting and kind of cool to work with, and I wanted to share a bit!


## libsyntax and syntex_syntax

The Rust compiler's own parsing code is in a library called libsyntax. Sadly for programs that aren't rustc, this is a private library inside the compiler source repository. If you're willing to use the nightly compiler, there are some incantations that will tell rustc to let you use these internal libraries, but it's not really encouraged. The Rust folks want to be able to make changes as they keep working on making Rust better, and not have to worry about breaking outside projects.

But someone maintains a copy in Crates.io called [syntex_syntax], with modifications to build with stable Rust. It's part of a suite of tools for generating Rust code, initially created for [a serialization framework][serde].

[syntex_syntax]: https://crates.io/crates/syntex_syntax
[serde]: https://github.com/serde-rs/serde

The upshot of this is that we can just add the line

~~~
syntex_syntax = "0.33"
~~~

to our `Cargo.toml` file, and then we can parse Rust code in our programs!


## A small example of working with the AST

To get an idea of what working with the AST looks like, let's look at a small example: printing the number of arguments for each function in a file.

syntex_syntax has a `Visitor` trait you can implement to walk an AST. The default implementation just walks the whole tree and doesn't do anything else. To write a custom visitor, you can implement the `visit` function for the node type you're interested in, and leave the rest out. If you're more used to Java-esque type systems, you can imagine inheriting from a base `DefaultVisitor` class, and overriding the functions relating to the syntactic elements you are interested in.

For our example, we override the implementation for `visit_fn` to store the funciton name and number of arguments in a hashmap. The [full code is up on GitHub][full-code], but here are the important parts:

[full-code]: https://github.com/kamalmarhubi/syntex-syntax-example

~~~
struct CountFnArgs<'a> {
    arg_counts: HashMap<String, usize>,
    // The codemap is necessary to go from a `Span` to actual line & column
    // numbers for closures.
    codemap: &'a CodeMap,
}

impl<'v, 'a> Visitor<'v> for CountFnArgs<'a> {
    fn visit_fn(&mut self,
                fn_kind: FnKind<'v>,
                fn_decl: &'v ast::FnDecl,
                block: &'v ast::Block,
                span: Span,
                _id: ast::NodeId) {
        let fn_name = match fn_kind {
            FnKind::ItemFn(id, _, _, _, _, _) |
            FnKind::Method(id, _, _) => id.name.as_str().to_string(),
            FnKind::Closure => format!("<closure at {}>", self.format_span(span)),
        };

        self.arg_counts.insert(fn_name, fn_decl.inputs.len());

        // Continue walking the rest of the funciton so we pick up any functions
        // or closures defined in its body.
        visit::walk_fn(self, fn_kind, fn_decl, block, span);
    }
}

fn count_fn_args(krate: &ast::Crate, codemap: &CodeMap) -> HashMap<String, usize> {
    let mut visitor = CountFnArgs {
        arg_counts: HashMap::new(),
        codemap: codemap,
    };
    visitor.visit_mod(&krate.module, krate.span, 0);

    visitor.arg_counts
}
~~~

Rust's pattern matching really shines when working with ASTs. You can get a small glimpse of it here: we match on the `FnKind` to see if the function is a method, a free function (`ItemFn`), or a closure. In the first two cases, we just pull out the function's name; in the third we get the source location to identify it better.

Here's the output for our little program running on its own source:

~~~
FUNCTION                         ARGS
<closure at 40:19-40:39>         2
count_fn_args                    2
format_loc                       1
format_span                      2
main                             0
parse                            2
visit_fn                         6
visit_mac                        2
~~~

Not bad!


## Things you can do with an AST

Being able to programmatically parse and work with source code opens up all kinds of cool possibilities. Here are a few.

### Format Rust code!

As I kind of mentioned at the start of this post, rustfmt uses the AST to format code. First it parses a file into an AST. Then it walks the tree, recursively printing each node according to a bunch of formatting fules. The [design document][rustfmt-design] gives bit more background on the approach, and why rustfmt works the way it does.

[rustfmt-design]: https://github.com/rust-lang-nursery/rustfmt/blob/master/Design.md#operate-on-the-ast


### Throughly test C APIs

The [libc crate][libc] is full of function declarations, struct definitions, and constants for working with system libraries. It has a really cool and thorough test suite that makes sure all the items match what's in the system C headers. It works by parsing the Rust source, and *generating C code* where it tests for equality of constants and matching type signatures and so on. It then generates some Rust code that calls the generated C code to check everything. So much code generation!

Here's a snippet of the generated C code to give you an idea of what it's like:

~~~
static int __test_const_O_RDONLY_val = O_RDONLY;
int* __test_const_O_RDONLY(void) {
    return &__test_const_O_RDONLY_val;
}


static int __test_const_O_WRONLY_val = O_WRONLY;
int* __test_const_O_WRONLY(void) {
    return &__test_const_O_WRONLY_val;
}


static int __test_const_O_RDWR_val = O_RDWR;
int* __test_const_O_RDWR(void) {
    return &__test_const_O_RDWR_val;
}
~~~

and it goes on and on and on for over 10,000 lines! The result is that the definitions in libc can be pretty safely relied on to match the definitions on the platform you're working with, which is really great for doing systems programming ([one of my favourite things][nix-post]).


### Other things you could do!

There are all kinds of other things we can do! For example, it should be possible to build a syntax-aware find-and-replace tool to make some refactorings easier. I'm imagining something like [FaceBook's jscodeshift tool for JavaScript][jscodeshift], or their [spatch] for PHP and some other languages. This would be great for situations like where you just changed the type of the third argument of that function from `String` to `&str` (string slice) and now you have to change all the call sites everywhere to add an `&` to take a reference.

[asref]: http://doc.rust-lang.org/std/convert/trait.AsRef.html
[convert]: http://doc.rust-lang.org/std/convert/index.html

[jscodeshift]: https://github.com/facebook/jscodeshift
[spatch]: https://github.com/facebook/pfff/wiki/Spatch

If we're willing to go a bit deeper into compiler internals—and require nighty Rust to build our project—we can also make tools that use type information. There's no reason something like [Hoogle] couldn't exist for Rust. Hoogle was amazing back when I used to write Haskell. You search by *type signature*. It's kind of magical to be able to say ["I have a a list and I want an int"][hoogle-search] and it would have the `length` function at the top of the results page.

[hoogle]: https://www.haskell.org/hoogle/
[hoogle-search]: https://www.haskell.org/hoogle/?hoogle=%5Ba%5D+-%3E+Int

[DXR] is an example of an exisiting project using type information in Rust. It's an indexing engine and web interface with fancy cross referencing of source code. As an example, you can [browse things around the `visit_fn` function in rustfmt][rustfmt-dxr]. It's really cool! This kind of interwingly source browsing can be really really powerful for understanding a new codebase.

I really love tools like these that make developers' lives easier. If you have any ideas for Rust, or examples from other languages to "borrow", please send them my way!



[libc]: https://github.com/rust-lang/libc
[nix-post]: http://kamalmarhubi.com/blog/2016/04/13/rust-nix-easier-unix-systems-programming-3/
[dxr]: https://github.com/mozilla/dxr
[rustfmt-dxr]: https://dxr.mozilla.org/rustfmt/source/src/visitor.rs#133

<br />

---

<small>*Thanks to Julia Evans for feedback on this post, and for suggesting I write it in the first place!*</small>
