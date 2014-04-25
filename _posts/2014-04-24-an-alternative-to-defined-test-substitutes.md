---
layout: post
title: "An alternative to defined Test substitutes"
description: ""
category: ruby
tags: [testing, interface]
---
{% include JB/setup %}

In the previous posts on Interfaces, I defined concrete Test substitute classes that implemented one or more Interface methods for a certain scenario.

But you can, if you prefer, define these in your tests. Or use a mixture of both approaches.

If you define this in a spec helper module

{% highlight ruby %}
module SpecHelpers
 def self.substitute(&block)
   Class.new(Object, &block).new
 end
end
{% endhighlight %}

For reference, say the interfaces are defined as...

{% highlight ruby %}
module Interfaces
  ServiceInterface = Interface.new(run: '*args')
  ActionInterface = Interface.new(create: '', no_price: '', price: 'amount')
  QueryInterface = Interface.new(fetch: '')
end
{% endhighlight %}

You can then use the substitute method like this...

{% highlight ruby %}
require_relative 'spec_helper'

describe PriceService do
  context 'unit tests' do
    before do
      @context = SpecHelpers.substitute { include Interfaces::ActionInterface }
      @next_service = SpecHelpers.substitute { include Interfaces::ServiceInterface }
    end

    subject { described_class.new(@context, price_query: @query, next_service: @next_service) }

    context "the service can't get a price and it calls back the action" do
      before do
        @query = SpecHelpers.substitute do
            include Interfaces::QueryInterface
            def fetch()
              super
              nil
            end
          end
      end

      it 'calls its collaborators' do
        subject.subscribe(@context).run
        expect(@query).to have_correctly_invoked_fetch
        expect(@context).to have_correctly_invoked_no_price
      end
    end

    context "the service can get a price and it invokes the next service" do
      before do
        @query = SpecHelpers.substitute do
            include Interfaces::QueryInterface
            def fetch()
              super
              900
            end
          end
      end

      it 'calls its collaborators' do
        subject.subscribe(@context).run
        expect(@query).to have_correctly_invoked_fetch
        expect(@next_service).to have_correctly_invoked_run
      end
    end
  end
end
{% endhighlight %}

P.S. I love Ruby
