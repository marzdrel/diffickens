---
layout: post
title:  "Backing Rails enums with Postgres ENUM type"
date:   "2020-08-16 18:38:19"
categories: rails
---
Enum in Rails was introduced in version 4.1. It's a handy feature, which lets
you describe any trait of an object in a very friendly and human
readable way. If you need to keep track of the state of an entity, you will
most likely add a field to the model and store a value there.

### Storing values as Integers

By default (and by common practice) Rails enums use integer type to store values. 

{% highlight ruby %}
class Post < ApplicationRecord
  enum status: { draft: 0, published: 1 }
end
{% endhighlight %}

With this approach `status` field is expected to store numbers. At the
same time all your ActiveRecord queries and operations will be enabled to use
to assigned labels instead of those numbers.

Actually more often then not you might encounter Rails Enums defined with Array
instead of Hash syntax. Even though Array syntax seems more pure and compact
you should really avoid it at all cost.

{% highlight ruby %}
class Post < ApplicationRecord
  enum status: [:draft, :published]
end

Post.statuses # => 
{% endhighlight %}

There are two main problem with the Array way. First of all you can't really
tell what type is used to store the values in the database. In most cases it
will be a number, but you will have to check the type of the column just to be
sure. 

Another, more severe issue is related to the actual values assigned to
provided labels. Rails will enumerate the Array and use the index of the given
field to back the value in the database. If someone decides to add another
label in the front of the Array (for example `removed`), all values will
get shifted. Meaning of this current setting in all existing records will get
changed.

{% highlight ruby %}
class Post < ApplicationRecord
  enum status: [:removed, :draft, :published]
end

Post.statuses # => 
{% endhighlight %}

There are many issues with number-backed enums. Regardless of the declaration
the readability gets crippled in many parts of the system. You can't tell at a
glance what is the meaning of underlying query when enums are translated to the
numbers by `ActiveRecord`.

{% highlight ruby %}

Post.removed.to_sql

{% endhighlight %}

When you have build a sql-query by hand, you have to manually translated enums
to numbers. There are still many places in ActiveRecord syntax, when the
automatic translation between labels and values cannot be performed.

{% highlight ruby %}

User
  .joins(<<~SQL.squish)
    LEFT JOINS posts 
      ON posts.user_id = user.id 
        AND post.status = 1
  SQL

{% endhighlight %}
