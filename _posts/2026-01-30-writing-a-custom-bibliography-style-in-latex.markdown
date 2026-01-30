---
layout: post
title: "Writing a custom bibliography style in LaTeX"
date: 2026-01-30 11:07:09 +08:00
categories: latex tutorial
published: true
---
I write a lot of LaTeX reports at work, since I like how specific I can be with the typesetting of my documents, and because it's partly programming, which is my entire jam.

I recently realised that, whilst using the `IEEETran` style, which I use for all my reports, since the in-text citations don't take up too much space, that the references don't actually display the date that content was accessed. This is filled into the `references.bib` file underneath the `urldate` parameter. Since the internet shifts around so much, and webpages disappear all the time, it's pretty necessary to fill this information in.

I couldn't find any simple way of doing it, so I decided that I would modify the existing `IEEETran.bst` style file, creating my own, named `IEEETranJ.bst`.

# Obtaining the original
The original can be found on CTAN, [here](https://ctan.org/tex-archive/macros/latex/contrib/IEEEtran/bibtex). There's already a couple of variations, but none of them do what I want them to do.

# Getting a guide to the bst language
So these `.bst` files use a really simple postfix language. I have to say, I'm actually a pretty big fan of this language; it does what it needs to quite well. Regardless, a quick guide is necessary to working with any new programming language in my opinion, and [Oren Patashnik's](https://au.mirrors.cicku.me/ctan/biblio/bibtex/base/btxhak.pdf) guide from February 8, 1988 is just the ticket.

# Making the changes necessary
So it turns out that `IEEETran` is already written quite nicely. We are going to copy many of the 'url' named functions and variables for urldate. For example:
```bst
% #0 turns off the display of urls
% #1 enables
FUNCTION {default.is.use.url} { #1 }
```
We can just appropriate this for `urldate`s:
```bst
% #0 turns off the display of urldates
% #1 enables
FUNCTION {default.is.use.urldate} { #1 }
```

Should be pretty easy to understand. This is a global options that we can modify to disable options for the style.

I want the string "(Accessed: yyyy-mm-dd)" or "(Accessed: dd Month yyyy)" to be printed inside the bibliography when I'm finished. The first step to that is to add the prefix "(Accessed:" to our entry. Again, we can copy from `url`:
```bst
% The default URL prefix.
FUNCTION {default.name.url.prefix}{ "[Online]. Available:" }
```
We can just add our own default prefix:
```bst
% The default urldate prefix.
FUNCTION {default.name.urldate.prefix}{ "(Accessed:"}
```

We need to tell the style to read the `urldate` field from the .bib file we write, otherwise that data will simply not be used:

```bst
ENTRY
  { address
    ...
    url
    urldate % <- Insert this
    ...
  }
  {}
  { label }
```

We also need to explicitly create an integer to hold our boolean flag from earlier:
```bst
INTEGERS { is.use.number.for.article
           is.use.paper
           is.use.url
           is.use.urldate % <- Insert this
           is.forced.et.al
           max.num.names.before.forced.et.al
           num.names.shown.with.forced.et.al
           is.use.alt.interword.spacing
           is.dash.repeated.names}
```

We also need to explicitly create a string to hold our prefix from earlier:
```bst
STRINGS { bibinfo
          longest.label
          oldname
          s
          t
          ALTinterwordstretchfactor
          name.format.string
          name.latex.cmd
          name.url.prefix
          name.urldate.prefix}
```

Now, we need to initialise these variables we've just created so that they hold the defaults:

```bst
FUNCTION {initialize.controls}
{ default.is.use.number.for.article 'is.use.number.for.article :=
  default.is.use.paper 'is.use.paper :=
  default.is.use.url 'is.use.url :=
  default.is.use.urldate 'is.use.urldate := % <- Insert this  
  default.is.forced.et.al 'is.forced.et.al :=
  default.max.num.names.before.forced.et.al 'max.num.names.before.forced.et.al :=
  default.num.names.shown.with.forced.et.al 'num.names.shown.with.forced.et.al :=
  default.is.use.alt.interword.spacing 'is.use.alt.interword.spacing :=
  default.is.dash.repeated.names 'is.dash.repeated.names :=
  default.ALTinterwordstretchfactor 'ALTinterwordstretchfactor :=
  default.name.format.string 'name.format.string :=
  default.name.latex.cmd 'name.latex.cmd :=
  default.name.url.prefix 'name.url.prefix :=
  default.name.urldate.prefix 'name.urldate.prefix := % <- Insert this
}
```

Note above is where we're really seeing the postfix notation coming out. We are using the assignment operator `:=` to set our integer/string variables to their default values. I'm unsure why we need the `'` for our right operand, but it seems to be required specifically for assignment.

Now we can create our actual formatting function. Note that I've probably done this somewhat wrong, but I couldn't waste to much time learning what everything in the file did. First, we start with the definition for `format.url`:

```bst
FUNCTION {format.url}
{ is.use.url
    { url empty$
      { "" }
      { this.to.prev.status
        this.status.std
        cap.yes 'status.cap :=
        name.url.prefix " " *
        "\url{" * url * "}" *
        punct.no 'this.status.punct :=
        punct.period 'prev.status.punct :=
        space.normal 'this.status.space :=
        space.normal 'prev.status.space :=
        quote.no 'this.status.quote :=
      }
    if$
    }
    { "" }
  if$
}
```
What does this function do? If the global `is.use.url` flag is enabled, and the url is nonempty, then we add the url, using the `\url{}` command (which you should recognise as normal LaTeX!) to input our URL into the reference entry.

Okay, now for our own:
```bst
FUNCTION {format.urldate}
{ is.use.urldate 
    { urldate empty$
        { "" }
        { this.to.prev.status
          this.status.tsd
          cap.yes 'status.cap :=
          name.urldate.prefix " " *
          "\DTMdate{" * urldate * "}" * ")" *
          punct.no 'this.status.punct :=
          punct.period 'prev.status.punct :=
          space.normal 'this.status.space :=
          space.normal 'prev.status.space :=
          quote.no 'this.status.quote :=
        }
    if$
    }
    { "" }
  if$
}
```
We're doing pretty much the same thing as `format.url`, except we use the `\DTMdate{}` function and close of our parentheses with a ')'. What if `\DTMdate` or `\url` aren't available though? Since we can write arbitrary latex inside this file, we handle this situation ourselves (although it's best to keep the amount of LaTeX in these files minimal). Inside the `begin.bib` function, we solve our problem:

```bst
FUNCTION {begin.bib}
{
  write$ newline$
  preamble$ empty$ 'skip$
    { preamble$ write$ newline$ }
  if$
  "\begin{thebibliography}{"  longest.label  * "}" *
  write$ newline$
  "\providecommand{\url}[1]{#1}"
  write$ newline$
  "\providecommand{\DTMdate}[1]{#1}" % <- Insert this line
  write$ newline$                    % <- Insert this line
  ...
}
```
We simply provide the command to dump `urldate` as it appears inside the .bib file if the `\DTMdate` function is not already provided elsewhere. For reference, this is provided by the `datetime2` package. We can simply change the way dates are formatted using this package so that we never have to interact with the `.bst` file again.

Now, we have to add our new formatting to each of the styles we want it to show up for. I want it to show up everywhere, so I add it everywhere:

```bst
FUNCTION {article}
{ std.status.using.comma
  ...
  format.url output
  format.urldate output % <- Insert this line
  fin.entry
  if.url.std.interword.spacing
}
```
Do the same for book, booklet, electronic, misc, etc. Voila, we now have the `urldate` formatted into all our references where this entry has been included inside of the `.bib` file!

# Other things
All those commands that end with `$` marks in .bst files are actually builtins. If they contain a letter in the name, then they end with a `$`, like `empty$`. If they don't contain a letter, they're written as-is, like `:=` or `*`.

The actual execution flow of the program is really simple, and only takes up a few lines:
```bst
READ

EXECUTE {initialize.controls}
EXECUTE {initialize.status.constants}
EXECUTE {banner.message}

EXECUTE {initialize.longest.label}
ITERATE {longest.label.pass}

EXECUTE {begin.bib}
ITERATE {call.type$}
EXECUTE {end.bib}

EXECUTE{completed.message}
```

`READ` just reads in the `.bib` file, locating the `ENTRY` fields that we speicified at the top of the file.

`EXECUTE` is just running functions that we've specified.

`ITERATE` is interesting, especially the second one. This does the actual generation of entries inside our references. The `call.type$` builtin will call the function associated with the type of the entry currently being read (types are things like "book", "conference", "article", "online"). `ITERATE` means that we call the function for every single entry in the `.bib` file. It's a pretty nice way to dispatch entries to the correct formatting function(s).

Did you know that `@online` and `@conference` aren't actual entry types? They're just aliases to `@electronic` and `@inproceedings`, respectively. We can simply define entry aliases to create "new" entries at will:
```bst
FUNCTION {conference}{inproceedings}
FUNCTION {online}{electronic}
FUNCTION {internet}{electronic}
FUNCTION {webpage}{electronic}
FUNCTION {www}{electronic}
FUNCTION {default.type}{misc}
```

This was a really fun detour, and I found myself loving the postfix stack language in these `.bst` files. It feels like the right solution to the problem, and I managed to do my adjustments in about 45 minutes with no prior knowledge.
