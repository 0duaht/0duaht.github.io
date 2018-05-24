---
title: "Manually saving PaperTrail Versions for a record"
layout: post
date: 2018-05-24 15:02
image: /assets/images/markdown.jpg
headerImage: false
tag:
- rails
- papertrail
- version
- manual
star: true
category: blog
author: tobi
description: Easily trigger saving a new papertrail-version for an ActiveRecord object. 
---

I ran into problems trying to do this earlier.

Most of the solutions I found online were for older versions of PaperTrail. The APIs being called on the solutions were either no longer available, been updated, or had been moved around.

Code to do this was basically:

{% highlight ruby %}
  def generate_version!(event = 'manual')
    data = {
      event: event,
      object: paper_trail.recordable_object,
      whodunnit: PaperTrail.whodunnit,
      item: self,
      object_changes: paper_trail.changes
    }

    PaperTrail::Version.create!(data.merge!(papertrail_metadata))
  end

  def papertrail_metadata
    paper_trail.merge_metadata_into({})
  end
{% endhighlight %}

Mixing this into an ActiveRecord class that `has_paper_trail` allows you call generate_version! on the object in order to create a PaperTrail version at that point in time.

This works with the base setup for PaperTrail. If you're integrating PaperTrail with a lotta mods, then you might need to tweak this to your use-case.