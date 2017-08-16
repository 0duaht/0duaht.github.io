---
title: "ActiveAdmin, Paperclip and Multiple uploads"
layout: post
date: 2017-08-15 18:02
image: /assets/images/markdown.jpg
headerImage: false
tag:
- rails
- active_admin
- paperclip
star: true
category: blog
author: tobi
description: Processing multiple uploads with a Paperclip model and ActiveAdmin form
---

Yesterday, I introduced a model into a Rails application I was working on. The model's job was simply to store a file and a few meta-data related to the file. For this project, we used [Paperclip](https://github.com/thoughtbot/paperclip) to handle file attachments.

Code for this model was basically:

{% highlight ruby %}
# == Schema Information
#
# Table name: ticket_files
#
#  id               :integer          not null, primary key
#  pdf_file_name    :string
#  pdf_content_type :string
#  pdf_file_size    :integer
#  pdf_updated_at   :datetime
#  source           :string
#  price            :decimal(8, 2)
#

class TicketFile
  has_attached_file :pdf, s3_protocol: :https
  validates :pdf, presence: true
end
{% endhighlight %}

The implementation for this was pretty basic. Included schema information introduced by [Annotate Gem](https://github.com/ctran/annotate_models). Each ticket-file is stored with a source, price and a pdf attachment. Nothing special here.

However, the problem I was trying to tackle was allow uploading multiple files at once, and then instantiate a new ticket-file with the same meta-data for each of the files uploaded. 

So, say a user wants to create five ticket-files, instead of filling in the source, price and pdf five different times, allow the pdf input field accept multiple file selections, and then instantiate five ticket-files when the input field is submitted.

To create an [ActiveAdmin](https://github.com/activeadmin/activeadmin) page for this, I had to add this file:

{% highlight ruby %}
ActiveAdmin.register TicketFile do
  actions :all
  permit_params :source, :price, pdf: []

  form do |f|
    f.inputs 'Details' do
      f.semantic_errors *f.object.errors.keys
      f.input :source
      f.input :price
      f.input :pdf, input_html: { multiple: true }
    end
    f.actions
  end

  show do |ticket|
    attributes_table do
      row :id
      row :source
      row :price
      row :pdf do
        ticket.pdf.url
      end
      row :created_at
      row :updated_at
    end
  end

  controller do
    def create
      meta_data = permitted_params.slice(:source, :price)
      permitted_params[:pdf].each do |pdf|
        TicketFile.create(meta_data.merge(pdf: pdf))
      end
      redirect_to admin_ticket_files_path
    end

    def permitted_params
      @permitted_params ||= params.require(:ticket_file).permit(
        :source, :price, pdf: []
      )
    end
  end
end
{% endhighlight %}

To explain the code above, first solution to tackling this was allowing multiple file-selections when creating a new ticket-file from ActiveAdmin.

{% highlight ruby %}
f.input :pdf, input_html: { multiple: true }
{% endhighlight %}

The second was adding in a custom implementation of the `create` action for the model to loop through uploaded files, and instantiate a new ticket-file at each iteration, using the same metadata, which is depicted in the controller block:

{% highlight ruby %}
controller do
  def create
    meta_data = ticket_file_params.slice(:source, :price)
    ticket_file_params[:pdf].each do |pdf|
      TicketFile.create(meta_data.merge(pdf: pdf))
    end
    redirect_to admin_ticket_files_path
  end

  def ticket_file_params
    @ticket_file_params ||= params.require(:ticket_file).permit(
      :source, :price, pdf: []
    )
  end
end
{% endhighlight %}

And there you go, easing UX for this in a very simple manner.