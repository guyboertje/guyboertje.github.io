---
layout: post
title: "More flesh on the Interface bones"
description: ""
category: ruby
tags: [testing, interface, dependency]
---
{% include JB/setup %}

I discovered, when preparing for a different post, that defining interfaces can be generalised.

One can subclass Module ([wat?](http://solnic.eu/2012/08/13/subclassing-module-for-fun-and-profit.html)) and provide a hash of signatures that define an interface...

{% highlight ruby %}
class Interface < Module
  def initialize(signatures)
    @signatures = signatures
  end

  def included(other)
    super
    define_interface
  end

  private

  def define_interface
    @signatures.each do |name, signature|
      test_ivar = "correctly_invoked_#{name.to_s.sub(/\?|\!\z/, '')}"
      class_eval <<-METHODS
  def #{name}(#{signature})
    # define in class, this is for testing
    @#{test_ivar} = true
  end
  def has_#{test_ivar}?() !!@#{test_ivar}; end
  METHODS
    end
  end
end
{% endhighlight %}

This means that one can define these Interface modules like this...

{% highlight ruby %}
module Interfaces
  PermittableInterface = Interface.new(permits?: '')
  ApplicableInterface = Interface.new(applicable?: '*args, &block')
  ServiceInterface = Interface.new(run: '*args')
  ActionInterface = Interface.new(create: '', failure: 'model', success: 'model', not_permitted: '')
end
{% endhighlight %}

Note: in the previous post [Unit testing, POODR and collaborator interactions]({% post_url 2014-04-18-testing-the-public-interface %}) I manually defined these Interfaces with a private method that was to be redefined in the included class but this is not necessary.

Then, in your test suite, you can define something like this...

{% highlight ruby %}
class BasePolicy
  include Interfaces::ApplicableInterface
  def applicable?() super; test_outcome; end
  private
  def test_outcome() fail "No Impl"; end
end

module NotApplicable
  private
  def test_outcome; false; end
end

module IsApplicable
  private
  def test_outcome; true; end
end

class DiscountablePolicy < BasePolicy
  include IsApplicable
end

class NonDiscountablePolicy < BasePolicy
  include NotApplicable
end
{% endhighlight %}

So rather than specifying mocks for dependencies, you can use real substitute objects that (should) have the same interface as the production object 

However, in the previous post I said that the spec would be immune to the details of the interface definition but this was naive, instead we get readible matchers...

{% highlight ruby %}
  expect(service).to have_correctly_invoked_run
  expect(discount_rule).to have_correctly_invoked_applicable
{% endhighlight %}

What if [Reek](https://github.com/troessner/reek/wiki) or [Rubocop](http://batsov.com/rubocop/) could be extended to warn for cases where an Interface has been included but not fully implemented?
