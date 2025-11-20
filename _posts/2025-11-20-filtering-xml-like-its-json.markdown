---
layout: post
title: "Filtering XML like it's JSON"
date: 2025-11-20 09:52:54 +08:00
categories: tutorial tool reference cheatsheet
published: false
---

**Note: I realised that yq also handles XML, and is functionally better than xq, so I swapped**.

`jq` is a tool that I don't need to use often, but always find myself enjoying when I need to quickly parse some JSON for a test. Whilst I was perusing [XKB](https://wiki.archlinux.org/title/X_keyboard_extension) to work out how keyboard descriptions are mapped to names internally, I found a file, `/usr/share/X11/xkb/rules/base.xml`. It seems that this file is the most complete record of all the available keyboard names.

Naturally, since this file is an XML file, I immediately tried `apt info xq`. To my surprise, [xq](https://github.com/sibprogrammer/xq) is exactly what I thought it was, an XML parser somewhat similar to jq (although it does its own thing). Reading up shortly from the GitHub page, it seems that it uses something called [XPath](https://en.wikipedia.org/wiki/XPath), which is a language designed specifically for querying XML, neat!.

There doesn't seem to be too much on complex XML queries floating on the internet (at first glance), but I did find this [data2type](https://www.data2type.de/en/xml-xslt-xslfo/xpath/xpath-introduction), which has an XPath tutorial I'm following to learn how I can query commands properly.

## The use case
So the file has some contents like this:
```xml
<layoutList>
  <layout>
    <configItem>
      <name>al</name>
      <!-- Keyboard indicator for Albanian layouts -->
      <shortDescription>sq</shortDescription>
      <description>Albanian</description>
      <countryList>
        <iso3166Id>AL</iso3166Id>
      </countryList>
      <languageList>
        <iso639Id>sqi</iso639Id>
      </languageList>
    </configItem>
    <variantList>
      <variant>
        <configItem>
          <name>plisi</name>
          <description>Albanian (Plisi)</description>
        </configItem>
      </variant>
    <!-- etc. -->
    </variantList>
  </layout>
  <!-- etc. -->
</layoutList>
```

The question is, how can I query this layout, using XPath, to get an output that maps a description to its name, like 'English (US)|us'?

First, I tried:
```bash
cat /usr/share/X11/xkb/rules/base.xml | xq -x '//layout/configItem/name'
```
Great, but I also need the description, too.
```bash
cat /usr/share/X11/xkb/rules/base.xml | xq -x '//layout/configItem/description'
```
Wonderful! But, thinking further, how do I mash these together? I could pipe the output into another tool and just mash it all together, but I'm not 100% everything lines up nicely. Also, what if I want some kind of variant mapping, too? We're going to need more advanced XPath functionality for our use case.

This was the point where I realised that yq could handle XML.

## Using yq instead
The [yq](https://github.com/mikefarah/yq) syntax is the same as the [jq](https://github.com/jqlang/jq) syntax. It seems that the tool reads in other formats and internally converts them into YAML/JSON (not sure which one it actually works with) and then does all the parsing on that format instead.

It's not too hard to work with yq once you understand how to use the '|' and '[]' operators, which act as a pipe and an array expander, respectively.

This gives us the command:
```bash
yq --xml-skip-directives -oy \
  '.xkbConfigRegistry.layoutList[][].configItem | .description + "|" + .name' \
  /usr/share/X11/xkb/rules/base.xml
```
Which simply reads through the hierarchy, and then concatenates every description and name together. Perfect.

Now, what if we also wanted to add variants to this, too?
