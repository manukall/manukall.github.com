---
layout: post
title: "Building a Simple Chat App in Elixir With Phoenix and RethinkDB"
date: 2015-04-25T19:47:13+02:00
---
In this post we will explore how to connect [RethinkDBs](http://rethinkdb.com/){:target="_blank"} changefeeds to [Phoenix's](http://www.phoenixframework.org){:target="blank"} channels. We are going to build a chat app that stores messages in a RethinkDB database.
When a user connects, we listen to a changefeed on the database.
As soon as a new message row is inserted into the database, we push the JSON we receive from the changefeed directly to the user.

For this, we will use the very young [exrethinkdb](https://github.com/hamiltop/exrethinkdb){:target="blank"} library. I'm assuming Elixir and Phoenix are already set up.

First we create a new phoenix application:

    mix phoenix.new phoenix_exrethinkdb_chat --no-ecto

Then we need to add exrethinkdb as a dependency. As it is not published on hex yet, we fetch it from Github. In `phoenix_exrethinkdb_chat/mix.exs` change the deps method to:

{% highlight elixir linenos=table %}
defp deps do
   [{:phoenix, "~> 0.11"},
    {:phoenix_live_reload, "~> 0.3"},
    {:exrethinkdb, github: "hamiltop/exrethinkdb"},
    {:cowboy, "~> 1.0"}]
end
{% endhighlight %}

Next we let mix install our new dependency. To do this run the following command (inside our new project's directory):

    mix deps.get

Because we do not want to reconnect to RethinkDB every time a new request happens, we will connect to it once when our application starts and keep a reference to the connection.
We will use an `Agent` for this.
Create the file `lib/phoenix_exrethinkdb_chat/repo.ex` with the following content:

{% highlight elixir linenos=table %}
defmodule PhoenixExrethinkdbChat.Repo do
  def start_link do
    conn = Exrethinkdb.local_connection
    Agent.start_link(fn -> conn end, name: __MODULE__)
    {:ok, self}
  end

  def run(query) do
    Agent.get(__MODULE__, fn conn ->
      Exrethinkdb.run conn, query
    end)
  end
end
{% endhighlight %}


To actually connect to the database when our application starts, we need to add our repo to the supervisor tree. In `lib/phoenix_exrethinkdb_chat.ex` add the following to the `children` array:

    worker(PhoenixExrethinkdbChat.Repo, []),


Next we are going to add our HTML. We just want the page to display a list of messages and one input each for the user's name and the message.
Replace `web/templates/page/index.html.eex` with the following code, which is pretty much stolen from [chrismccord's phoenix chat app](https://github.com/chrismccord/phoenix_chat_example):

{% highlight html linenos=table %}
<div id="messages" class="container">
</div>
<div id="footer">
  <div class="container">
    <div class="row">
      <div class="col-sm-2">
        <div class="input-group">
          <span class="input-group-addon">@</span>
          <input id="username" type="text" class="form-control"
          placeholder="username">
        </div><!-- /input-group -->
      </div><!-- /.col-lg-6 -->
      <div class="col-sm-10">
        <input id="message-input" class="form-control" />
      </div><!-- /.col-lg-6 -->
    </div><!-- /.row -->
  </div>
</div>
{% endhighlight %}

We're also going to shamelessly steal the javascript from the example chat app. Put the following into `web/static/js/app.js`:

{% highlight js linenos=table %}
import {Socket, LongPoller} from "phoenix"

class App {

  static init(){
    var socket     = new Socket("ws://" + location.host +  "/ws")
    socket.connect()
    var $status    = $("#status")
    var $messages  = $("#messages")
    var $input     = $("#message-input")
    var $username  = $("#username")

    socket.onClose( e => console.log("CLOSE", e))

    socket.join("rooms:lobby", {})
      .receive("ignore", () => console.log("auth error") )
      .receive("ok", chan => {

        chan.onError( e => console.log("something went wrong", e) )
        chan.onClose( e => console.log("channel closed", e) )

        $input.off("keypress").on("keypress", e => {
          if (e.keyCode == 13) {
            chan.push("new:msg", {user: $username.val(), body: $input.val()})
            $input.val("")
          }
        })

        chan.on("new:msg", msg => {
          $messages.append(this.messageTemplate(msg))
          scrollTo(0, document.body.scrollHeight)
        })

        chan.on("user:entered", msg => {
          var username = this.sanitize(msg.user || "anonymous")
          $messages.append(`<br/><i>[${username} entered]</i>`)
        })
      })
      .after(10000, () => console.log("Connection interruption") )
  }

  static sanitize(html){ return $("<div/>").text(html).html() }

  static messageTemplate(msg){
    let username = this.sanitize(msg.user || "anonymous")
    let body     = this.sanitize(msg.body)

    return(`<p><a href='#'>[${username}]</a>&nbsp; ${body}</p>`)
  }

}

$( () => App.init() )

export default App
{% endhighlight %}

This needs jQuery. Download [this version](https://github.com/chrismccord/phoenix_chat_example/blob/46a9112a67010ccad283d6a70dd7426228498231/web/static/vendor/jquery.min.js){:target="_blank"} directly from chrismccord's repo to `web/static/vendor/jquery.min.js`.


You can start the server by running `mix phoenix.server` now and open <http://localhost:4000>{:target="_blank"}.
RethinkDB needs to be started for this. For toy projects like this, I prefer to start a local instance by simply running `rethinkdb` inside the projects folder.

If you check your browser console, you will see an error telling you that the app can't connect to your server via websocket. Let's fix that by adding a channel route.

Open `web/router.ex` and add the following:

{% highlight elixir linenos=table %}
socket "/ws", PhoenixExrethinkdbChat do
  channel "rooms:*", RoomChannel
end
{% endhighlight %}

This tells the router to listen for websocket connections at `/ws` and use the `PhoenixExrethinkdbChat.RoomChannel` module for all requests to channels matching `rooms:*`.

Next we need to create this module. Create the file `web/channels/room_channel.ex` with the following content:

{% highlight elixir linenos=table %}
defmodule PhoenixExrethinkdbChat.RoomChannel do
  use Phoenix.Channel
  require Logger
  alias Exrethinkdb.Query
  alias PhoenixExrethinkdbChat.Repo

  def join("rooms:lobby", message, socket) do
    send(self, :after_join)
    {:ok, socket}
  end

  def handle_info(:after_join, socket) do
    q = Query.table("messages")
    result = Repo.run(q)
    Enum.each(result.data, fn message -> push socket, "new:msg", message end)

    changes = Query.changes(q)
    |> Repo.run
    Task.async fn ->
      Enum.each(changes, fn change ->
        push socket, "new:msg", change["new_val"]
      end)
    end

    {:noreply, socket}
  end

  def handle_in("new:msg", msg, socket) do
    q = Query.table("messages")
    |> Query.insert(%{user: msg["user"], body: msg["body"]})
    Repo.run(q)

    {:reply, {:ok, msg["body"]}, assign(socket, :user, msg["user"])}
  end
end
{% endhighlight %}

The join method get's called when users open the website. It allows everyone to join the channel "rooms:lobby" and sends a `after_join` message. This is an Elixir/Erlang message (see <http://elixir-lang.org/getting-started/processes.html#send-and-receive>{:target="_blank"}) - not a chat message. This message is handled at line 12. We first send all existing chat-messages back to the browser (line 15). Then we subscribe to the messages-table's changefeed and send each new message as it comes (lines 17-23).


The last step before we can try our application is setting up the database. For now, it is easiest to just create the table manually. Run `iex -S mix` in the console to start IEx. In the IEx shell execute `Exrethinkdb.Query.table_create("messages") |> PhoenixExrethinkdbChat.Repo.run`.

This is all. Restart your server and reload the browser page. In IEx run the following and watch the new message appear in your browser.

`Exrethinkdb.Query.table("messages") |> Exrethinkdb.Query.insert(%{user: "kai", body: "hi"}) |> PhoenixExrethinkdbChat.Repo.run`
