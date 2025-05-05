---
layout: post
title:  "Creating a Derive proc-macro in Rust"
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

#[proc_macro_derive(EnvArgs)]
/// A derive proc macro meant to reduce the amount of boilerplate for
/// environment variable parsing and printing.
pub fn env_display(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    env::expand_derive_envargs(input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}
```

This is our entry point for parsing the struct attached to the derive proc-macro. All we do is use `syn` to parse the input tokens into a useful data structure, and parse that data structure to the `env.rs` file to do all the manual processing, expecting a TokenStream Result to be returned. The `.into()` call converts the wrapper `proc_macro2::TokenStream` back into a `proc_macro::TokenStream`. You can image that if you had many derive proc macros, this file would be used to delegate the processing the relevant module, which in my opinion is the nicest way of organising this.

Let's move on to the `env` module to implement the rest of the pipeline.

We want to start this module by defining the function `expand_derive_envargs` from the `lib.rs` file that we were using as the entry point to our proc-macro. This function should contain the high level steps of our macro and should return the final result to the caller.

```rust
use proc_macro2::TokenStream;
use quote::quote;
use syn::DeriveInput;
//...

pub fn expand_derive_envargs(input: DeriveInput) -> syn::Result<TokenStream> {
    // Parse the ast according to our rules
    let fields = parse_fields(&input)?;

    // Requirements for impl definition
    let ident = &input.ident;
    let (impl_generics, type_generics, where_clause) = &input.generics.split_for_impl();

    // Finish creating our traits
    let output = quote! {
        impl #impl_generics std::fmt::Display for #ident #type_generics #where_clause {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                Ok(())
            }
        }
    };

    Ok(output)
}
```
There it is. For now, we're missing the actual functionality where we implement the display function properly, but this is a useful skeleton (which is basically always present for a derive proc-macro). I'll explain the `let ident = &input.ident;` line and below. These lines are just boilerplate. The generics line, making use of the `split_for_impl();` allows our proc-macro to support structs that use generics and lifetimes. Inside the `quote!` block, we've made use of these variables in the `impl` header of the Display trait. it's a bit of boilerplate, but without it we lose flexibility.

Let's go through what the `parse_fields` function is doing.

```rust
use syn::{punctuated::Punctuated, token::Comma, Data, DeriveInput, Field, Fields};
// ...

fn parse_fields(input: &DeriveInput) -> Punctuated<Field, Comma> {
    let fields = match input.data {
        Data::Struct(ref data_struct) => match data_struct.fields {
            Fields::Named(ref fields_named) => fields_named.named.clone(),
            _ => {
                return Err(syn::Error::new_spanned(
                    input,
                    "EnvArgs only supports named struct fields.",
                ))
            }
        },
        _ => {
            return Err(syn::Error::new_spanned(
                input,
                "EnvArgs can only be used on structs.",
            ))
        }
    };
}
```
Since we only want to support structs, `parse_fields` is making sure that we have a struct with named fields, and returning those fields back to the caller.

Believe it or not, we actually have a working derive proc-macro. It isn't useful, but let's hook it up to `enver` and watch it do nothing **live**. It's also a good time to show what the problems are with the proc-macro so far so that it's more obvious why we're addressing them later on.

Let's make sure that `env_macros` is in our main workspace Cargo.toml:
```toml
[workspace]
resolver="2"
members = [ "env_macros", "enver"
]

[workspace.dependencies]
env_macros = {path="env_macros"}
```

Once that's set up, we can now access env_macros from enver, which we'll do so that we can use our proc-macro. Edit the `enver` `Cargo.toml`:
```toml
[dependencies]
# dotenvy = Your version
env_macros.workspace = true
```

Now we have to change the `main.rs` file in `enver` to contain the following:
```rust
use env_macros::EnvArgs;
use std::collections::HashMap;
use std::env::var;

#[derive(EnvArgs)]
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

fn main() {
    dotenvy::dotenv().expect("Unable to find .env file in current directory");
    let args = EnvArgs::get_env_args();
    println!("Displaying environment variables");
    print!("{}", args);
}
```

Notice that we've removed the Display impl because it's auto-generated? Go ahead and `cargo run` our project, you should get the following output:
```bash
Displaying environment variables
```
Not particularly useful, but it will be very soon. Let's move back to `env_macros` and continue implementing the display trait and get some functionality going.

I'm going to use a technique here which is applicable to any trait you may want to implement. We are going to grow a vector line-by-line, and then use functionality from the `quote` crate to place the lines of our vector directly into the `fmt` function body. It will make sense soon, I promise.

To create this vector, we are going to write a function named `fmt_fn_lines` which returns, as you may have guessed, the lines that make up the `fmt` function, inside of a vector. The function is shown below, and I'll explain what it means further down.

```rust
use syn::{
    punctuated::Punctuated, spanned::Spanned, token::Comma, Data, DeriveInput, Field, Fields, Type,
};
// ...

fn fmt_fn_lines(fields: &Punctuated<Field, Comma>) -> syn::Result<Vec<TokenStream>> {
    let mut lines = vec![];

    lines.push(quote! {
        let padding: usize = 23;
    });

    for field in fields.iter() {
        let field_ident = field.ident.as_ref().unwrap();
        let field_name = field_ident.to_string();
        let field_name_uc = field_name.to_ascii_uppercase();

        match &field.ty {
            Type::Path(_) => {
                lines.push(quote! {
                    writeln!(f,
                        "{:<width$}: {}",
                        #field_name_uc,
                        self.#field_ident,
                        width=padding)?;
                });
            }
            _ => {
                return Err(syn::Error::new(
                    field.span(),
                    "Only Path type arguments are allowed",
                ));
            }
        }
    }

    Ok(lines)
}
```
Woah that's a bit of code, what is it doing?
First, we make a vector and chuck that padding line in from the old fmt function we wrote manually. Then, we loop through every field in our struct, get it's identifier, and it's identifier as a SCREAMING_SNAKE_CASE string. For each field, we ensure that it is a normal type (not an array or a raw pointer or something unusual like that), and then we Generate our `writeln!` call for every field in the struct. For now, that's it! Let's return to the `expand_derive_envargs` function to integrate everything.

> Just a quick note concerning the difference between an `Ident` and `String` for use with the quote! macro. An `Ident`, when interpolated with the `#` symbol, will be placed as-is. This means a field named `important_field` will be input as `important_field`. A `String`, however, will be interpolated as `"important_field"`. It's useful to remember this when playing around with the quote! macro.

Now we just have to plug the fmt_lines we're generating into the `fmt` function definition, like so:

```rust
pub fn expand_derive_envargs(input: DeriveInput) -> syn::Result<TokenStream> {
    // Parse the ast according to our rules
    let fields = parse_fields(&input)?;

    // Generate the `fmt` function lines
    let fmt_lines = fmt_fn_lines(&fields)?;

    // Requirements for impl definition
    let ident = &input.ident;
    let (impl_generics, type_generics, where_clause) = &input.generics.split_for_impl();

    // Finish creating our traits
    let output = quote! {
        impl #impl_generics std::fmt::Display for #ident #type_generics #where_clause {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                #(#fmt_lines)*
                Ok(())
            }
        }
    };

    Ok(output)
}
```

Okay, wonderful. Now we have a simple derive proc-macro that works perfectly for simple types. It doesn't quite meet our requirements, but let's test it anyway. Head back over to the `enver` crate and try and run the program. You should get this error:
```
error[E0277]: `HashMap<&str, String>` doesn't implement `std::fmt::Display`
 --> enver/src/main.rs:5:10
  |
5 | #[derive(EnvArgs)]
  |          ^^^^^^^ `HashMap<&str, String>` cannot be formatted with the default formatter
  |
```
That's okay for now. Let's just do a test with `unimportant_variable` commented out:
```rust
use env_macros::EnvArgs;
use std::collections::HashMap;
use std::env::var;

#[derive(EnvArgs)]
struct EnvArgs /*<'a>*/ {
    important_variable: String,
    more_important_variable: u64,
    // unimportant_variable: HashMap<&'a str, String>,
}

impl EnvArgs /*<'_>*/ {
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
            // unimportant_variable,
        }
    }
}

fn main() {
    dotenvy::dotenv().expect("Unable to find .env file in current directory");
    let args = EnvArgs::get_env_args();
    println!("Displaying environment variables");
    print!("{}", args);
}
```
Running the program now should net you some new output:
```
Displaying environment variables
IMPORTANT_VARIABLE     : I am so very important
MORE_IMPORTANT_VARIABLE: 420
```
Not bad at all! As I said, this macro is finished for simple structs that only contain primitives, but we want more. It's time to introduce **attributes**. We can use attributes to mark struct fields that require special processing. Then we can use some standard control flow to choose the special processing when required. Time to go back to `env_macros` again. We need to add a new dependency, `deluxe`:
```bash
cargo add deluxe
```
`deluxe` handles parsing attributes attached to structs and makes it really easy to operate on them. I haven't even attempted attribute parsing without it so far. Now we need to edit `lib.rs` to tell it how we mark our attributes.
```rust
use proc_macro::TokenStream;
use syn::{parse_macro_input, DeriveInput};

mod env;

#[proc_macro_derive(EnvArgs, attributes(envargs))]
/// A derive proc macro meant to reduce the amount of boilerplate for
/// environment variable parsing and printing.
pub fn env_display(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    env::expand_derive_envargs(input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}
```
Note that we've added the `attrbutes(envargs)` parameter to `proc_macro_derive`. `envargs` is the name we wish to use when annotating struct field attributes. There are many small changes required to get attributes working and they are listed below.  All this is done inside `env.rs`. I'll explain everything below it.

```rust
use std::collections::HashMap;
use proc_macro2::TokenStream;
use quote::quote;
use syn::{
    punctuated::Punctuated, spanned::Spanned, token::Comma, Data, DeriveInput, Field, Fields, Type,
};

#[derive(deluxe::ExtractAttributes)]
#[deluxe(attributes(envargs))]
struct EnvArgsFieldAttributes {
    #[deluxe(default)]
    map: Vec<String>,
}

fn extract_envargs_field_attrs(
    fields: &mut Punctuated<Field, Comma>,
) -> deluxe::Result<HashMap<String, EnvArgsFieldAttributes>> {
    let mut field_attrs: HashMap<String, EnvArgsFieldAttributes> = HashMap::new();

    for field in fields.iter_mut() {
        let field_name = field.ident.as_ref().unwrap().to_string();
        let attrs: EnvArgsFieldAttributes = deluxe::extract_attributes(field)?;
        field_attrs.insert(field_name, attrs);
    }

    Ok(field_attrs)
}

pub fn expand_derive_envargs(input: DeriveInput) -> syn::Result<TokenStream> {
    let mut fields = parse_fields(&input)?;

    //Parse envargs field attributes
    let field_attrs: HashMap<String, EnvArgsFieldAttributes> =
        extract_envargs_field_attrs(&mut fields)?;

    // Generate the `fmt` function lines
    let fmt_lines = fmt_fn_lines(&fields, &field_attrs)?;

    // Requirements for impl definition
    let ident = &input.ident;
    let (impl_generics, type_generics, where_clause) = &input.generics.split_for_impl();

    // Finish creating our traits
    let output = quote! {
        impl #impl_generics std::fmt::Display for #ident #type_generics #where_clause {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                #(#fmt_lines)*
                Ok(())
            }
        }
    };

    Ok(output)
}
```
To begin, we have a new struct which represents the potential attributes that a field may have. The `#[deluxe(default)]` attribute sets the struct value to it's `default::Default()` value, which in the case of a `Vec`, is just an empty vector. The `extract_envargs_field_attrs` function loops through all fields of the struct and collects their attribute values in a hashmap to make them easier to work with. Finally, we call this function inside of `expand_derive_envargs` and parse the field attributes to the `fmt_fn_lines` function. Now we have everythig we need to fix our hashmap problem inside of `fmt_fn_lines`. Let's adjust that function now.

```rust
fn fmt_fn_lines(
    fields: &Punctuated<Field, Comma>,
    field_attrs: &HashMap<String, EnvArgsFieldAttributes>,
) -> syn::Result<Vec<TokenStream>> {
    let mut lines = vec![];

    lines.push(quote! {
        let padding: usize = 23;
    });

    for field in fields.iter() {
        let field_ident = field.ident.as_ref().unwrap();
        let field_name = field_ident.to_string();
        let field_name_uc = field_name.to_ascii_uppercase();
        let map: bool = !field_attrs[&field_name].map.is_empty();

        match &field.ty {
            Type::Path(_) => {
                if map {
                    lines.push(quote! {
                        writeln!(f,
                            "{:<width$}: {:?}",
                            #field_name_uc,
                            self.#field_ident.values(),
                            width=padding)?;
                    });
                } else {
                    lines.push(quote! {
                        writeln!(f,
                            "{:<width$}: {}",
                            #field_name_uc,
                            self.#field_ident,
                            width=padding)?;
                    });
                }
            }
            _ => {
                return Err(syn::Error::new(
                    field.span(),
                    "Only Path type arguments are allowed",
                ));
            }
        }
    }

    Ok(lines)
}
```
So we've changed the signature, pulled out our struct field attribute that we needed, and added some control flow to handle the new case. If something is annotated as a *map*, then we know that we should instead print out the values of that map and print those instead. This is my favourite part of the process because now you can easily see just how powerful these attributes can be. Let's try running the program from the `enver` crate. First, we need to make some small changes (like returning `unimportant_variable`):
```rust
use env_macros::EnvArgs;
use std::collections::HashMap;
use std::env::var;

#[derive(EnvArgs)]
struct EnvArgs<'a> {
    important_variable: String,
    more_important_variable: u64,
    #[envargs(map("foo", "bar"))]
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

fn main() {
    dotenvy::dotenv().expect("Unable to find .env file in current directory");
    let args = EnvArgs::get_env_args();
    println!("Displaying environment variables");
    print!("{}", args);
}
```
Note that we've also annotated `unimportant_variable` as a map with the "foo" and "bar" keys. This will be useful when we generate `get_env_args()` a bit later. Run the program and you should get some exciting output!
```
Displaying environment variables
IMPORTANT_VARIABLE     : I am so very important
MORE_IMPORTANT_VARIABLE: 420
UNIMPORTANT_VARIABLE   : ["I am the fooest of them all", "I'm not your bar"]
```
That's no small feat, we've auto-generated the `fmt` function and it even works on our `HashMap` field, too. Take a moment to enjoy the victory, and continue on when you want to implement `get_env_args()`.

### Implementing *get_env_args*
The good news is that we already have all the boilerplate we need in place, so implementing this function should be as easy writing all the lines we need into a vector. There's a small caveat to this function, since we have to return an Instance of `Self` at the end of the function, but it only adds a bit of mental overhead. Time to begin.

Let's add the filler to `expand_derive_envargs`:
```rust
pub fn expand_derive_envargs(input: DeriveInput) -> syn::Result<TokenStream> {
    // Parse the ast according to our rules
    let mut fields = parse_fields(&input)?;

    //Parse envargs field attributes
    let field_attrs: HashMap<String, EnvArgsFieldAttributes> =
        extract_envargs_field_attrs(&mut fields)?;

    // Generate the `fmt` function lines
    let fmt_lines = fmt_fn_lines(&fields, &field_attrs)?;
    let get_env_arg_lines = get_env_arg_fn_lines(&fields, &field_attrs)?;

    // Requirements for impl definition
    let ident = &input.ident;
    let (impl_generics, type_generics, where_clause) = &input.generics.split_for_impl();

    // Finish creating our traits
    let output = quote! {
        impl #impl_generics std::fmt::Display for #ident #type_generics #where_clause {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                #(#fmt_lines)*
                Ok(())
            }
        }

        impl #impl_generics #ident #type_generics #where_clause {
            fn get_env_args() -> Self {
                #(#get_env_arg_lines)*
            }
        }
    };

    Ok(output)
}
```
We've put in the `get_env_arg_lines` function and added the new `impl` block inside the `quote!` macro at the bottom. Now we just need to write `get_env_arg_lines`. Let's do that now:
```rust
fn get_env_arg_fn_lines(
    fields: &Punctuated<Field, Comma>,
    field_attrs: &HashMap<String, EnvArgsFieldAttributes>,
) -> syn::Result<Vec<TokenStream>> {
    let mut lines = vec![];
    for field in fields.iter() {
        let field_ident = field.ident.as_ref().unwrap();
        let field_name = field_ident.to_string();
        let field_name_uc = field_name.to_ascii_uppercase();
        let map: &Vec<String> = &field_attrs[&field_name].map;

        match &field.ty {
            Type::Path(path) => {
                if !map.is_empty() {
                    lines.push(quote! {
                        let mut #field_ident: HashMap<&str, String> = HashMap::new();
                    });
                    for key in map.iter() {
                        let key_uc = key.to_ascii_uppercase();
                        let var = format!("{}_{}", field_name_uc, key_uc);
                        let var_err = format!(
                            "No value found for environment variable {}",
                            var
                        );
                        lines.push(quote! {
                            #field_ident.insert(
                                #key,
                                std::env::var(#var).expect(#var_err)
                            );
                        });
                    }
                } else {
                    let type_ident =
                        path.path.get_ident().ok_or(syn::Error::new(
                            path.span(),
                            "Couldn't find type name at path",
                        ))?;
                    let env_error = format!(
                        "No value found for environment variable {}",
                        field_name_uc
                    );
                    let parse_error = format!(
                        "Type of {} was not {}",
                        field_name_uc, type_ident
                    );
                    lines.push(quote! {
                     let #field_ident = std::env::var(#field_name_uc)
                         .expect(#env_error)
                         .parse::<#type_ident>()
                         .expect(#parse_error);
                    });
                }
            }
            _ => {
                return Err(syn::Error::new(
                    field.span(),
                    "Only Path type arguments are allowed",
                ));
            }
        }
    }

    Ok(lines)
}
```
If we have the map attribute defined, then we loop over all the variables from the attribute and generate an environment variable read, ensuring that it is ready as a string. If it isn't a map variable, then we just parse the environment variable as the type it should be. I'd agree with you if you said this function was too long, so it's up to you, the reader, to decide how to clean it up. I'm going to continue on, though.

We still haven't generated the `Self` construction block, so we'll do that in another function, and then feed it into the end of `lines` vector we've made. Let's call that function from `get_env_arg_lines`.
```rust
// fn get_env_arg_lines(...) {
    ...
    
    lines.push(get_self_invocation(fields));
    
    Ok(lines)
}

```
And let's implement `get_self_invocation`. It's just going to loop through all fields of the struct and generate a shorthand `Self` block from them.

```rust
fn get_self_invocation(fields: &Punctuated<Field, Comma>) -> TokenStream {
    let mut lines = vec![];
    for field in fields.iter() {
        let field_ident = field.ident.as_ref().unwrap();

        if let Type::Path(_) = &field.ty {
            lines.push(quote! { #field_ident });
        }
    }

    quote! { Self { #(#lines),*} }
}
```
I'm being a bit lazy in this function with error checking, but I assume it's only being called at the end of `get_env_arg_lines`, after all the existing error checking.

That's the entire implementation. Let's go over to `enver` and change the file so it contains this:
```rust
use env_macros::EnvArgs;
use std::collections::HashMap;

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
Now run the program and you should get the output:
```
Displaying environment variables
IMPORTANT_VARIABLE     : I am so very important
MORE_IMPORTANT_VARIABLE: 420
UNIMPORTANT_VARIABLE   : ["I am the fooest of them all", "I'm not your bar"]
```
Finished!

### Next Steps
Whilst I have done it myself, I didn't want to put it here. You can also nest multiple structs that derive from the same proc-macro. You'll need an attribute, and a trait containing the shared functionality would be preferred to just placing the function on each of the structs. All you need to do is generate the call to the inner struct and have it return it's output up to the outer-most struct. This would mean that you could have multiple EnvArgs structs nested that will display all their fields and create themselves from the required environment variables as needed.

## Conclusion
This metaprogramming stuff is confusing as hell, but once you get it working, you'll realise just how powerful it is when it suits the problem.

The resources that helped me are:
- A great [Derive proc-macro with attributes implementation video](derive_macro_w-attr)
- The [serde_derive source](serde-derive)
- The [syn crate docs](syn docs)
- The [quote crate docs](quote docs)


[derive_macro_w-attr]: https://www.youtube.com/watch?v=GFijwucFJqw
[serde-derive]: https://github.com/serde-rs/serde/tree/master/serde_derive
[syn docs]: https://docs.rs/syn/latest/syn/
[quote docs]: https://docs.rs/quote/latest/quote/
