---
title: "Rails, ActiveRecord and Callbacks"
layout: post
date: 2017-08-14 16:12
image: /assets/images/markdown.jpg
headerImage: false
tag:
- rails
- active_record
- callbacks
star: true
category: blog
author: tobi
description: Markdown summary with different options
---

A couple hours ago, I wrote a very basic callback method on an ActiveRecord model in a Rails app. I was working on a ticketing system that assigns tickets to new orders.

To do this, I had tickets stored in a different model, and then these were assigned to new orders in a post-create callback on orders.

Given my history with using `after_create` and background jobs, aptly described [here](http://www.justinweiss.com/articles/a-couple-callback-gotchas-and-a-rails-5-fix/), I was using hooks triggered by `after_commit on: :create`

Code was basically:

{% highlight ruby %}
class Order
  after_commit :attach_ticket_files, on: :create

  def attach_ticket_files
    valid_tickets = event.ticket_files.available.equal_or_below(price)
                         .limit(quantity)

    return if valid_tickets.empty?
    return if valid_tickets.count < quantity

    valid_tickets.update_all(order_id: id)
    update_fulfillment_fields_and_mail_user
  end

  def update_fulfillment_fields_and_mail_user
    update(auto_fulfilled: true)
    OrderMailer.etickets_ready(id).deliver_later
  end
end
{% endhighlight %}

Pretty straightforward, yeah? Tried testing, and then I started getting weird vibes off the log. The callback was being triggered multiple times, depending on the number of tickets available.

Put up some break-points, and then discovered the callback was being triggered each time this line ran:

{% highlight ruby %}
  update(auto_fulfilled: true)
{% endhighlight %}

Did a Google-search and ran into an issue already raised for this [here](https://github.com/rails/rails/issues/14493).

Apparently calling `update` from a callback being run for `after_commit on: :create` re-triggers all callbacks defined this way, which'll leave you with an infinite loop if the update action is being called unconditionally.

Although, this screams bug since these actions should only be run once, after a new record is saved, and the transaction committed, the only explanation plausible about this is that even though the commit has been completed, the create action which the commit is based off, wasn't complete before the update was triggered.

The work-around I ended up using was changing the update line to its skimpy counterpart:

{% highlight ruby %}
  update_columns(auto_fulfilled: true)
{% endhighlight %}

This performs the update without running any validations, callbacks or changing the `updated_at` column.