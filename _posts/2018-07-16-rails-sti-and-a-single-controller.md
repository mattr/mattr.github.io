---
title: Rails, STI, and a single controller
layout: post
---

At [Katalyst](https://katalyst.com.au), we recently set up a [code challenge](https://github.com/katalyst/krumblr) for prospective employees. It's not limited to them by any means; if you're interested in Rails, it's a fun coding challenge. So fun, in fact, that I decided to try it out for myself. The code may eventually make its way to github; however, for now it exists only in a private repository on [Bitbucket](https://bitbucket.org) so as not to influence any prospective candidates. (Note to prospective candidates, if you found your way here: well done. Use all the resources you can find!)

I have specific ideas for what I want to achieve with this: a deep dive into some of the gems we use regularly ([Devise](https://github.com/plataformatec/devise), [FriendlyId](https://github.com/), [SimpleForm]() etc.); a chance to try out some of Rails' lesser known features; and a chance to play with some PostgreSQL features.

One of these was a deeper look at single-table inheritance (STI). The code project, Krumblr, supports the notion of 'post types', that is, you can create different types of posts based on a single ideal: text, video, image, etc. (It's not a new idea, in fact I unashamedly stole it from a Rails Rumble -- I think? -- challenge from a decade or so ago. At least I think that's where it was from; sadly [railsrumble.com](http://railsrumble.com) doesn't resolve, and the app itself disappeared from the internet after a while; I assume it wasn't worth the continued hosting costs at the time.)

So why not have child types of our base `Post` type which implement each of these? It is, perhaps, the exact use-case that STI is designed to support. The code challenge requires at minimum a post that can handle text. I won't go into the database design, but let's say that each post has its own unique attributes which store the data (e.g. in image and video posts, it would be the URL).

So `Post` becomes an abstract class (figuratively, not literally; adding `self.abstract_class = true` will actually break STI in Rails<sup id="fn_link_1">[1](#fn_1)</sup>).

## Question 1: How do we prevent the base class from being stored?

There's a couple of things we can do here.

1. We can raise an exception on initialize on the base class.

   ```ruby
   class Post < ApplicationRecord
      def initialize(*args)
       raise "not allowed" if self.class == Post
       super
     end
   end
   ```
2. We can ensure that a `Post` is never created by validating the presence of `type`, the column used by Rails to handle STI in the database. The base class doesn't have a value in this column; all children will have the class name as a string (e.g. `"TextPost"`).

   ```ruby
   class Post < ApplicationRecord
     # ...
     validates_presence_of :type
     # ...
   end
   ```

Option 1 is great if we don't ever want to initialize the base class, i.e. if we know (or want to enforce) the child classes type at initialization. Option 2 gives us a bit mor flexibility -- we can initialize a new post, but we can't save it unless it's got a `#type`.

In this situation, option 2 wins, since I want to be able to initialize a class without necessarily knowing its type.

## Question 2: How do we handle routing?

Every item that inherits from `ActiveRecord::Base`<sup id="fn_link_2">[2](#fn_2)</sup> has a method called `#model_name`. This returns an `ActiveModel::Name` object, which provides the values for things like `params`, polymorphic paths in `link_to`, `I18n` references etc. This is instantiated on each child class as well, so a `TextPost` class expects URLs like `text_posts_path`. It turns out that this can be overridden [fairly]() [easily]() for child classes:

```ruby
class Post < ApplicationRecord
  def self.inherited(child)
    child.model_name
      Post.model_name
    end
  end
end
```
Boom. We're back to using `posts_path` for everything, which also means we can continue to use polymorphic paths for our app, such as

```ruby
form_for [@blog, @post] do |f|
```
But wait, I hear you cry, what about our params?

## Question 3: How do we handle `params`?

Anyone familiar with Rails would know about `strong_parameters`.<sup id="fn_link_3">[3](#fn_3)</sup> It looks something like this:

```ruby
class PostsController < ApplicationController
  # ...
  def post_params
    params.require(:post).permit(:title, :body)
  end
  # ...
end
```


---
<sup id="fn_1">[1](#fn_link_1)</sup> That's a feature, not a bug. `self.abstract_class = true` is basically the *opposite* use case from STI; it's designed for when we have a base class that does not have a database table supporting it (i.e. abstract, not stored, such as `ApplicationRecord`). In our case, we have a single database-backed type with children.

<sup id="fn_2">[2](#fn_link_2)</sup> Actually it's anything that includes `ActiveModel::Name`, which in turn includes anything that inherits `ActiveModel::Model`, so you can get all that naming goodness even if you have non-persisted classes.

<sup id="fn_3">[3](#fn_link_3)</sup> I actually started with Rails back when you white- or blacklisted your `params` in the model; if you didn't have anything, all `params` were permitted by default. The move to `strong_parameters` is without a doubt a move for the better.