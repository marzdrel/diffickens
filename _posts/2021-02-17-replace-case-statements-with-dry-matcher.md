---
layout: post
title:  "Replace case-statements with dry-matchers"
date:   2021-02-17 21:10:57 +0200
categories: rails
---

Gem [`dry-matcher`](https://dry-rb.org/gems/dry-matcher/0.8/) offers very flexible and robust pattern matching API for
Ruby. While the gem has many interesting use-cases I would like to focus on
very specific one: Using a matcher instead of standard conditional
structures like `if` or `case`.

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
  when "CN"
    raise "We only sell digital products in China!"
  end
end
{% endhighlight %}

The shipping cost calculation isn't probably the most sophisticated example,
but you get the idea.  This logic could easily get way more complex. What's more
important - this type of county-based branching can be easily required in many
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
better programming pattens to serve different cases. In the end though you will
generally end up with some kind of pattern matching. Logic will branch on the
provided value.

Code used in those examples is hard to maintain. Imagine, that you need to add
support for another country with another set of custom rules. If the first
`case` example will receive non-supported country the logic will silently
fail. This method will return `nil`. This might cause an exception later
in a totally different part of the code. Either way - in most cases - failing
is probably a good thing. At least you will know that there is an error,
which needs to be fixed. But imagine that the `nil` value will get coerced to a
zero somewhere along the way. This means that shipping in new country will
automatically be free.

The second `if` example is pretty obvious. Every new country lands in the
`else` clause and adopts the logic provided for that branch.

Of course you might easily patch that problem. It's enough to provide an `else`
clause with exception. This way any unexpected value will stop the execution.
This gets cumbersome pretty fast and many people tend to skip such fail-branches
while rushing the delivery of new features. In general we should aim for
[failing fast](https://martinfowler.com/ieeeSoftware/failFast.pdf), but
sometimes we just get bored by copy-pasting a boilerplate code.

Also, your app needs very thorough end-to-end testing suite in order to catch
such branching errors before going into production.

## The solution

Let's introduce a `dry-matcher` for countries in the app. First we need to
provide some definitions. Do not get scared by the wall-of-code down below. Let's
just treat this as a necessary boiler-plate, which needs to placed
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

  Countries = Strict::String.enum("US", "DE", "GB", "CN")
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
    match.cn { raise "We only sell digital products in China!" }
  end
end
{% endhighlight %}

The call to matcher yields an object which defines a branch. If the country
value is `DE`, then the value from block `{ 5 + weight * 0.6 }` will get
returned from this method. The syntax is pretty compact and obvious, but
that's not the reason we want to replace the previous `case` example.

What happen if unexpected value gets passed to the matcher?

{% highlight ruby %}

shipping_cost("FR", 10)
# Traceback (most recent call last):
# [...]
# Dry::Types::ConstraintError ("FR" violates constraints
# (included_in?(["US", "DE", "GB", "CN"], "FR") failed))
{% endhighlight %}

Unless the country code is included in type definition, the call will
immediately fail. The error will be very clear and will point directly to the
cause. It doesn't require a custom `else` clause in every branching, which is
already good trade-off.

The best part comes to play, when you decide to introduce a new market. First
you will probably add another value to the type definition, so lets put `FR` at
the end of list.

{% highlight ruby %}

module Types
  include Dry::Types()

  Countries = Strict::String.enum("US", "DE", "GB", "CN", "FR")
end

{% endhighlight %}

For now that is the only change we made to the app. Our code is polluted with
logic branching over the value of country. But instead of `case` or
`if` we now use matchers, just like the one defined above. What happens if
the code runs through one of those matchers? Let's try the same example and
calculate cost for "DE" (yes, "DE", not the new country code).

{% highlight ruby %}

shipping_cost("DE", 10)
# Traceback (most recent call last):
# [...]
# Dry::Matcher::NonExhaustiveMatchError (cases +fr+ not handled)

{% endhighlight %}

Every executed matcher will now fail, demanding to declare logic for newly
added market. You don't even need to pass the new market to actually get the
failure. All you need now is a basic suite of unit tests. All you need to do is
extend the type. Every matcher covered by your test suite will now fail the
tests. This way you will have a clear view on where you actually need to add
code with logic for new values.

Whenever you tend to branch code on the same set of values, consider using this
approach. The next time you will extend the set with new value, you will thank
yourself.

## Custom implementation

Even if this specific use-case is appealing, you might sill hesitate to
introduce yet another external dependency to your app. That's a perfect valid
concern, especially if you don't want to use `dry-matcher` for anything else.

So how hard would be to implement this kind of construct by yourself?
We know exactly what we want and how it should work. It should be easy
to define set of tests.

- Valid definition with expected country should yield result from given
block. This is the obvious happy path we expect to use.
- Happy path should also support typical edge cases, like a empty block or a
nil value.
- Every block should be able to be defined only once. Multiple definitions
should raise
exception.
- Missing block for defined value should raise exception.
- Passing unknown value to the matcher should raise exception.
- Definition of a block for unknown value should raise exception.

{% highlight ruby %}

require "test/unit"

class CountryMatcherTest < Test::Unit::TestCase
  def test_valid_case
    result = CountryMatcher.call("DE") do |match|
      match.us { "US" }
      match.de { "DE" }
      match.gb { "GB" }
    end

    assert_equal result, "DE"
  end

  def test_valid_case_with_nil_result
    result = CountryMatcher.call("DE") do |match|
      match.us { "US" }
      match.de {}
      match.gb { "GB" }
    end

    assert_equal result, nil
  end

  def test_multiple_definitions
    result = proc do
      CountryMatcher.call("DE") do |match|
        match.us { "US" }
        match.de { "DE" }
        match.de { "DE" }
        match.gb { "GB" }
      end
    end

    assert_raise(CountryMatcher::MultipleDefinitionError) { result.call }
  end

  def test_missing_definition
    result = proc do
      CountryMatcher.call("DE") do |match|
        match.de { "DE" }
        match.gb { "GB" }
      end
    end

    assert_raise(CountryMatcher::MissingDefinitionError) { result.call }
  end

  def test_unknown_country
    result = proc do
      CountryMatcher.call("XX") do |match|
        match.de { "DE" }
        match.us { "US" }
        match.gb { "GB" }
      end
    end

    assert_raise(CountryMatcher::UnknownValueError) { result.call }
  end

  def test_unknown_case
    result = proc do
      CountryMatcher.call("DE") do |match|
        match.xx { "XX" }
        match.de { "DE" }
        match.us { "US" }
        match.gb { "GB" }
      end
    end

    assert_raise(NoMethodError) { result.call }
  end
end

{% endhighlight %}

Writing this requirements was pretty straightforward. I encourage everyone
to try to implement this logic on their own. The idea of doing my own
implementation as a thought exercise came to me when writing this article. I've
decided to give it a go and here's what I came with. I haven't really spend too
much time thinking about, so beware of any bugs or weird edge-cases.

{% highlight ruby %}

class CountryMatcher
  COUNTRIES = ["DE", "GB", "US"]

  class MultipleDefinitionError < StandardError; end
  class MissingDefinitionError < StandardError; end
  class UnknownValueError < StandardError; end

  class Match
    def initialize(country)
      @country = country
      @calls = []
    end

    COUNTRIES.each do |country|
      define_method country.downcase do |&block|
        if @calls.include?(country)
          raise MultipleDefinitionError, country.inspect
        end

        @calls << country
        @result = block.call if @country == country
      end
    end

    def verify_and_return
      difference.then do |missing|
        if missing.any?
          raise MissingDefinitionError, missing.inspect
        end
      end

      if !COUNTRIES.include?(@country)
        raise UnknownValueError, @country.inspect
      end

      @result
    end

    private

    def difference
      COUNTRIES.difference(@calls) +
        @calls.difference(COUNTRIES)
    end
  end

  def self.call(*args, **kwargs, &block)
    new(*args, **kwargs).call(&block)
  end

  def initialize(value)
    @value = value
  end

  def call
    Match.new(@value).then do |match|
      yield match

      match.verify_and_return
    end
  end
end

{% endhighlight %}

In case you don't know, you can just concatenate the tests with implementation
and execute it locally. This will run all the test for you without installing
any external dependencies. Otherwise you might just use this
[gist](https://gist.github.com/marzdrel/00fe5d497d10f361158242e5564d5477).

## Polymorphism

Some
[readers](https://www.moncefbelyamani.com/refactoring-technique-replace-conditional-with-polymorphism/)
pointed out, that polymorphism is a
[better solution](https://gist.github.com/reborg/dc8b0c96c397a56668905e2767fd697f#why-no-pattern-matching)
than any kind of pattern matching. While in general I agree, there are few important
points worth mentioning in this context. Ruby doesn't support multi-dispatch
polymorphism. Any kind of generic data coming from API or user input needs to be
transformed using some kind of dispatch. Building objects from stings without any
kind of guarding the input might lead to security concerns. Aside from, that unexpected
values generate very unclear error messages. Introducing new value for given type
immediately fails every code path coming through the matcher. No new test cases
for new value is required. This is one of the features of `dry-matcher` I wanted to focus on.

## Final Words

Maintaining a large code-base can be pain, but right tools and ideas can ease
the pain a lot. Gems `dry-matcher`, `dry-types` offers much more than explained
here. Theres plenty of great ideas buried in whole `dry-*`
family, so everyone might find something really use-full. I encourage everyone
to check the [dry-rb](https://dry-rb.org) website.
