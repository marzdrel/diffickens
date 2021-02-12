---
layout: post
title:  "Replace your case-statements with dry-matchers [draft]"
date:   2020-10-05 20:52:57 +0200
categories: rails
---
You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

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

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
