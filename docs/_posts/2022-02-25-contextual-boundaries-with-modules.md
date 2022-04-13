---
title:  "Contextual boundaries with modules"
date:   2022-02-25 08:55:11 -0500
toc: true
toc_sticky: true
author: Tim Overly
---

We have a monolith and we think it is right choice.  The intercetion of our business domain and engineering constraints won't be the number of transnations we can run per second.  It will be the number of PR's we can ship in a week over the next two years.  With that in mind, developer speed is the highest priority, and we think that is a monolith.

# Problem to avoid

The problem that many design patterns are seeking to avoid is unessassary coupling.  Coupling if left untended can grow into a tangled ivy patch of interwoven vines of code.  When you go to pull on a small piece of that code you quickly learn that you are also pulling out pieces that are around the corner in your neighbors yard.  Which is guarded by a 180 lb. ill tempered and slobbery dog.  See how code coupling could lead to you loosing a finger?  Shame on you.

Solving unessarray coupling involves boundaries.  The best boundaries only let what you want through and not what you don't.  If you can design your boundaries property you get the benifiet you want with the smallest cost you can pay for those benifets.  Monoliths are particularly suseptable to code coupling overtime because they are (by definition) without boundaries.  There is nothing stopping code from reaching across the entire codebase to entrap other code.

# Taking the solution too far

I believe you can take many things to the point of dimisishing or even negative returns.  This is possible with boundaries, you can create boundaries in your code that are so numerous and deep as to eliminate any gains you have made with your decoupling.  I have been careful to speak of unessarary coupling, because there are many processes are are deeply coupled because that is the nature of the process.  My first and last name are coupled because they _are_ coupled when you are speaking about me.  If you are decoupling things that are by nature truly coupled, different pain will ensue.  You will simply move the logic and work up into the corrdination layer and you will not an improved design that is easier to modify over time.

_Where_ to draw your boundaries is actually much more difficult in many ways than _how_ to draw your bondaries.  However I have also found that if you don't have a method in your organization or codebase to draw boundaries, you never will.  That means first, set up a _how_ then you can at least have the discussion or experiement with the _where_.

# What we are optimizing for

## Macro level

Our solution was a solution for the macro level of coupling.  We would not be addressing coupling at a low level like within a class or between two items that are very closely related.

## Maintainable

This solution should be scalable across and should not require people to _remember_ to do something in a code review.

## Developer joy

This is a concept that is more of a feeling.  Does this solution inspire joy in our developers?  We should be looking to maximize this joy.

# Solution

First there are other thoughts out there on this topic.  The [article](https://medium.com/airtribe/enforcing-modularity-inside-a-rails-monolith-f856adb54e1d) by Thomas Pagram at Airtasker was a great starting point for us. A couple of the points that we ended with here at Empora were inspired by their work.  One is Linting is a great way to enforce boundaries in our code base.  It is quick and the developer gets realtime feedback in thier editor about what is happening.  The second was the use of the use of inbound and outbound serializers to only pass PORO's across the boundaries.

When we started on our solution we asked "what kinds of interactions do we need to have that cross our boundaries?".  Where this question tooks us was to three types of items.

1. Events - these are notification or broadcast to the monolith at large.
2. Queries - these are requests for data from another boundary.
3. Commands - these are actions that are called on another boundary.

If we could design a solution that would allow us to enforce the use of only these objects when referencing different boundaries, and those would only allow us to pass data and not deep references, it would put us on the road to having decoupled code at the macro level.

## Modules as boundaries

As the Airtasker article also stated, we found using rails engines as boundaries to be pretty superficial.  Because of this, we decided that it wasn't worth the additiona time and effort to set them up and instead we would use the basic ruby `module` as the boundary.  This would have the added benefit of eliminating naming conflicts because no two classes could exist in the same mondule.  For example you could have two `User` classes in two different engines, but if we enforce modules then we would be forced to have `Foo::User` and `Bar::User`.

Below is a digram of how the calls from three different core types interact with the module boundaries.

![solution_overview](https://docs.google.com/drawings/d/e/2PACX-1vTMbeiOPnYuzfM89ksBLOtdRozAVQr7RBSTOHQkgw6gWtvxDa_ahSWCmNeECwjfX-NHlutrdHenzMVV/pub?w=586&h=582)

### Events

Events are probably the most conventional items we have in the list because it is pretty standard

### Commands

### Queries

We have a REST API in our server, which means we already will have a way to serialize and deserialize our models.  The trick is, we don't want to send anything over the http connection, so we set up a series of classes that make that easier.  For us we used the render from our include JSONAPI gem.

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

We can mock the results of a given query in testing by returning the simple PORO that is created from this serialization.  To ensure that the result of the query's PORO and the mocked PORO is the same we testing to ensure we have the same signature by asserting shared examples against the two outputs. Like the following:

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

To enforce our boundary conditions then becomes pretty easy.  Only allow your code to call the three allowed types of classes (queries, commands, and events) if that class is in a different module.

# Tracking and Ensuring Progress

It is hard to make sustained progress against a goal unless you can see how you are making progress.  Again Rubocop to the rescue!  A new rubocop rule usually some requires `Exclude`'s that are needed to get the existing issues to grandfather in.  This gives a perfect new place to run a report or carry out an action to help move the migration along.

We made use of a nice Github action plugin by [dorny](https://github.com/dorny/paths-filter) that gives you the changed files in a PR and used that against the custom rules we have that still have an a list of `Excluded` files.  Here is the entire script that gives a particular PR a pass or fail.

{% highlight ruby %}
require 'yaml'
require 'json'

changed_files = JSON.parse(ARGV[0])
rubocop_config = YAML.load_file('.rubocop.yml')

offending_files = []
rubocop_config.select{ |k, v| k.include?("Empora") && v["Exclude"] }.each do |name, value|
  value["Exclude"].each do |exluded_file|
    next unless changed_files.any? { |changed_file| changed_file.include? exluded_file }

    offending_files << {"rule": name, "file": exluded_file}
  end
end

if offending_files.length > 0
  puts "Changed files have rubocop rule exclusions."
  puts "Look into migrating these files to remove the offences:"
  puts JSON.pretty_generate(offending_files)
  exit(1)
else
  exit(0)
end
{% endhighlight %}

# Conclusion
