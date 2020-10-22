# Chatty

A simple rails starter for playing around with ActionCable.

Clone, or recreate:

-------------------

`rails new chatapp --database=postgresql --skip-spring --skip-turbolinks --skip-test`

##Create a View

Include jquery in Gemfile and application.js

```
#Gemfile

...

gem 'jquery'

...
```

```
#app/assets/javascripts/application.js

...

//= require jquery
//= require jquery_ujs

...
```

---------------------------

Create a RoomsController with a show action.
Set the root route to the rooms show
```ruby
#config/routes.rb

root to: 'rooms#show'
```
Create corresponding view with the following erb:

```html
#app/views/rooms/show.html.erb

<h1>Chat room</h1>

<form>
  <label>Say something:</label><br>
  <input type="text" data-behavior="room_speaker">
</form>

```

## Setup for ActionCable

Mount actioncable in routes

```ruby
#config/routes.rb
...

mount ActionCable.server => '/cable'
```
--------------------------
Verify cable.js:

```js
#app/assets/javascript/cable.js

(function() {
  this.App || (this.App = {});

  App.cable = ActionCable.createConsumer();

}).call(this);
```
-------------------------------
Add application meta tag
```html
#app/views/layouts/application.html.erb

<head>
...

<%= action_cable_meta_tag %>

...
</head>
```


## Create a Channel
```bash
rails g channel room speak
```

```ruby
 #app/channels/room_channel.rb

class RoomChannel < ApplicationCable::Channel
  def subscribed
    # stream_from "some_channel"
  end

  def unsubscribed
    # Any cleanup needed when channel is unsubscribed
  end

  def speak
  end
end
```
The subscribed method is a default that’s called when a client connects to the channel, and it’s usually used to 'subscribe' the client to listen to changes.

The speak action is a custom action that we created when we ran the generator. It will be used to receive data from its client-side representation.

```coffeescript
 #app/assets/javascripts/channels/room.coffee

App.room = App.cable.subscriptions.create "RoomChannel",
  connected: ->
    # Called when the subscription is ready for use on the server

  disconnected: ->
    # Called when the subscription has been terminated by the server

  received: (data) ->
    # Called when there's incoming data on the websocket for this channel

  speak: ->
    @perform 'speak'
```

In the JavaScript file, the client subscribes to the server through App.room = App.cable.subscriptions.create "RoomChannel".There are three default methods; connected , disconnected (which, as you might have guessed, handles the state of the connection), and received (which will handle how received data from the server-side will be handled). The speak method in the JavaScript file will be used to send data to its server-side representation.

Start your rails server by typing rails s and go to your JavaScript console. ActionCable client-side

Typing App.cable will show you what the actual representation of ActionCable looks like client-side. App.room is the channel we just created. By typing App.room.speak we call the speak() function from the room.coffee file, which then transmits data to the speak method in room_channel.

*try it by rails s*

## Transmit Data

The speak function should accept a parameter

```coffeescript
#app/assets/javascripts/channels/room.coffee

App.room = App.cable.subscriptions.create "RoomChannel",
  #rest of the code
  speak: (message) ->
    @perform 'speak', message: message
```
This will send a message object to the server side action #speak

```ruby
 #app/channels/room_channel.rb

  def speak(data)
    ActionCable.server.broadcast "room_channel", message: data['message']
  end
```

Now, the speak method will take the message and broadcast it to room_channel. But how do we get the broadcasts from it? In order to do that, we’ll assign all the subscribers to listen to it by using stream_from in the subscribed method.

```ruby
 #app/channels/room_channel.rb

class RoomChannel < ApplicationCable::Channel
  def subscribed
     stream_from "room_channel"
  end
  
  ...
  
end
```

Essentially, room_channel is the environment in the ActionCable server where the data comes and gets bounced back to all clients. We can receive the data from subscribed by using the received() function in room.coffee. Here’s how:

```coffeescript
#app/assets/javascripts/channels/room.coffee

  received: (data) ->
    alert(data['message'])

  #speak function

$(document).on 'keypress', '[data-behavior~=room_speaker]', (event) ->
  if event.keyCode is 13 # return/enter = send
    App.room.speak event.target.value
    event.target.value = ''
    event.preventDefault()
    
```

## List messages

We're going to update our show page to include a div for messages that we will populate with a message partial whenever a message is sent:

```html
#app/views/rooms/show.html.erb

<h1>Chat Room</h1>

<div id="messages">
  
</div>

<form>
  <label>Say Something</label><br>
  <input type="text" data-behavior="room_speaker">
</form>
```

```html
#app/views/messages/_message.html.erb

<div class=“message”>
  <p><%= message %></p>
</div>
```

Now, instead of creating an alert, lets reconfigure this to log all the messages sent

```ruby 
#app/channels/room_channel.rb

class RoomChannel < ApplicationCable::Channel
  def subscribed
    stream_from "room_channel"
  end

  def unsubscribed
    # Any cleanup needed when channel is unsubscribed
  end

  def speak(data)
    ActionCable.server.broadcast "room_channel", message_partial: render_message(data['message'])
  end

  private

  def render_message(message)
    ApplicationController.render(partial: 'messages/message', locals: {message: message })
  end
end

```

```coffeescript
#app/assets/javascripts/channels/room.coffee

...

  received: (data) ->
    # Called when there's incoming data on the websocket for this channel
    $('#messages').append data['message_partial']
    
...
```

