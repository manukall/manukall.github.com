---
layout: post
title: "Cleaning Up Your Asynchronous Elixir Tasks in Your Tests"
date: 2015-08-03T12:53:22+02:00
---

Asynchronous tasks in elixir are a great way of performing actions that your main code does not have to wait for.

If you start them using `Task.start(...)` they will not be linked to the current process. This means an error in the task can not crash your current process.

In tests this can lead to unexpected behaviour though, as tasks might still be executing when your test has already finished.
I ran into this, when a task was still trying to access database resources that were already deleted by the tests `on_exit` function.

To work around that, I added a `Task.Supervisor` to my projects supervision hierarchy and started my tasks through that.
This allows you to get a reference to all tasks in your tests and handle them by either waiting for or killing them.

To do that, just add the supervisor to the `children` array in your apps `start` callback:

{% highlight elixir %}
children = [
  ...
  supervisor(Task.Supervisor, [[name: MyApp.TaskSupervisor]]),
  ...
]
{% endhighlight %}


And in your tests handle the tasks accordingly:

{% highlight elixir %}
Task.Supervisor.children(MyApp.TaskSupervisor)
|> Enum.map(fn child -> Task.Supervisor.terminate_child(MyApp.TaskSupervisor, child) end)
{% endhighlight %}
