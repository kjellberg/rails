# Action Cable – Integrated websockets for Rails

Action Cable seamlessly integrates websockets with the rest of your Rails application.
It allows for real-time features to be written in Ruby in the same style
and form as the rest of your Rails application, while still being performant
and scalable. It's a full-stack offering that provides both a client-side
JavaScript framework and a server-side Ruby framework. You have access to your full
domain model written with ActiveRecord or your ORM of choice.

## Terminology

A single Action Cable server can handle multiple connection instances. It has one
connection instance per websocket connection. A single user may well have multiple
websockets open to your application if they use multiple browser tabs or devices.
The client of a websocket connection is called the consumer.

Each consumer can in turn subscribe to multiple cable channels. Each channel encapsulates
a logical unit of work, similar to what a controller does in a regular MVC setup. So
you may have a `ChatChannel` and a `AppearancesChannel`. The consumer can be subscribed to either
or to both. At the very least, a consumer should be subscribed to one channel.

When the consumer is subscribed to a channel, they act as a subscriber. The connection between
the subscriber and the channel is, surprise-surprise, called a subscription. A consumer
can act as a subscriber to a given channel any number of times. For example, a consumer
could subscribe to multiple chat rooms at the same time. (And remember that a physical user may
have multiple consumers, one per tab/device open to your connection).

Each channel can then again be streaming zero or more broadcastings. A broadcasting is a
pubsub link where anything transmitted by the broadcaster is sent directly to the channel
subscribers who are streaming that named broadcasting.

As you can see, this is a fairly deep architectural stack. There's a lot of new terminology
to identify the new pieces, and on top of that, you're dealing with both client and server side
reflections of each unit.


## A full-stack example

The first thing you must do is define your `ApplicationCable::Connection` class in Ruby. This
is the place where you authorize the incoming connection, and proceed to establish it
if all is well. Here's the simplest example starting with the server-side connection class:

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    protected
      def find_verified_user
        if current_user = User.find(cookies.signed[:user_id])
          current_user
        else
          reject_unauthorized_connection
        end
      end
  end
end
```

Then you should define your `ApplicationCable::Channel` class in Ruby. This is the place where you put
shared logic between your channels.

```ruby
# app/channels/application_cable/channel.rb
module ApplicationCable
  class Channel < ActionCable::Channel::Base
  end
end
```

This relies on the fact that you will already have handled authentication of the user, and
that a successful authentication sets a signed cookie with the `user_id`. This cookie is then
automatically sent to the connection instance when a new connection is attempted, and you
use that to set the `current_user`. By identifying the connection by this same current_user,
you're also ensuring that you can later retrieve all open connections by a given user (and
potentially disconnect them all if the user is deleted or deauthorized).

The client-side needs to setup a consumer instance of this connection. That's done like so:

```javascript
//app/assets/javascripts/application.js

//= require cable
```

```coffeescript
# app/assets/javascripts/application_cable.coffee

@App = {}
App.cable = Cable.createConsumer "ws://cable.example.com"
```

The ws://cable.example.com address must point to your set of Action Cable servers, and it
must share a cookie namespace with the rest of the application (which may live under http://example.com).
This ensures that the signed cookie will be correctly sent.

That's all you need to establish the connection! But of course, this isn't very useful in
itself. This just gives you the plumbing. To make stuff happen, you need content. That content
is defined by declaring channels on the server and allowing the consumer to subscribe to them.


## Channel example 1: User appearances

Here's a simple example of a channel that tracks whether a user is online or not and what page they're on.
(That's useful for creating presence features like showing a green dot next to a user name if they're online).

First you declare the server-side channel:

```ruby
# app/channels/appearance_channel.rb
class AppearanceChannel < ApplicationCable::Channel
  def subscribed
    current_user.appear
  end

  def unsubscribed
    current_user.disappear
  end

  def appear(data)
    current_user.appear on: data['appearing_on']
  end

  def away
    current_user.away
  end
end
```

The `#subscribed` callback is invoked when, as we'll show below, a client-side subscription is initiated. In this case,
we take that opportunity to say "the current user has indeed appeared". That appear/disappear API could be backed by
Redis or a database or whatever else. Here's what the client-side of that looks like:

```coffeescript
# app/assets/javascripts/cable/subscriptions/appearance.coffee
App.appearance = App.cable.subscriptions.create "AppearanceChannel",
  connected: ->
    # Called once the subscription has been successfully completed

  appear: ->
    @perform 'appear', appearing_on: @appearingOn()

  away: ->
    @perform 'away'

  appearingOn: ->
    $('main').data 'appearing-on'

$(document).on 'page:change', ->
  App.appearance.appear()

$(document).on 'click', '[data-behavior~=appear_away]', ->
  App.appearance.away()
  false
```

Simply calling `App.cable.subscriptions.create` will setup the subscription, which will call `AppearanceChannel#subscribed`,
which in turn is linked to original `App.consumer` -> `ApplicationCable::Connection` instances.

We then link `App.appearance#appear` to `AppearanceChannel#appear(data)`. This is possible because the server-side
channel instance will automatically expose the public methods declared on the class (minus the callbacks), so that these
can be reached as remote procedure calls via `App.appearance#perform`.

Finally, we expose `App.appearance` to the machinations of the application itself by hooking the `#appear` call into the
Turbolinks `page:change` callback and allowing the user to click a data-behavior link that triggers the `#away` call.


## Channel example 2: Receiving new web notifications

The appearance example was all about exposing server functionality to client-side invocation over the websocket connection.
But the great thing about websockets is that it's a two-way street. So now let's show an example where the server invokes
action on the client.

This is a web notification channel that allows you to trigger client-side web notifications when you broadcast to the right
streams:

```ruby
# app/channels/web_notifications.rb
class WebNotificationsChannel < ApplicationCable::Channel
   def subscribed
     stream_from "web_notifications_#{current_user.id}"
   end
 end
```

```coffeescript
# Somewhere in your app this is called, perhaps from a NewCommentJob
ActionCable.server.broadcast \
  "web_notifications_1", { title: 'New things!', body: 'All shit fit for print' }

# Client-side which assumes you've already requested the right to send web notifications
App.cable.subscriptions.create "WebNotificationsChannel",
  received: (data) ->
    new Notification data['title'], body: data['body']
```

The `ActionCable.server.broadcast` call places a message in the Redis' pubsub queue under the broadcasting name of `web_notifications_1`.
The channel has been instructed to stream everything that arrives at `web_notifications_1` directly to the client by invoking the
`#received(data)` callback. The data is the hash sent as the second parameter to the server-side broadcast call, JSON encoded for the trip
across the wire, and unpacked for the data argument arriving to `#received`.


## Configuration

The only must-configure part of Action Cable is the Redis connection. By default, `ActionCable::Server::Base` will look for a configuration
file in `Rails.root.join('config/redis/cable.yml')`. The file must follow the following format:

```yaml
production: &production
  :url: redis://10.10.3.153:6381
  :host: 10.10.3.153
  :port: 6381
  :timeout: 1
development: &development
  :url: redis://localhost:6379
  :host: localhost
  :port: 6379
  :timeout: 1
  :inline: true
test: *development
```

This format allows you to specify one configuration per Rails environment. You can also change the location of the Redis config file in
a Rails initializer with something like:

```ruby
ActionCable.server.config.redis_path = Rails.root('somewhere/else/cable.yml')
```

The other common option to configure is the log tags applied to the per-connection logger. Here's close to what we're using in Basecamp:

```ruby
ActionCable.server.config.log_tags = [
  -> request { request.env['bc.account_id'] || "no-account" },
  :action_cable,
  -> request { request.uuid }
]
```

For a full list of all configuration options, see the `ActionCable::Server::Configuration` class.

Also note that your server must provide at least the same number of database connections as you have workers. The default worker pool is set to 100, so that means you have to make at least that available. You can change that in `config/database.yml` through the `pool` attribute.

## Starting the cable server

As mentioned, the cable server(s) is separated from your normal application server. It's still a rack application, but it is its own rack
application. The recommended basic setup is as follows:

```ruby
# cable/config.ru
require ::File.expand_path('../../config/environment',  __FILE__)
Rails.application.eager_load!

require 'action_cable/process/logging'

run ActionCable.server
```

Then you start the server using a binstub in bin/cable ala:
```
#!/bin/bash
bundle exec puma -p 28080  cable/config.ru
```

That'll start a cable server on port 28080. Remember to point your client-side setup against that using something like:
`App.cable.createConsumer('ws://basecamp.dev:28080')`.

Note: We'll get all this abstracted properly when the framework is integrated into Rails.


## Dependencies

Action Cable is currently tied to Redis through its use of the pubsub feature to route
messages back and forth over the websocket cable connection. This dependency may well
be alleviated in the future, but for the moment that's what it is. So be sure to have
Redis installed and running.

The Ruby side of things is built on top of [faye-websocket](https://github.com/faye/faye-websocket-ruby) and [celluloid](https://github.com/celluloid/celluloid).



## Deployment

Action Cable is powered by a combination of EventMachine and threads. The
framework plumbing needed for connection handling is handled in the
EventMachine loop, but the actual channel, user-specified, work is handled
in a normal Ruby thread. This means you can use all your regular Rails models
with no problem, as long as you haven't committed any thread-safety sins.

But this also means that Action Cable needs to run in its own server process.
So you'll have one set of server processes for your normal web work, and another
set of server processes for the Action Cable. The former can be single-threaded,
like Unicorn, but the latter must be multi-threaded, like Puma.


## Alpha disclaimer

Action Cable is currently considered alpha software. The API is almost guaranteed to change between
now and its first production release as part of Rails 5.0. Real applications using the framework
are all well underway, but as of July 8th, 2015, there are no deployments in the wild yet.

So this current release, which resides in rails/actioncable, is primarily intended for
the adventurous kind, who do not mind reading the full source code of the framework. And it
serves as an invitation for all those crafty folks to contribute to and test what we have so far,
in advance of that general production release.

Action Cable will move from rails/actioncable to rails/rails and become a full-fledged default
framework alongside Action Pack, Active Record, and the like once we cross the bridge from alpha
to beta software (which will happen once the API and missing pieces have solidified).

Finally, note that testing is a unfinished/unstarted area of this framework. The framework
has been developed in-app up until this point. We need to find a good way to test both the framework
itself and allow the user to test their connection and channel logic.


## License

Action Cable is released under the MIT license:

* http://www.opensource.org/licenses/MIT


## Support

Bug reports can be filed for the alpha development project here:

* https://github.com/rails/actioncable/issues
