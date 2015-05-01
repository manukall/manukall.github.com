---
layout: post
title: "Automatically Building Your Phoenix Assets With Webpack"
date: 2015-05-01T12:26:16+02:00
---

By default [Phoenix](http://www.phoenixframework.org/){:target="_blank"} (currently version 0.12) comes with support for [brunch.io](http://brunch.io/){:target="_blank"}.
I prefer using [Webpack](https://webpack.github.io/){:target="_blank"} to build my assets, though.
Fortunately creating a Phoenix app that uses Webpack instead of brunch.io is pretty easy:

Create your new app with the `--no-brunch` flag:

{% highlight bash %}
mix phoenix.new /tmp/my_app --no-brunch
{% endhighlight %}

Initialize npm and install Webpack:

{% highlight bash %}
cd /tmp/my_app
npm init
npm install --save-dev webpack
{% endhighlight %}

Add webpack to the watchers array in your development config (`/tmp/my_app/config/dev.exs`):

{% highlight elixir %}
config :my_app, MyApp.Endpoint,
  ...
  watchers: [{Path.expand("node_modules/webpack/bin/webpack.js"), ["--watch", "--colors", "--progress"]}]
{% endhighlight %}

Add a webpack config (`/tmp/my_app/webpack.config.js`)

{% highlight js %}
module.exports = {
    entry: "./web/static/js/app.js",
    output: {
        path: "./priv/static/js",
        filename: "bundle.js"
    }
};
{% endhighlight %}

Include the generated js file in your application template (`/tmp/my_app/web/templates/layout/application.html.eex`).

Replace
{% highlight html+erb %}
<script src="<%= static_path(@conn, "/js/app.js") %>"></script>
<script>require("web/static/js/app")</script>
{% endhighlight %}

with
{% highlight html+erb %}
<script src="<%= static_path(@conn, "/js/bundle.js") %>"></script>
{% endhighlight %}

Create the source javascript file (`/tmp/my_app/web/static/js/app.js`):

{% highlight js %}
console.log("Hello from Webpack");
{% endhighlight %}

Start the server (`mix phoenix.server`), [open your browser](http://localhost:4000/){:target="_blank"} and check the console.
