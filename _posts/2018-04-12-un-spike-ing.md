---
title:  "Un-spike-ing"
date:   2018-04-12
categories: spikes refactoring
layout: posts
---

Writing code is often the best way to figure out what exactly needs to be done to solve a problem. In the Agile world, they call this a [Spike](http://agiledictionary.com/209/spike/).

A spike is when a software developer sits done and starts trying little bits of code in order to build a map of all they code they are going to need to eventually write.

It’s a way of quickly hacking out code to do stuff like:
- querying the database to see what’s in there and how it fits together
- calling 3rd party APIs to see how they work
- Testing the algorithm that Fred said would work in Slack to see if he’s right
- Anything else that might need verified before committing to a design of a solution.

Spikes usually result in a bunch of messy code that kinda/sorta does most of the things that the product is going to need to do in the end. To most programmers, it feels good to do a spike. The gratification that comes from building a production ready system comes much more quickly in a spike. Because of that, there’s a temptation to feel that most of the work is done when the spike is done.

Something like, “I just need to clean up the code from the Spike and open a PR”, is often said by a developer. This creates the impression that there’s not much work left to do. The bad news is that that impression is often false. The good news is that I’m going to tell you how to deal with that bad news right now.

## What’s tangibly different between Spike code and Production code?

### All the edge cases. I mean <span class="underline">all</span> of them.

One of the main differences between doing a spike and writing production code is what one does when an edge case becomes clear.

In production code, I immediately write a test and/or alter the code to cover that scenario. In a spike, I just keep on rolling. At best, I’ll make a note to myself about it.

These build up over time. Three days worth of hacking out spike code can leave a lot more of these un-handled cases in the code then you might expect.

### Long term maintainability

Code written during a spike often has much less thought about maintainability put into it.

Here’s an example from a spike I recently did that involved pulling all data about cohorts of students from an internal API.

API limit thing

```ruby
def cohort_ids
  open = cohorts_api_client.search_cohorts(status: 'open', limit: 5000)
  active = cohorts_api_client.search_cohorts(status: 'active', limit: 5000)
  upcoming = cohorts_api_client.search_cohorts(status: 'upcoming', limit: 5000)
  (open + active + upcoming).map(&:cohort_id).map(&:to_i).sort
end
```

We had significantly less than 5000 cohorts at the time. So for the purposes of my spike, setting the limit to 5000 and making one call was fine. But what would happen if we had more than 5000 cohorts with any one particular status? This code would miss them. The correct thing to do is to page through the results until there are none left.

The code that actually made it into production looked like this

```ruby
BATCH_SIZES = 100

def all
  %w(open active upcoming).map do |status|
    fetch(status: status)
  end.inject(&:+).uniq
end

protected

def fetch(status:, limit: BATCH_SIZES, offset: 0)
  batch = cohorts_api_client.search_cohorts(status: status, limit: limit, offset: offset)
  if batch.count == limit
    batch + fetch(status: status, limit: limit, offset: offset + limit)
  elsif batch.count < limit
    batch
  end
end
```

That’s a pretty important refactor. It’s also exactly the type of thing that can be easily missed when a coder is rushing to get spike code into production. The spike version of the code will work fine for now. But some day in the future (probably at the worst possible moment), there will be a big influx of new cohorts and that code is going to fall on its face.

### Testability

Spike code is typically not driven by writing tests, or the type of tests used to drive it are at a very high level.

For the spike code listed above, I drove the development of that with one high level integration test that involved running all of the code in the spike end to end. Driving code with unit tests, rather than integration tests, leads to much better designs. This ends up being a major cause of the less-than-production-ready-ness of spike code.

## What would the PR look like?

Another great thing to think about is what the Pull Request is going to look like. In most cases, without some deliberate effort it is going to be **way** too big.

I like to compare the branch with master (or develop) on Github so that I can see exactly what it would be like. That’s a good starting point for understanding that the spike code is in fact, not production ready.

## How do we get out of this mess?

### Take relevant bits of code and put them into some sort of notes document.

I’ll spare you a rant about how I love emacs/org-mode for this sort of thing with all my heart…

### Look at each one and type up what it does, how it does it, and (most importantly) what’s not perfect about it.

You might think you already know these things and can skip this step, but don’t!

### Create a new branch off of master (or develop or whatever) that does not include your spike code.

### Decide the smallest chunk of functionality that would make for a complete Pull Request

In my most recent spike, I’m aiming to turn it into pull requests that go the basically the following

-   Setup the credentials for accessing the APIs involved
-   Pulled the data from the apis
-   Setup the OO design to do the computations on that data
-   One PR for each field that was computed

This will be somewhere between 5 and 10 PRs

### Build that functionality. Reach for the code snippets in your notes from your spike as you need them.

### Take a special pass with each snippet you copy in to make sure all paths through are code are fully covered with unit tests and any short comings that you wrote up earlier are addressed.

### Open that PR, get feedback, make changes, merge.

### Go back to \`Create a new branch…\` and repeat until all the work is done

## There’s an art to this…

It’s not always going to be easy to do this. For example, Sometimes it will be really hard to break your spike into a bunch sane small PRs. Over time one develops a sense of taste for what’s going to work out best. The only way to develop that sense of taste is to try your best every time you find yourself in this situations.

Good Luck!
