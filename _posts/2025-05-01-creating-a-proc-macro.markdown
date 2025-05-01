---
layout: post
title:  "Creating a proc-macro"
date:   2025-05-01 14:25:03 +08:00
categories: tutorial rust linux
published: true
---
# Background
I've recently been working on a Rust codebase with 5 binaries that communicate with one another via Kafka. To configure each of the binaries, they read in a .env file and parse the information in that .env file into a struct for better ergonomics. By the 5th binary, I realised that I was reimplementing almost exactly the same code to read in the environment variables, parse them into the struct, and then implement the display trait on that struct so that the binary would show the current configuration on boot.

I knew that I wouldn't be able to automate what I was doing with a common trait or anything like that, since the implementation of the struct was largely based on the name of the fields within the struct itself. In my case, metaprogramming by creating a custom derive proc-macro seemed like the way to go. I don't want to include the original code for what I was working on in this blog post, so I'll instead create equivalent fake code that illustrates my solution as well as the problems I was running into.

# How to begin
Enter a new empty directory of your choosing. Here we'll create a cargo workspace, and a crate called enver that loads in some environment variables and prints them to stdout.
```bash
echo -e "[workspace]\nresolver = \"2\"" > Cargo.toml
cargo new enver
cd enver
cargo add dotenvy
```
Now that we have our project, let's fill in the functionality of enver.
Inside `src/main.rs` add the following:
```rust
use std::collections::HashMap;
use std::env::var;
use std::fmt::Display;

struct EnvArgs<'a> {
    important_variable: String,
    more_important_variable: u64,
    unimportant_variable: HashMap<&'a str, String>,
}

impl EnvArgs<'_> {
    fn get_env_args() -> Self {
        let important_variable = var("IMPORTANT_VARIABLE")
            .expect("Couldn't find environment variable IMPORTANT_VARIABLE");
        let more_important_variable = var("MORE_IMPORTANT_VARIABLE")
            .expect("Couldn't find environment variable MORE_IMPORTANT_VARIABLE")
            .parse::<u64>()
            .expect("Unable to parse MORE_IMPORTANT_VARIABLE as type u64");
        let unimportant_variable_foo = var("UNIMPORTANT_VARIABLE_FOO")
            .expect("Couldn't find environment variable UNIMPORTANT_VARIABLE");
        let unimportant_variable_bar = var("UNIMPORTANT_VARIABLE_BAR")
            .expect("Couldn't find environment variable UNIMPORTANT_VARIABLE");

        let unimportant_variable = HashMap::from([
            ("UNIMPORTANT_VARIABLE_FOO", unimportant_variable_foo),
            ("UNIMPORTANT_VARIABLE_BAR", unimportant_variable_bar),
        ]);

        Self {
            important_variable,
            more_important_variable,
            unimportant_variable,
        }
    }
}

impl Display for EnvArgs<'_> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        // Length of the MORE_IMPORTANT_VARIABLE key
        let padding: usize = 23;
        writeln!(
            f,
            "{:<width$}: {}",
            "IMPORTANT_VARIABLE",
            self.important_variable,
            width = padding
        )?;
        writeln!(
            f,
            "{:<width$}: {}",
            "MORE_IMPORTANT_VARIABLE",
            self.more_important_variable,
            width = padding
        )?;
        writeln!(
            f,
            "{:<width$}: {:?}",
            "UNIMPORTANT_VARIABLE",
            self.unimportant_variable.values(),
            width = padding
        )?;
        Ok(())
    }
}

fn main() {
    dotenvy::dotenv().expect("Unable to find .env file in current directory");
    let args = EnvArgs::get_env_args();
    println!("Displaying environment variables");
    print!("{}", args);
}
```
Have a good read of the code that we just added. You'll notice a few things. Mainly, we're repeating ourselves quite a lot, even though the name of our fields inside the `EnvArgs` struct are the same as the environment variable they represent (excluding the HashMap, which we'll get to later, among other things). Realistically, we should be able to generate not only the display function, but also the get_env_args function, too.

For now, try to run the program, you should get the following error:
```
thread 'main' panicked at enver/src/main.rs:67:23:
Unable to find .env file in current directory: Io(Custom { kind: NotFound, error: "path not found" })
```
Pretty easy to understand, we should make a .env file with the variables we care about.
Let's make `.env` in the current directory.
```bash
IMPORTANT_VARIABLE="I am so very important"
MORE_IMPORTANT_VARIABLE="420"
UNIMPORTANT_VARIABLE_FOO="I am the fooest of them all"
UNIMPORTANT_VARIABLE_BAR="I'm not your bar"
```

Now when we run the program, it doesn't crash, and we get a nice printout:
```
Displaying environment variables
IMPORTANT_VARIABLE     : I am so very important
MORE_IMPORTANT_VARIABLE: 420
UNIMPORTANT_VARIABLE   : ["I'm not your bar", "I am the fooest of them all"]
```

## Creating our derive proc-macro crate
By the end of this section, we'll have replaced the code in Enver with the following:
```rust
#[derive(EnvArgs)]
struct EnvArgs<'a> {
    important_variable: String,
    more_important_variable: u64,
    #[envargs(map("foo", "bar"))]
    unimportant_variable: HashMap<&'a str, String>,
}

fn main() {
    dotenvy::dotenv().expect("Unable to find .env file in current directory");
    let args = EnvArgs::get_env_args();
    println!("Displaying environment variables");
    print!("{}", args);
}
```
The above code will do the exact same thing it did before, with far less overhead to the developer.

That was nice, but it starts getting real tedious when you need to do it more than once, so let's create our proc-macro crate (rust forces you to make a new crate for proc-macros, since we have to opt into a specific compiler api to do so).
```bash
cd ..
cargo new --lib env_macros
cd env_macros
```
We'll need to edit `Cargo.toml` manually to set to opt in to proc-macros. Let's also add our dependencies while we're at it.
```toml
[package]
name = "env_macros"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
proc-macro2 = "1.0"
quote = "1.0"
syn = "2.0"
```

There are plenty of resources on the internet regarding the functionality of the `syn`, `quote`, and `proc-macro2` crates, but I'll briefly explain them. `syn` Parses rust tokens into data structures that are fairly easy to operate on, `quote` converts rust code back into tokens that serve as our final output, and `proc-macro2` is a wrapper around `proc-macro` that allows us more flexibility.

Although it adds more complexity to the example I provide here, to keep the code closer to a real life scenario, I'm not going to dump everything into a single file, to make it evident how you can choose to structure the layout of your crate.

Let's add everything we need to `src/lib.rs` to get started.
```rust
use proc_macro::TokenStream;
use syn::{parse_macro_input, DeriveInput};

mod env;

#[proc_macro_derive(EnvArgs, attributes(envargs))]
/// A derive proc macro meant to reduce the amount of boilerplate for
/// environment variable parsing and printing.
pub fn env_display(input: TokenStream) -> TokenStream {
    let mut input = parse_macro_input!(input as DeriveInput);
    env::expand_derive_envargs(input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}
```

This is our entry point for parsing the struct attached to the derive proc-macro. All we do is use `syn` to parse the input tokens into a useful data structure, and parse that data structure to the `env.rs` file to do all the manual processing, expecting a TokenStream Result to be returned. The `.into()` call converts the wrapper `proc_macro2::TokenStream` back into a `proc_macro::TokenStream`. You can image that if you had many derive proc macros, this file would be used to delegate the processing the relevant module, which in my opinion is the nicest way of organising this.

Let's move on to the `env` module to implement the rest of the pipeline.

[Derive proc-macro Attributes Video]: https://www.youtube.com/watch?v=GFijwucFJqw
[serde-derive]: https://github.com/serde-rs/serde/tree/master/serde_derive
[syn docs]: https://docs.rs/syn/latest/syn/
[quote docs]: https://docs.rs/quote/latest/quote/
