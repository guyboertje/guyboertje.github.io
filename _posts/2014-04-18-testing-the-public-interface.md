---
layout: post
title: "Unit testing, POODR and collaborator interactions"
description: ""
category: 
tags: []
---
{% include JB/setup %}

I recently read the POODR book. In it Sandi Metz talks about verifying incoming and outgoing messages of the public interface of the object being tested.

Its the Interface part that I want to focus on.

Consider this sequence diagram.

![action](/assets/images/action.svg)

The methods invoked are the API of each object

We want to create a test for the Action object

{% highlight ruby linenos %}
describe Action do
  context 'unit test using a mock' do
    before do
      @service = double('service')
      @persistor = Object.new
    end

    subject { described_class.new(@service, persistor: @persistor) }

    it 'calls its collaborators' do
      expect(@service).to receive(:run).with(@persistor)
      subject.create
    end
  end
end
{% endhighlight %}

Here we verify the behaviour by specifying the method that should be called and the args

However there is a way to construct the spec to be immune to the API method names

Consider this spec

{% highlight ruby linenos %}
describe Action do
  context 'unit test using a substitute implementation of the interface' do
    before do
      @service = SpecHelpers::ActionService.new
    end

    subject { described_class.new(@service) }

    it 'calls its collaborators' do
      subject.create
      expect(@service.invoked_correctly?).to be_true
    end
  end
end
{% endhighlight %}

If we define a module that represents the interface e.g. Runnable.rb

{% highlight ruby linenos %}
module Runnable
  def run(*args, &block)
    internal_run(*args, &block)
  end

  private

  def internal_run(*args, &block)
    # override in class
    # this is for testing
    @invoked_correctly = true
  end
end
{% endhighlight %}

Then we include this in any class that can be injected into the Action initializer. Perhaps we have a SubscribeService and an UnsubscribeService, but we also include this in our SpecHelpers::ActionService class too.  By overriding the method internal_run (you can call this method anything you want) in all included classes, they can be said to implement the interface.

{% highlight ruby linenos %}
module SpecHelpers
  class ActionService
    include Runnable
    def invoked_correctly?() !!@invoked_correctly; end
  end
end
{% endhighlight %}

and

{% highlight ruby linenos %}
class Service
  include Runnable

  private

  def internal_run(context)
    puts 'Service did something with context: ' + context.class.name
  end
end
{% endhighlight %}

The default implementation of internal_run in the module sets the ivar @invoked_correctly. You can be pretty specific about what invoked_correctly means, in terms of arguments passed and it can document how you expect the Interface to be used, directly in the interface and not scattered around in tests or implementations.

Or you could alse have a module just for testing that provides the invoked_correctly? method

{% highlight ruby linenos %}
module SpecHelpers::InvokedCorrectly
  def invoked_correctly?()
    !!@invoked_correctly
  end
end
{% endhighlight %}

then the SpecHelpers::ActionService would be

{% highlight ruby linenos %}
module SpecHelpers
  class ActionService
    include Runnable
    include InvokedCorrectly
  end
end
{% endhighlight %}

So if any aspect of the interface definition changed, as long as you you also changed the test helper implementations too (they are proper implementations after all) your spec should still pass.

Does this seem like too much ceremony? Some would say yes; I say no - but it works best if you always pass the dependancies into the class somehow.

so the the Service class might look like this
{% highlight ruby linenos %}
class Service
  include Runnable
  include Callbackable

  def initialize(resource, context,
      policy: CreateResourcePolicy.new(context, resource),
      persistor: ResourcePersistor.new(context, resource)
    )
    @resource, @context = resource, context
    @policy, @persistor = policy, persistor
  end

  private

  def internal_run
    callback(:denied) and return unless @policy.permit?
    if @persistor.persist
      callback :created
    else
      callback :errored
    end
  end
end
{% endhighlight %}

Here Policy and Persistor specific classes also implement interface modules.

If a set of test implementations e.g FailPolicy, FailPersistor, RaisesPersistor are built, these can be pushed into specs at any point to simulate variations.

