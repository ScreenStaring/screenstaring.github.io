---
layout: post
title:  "Active Record's find_each & find_in_batches: Use error_on_ignored_order!"
date:   2020-11-18
category: blog
---

```
grep 'Scoped order is' log/*.log
```

Is your Rails application emitting this warning? Were you aware of this?


Several responses to [a recent question on Reddit](https://www.reddit.com/r/ruby/comments/jw8xli/how_to_deal_with_huge_object_allocations_caused/) about Active Record performance
suggested using [`.find_each`](https://api.rubyonrails.org/v6.0/classes/ActiveRecord/Batches.html#method-i-find_each) and
[`.find_in_batches`](https://api.rubyonrails.org/v6.0/classes/ActiveRecord/Batches.html#method-i-find_in_batches)
(there's also [`.in_batches`](https://api.rubyonrails.org/v6.0/classes/ActiveRecord/Batches.html#method-i-in_batches)).
While these methods are well-known, the trivial ways to prevent the subtle bug they can cause seem to be unknown or ignored.

## The Problem

You create an Active Record Relation with a defined ordering, e.g., `Model.order("some_column desc")`, then call `.find_each` or a similar
batch method on it. You expect this to return records ordered by `some_column` in descending order. It doesn't.

This is no secret. From [the docs](https://api.rubyonrails.org/v6.0/classes/ActiveRecord/Batches.html):

    NOTE: It's not possible to set the order. That is automatically set to ascending on the primary key (“id ASC”)
    to make the batch ordering consistent. Therefore the primary key must be orderable, e.g. an integer or a string.

When an order is set Active Record's default behavior is to output a warning:

    Scoped order is ignored, it's forced to be batch order.

Not ideal.

While a basic case like `MyModel.order("some_column").find_each` may be easy to identify, cases where the ordered Relation
is passed as an argument to other methods that then call one of the batch retrieval methods on it can be a bigger source of problems
(and technically it's not mutating the caller's argument).

Here's a somewhat recent case I encountered where a reservation should be made on the first available opening. "First available opening" depends on the
applicable scheduling algorithm. If the user requesting the reservation is a high roller, i.e. `PremiumMember`, they'll have access to
different set of openings than other users.

The names and most of the code have been changed/removed to protect the guilty.
Note that [the strategy pattern](https://en.wikipedia.org/wiki/Strategy_pattern) is used to apply the user-appropriate algorithm:
```rb
class ReservationStrategy
  # snip irrelevant code...
  def reserve(request)
    raise NotImplementedError
  end

  protected

  def make_reservation(openings, request)
    openings.find_each do |reservation|
      # reserve the first opening based on some common scheduling logic then break
    end
  end
end

class FirstComeFirstServed < ReservationStrategy
  def reserve(request)
    make_reservation(
      Reservation.soonest_available(request.time, request.quantity),
      request
    )
  end
end

class PremiumMember < ReservationStrategy
  def reserve(request)
    openings = Reservation.or(Reservation.open, Reservation.booked_on_or_around(request.time))
    # some propriety and confidential logic on openings and request that results in an ordering
    openings = openings.order(propriety_and_confidential_ordering)

    make_reservation(openings, request)
  end
end
```

And used like so:
```rb
strategy = user.reservation_strategy.constantize.new(user)
reservation = strategy.reserve(request)
```
The problem happens in `ReservationStrategy`. Since `find_each` is used the results are ordered by `id` and **not** the order defined by the
`ReservationStrategy` subclass. This can result in a reservation that's not made on the correct/soonest opening.

But the warning was output...

Tests?

Yes a test can catch this but only if you create Reservation records in non-chronological order (a good rule of thumb, really).
And if your test coverage is not 100% (whose is?), an order-breaking, bug causing call to `find_each` can slip through.

## Prevention Via `error_on_ignored_order`

I don't think the current default behavior is ideal for most (all?) scenarios.
Instead of outputting a warning when the ordering is discarded one can force the batch methods to raise an exception. There are 3 ways to do this:
1. [Set the `error_on_ignore` argument to `true`](https://api.rubyonrails.org/v6.0/classes/ActiveRecord/Batches.html#method-i-find_each)
1. Set `Model.error_on_ignored_order` to `true`
1. [Set `error_on_ignored_order` to `true` globally](https://guides.rubyonrails.org/v6.0/configuring.html#configuring-active-record)


I prefer 3. Set it and forget it. It's a sane default. Now calls like this:
```
Reservation.order("availabe_at desc").find_each { }
```

Result in:
```
ArgumentError (Scoped order is ignored, it's forced to be batch order.)
```

No silent warnings. No _silent_ bugs. Instead of scheduling the wrong date we don't schedule at all. You may even get a test failure somewhere.
Yay.

The cases where you want the warning are few –if any. When you do just override the default at the call site (option 1) or class-wide (option 2).

## Is _Anybody_ Overriding Active Record's Default?

This functionality [was added in Rails 5](https://github.com/rails/rails/pull/23417) but is anyone using it?
I have not seen it used in any of the many Rails applications I've worked on since 2016 when the option was added. At least not initially.
And a large portion of these applications had warnings in their logs.

Some quick DuckDuckGoing turns up only a mention of it on [fsainz.com](https://www.fsainz.com/2016/10/29/find-each-warnings/).
Nothing else outside of docs.

StackOverflow? Nothing: [1](https://stackoverflow.com/search?q=error_on_ignored_order), [2](https://stackoverflow.com/search?q=error_on_ignore)

GitHub? Nope: [1](https://github.com/search?q=%22active_record+error_on_ignored_order%22+language%3Aruby&type=Code),
[2](https://github.com/search?q=error_on_ignored_order+-repo%3Arails%2Frails+-filename%3Abatches_test.rb+-filename%3Acore.rb+-filename%3Abatches.rb&type=Code),
[3](https://github.com/search?l=Ruby&q=error_on_ignore+-repo%3Arails%2Frails+-filename%3Abatches_test.rb+-filename%3Acore.rb+-filename%3Abatches.rb&type=Code)

[The venerable Rubocop](https://rubystyle.guide/) has no rule for it. Maybe it should?

For applications created before Rails 5 I understand. Code was ported to be Rails 5 compatible, tests passed, and folks moved on to
whatever the backlog bequeathed. But for new applications, having this raise an error is an important bug preventing default. Does anyone else think so?
It's not clear.
