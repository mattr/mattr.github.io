---
layout: post
title: Using FactoryBot in Rails Engines
---

I'm currently extracting a plugin used in a [Spina CMS](http://spinacms.com) site into a reusable module. And since it relies on a relation to an existing class in Spina, fixtures become a nightmare (since it relies on the users, and there's no easy way to set password digests using fixtures).

Enter `FactoryBot` (formerly `FactoryGirl`).

Unfortunately, even after following all the steps to install it, I kept getting the following error:

```
ArgumentError: Factory not registered: post
```

No luck searching for an answer. Plenty for it not working with RSpec or some other setup, but nothing with Minitest.

Thankfully, you can load `FactoryBot` in the console... (I've replaced the full path to the gem to `...` here readability.)

```ruby
>  FactoryBot.find_definitions
=> [".../test/dummy/factories", ".../test/dummy/test/factories", ".../test/dummy/spec/factories"]
```

Wait. That's not right. Except it sort of is. If you run `Rails.root` in your console, you'll see that it is actually `test/dummy` from your gem. So our factories simply aren't where `FactoryBot` is expecting them.

Add the following line into `test_helper.rb`, just above the `FactoryBot.find_definitions` line:

```ruby
FactoryBot.definition_file_paths << File.expand_path('../factories', __FILE__)
```

That puts the `test/factories` directory of your gem into the load path. Suddenly everything is working. Well, the tests are failing in the expected manner, at least.

