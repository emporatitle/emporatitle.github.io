---
layout: post
title:  "Contextual boundaries with modules"
date:   2022-02-25 08:55:11 -0500
---

We have a monolith and we think it is right choice.  The intercetion of our business domain and engineering constraints won't be the number of trasnations we can run per second.  It will be the number of PR's we can ship in a week over the next two years.

I say that not discount the issues that can also arise from a monolith.  People around me know a favorite saying of mine is "It isn't the alway's or the nevers' that are issue.  It is the sometime's".  In this case, we don't always want to have contextural boundaries between systems (microservices) and we never want to have boundaries between systems (monolith).  We sometimes, what to have boundaries.  No getting around it, it is hard.

# Problem to avoid

# Taking the solution too far

# What we are optimizing for

## Developer joy

# Solution

## Reusable

https://medium.com/airtribe/enforcing-modularity-inside-a-rails-monolith-f856adb54e1d


{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}
