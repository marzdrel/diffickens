---
layout: post
title:  "Backing Rails enums with Postgres ENUM type"
date:   "2020-08-16 18:38:19"
categories: ruby rails sql postgres
---
Enum in Rails was introduced in version 4.1. It's a handy feature, which lets
you describe any feature or trait of an object in a very friendly and human
readable way. If you need to keep track of the state of an entity, you will
most likely add a field to the model and store a value there.

By default and by common practice Rails enums use integer type to store values. 

{% highlight ruby %}

class Post < ApplicationRecord
  enum status: { draft: 0, published: 1 }
end

{% endhighlight %}

With this approach `status` field will be expected to store numbers. At the
same time all your ActiveRecord queries and operations are able to assigned
labels instead of those numbers.
