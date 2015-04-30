---
layout: post
title: "Saving Base64 Encoded Data as Paperclip 4 Attachments"
date: 2015-04-30T08:57:54+02:00
---

Recently I had to implement part of the Rails backend of an app that allowed the user to upload pictures.
Those pictures are base64 encoded in the browser and sent to the API together with their content type and file name.

Request parameters would look like this:
{% highlight JSON %}
picture: {
  data: "base64encodedimage",
  content_type: "image/jpg",
  filename: "image.jpg"
}
{% endhighlight %}

The app already used [paperclip](https://github.com/thoughtbot/paperclip){:target="_blank"} 4, so we were going to use that here as well.

After some stumbling around and some more `Paperclip::AdapterRegistry::NoHandlerError`s, it turned out that it's actually quite easy to make paperclip happy.

With the usual model code (we have a `Picture` model with an attachment called `image`), just turn the parameters into a valid data URI in the controller action and manually pick the adapter:

{% highlight ruby linenos=table %}
picture_params = params[:picture]
encoded_picture = picture_params[:data]
content_type = picture_params[:content_type]
image = Paperclip.io_adapters.for("data:#{content_type};base64,#{encoded_picture}")
image.original_filename = picture_params[:filename]
@picture.image = image
{% endhighlight %}
