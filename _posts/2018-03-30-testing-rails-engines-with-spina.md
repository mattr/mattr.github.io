---
layout: post
title: Testing Rails Engines with Spina
---

As mentioned in the [previous post](/2018/03/29/factory-bot-in-rails-engine/), I've been working on a Rails plugin gem for use in [Spina CMS](http://spinacms.com). After a couple of tweaks (defining factories for users and accounts, since the user is linked to the blog post as an author, and the account is required for the admin controller inheritance) I was ready to start testing again.

... and hit a major issue. I kept getting an issue saying `plugins` was not defined.
```bash
ActionView::Template::Error: undefined method `plugins' for nil:NilClass
```

Applying a backtrace on the test, it tells me that it's on line 22 of /layouts/spina/admin/admin.html.haml, which we can find in the spina gem:
```haml
- Spina::Plugin.all.each do |plugin|
```

`Spina::Plugin` is definitely defined, however (can output to the console in the test, for example). It's actually the following line that's the problem:
```haml
- if current_theme.plugins.include? plugin.name
```

`current_theme` is the culprit here. In our `accounts.rb` factory file, the account is defined as (taken wholesale from the `spina` gem):
```ruby
FactoryBot.define do
  factory :account, class: Spina::Account do
    name 'My Website'
    preferences { { theme: 'demo' } }
  end
end
```
Spina defines themes under `config/initializers/themes`. This doesn't exist in the default app created under `test/dummy`, nor the `spina.rb` initalizer.

So you need to add them to the dummy app to load things correctly. I haven't tried with the rake tasks/rails generators, but you should be able to do it by calling `rails g spina:install` from *inside* `test/dummy` (don't do it from the root, or it will install Spina in your engine rather than in the test app). Whichever of the themes you choose, it should match your theme definition in your factory (and so should the site name, for consistency). Or if you want to do it manually, you'll need the following:

```ruby
# config/initializers/spina.rb
Spina.configure do |config|
  # Set locales
  config.locales = [:en]
  # Run `rake spina:update_translations` after you add any new locale.

  # Important Note
  # ==============

  # You MUST restart your server before changes to this file
  # will take effect.

  # Specify a backend path. Defaults to /admin.
  # config.backend_path = 'admin'

  # Pages Options
  # ===============

  # Note that you might need to remove cached asset after changing this value
  # config.max_page_depth = 5
end
```
I'm not 100% sure that this is required for the test app, but really, it should reflect a real-world case as much as possible, right?

And for the theme, we'll just use the default theme:

```ruby
# config/initializers/themes/default.rb
::Spina::Theme.register do |theme|

  theme.name = 'default'
  theme.title = 'Default Theme'

  theme.page_parts = [{
    name:           'text',
    title:          'Text',
    partable_type:  'Spina::Text'
  }]

  theme.view_templates = [{
    name:       'homepage',
    title:      'Homepage',
    page_parts: ['text']
  }, {
    name: 'show',
    title:        'Default',
    description:  'A simple page',
    usage:        'Use for your content',
    page_parts:   ['text']
  }]

  theme.custom_pages = [{
    name:           'homepage',
    title:          'Homepage',
    deletable:      false,
    view_template:  'homepage'
  }]

  theme.navigations = [{
    name: 'mobile',
    label: 'Mobile'  
  }, {
    name: 'main',
    label: 'Main navigation',
    auto_add_pages: true
  }]

end
```
Since our theme in tests is named `demo` we either need to change this to `default`, or alternatively, we can copy/move `default.rb` in the themes folder to `demo.rb` and update the name.

As long as you have a valid theme definition here, controller tests will start to work.

You can check out the work-in-progress gem [here](https://github.com/mattr/spina_blog).
