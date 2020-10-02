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

## Storing values as Integers

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
still cannot use bind variables in `JOIN` syntax (as you can with `WHERE`), so
there is no other way to build the query.

Instad of explicit values you might be better of using values extracted from
the definition in the model. To be honest this is probably what you should be
doing if you are stuck with number based enums anyway. This way you can still
avoid "knowledge duplication". Any change of the values might require less
changes in the system. What's most important you can still see what is actually
going on here without checking the status definition in the model.

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
you won't be able to tell what statuses are used by the query.

## Storing values as Strings

If you define an enum backed by a string, you can define any identifier for
the values. Most likely you might want to use exactly the same identifier as the
one used in the Rails enum. 

{% highlight ruby %}
class Post < ApplicationRecord
  enum status: { draft: "draft", published: "published" }
end

Post.statuses 
# => {"draft"=>"draft", "published"=>"published"}
{% endhighlight %}

This gets a bit mouthful, but there is no easier way to define this kind
of definition as of Rails 6. Using stings has some serious readability
benefits.

{% highlight ruby %}
User
  .joins(<<~SQL.squish)
    LEFT JOINS posts 
      ON posts.user_id = user.id 
        AND post.status = 'published'
  SQL
{% endhighlight %}

Whenever you spot a raw SQL in specs or logs, or when you might need to
look at your raw tables in SQL client you will immidetly understand
what the query is doing. There is no hidden meaning, no need to
reference identifiers in other places. 

One serious drawback to this approach is of course the storage
used. With enums you can just use `smallint` type (because you wont need
more than 65k different enum values anyway). This will consume just 2 bytes per
row of your table. Strings on the other hand will much more space depending on
how long identifiers will you use.

If your table stores millions+ of records and you need an enum there, you
might want to sacrifice readability to save some space.

Another issue with string based enums raise when you tend to manually encode
the values in queries. And using strings will make you do that a lot
more. Typo in long identifier sometimes gets unnoticed as you just get
no rows as the result. Even though same might happen with
integers, it happens to me a lot more often with string based enums.

## Enter ENUM Type in Postgres

As stated by Postgres documentation: _Enumerated (enum) types are data
types that comprise a static, ordered set of values. They are equivalent
to the enum types supported in a number of programming languages. An
example of an enum type might be the days of the week, or a set of
status values for a piece of data._

There is no direct way to create an `ENUM` in Rails as for now, but you can
easily add some manual SQL in order to perform this.

{% highlight ruby %}
class CreatePostsStatusesEnum < ActiveRecord::Migration[6.0]
  def up
    execute <<-SQL.squish
      CREATE TYPE posts_statuses_enum 
        AS ENUM('draft', 'published');
    SQL
    
    change_column(
      :posts,
      :status,
      "posts_statuses_enum USING status::posts_statuses_enum",
    )
  end

  def up
    execute <<-SQL.squish
      DROP TYPE posts_statuses_enum;
    SQL
    
    change_column(:posts, :status, :string)
  end
end
{% endhighlight %}

This migration assumes that you already have a Rails enum backed by a
string. It creates Postgres `ENUM` type for your Posts and then changes
the type of the column in posts to newly defined enum. As long as your
current string values are consistent and contain only defined values,
the conversion will be straightforward. Otherwise Postgres will fail to
perform the change throwing an error and you will have to remove/fix 
any invalid entries.

{% highlight ruby %}
class Post < ApplicationRecord
  enum status: { draft: "draft", published: "published" }
end

Post.statuses 
# => {"draft"=>"draft", "published"=>"published"}
{% endhighlight %}

The same definition used previously for strings still works here as
before, so no changes in the model code are required. So, why did we
ever bother to use `ENUM` type in this case?

{% highlight ruby %}
Post.where("state = 'pubilshed'")
# => PG::InvalidTextRepresentation: ERROR: invalid input 
#    value for enum posts_statuses_enum: "pubilshed"

Post.published.to_sql 
# => SELECT "posts".* FROM "posts" WHERE "posts"."status" = 'published'
{% endhighlight %}

Whenever you will try to read or write the value from the enum field,
you will get an exception if the value is outside defined scope. No typo
will cause silent failure anymore. No more invalid results looking legit
just because there was typo or `nil` leaking from somewhere else.

As a side-benefit, ENUMs consume only four bytes on disk. They are pretty
much on-pair with integers when it comes to space, but hold all the
readability benefits of strings and provide extra safety from various
coding mistakes. 

## Final words

There is a gem
(activerecord-postgres_enum)[https://github.com/bibendi/activerecord-postgres_e
num] which might lower the friction for using ENUMs in your models, but I would
recommend against using external dependency just to save some cut'n'paste code
in your migrations. After introducing your first `ENUM` everything else can be
just duplicated with the magic of your editor.

You must choose names for your `ENUM` types carefully. If you define a lot of
different types you might get overwhelmed quickly. I would suggest to prefix
your names with table and column name if your `ENUM` is defined for one table
only. If you want to define an enum for more widespread use, some more generic
name like `enum_languages` can be more appropriate.
