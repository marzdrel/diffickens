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

Post.statuses 
# => {"draft"=>0, "published"=>1}
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

Post.statuses 
# => {"draft"=>0, "published"=>1}
{% endhighlight %}

There is one very dangerous issue related to the actual values assigned to
provided labels. Rails will enumerate the Array and use the index of the given
field to back the value in the database. If someone decides to add another
label in the front of the Array (for example `removed`), all values will
get shifted. Meaning of this current setting in all existing records will get
changed.

{% highlight ruby %}
class Post < ApplicationRecord
  enum status: [:removed, :draft, :published]
end

Post.statuses 
# => {"removed"=>0, "draft"=>1, "published"=>2}
{% endhighlight %}

There are many issues with number-backed enums. Regardless of the declaration
the readability gets crippled in many parts of the system. You can't tell at a
glance what is the meaning of underlying query when enums are translated to the
numbers by `ActiveRecord`.

{% highlight ruby %}

Post.removed.to_sql 
# => SELECT "posts".* FROM "posts" WHERE "posts"."status" = 0

{% endhighlight %}

When you have to build a sql-query by hand, you have to manually translate enums
to numbers. There are still many places in ActiveRecord syntax, where the
automatic translation between labels and values cannot be performed.

{% highlight ruby %}

User
  .joins(<<~SQL.squish)
    LEFT JOINS posts 
      ON posts.user_id = user.id 
        AND post.status = 1
  SQL

{% endhighlight %}

This particular query could be replaced with one using a `WHERE` clause, but
there are valid cases, when you might want to use custom syntax in other parts
of your SQL-query. Lets stick to this as an example for now. As for Rails 6 you
still cannot use bind wariables in `JOIN` syntax (as you can with `WHERE`), so
there is no other way to build the query.

Instad of explicit values you might be better of using values extracted from
the definition in the model. To be honest this is probably what you should be
doing if you are stuck with number based enums anyway. This way you can still
avoid "knowledge duplication". Any change of the values might require less
changes in the system.  Whats most imporant you can still see what is actually
going on here with refereing to the status definition in the model.

{% highlight ruby %}

status = Post.statuses.fetch("published")

User
  .joins(format(<<~SQL.squish, status: status))
    LEFT JOINS posts 
      ON posts.user_id = user.id 
        AND post.status = %<status>s
  SQL

{% endhighlight %}

This isn't all that bad, but if you ever end up somewhere looking at final query
you won't be able to what statuses are used by the query.

### Storing values as Strings

If you define an enum backed by a string, you can define any indentifier for
the values. Mostlikey you might want to use exaclty the same indentifier as the
one used in the Rails enum. This gets a bit mouthful, but there is no easier
way to define this kind of definition as of Rails 6. Using stings has some
serious readability definitions.

{% highlight ruby %}
class Post < ApplicationRecord
  enum status: { draft: "draft", published: "published" }
end

Post.statuses 
# => {"draft"=>"draft", "published"=>"published"}
{% endhighlight %}
