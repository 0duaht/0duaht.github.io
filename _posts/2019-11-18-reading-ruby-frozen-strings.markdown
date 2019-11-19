---
title: "Reading Ruby: Frozen Strings"
layout: post
date: 2019-11-18 18:10
image: /assets/images/markdown.jpg
headerImage: false
tag:
- rails
- ruby
- strings
star: true
category: blog
author: tobi
description: Understanding frozen strings, and shortcuts to getting mutable/immutable copies of a (frozen) string
---

A lot's happened since my last article. I've always wanted to be more consistent with my writing, but life gets in the way, so I'm truly sorry if this came later than you expected.

I've decided to start spending a couple minutes, sometimes hours daily, reading open-source Ruby code written by other people. Open source work has always been a good avenue to learn stuff real quick, but I've gotten pretty lazy in the past couple years so I've not been doing too much external code reading.

I typically review a lot of code written at work daily, and sometimes might have to roll up my sleeves to push a quick fix for one of the libraries we use at work. However, I'd like to spend more time deliberately reading code from other people. So if you run into any interesting Ruby libraries out there, please send them my way.

Enough with the digression. Today, I started looking at [railties](https://github.com/rails/rails/tree/e0bfe78e462e489be519c0f628c913fa78fd079a/railties). Railties is sort of a glu for piecing together some of the magic you get from Rails - bootstrapping a new Rails app, managing the cli interface and handling core Rails generators.

While reading, I stumbled on [this](https://github.com/rails/rails/blob/e0bfe78e462e489be519c0f628c913fa78fd079a/railties/lib/rails/source_annotation_extractor.rb#L58) - a plus sign before a new string, and was initially confused as to whether it was a typo, or some optimization trick to initializing the string. Here's what the snippet looked like:

{% highlight ruby %}
  def to_s(options = {})
    s = +"[#{line.to_s.rjust(options[:indent])}] "
    s << "[#{tag}] " if options[:tag]
    s << text
  end
{% endhighlight %}

First thing I did after writing off the possibility of it being a typo was try several Google search keyword combinations to get a quick explanation of why strings are initialized that way.

Multiple searches after, and several regex/string-formatting related results after, I realized searching about this syntax would not yield any relevant results. So, I opened up the Ruby documentation for the String class, which in hindsight, should be what I ought to have started with. Barely even scrolled before I saw an explanation for why strings are initialized that way, as well as a similar syntax involving the `-` sign instead of the `+` sign.

Before going into an explanation of what the purpose for prefixing a String declaration there was, I'll talk a little about frozen strings in Ruby, why it's typically a good idea to freeze strings and why it matters if you're working on a large project, or a project (that will be) used by a lot of users.

Strings in most languages are a sequence of characters represented by using quotes or enclosed in some bracketing syntax. Because a new memory space is used to store each new string, memory usage grows linearly with the number of new strings initialized. To optimize memory usage, especially in long-running apps like a web server or a project that initializes a lot of strings, strings would typically be assigned to a variable, and then the variable is returned or used, instead of a string literal. So for example in the snippet below:


{% highlight ruby %}
  def generate_version!(event = 'manual')
    puts event.object_id
  end
{% endhighlight %}


Each time `generate_version!` is called without an argument, `event` points to a new string instance, initialized with a new memory block as well. So each time this is run, a new object_id is being printed. A more optimized version of this could be something similar to:


{% highlight ruby %}
  MANUAL_EVENT = 'manual'

  def generate_version!(event = MANUAL_EVENT)
    puts event.object_id
  end
{% endhighlight %}


Here, the same object_id is printed to the console each time `generate_version!` is called, which means this string is optimized for memory usage, even if it's called a million different times.

What happens though if some operation within the method inadvertently updates the string, like:

{% highlight ruby %}
  MANUAL_EVENT = 'manual'

  def generate_version!(event = MANUAL_EVENT)
    event << '1'
    puts event.object_id
  end
{% endhighlight %}

The same object_id is still printed to the console, however, MANUAL_EVENT gets a 1 appended to its value at each call, which might not be the intended effect when passing the same string reference to a different method, or an external library.

One way to fix this is using String's freeze method, like:

{% highlight ruby %}
  def generate_version!(event = 'manual'.freeze)
    event << '1' # throws an error
    puts event.object_id
  end
{% endhighlight %}

This optimizes memory usage, and also throws an error whenever any operation tries to modify the same String.

To avoid having to freeze every string literal each time it's initialized, Ruby 2.3.0 and later provide an opt-in option to make all string literals frozen by default, and it's been used, or currently being added across several libraries for that extra memory boost while retaining consistency.

To enable this, add a pragma comment at the top of the Ruby file/code you want to enable this for, and all string literals would be frozen by default:

{% highlight ruby %}
  # frozen_string_literal: true

  def generate_version!(event = 'manual')
    event << '1' # throws an error
    puts event.object_id
  end
{% endhighlight %}

Now, back to [railties](https://github.com/rails/rails/tree/e0bfe78e462e489be519c0f628c913fa78fd079a/railties). While reading railties code, I saw the frozen-string-literal declaration at the top, but didn't make any connection to why strings in that file were prefixed with the plus sign until after I had checked out the String documentation and understood the reason for doing so.

Prefixing a (frozen) string with the + sign is typically used to return a non-frozen duplicate of the same string if it's frozen, or the same string if it's not frozen. To explain better:

{% highlight ruby %}
  MANUAL = 'manual'.freeze
  MANUAL.object_id != (+MANUAL).object_id

  MANUAL = 'manual'
  MANUAL.object_id == (+MANUAL).object_id
{% endhighlight %}

The - sign is typically used to return a frozen copy of the string. If a pre-existing copy of the string exists, it returns that copy, but if there's none, it returns a new frozen copy of the string.

So, bringing back the same snippet that started it all:

{% highlight ruby %}
  def to_s(options = {})
    s = +"[#{line.to_s.rjust(options[:indent])}] "
    s << "[#{tag}] " if options[:tag]
    s << text
  end
{% endhighlight %}

The + sign was added there in order to assign a mutable copy of the string literal. If the + sign wasn't there, the second line would fail because the String literal, when initialized, is frozen as a result of the frozen-string-literal declaration at the top of the file.