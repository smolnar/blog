---
layout: post
title: Eagerloading Polymorphic Associations in Rails
author: samuel
keywords: rails, rails 5, polymorphic, polymorphic associations, eagerloading, eagerloading polymorphic associations, nested associations, includes, includes polymorphic, includes polymorphic rails, n + 1 queries
---

_Originally posted in [Intuo Engineering](https://nerds.intuo.io/2018/03/01/eagerload-polymorphic-associations-in-rails.html)_

Eagerloading an association in ActiveRecord is a very handy feature that helps to cut down on the amount queries made to a database and successfully avoid N + 1 queries. When using ActiveRecord as an ORM, it's also very easy to load nested associations, e.g.

```ruby
posts = Post.all.includes(comments: { user: :role })
```

will fetch all posts and eagerload `comments` association with `user` and `role` association. As a result, calling `posts[0].commens[0].user.role` will make no additional SQL query.

This works very well with traditional ActiveRecord associations, but becomes tricky for _polymorphic_ associations. Let's assume that you have a model hierarchy such as this one.

```ruby
class View < ActiveRecord::Base
  belongs_to :viewable, polymorphic: true # viewable can be Post or Image
end

class Post < ActiveRecord::Base
  belong_to :author
  has_many :comments
end

class Image < ActiveRecord::Base
  belongs_to :user
end
```

If you'd like to call `View.all[0].viewable.user` on a view for an image, it will first fetch the viewable resource, `Image`, and then it will fetch `User` for the `user_id` specified in `Image`. So, in total 2 additional SQL queries.

Rails is smart enough to enable you to eagerload the `viewable` association as `View.includes(:viewable)`, but nothing more beyond that point. You cannot do `View.includes(viewable: :user)`, since how would it figure out from which model to preload `user` - `Post` or `Image`?

Luckily there's a workaround. You can define the association directly as

```ruby
class View < ActiveRecord::Base
  belongs_to :viewable, polymorphic: true # viewable can be Post or Image

  belongs_to :post
end
```

But since the `post_id` is not defined on the `View`, you need to include it and that is where `LEFT OUTER JOIN` comes to the rescue.

```ruby
class View < ActiveRecord::Base
  belongs_to :viewable, polymorphic: true # viewable can be Post or Image

  belongs_to :post

  def self.with_post
    joins(
      "
      LEFT OUTER JOIN posts ON posts.id = views.viewable_id AND views.viewable_type = 'Post'
    ",
    ).select('views.*, posts.id as post_id')
  end
end
```

And by calling `View.with_post.includes(post: [:author, :comments])`, you can eagerload any nested associations. The only difference now is that you need to call `view.post` to get the post, instead of calling `view.viewable` as before.

_Note that calling `includes(:post)` without `with_post` or calling `post` on a `view` instance will either result in an error or it will return `nil`, since no `post_id` was defined on the `view`._

Let us know in the comments below if you have any idea on how to improve this or if you have found a better way how to tackle this problem! :beers:t
