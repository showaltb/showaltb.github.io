---
title: "Decorate Devise current_user with Draper"
layout: post
category: Ruby on Rails
---

[Draper](https://github.com/drapergem/draper) is awesome.

I normally use the handy `decorates_assigned` method to decorate model
objects in controllers.

[Devise](https://github.com/plataformatec/devise) makes the
currently signed in user available through a `current_user` (or
`current_admin`, etc.) method. To decorate this with your Draper
`UserDecorator`, just add the following method to
`ApplicationController`:

```ruby
protected

def current_user
    super&.decorate
end
```

This just calls `decorate` on the model object returned by the Devise
helper. The `&.` is Ruby 2.3's [safe
navigation](http://mitrev.net/ruby/2015/11/13/the-operator-in-ruby/)
method and ensures that the call to `decorate` doesn't raise an
exception if no user is signed in (`current_user` is nil). For older
Ruby versions, you can use Rails' `#try` method:

```ruby
super.try(:decorate)
```

<i>h.t. [Ariejan de Vroom](https://ariejan.net/2012/04/14/decorating-devise-s-current_user-with-draper/) for the original idea.</i>
