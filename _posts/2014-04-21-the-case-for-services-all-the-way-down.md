---
layout: post
title: "The case for Services all the way down"
description: ""
category: ruby
tags: [services, testing, interface, dependency]
---
{% include JB/setup %}

When using Rails you may deside you want to move from framework to ruby ASAP, so in the POST/PUT/PATCH actions you would invoke a service object directly after establishing your context (resource, current_user, params, etc.) Lets assume you want to create an Order and that the process is composed of a number of co-ordinated services.  One might design this a hub and spoke or a pipeline.

The sub-processes might be: 1) Create User, 2) Create Order, 3) Create Fulfillment.

The CreateUser process, designed as a service, might send a welcome email and hit the Accounts endpoint.
The CreateOrder service might hit the Accounts endpoint and a Sales Intelligence endpoint
The CreateFulfillment service might hit the Accounts endpoint and an external Provider endpoint.

Hub and spoke

![spoke](/assets/images/spoke.svg)

Here the Action class co-ordinates the flow via callbacks, all part of the public interface.

Pipeline

![pipeline](/assets/images/pipeline.svg)

Here the Action class must setup the flow by telling each service who the next service is.

In reality each service would probably be a wrapper e.g. FindOrCreateUserService, because the CreateOrder action would be done as asynchronously (i.e. outside of a logged-in user session) or the order has been partially processed.

Writing a spec for the Action class in either design, requires quite a lot of setup.

The spec would have to assemble substitutes to pass into the Action class constructor.

In the Pipeline case, I'm beginning to think that we need some value objects, one to carry our create context (e.g. order params, user, callback target) and another for the services and their dependencies.

{% highlight ruby %}
class FindOrCreateUserService
  include Wisper::Publisher
  include Interfaces::ServiceInterface

  def initialize(context, dependencies)
    @context = context
    @user_query = dependencies.user_query || UserQuery.new(context)
    @user_service = dependencies.user_service || NewUserService.new(context, dependencies)
    @next_service = dependencies.next_service
  end

  def run(*)
    @context.user = @user_query.fetch

    if @context.user.nil?
      @user_service.subscribe(self).run
    end
    
    if @next_service
      @next_service.run
    else
      publish :user_created, @context.user
    end
  end

  def created(user)
    @context.user = user
  end
end
{% endhighlight %}

