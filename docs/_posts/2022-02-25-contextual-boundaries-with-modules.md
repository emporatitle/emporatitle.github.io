---
title:  "Contextual boundaries with modules"
date:   2022-02-25 08:55:11 -0500
toc: true
toc_sticky: true
author: Tim Overly
---

We have a monolith and we think it is right choice.  The intersection of our business domain and engineering constraints won’t be the number of transactions we can run per second.  It will be the number of PRs we can ship in a week over the next two years.  With that in mind, developer speed is the highest priority, and we think that is a monolith.

# Problem to avoid

The problem that many design patterns are seeking to avoid is unnecessary coupling.  Coupling, if left untended, can grow into a tangled ivy patch of interwoven vines of code.  When you go to pull on a small piece of that code you quickly learn that you are also pulling out pieces that are around the corner in your neighbors yard.  Which is guarded by a 180 lb. ill-tempered and slobbery dog.  See how code coupling could lead to you losing a finger?  Shame on you.

Solving unnecessary coupling involves boundaries.  The best boundaries only let what you want through and not what you don’t.  If you can design your boundaries properly you get the benefit you want with the smallest cost you can pay for those benefits.  Monoliths are particularly susceptible to code coupling over time because they are (by definition) without boundaries.  There is nothing stopping code from reaching across the entire codebase to entrap other code.

# Taking the solution too far

I believe you can take many things to the point of diminishing or even negative returns.  This is possible with boundaries - you can create boundaries in your code that are so numerous and deep as to eliminate any gains you have made with your decoupling.  I have been careful to speak of unnecessary coupling, because there are many processes are are deeply coupled because that is the nature of the process.  My first and last name are coupled because they _are_ coupled when you are speaking about me.  If you are decoupling things that are by nature truly coupled, different pain will ensue.  You will simply move the logic and work up into the coordination layer and you will not have an improved design that is easier to modify over time.

_Where_ to draw your boundaries is actually much more difficult in many ways than _how_ to draw your boundaries.  However, I have also found that if you don’t have a method in your organization or codebase to draw boundaries, you never will.  That means first, set up a _how_ then you can at least have the discussion or experiment with the _where_.

# What we are optimizing for

## Macro level

Our solution was a solution for the macro level of coupling.  We would not be addressing coupling at a low level like within a class or between two items that are very closely related.

## Maintainable

This solution should be scalable across and should not require people to _remember_ to do something in a code review.

## Developer joy

This is a concept that is more of a feeling.  Does this solution inspire joy in our developers?  We should be looking to maximize this joy.

# Solution

First there are other thoughts out there on this topic.  The [article](https://medium.com/airtribe/enforcing-modularity-inside-a-rails-monolith-f856adb54e1d) by Thomas Pagram at Airtasker was a great starting point for us. A couple of the points that we ended up with at Empora were inspired by their work.  One is: Linting is a great way to enforce boundaries in our code base.  It is quick and the developer gets realtime feedback in their editor about what is happening.  The second was the use of the use of inbound and outbound serializers to only pass PORO’s across the boundaries.

When we started on our solution we asked "what kinds of interactions do we need to have that cross our boundaries?".  Where this question took us was to three types of items.

1. Events - these are notification or broadcast to the monolith at large.
2. Queries - these are requests for data from another boundary.
3. Commands - these are actions that are called on another boundary.

If we could design a solution that would allow us to enforce the use of only these objects when referencing different boundaries, and those would only allow us to pass data and not deep references, it would put us on the road to having decoupled code at the macro level.

## Modules as boundaries

As the Airtasker article also stated, we found using rails engines as boundaries to be pretty superficial.  Because of this, we decided that it wasn’t worth the additional time and effort to set them up and instead we would use the basic ruby `module` as the boundary.  This would have the added benefit of eliminating naming conflicts because no two classes could exist in the same module.  For example you could have two `User` classes in two different engines, but if we enforce modules then we would be forced to have `Foo::User` and `Bar::User`.

Below is a diagram of how the calls from three different core types interact with the module boundaries.

![solution_overview](https://docs.google.com/drawings/d/e/2PACX-1vTMbeiOPnYuzfM89ksBLOtdRozAVQr7RBSTOHQkgw6gWtvxDa_ahSWCmNeECwjfX-NHlutrdHenzMVV/pub?w=586&h=582)

### Events

Events are probably the most conventional items we have in the list because it is pretty standard in the Rails ecosystem to have some kind of pub/sub infrastructure.  In our case we made use of the railseventstore.org gem and project.  This has some nice to haves that help the project overall, including having the ability to have coorilation and causation ids on all events.  These additions make it easier to track a chain of events with more complicated interactions.

We have implemented a core event class to make it easier to pubish events with the rails event store and it looks like:

{% highlight ruby %}
module Core
  class Event < RailsEventStore::Event
    def self.publish(**kwargs)
      event_from(**kwargs).publish
    end

    def publish
      Rails.configuration.event_store.publish(self, stream_name: stream_name)
    end
  end
end
{% endhighlight %}

This core class can then be extended like the following:

{% highlight ruby %}
module Foo
  module Events
    class BarCreated < Core::Event
      attr_accessor :bar_id, :data

      def self.event_from(bar:)
        event = new(data: { bar_id: bar.id })
        event.bar_id = bar.id

        event
      end

      def stream_name
        "bar_#{@bar_id}"
      end
    end
  end
end
{% endhighlight %}

These classes make it very consistent and easy to fire events and prevents leaking database models across the boundaries with events.

{% highlight ruby %}
Foo::Events::BarCreated.publish(bar: bar_instance)
{% endhighlight %}

### Commands

Our commands are designed for when you want to proactivly make a change, i.e. a write, update, or delete.  We made the decision to make use of commands both across boundaries and also within a module.  This was done for consistency and also as a way to help controllers stay small.

In terms of strurcture our commands are pretty straightforward.  They are simple classes with a standard interface and response structure.  Here is the base class that we have settled on:

{% highlight ruby %}
{% endhighlight %}

### Queries

We have a REST API in our server, which means we already will have a way to serialize and deserialize our models.  The trick is, we don’t want to send anything over the http connection, so we set up a series of classes that make that easier.  For us we used the render from our include JSONAPI gem.

{% highlight ruby %}
module Core
  class Query
    RENDERER = JSONAPI::Serializable::Renderer.new

    NotImplementedError = Class.new(StandardError)

    def self.class_to_serializer_map
      raise NotImplementedError, 'The mapping of the model to serializer should be overridden in a module base query.'
    end

    def self.render(object)
      RENDERER.render(object, class: class_to_serializer_map)
    end
  end
end
{% endhighlight %}

This class would then be extended in each module like to indicate which serializer to use:

{% highlight ruby %}
module Foo
  class BaseQuery < Core::Query
    CLASS_TO_SERIALIZER_MAPPING = {
      'Foo::User': Foo::Queries::UserSerializer
    }.freeze

    def self.class_to_serializer_map
      CLASS_TO_SERIALIZER_MAPPING
    end
  end
end
{% endhighlight %}

Finally the resulting query interface:

{% highlight ruby %}
module Foo
  class UserQuery < Foo::BaseQuery
    def self.find(id:)
      render Foo::User.find(id: id)
    end
  end
end
{% endhighlight %}

We can mock the results of a given query in testing by returning the simple PORO that is created from this serialization.  To ensure that the result of the query’s PORO and the mocked PORO is the same we testing to ensure we have the same signature by asserting shared examples against the two outputs. Like the following:

{% highlight ruby %}
RSpec.describe Foo::UserQuery do
  shared_examples 'a user serializer' do
    it 'returns a valid user' do
      expect(subject.dig(:data, :id)).to be_present
    end
  end

  describe 'factory matches query' do
    subject { build(:user_query) }

    it_behaves_like 'a user serializer'
  end

  describe '.find' do
    subject { described_class.find(id: user.id) }

    it_behaves_like 'a user serializer'
  end
end
{% endhighlight %}

## Enforcement

To enforce our boundary conditions when we have specified specific gateways becomes pretty easy.  What we need to do is only allow our code to call the three permitted types of classes (queries, commands, and events) if that class is in a different module.  This enforement can be done with linting rules written in Rubocop, which gives the developer instant feedback on violations.

To enforce all the important pieces of our boundaries, we have the two rules: `ControllerDenyWrite` and `CrossModuleReference`.  The first rule, simply prevents write operations in controller classes to help keep them smaller and move that logic to commands.  The second is more involved, but prevents calling any module withing the app from a different module.  For example code within `Foo::User` cannot call `Bar::Service` directly, it must use one of the three permitted class types: `Query`, `Command`, or `Event`.

### Tracking and Ensuring Progress

It is hard to make sustained progress against a goal unless you can see how you are making progress.  Again Rubocop to the rescue!  A new rubocop rule usually some requires `Exclude`’s that are needed to get the existing issues to grandfather in.  This gives a perfect new place to run a report or carry out an action to help move the migration along.

We made use of a nice Github action plugin by [dorny](https://github.com/dorny/paths-filter) that gives you the changed files in a PR and used that against the custom rules we have that still have an a list of `Excluded` files.  Here is the entire script that gives a particular PR a pass or fail.

{% highlight ruby %}
require 'yaml'
require 'json'

changed_files = JSON.parse(ARGV[0])
rubocop_config = YAML.load_file('.rubocop.yml')

offending_files = []
rubocop_config.select{ |k, v| k.include?("Empora") && v["Exclude"] }.each do |name, value|
  value["Exclude"].each do |excluded_file|
    next unless changed_files.any? { |changed_file| changed_file.include? excluded_file }

    offending_files << {"rule": name, "file": excluded_file}
  end
end

if offending_files.length > 0
  puts "Changed files have rubocop rule exclusions."
  puts "Look into migrating these files to remove the offenses:"
  puts JSON.pretty_generate(offending_files)
  exit(1)
else
  exit(0)
end
{% endhighlight %}

# Conclusion

I wrote this article, but it is folly to think all these ideas and efforts are mine.  I am proud to be part of a team of very intelligent and capable engineers. We did this work as a team with ideas and contributions coming from all members.