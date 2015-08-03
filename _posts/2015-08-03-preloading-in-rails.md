---
layout: post
title: Preloading in Rails
---

##### **TLDR**; You may need to manually choose between [`preload`](http://apidock.com/rails/ActiveRecord/QueryMethods/preload) and [`eager_load`](http://apidock.com/rails/ActiveRecord/QueryMethods/eager_load) instead of using [`includes`](http://apidock.com/rails/ActiveRecord/QueryMethods/includes) if ActiveRecord gets confused.

Usually when I'm optimizing a long running query I'm looking for things like index misses or opportunities to eager load with [`includes`](http://apidock.com/rails/ActiveRecord/QueryMethods/includes).  Today I decided to finally make peace with the quirky AR preloading system &mdash; specifically a case when it creates a broken query.

Here's a contrived example to illustrate

{% highlight ruby %}
# app/blog.rb
class Blog < ActiveRecord::Base
  has_many :posts
  has_many :authors, through: :posts
end

# app/post.rb
class Post < ActiveRecord::Base
  belongs_to :blog
  belongs_to :author
end

# app/author.rb
class Author < ActiveRecord::Base
  has_many :posts
end

{% endhighlight %}

Simple enough.

Let's say we want to build a view that shows blogs containing posts by a specific author, and the latest post (by any author) for that blog:

{% highlight ruby %}
# app/blog.rb
class Blog < ActiveRecord::Base
  has_many :posts
  has_one :latest_post, ->{ order created_at: :desc }, class_name: 'Post'

  has_many :authors, through: :posts
  scope :with_author, ->(author){ joins(:authors).where "authors.id = ?", author.id }
end
{% endhighlight %}

So now we're producing SQL like

{% highlight ruby %}
blog.latest_post
{% endhighlight %}

{% highlight sql %}
SELECT "posts".*
FROM "posts"
WHERE "posts"."blog_id" = 1
ORDER BY "posts"."created_at" DESC
LIMIT 1
{% endhighlight %}

and

{% highlight ruby %}
Blog.with_author author
{% endhighlight %}

{% highlight sql %}
SELECT "blogs".*
FROM "blogs"

INNER JOIN "posts"
  ON "posts"."blog_id" = "blogs"."id"

INNER JOIN "authors"
  ON "authors"."id" = "posts"."author_id"

WHERE (authors.id = 1)
{% endhighlight %}

Looks good, but there is an N<sup>2</sup> query here when you combine them

{% highlight ruby %}
Blog.with_author(author).collect &:latest_post
{% endhighlight %}

{% highlight sql %}
SELECT "blogs".* FROM "blogs" INNER JOIN "posts" ON "posts"."blog_id" = "blogs"."id" INNER JOIN "authors" ON "authors"."id" = "posts"."author_id" WHERE (authors.id = 1)
SELECT "posts".* FROM "posts" WHERE "posts"."blog_id" = 1 ORDER BY "posts"."created_at" DESC LIMIT 1
SELECT "posts".* FROM "posts" WHERE "posts"."blog_id" = 1 ORDER BY "posts"."created_at" DESC LIMIT 1
SELECT "posts".* FROM "posts" WHERE "posts"."blog_id" = 1 ORDER BY "posts"."created_at" DESC LIMIT 1
SELECT "posts".* FROM "posts" WHERE "posts"."blog_id" = 2 ORDER BY "posts"."created_at" DESC LIMIT 1
SELECT "posts".* FROM "posts" WHERE "posts"."blog_id" = 2 ORDER BY "posts"."created_at" DESC LIMIT 1
SELECT "posts".* FROM "posts" WHERE "posts"."blog_id" = 2 ORDER BY "posts"."created_at" DESC LIMIT 1
-- ...
{% endhighlight %}

So we'll just eager load our associations with

{% highlight ruby %}
Blog.includes(:latest_post, :authors).with_author(author).collect &:latest_post
{% endhighlight %}

AR makes the query look insane, but here's the less ugly version

{% highlight sql %}
SELECT
  -- Blog attributes
  "blogs"."id" AS t0_r0,
  "blogs"."title" AS t0_r1,
  "blogs"."created_at" AS t0_r2,
  "blogs"."updated_at" AS t0_r3,

  -- Post attributes
  "posts"."id" AS t1_r0,
  "posts"."title" AS t1_r1,
  "posts"."blog_id" AS t1_r2,
  "posts"."author_id" AS t1_r3,
  "posts"."created_at" AS t1_r4,
  "posts"."updated_at" AS t1_r5,

  -- Author attributes
  "authors"."id" AS t2_r0,
  "authors"."name" AS t2_r1,
  "authors"."created_at" AS t2_r2,
  "authors"."updated_at" AS t2_r3

FROM "blogs"

-- Get the posts for the blog
INNER JOIN "posts"
  ON "posts"."blog_id" = "blogs"."id"

-- Get the authors for those posts
INNER JOIN "authors"
  ON "authors"."id" = "posts"."author_id"

-- ?????
-- umm.... latest posts I guess?
-- ?????
LEFT OUTER JOIN "posts" "latest_posts_blogs"
  ON "latest_posts_blogs"."blog_id" = "blogs"."id"

WHERE (authors.id = 1)
{% endhighlight %}

Okay, so that doesn't look right at all, but let's test it to be sure

{% highlight ruby %}
blog = Blog.create title: 'blurg'
author = Author.create name: 'somebody'
old_post = Post.create blog: blog, author: author, title: 'old post'
new_post = Post.create blog: blog, author: author, title: 'new post'

blog.latest_post.title
# => "new post"

# preloading the post without filtering
Blog.includes(:latest_post)
    .where(id: blog.id).first.latest_post.title
# => "new post"

#  filtering and preloading only the latest post
Blog.includes(:latest_post).with_author(author)
    .where(id: blog.id).first.latest_post.title
# => "new post"

# filtering and preloading latest post and authors
Blog.includes(:latest_post, :authors).with_author(author)
    .where(id: blog.id).first.latest_post.title
# => "old post"
{% endhighlight %}

Right, so it's specifically when we're trying to eager load an association that's joined that's screwing things up.

Let's look at the documentation here:

> [`includes(*args)`](http://apidock.com/rails/ActiveRecord/QueryMethods/includes): "Specify relationships to be included in the result set"

> [`preload(*args)`](http://apidock.com/rails/ActiveRecord/QueryMethods/preload): "Allows preloading of args, in the same way that includes does"

> [`eager_load(*args)`](http://apidock.com/rails/ActiveRecord/QueryMethods/eager_load): "Forces eager loading by performing a LEFT OUTER JOIN on args"

That's not terribly useful to disambiguate the differences between these methods.

This article ([3 ways to do eager loading (preloading) in Rails 3 & 4](http://blog.arkency.com/2013/12/rails4-preloading/)) has some useful information, but specifically it clues us into a few things

- [`includes`](http://apidock.com/rails/ActiveRecord/QueryMethods/includes) can dynamically choose between [`preload`](http://apidock.com/rails/ActiveRecord/QueryMethods/preload) and [`eager_load`](http://apidock.com/rails/ActiveRecord/QueryMethods/eager_load)
- [`preload`](http://apidock.com/rails/ActiveRecord/QueryMethods/preload) eager loads with separate queries
- [`eager_load`](http://apidock.com/rails/ActiveRecord/QueryMethods/eager_load) joins it to the current query

#### Contrary to the documentation of [`preload`](http://apidock.com/rails/ActiveRecord/QueryMethods/preload): it will actually **generate a separate query**.


So let's try it

{% highlight ruby %}
Blog.preload(:latest_post).includes(:authors).with_author(author)
    .where(id: blog.id).first.latest_post.title
# => "new post"
{% endhighlight %}

Cool, let's check the SQL it's generating with that query from before

{% highlight ruby %}
Blog.preload(:latest_post).includes(:authors).with_author(author)
    .collect &:latest_post
{% endhighlight %}

{% highlight sql %}
SELECT
  -- Blog attributes
  "blogs"."id" AS t0_r0,
  "blogs"."title" AS t0_r1,
  "blogs"."created_at" AS t0_r2,
  "blogs"."updated_at" AS t0_r3,

  -- No post attributes this time

  -- Author attributes
  "authors"."id" AS t1_r0,
  "authors"."name" AS t1_r1,
  "authors"."created_at" AS t1_r2,
  "authors"."updated_at" AS t1_r3

FROM "blogs"

-- Get the posts for the blog
INNER JOIN "posts"
  ON "posts"."blog_id" = "blogs"."id"

-- Get the authors for those posts
INNER JOIN "authors"
  ON "authors"."id" = "posts"."author_id"

-- No wacky aliased latest_posts_blogs join this time either

WHERE (authors.id = 1)


-- A separate eager-load query for the latest posts
SELECT "posts".*
FROM "posts"
WHERE "posts"."blog_id" IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
ORDER BY "posts"."created_at" DESC
{% endhighlight %}

Hey, it works!

So the moral of the story is

##### You may need to manually choose between [`preload`](http://apidock.com/rails/ActiveRecord/QueryMethods/preload) and [`eager_load`](http://apidock.com/rails/ActiveRecord/QueryMethods/eager_load) instead of using [`includes`](http://apidock.com/rails/ActiveRecord/QueryMethods/includes) if AR doesn't generate queries that make sense.
