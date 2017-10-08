---
title: "Metaprogramming Callbacks in Rails"
layout: post
date: 2017-10-08 20:10
image: /assets/images/markdown.jpg
headerImage: false
tag:
- rails
- active_record
- callbacks
star: true
category: blog
author: tobi
description: Using Ruby's metaprogramming powers in modularising ActiveRecord callbacks in Rails.
---

I've been working with Agora, a really interesting start-up, part-time for a while now. Over at Agora, we act as ticket matchmakers by finding concert tickets at the price users are willing to pay. Contrasted against traditional ticketing platforms like StubHub and Seatgeek, the price for tickets is wholly determined by how much sellers want to price their tickets, with no input from customers wanting to buy tickets.

For each concert event available on Agora, we store cheapest price information for varying quantities between one and four. So at each point in time, an event could have different prices per ticket, for one, two, three or four tickets. The reason prices could be different is because we aggregate pricing information (basically tickets) from several sources, and crunch down in finding the cheapest options, or combinations for these quantities.

We had a `PriceInfo` model that just stored this data - price, and ticket-source for each of the different quantities we supported buying - 1, 2, 3 and 4. However, apart from just finding cheapest prices, we also store data about ticket prices over time. This data is useful in a lot of ways. From analysing price movements over time, to making data-backed forecasts of future pricing for that event, or answering questions such as how much disparity occurs between several ticket sources, and at what times these happen.

To scale down on the volume of data we store, we only store points at which price changes occur. Each time pricing for any quantity changes, we create a `PriceHistory` record with details about that change. 

Schema, alongsides definition for both models look similar to :

{% highlight ruby %}
# == Schema Information
#
# Table name: price_infos
#
#  id             :integer not null, primary key
#  price_for_1    :integer
#  price_for_2    :integer
#  price_for_3    :integer
#  price_for_4    :integer
#  source   :string

class PriceInfo < ApplicationRecord
  has_many :price_histories
end
{% endhighlight %}

{% highlight ruby %}
# == Schema Information
#
# Table name: price_histories
#
#  id       :integer not null, primary key
#  price    :integer
#  quantity :integer
#  price_info_id :integer not null

class PriceHistory < ApplicationRecord
  belongs_to :price_info
end
{% endhighlight %}

For each ticket-source, there's an associated `PriceInfo` model. Each time any of `price_for_1` down to `price_for_4` changed, a new `PriceHistory` would be created using the new price, the quantity, and linked to that ticket-source, through the `PriceInfo`. So say the price for one ticket dropped from $80 to $65 on TicketNetwork, a new price-history would be created using price - $65, quantity - 1, and a reference to the `PriceInfo` model for TicketEvolution.

Essentially, the process of creating a new price-history needed to be performed only if the any of the pricing fields changed.

A naive approach to achieve this would be to have different after-save callbacks defined for each `price_for_*` column. Apart from being redundant and lengthy, changing collective functionality would mean modifying different definitions, even though post-save behaviour for all four columns was exactly the same.

To tackle this in a concise manner, I used Ruby's meta<s>physical</s>programming powers:

{% highlight ruby %}
module PriceChangeWrapper
  extend ActiveSupport::Concern

  included do
    def self.price_loop
      1.upto(4) do |number|
        yield(number)
      end
    end

    price_loop do |quantity|
      define_method("price_update_callbacks_for_#{quantity}") do
        price = send("price_for_#{quantity}").to_f
        return unless price.positive?

        post_save_processing(quantity, price)
      end
    end

    price_loop do |quantity|
      after_save "price_update_callbacks_for_#{quantity}".to_sym, if: "price_for_#{quantity}_changed?"
    end
  end
end
{% endhighlight %}

{% highlight ruby %}
class PriceInfo < ApplicationRecord
  include PriceChangeWrapper
  has_many :price_histories

  def post_save_processing(quantity, price)
    price_histories.create(price: price, quantity: quantity)
  end
end
{% endhighlight %}

We use the `included` block, made available by `ActiveSupport::Concern`. This block is called each time the `PriceChangeWrapper` module is included in any class. In the block, we define a reusable `price_loop` method to perform operations for quantities 1, 2, 3, and 4.

The first call to `price_loop` uses Ruby's `define_method` class-method to create instance methods for all four quantities. These instance methods serve as the callback methods each time pricing for a particular quantity changes. The functionality there is pretty basic. Get the new price, and call post_save_processing only if the price is positive, with the quantity and new price as arguments.

The second call to `price_loop` records the after-save callbacks that are called by ActiveRecord each time the model is saved or updated. The caveat for calling each method is dependent on whether the price for that particular quantity changes. So, if only pricing for 2, and 4 tickets changed, only those two callback methods - `price_update_callbacks_for_2` and `price_update_callbacks_for_4` will be called.

With this approach, we could easily add in new functionality when pricing changes, such as sending out notifications to users who want to purchase at pricing thresholds close to that price, or auto-purchasing tickets for users whose preferred purchase price is more than, or equal to the new price.