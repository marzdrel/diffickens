---
layout: post
title:  "Replace case-statements with dry-matchers [draft]"
date:   2020-10-05 20:52:57 +0200
categories: rails
---

Gem `dry-matcher` offers very flexible and robust pattern matching API for
Ruby. While the gem has many interesting use-cases I would like to focus on
very specific one.

## The problem

Let's assume your code has multiple points, where logic need to support
branching for the same specific set of values. This might sound a bit too
generic, so lets dive into a first-class real world example.

Your app supports some business activity in multiple countries. Some of
the differences might be reflected by a simple database table. Other, like
translations could be provided by flat files. Even if you could abstract away
many parts of those differences, you won't be able to avoid custom code for
different countries.

Usually this can be done using simple case statements.

{% highlight ruby %}
def shipping_cost(country, weight)
  case country
  when "DE"
    5 + weight * 0.6
  when "GB"
    10
  when "US"
    10
  when "CH"
    raise "We only sell digital products in China!"
  end
end
{% endhighlight %}

The shipping cost calculation isn't probably the most sophisticated example,
but you get the idea.  This logic could easily get way more complex. What's more
imporant - this type of county-based branching can be easily required in many
different parts of your application.

In some other part of your app someone could code this logic for different case
with just one branch.

{% highlight ruby %}
def maximum_weight(country)
  if country == "DE"
    25
  else
    30
  end
end
{% endhighlight %}

Of course those samples look very primitive. You might want to introduce
better programming pattens to serve different cases. In the and though you will
generally end up with some kind of pattern maching. Logic will branch on the
provided value.

Code used in those examples is hard to maintain. Imagine, that you need to add
support for another country with another set of custom rules. If the first
`case` example will receive non-supported country the logic will silently
fail. This method will return `nil`. This might cause an exception later
in a totaly different part of the code. Either way - in most cases - this
is probably a good thing. At least you will know that there is an error,
which needs to be fixed. Imagine that the `nil` value will get coerced to a
zero somewhere along the way. This means that shipping in new country will
automatically be free.

The second `if` example is pretty obvious. Every new country lands in the
`else` clause and adopts the logic provided for that branch.

Of course you might easily patch that problem. It's enough to provide an `else`
clause with exception. This way any unexpected value will stop the execution.
This gets cumbersome pretty fast and many people tend to skip such fail-branches
while rushing the delivery of new features.

Also, your app needs very thourgh end-to-end testing suite in order to catch such
branching errors for new or unexpected scenarios before going into production.

## The solution

Let's introduce a `dry-matcher` for countries in the app. First we need to
provide some definitions. Do not get scared by the wall-of-code down below. Let's
just treat this as a nessecery boiler-plate, which needs to placed
somewhere. Let's even ignore the details for now.

There's a inline bundler setup at the top, which allows to keep this
single-file code executable. Aside from that there are two more things. First
there is a `Type` definition for your countries. Finally there is the cryptic
matcher.

Let's just scroll past that for now, and keep reading.

{% highlight ruby %}

require "bundler/inline"

gemfile do
  source "https://rubygems.org"

  gem "dry-matcher"
  gem "dry-types"
end

module Types
  include Dry::Types()

  Countries = Strict::String.enum("US", "DE", "GB", "CH")
end

class CountryMatcher < Dry::Matcher
  def self.call(value, &block)
    new(value).call(&block)
  end

  def initialize(value)
    self.value = value

    super(**Hash[countries])
  end

  def call
    super(value)
  end

  private

  def countries
    Types::Countries.values.map do |country|
      [country.downcase.to_sym, defined_match(country)]
    end
  end

  def defined_match(expected)
    Dry::Matcher::Case.new(
      match: lambda do |value|
        Types::Countries[value] == expected
      end
    )
  end

  attr_accessor :value
end

{% endhighlight %}

What this code actually does? Well it lets you define the previous `case` example
using a DSL-like syntax.

{% highlight ruby %}
def shipping_cost(country, weight)
  CountryMatcher.call(country) do |match|
    match.de { 5 + weight * 0.6 }
    match.gb { 10 }
    match.us { 10 }
    match.ch { raise "We only sell digital products in China!" }
  end
end
{% endhighlight %}

The call to matcher yields an object which defines a branch. If the
country value is `DE`, then the value from block `5 + weight * 0.6` will get
returned from this method. The sytnax is pretty compact and resonable, but
that's not the reason we wan't to replace the previous `case` example.

What happen if unexpected value gets passed to the matcher?

{% highlight ruby %}

shipping_cost("FR", 10)
# Traceback (most recent call last):
# [...]
# Dry::Types::ConstraintError ("FR" violates constraints
# (included_in?(["US", "DE", "GB", "CH"], "FR") failed))
{% endhighlight %}

Unless the country code is included in type definition, the call will
immediately fail. The error will is very clear and points directly to the
cause. It doesn't require a custom `else` clause in every branching, which is
already good.

The best part comes to play, when you decide to introduce a new market. First
you will probably add another value to the type definition, so lets put `FR` at
the end of list.

{% highlight ruby %}

module Types
  include Dry::Types()

  Countries = Strict::String.enum("US", "DE", "GB", "CH", "FR")
end

{% endhighlight %}

For now thats the only change we made to the app. Our code is polluted with
many places where the logic branches on the country. But instead of `case` or
`if` we now use matchers, just like the one we defined above. What happens if
the code runs through one of those matchers? Let's try the same example and
calculate cost for "DE" (yes, "DE", not the new country).

{% highlight ruby %}

shipping_cost("DE", 10)
# Traceback (most recent call last):
# [...]
# Dry::Matcher::NonExhaustiveMatchError (cases +fr+ not handled)

{% endhighlight %}

Every executed matcher will now fail, demanding that you need to declare logic
for new added market. You don't even need to pass the new market to actally
get the failure. All you need now is a basic suite of unit tests. All you need
to do is extend the type. Every matcher hit by your test will now fail the
tests. This way you will have a clear view on where you actually need to add
code with logic for new values.

Whenever you tend to branch code on the same set of values, consider using this
approch. The next time you will extend the set with new value, you will thank
yourself.

## Final words

[...]

