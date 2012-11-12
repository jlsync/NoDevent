[![build status](https://secure.travis-ci.org/cconstantine/NoDevent.png)](http://travis-ci.org/cconstantine/NoDevent)
[![IRC](https://fighting-mongeese.nko3.jit.su/gifs/nodejitsu/chat.gif)](http://http://githubchat.us/chats/nodejitsu)

NoDevent
=======
A system for notifying browser clients about events happening in a Ruby On Rails (or other non-evented backend) such as private messages, new discussion posts, or interesting resque jobs.  The goal is to reduce or elliminate the need for polling from the browser.  

See a demo at http://demo.nodevent.com

# Motivation

Both node.js and Rails are great systems.  I think rails is great for its maturity and feature set, and Node.js is great for socket.io.  There are times when I'm working in a Rails app, and I want some kind of socket.io interaction with the browser.  With NoDevent I can write all of my application logic in Rails, and syndicate the socket.io events out to the browser.  With careful organization of NoDevent events you can extend the View of MVC into the browser and have confidence that that when a Model changes, all Views of it change too.  

Quick* Start (Rails)
-----------
1) Install Node.js (http://nodejs.org/)

2) Install and start NoDevent with npm:

    sudo npm install -g nodevent
    npm start -g nodevent

3) Add the following to the gemspec

    gem 'nodevent'

nodevent expects a $redis global to exist and be a connect to your redis server.

4) Send some events

All events that will be sent out via NoDevent are published from redis.  To send an event you must know the namespace, room, event name, and message (if any).

The NoDevent appliance expects to get a message published in the redis channel coresponding to the nodevent namespace.  The data posted is expected to be a json object
with the keys room, event, message.


```redis
publish "the_namespace" "{room: 'the_room', event: 'the_event', message: 'the_message'}"
```

See http://redis.io/commands/publish for more information on the redis publish command

ActiveRecord Models:
```ruby
class SomeModel < ActiveRecord::Base
  include NoDevent::Base
end

#Emit a model instance as a json object to that model
SomeModel.first.emit('the_event', SomeModel.first)

#Or just emit a message
NoDevent::Emitter.emit('the_room', 'the_event', 'the_message')

#You can also use an active record model as the room.
NoDevent::Emitter.emit(SomeModel.first, 'the_event', 'the_message')
```

3) Connect and listen for events in the browser
Load the socket.io layer:
```
<%= javascript_include_nodevent %>
```

Connect to a room and listen for events
```javascript
  var room = NoDevent.room('theroom');
  room.join();
  theroom.on("the_event",function(data){
    /* Do stuff with data */
  });
  
```

Thats it!

*Ok, so it's not that quick.


# Documentation

## Appliance

The NoDevent server appliance is provided as an npm module/server.  Simply install the 'nodevent' module and you can 'npn start nodevent' to get started.  The appliance is stateless with respect to you rapplication, so you can run as many instances as you like.

### Configuration

NoDevent is configured via /etc/nodevent.json.  If you are running from source you can also provide the file to read as a command line parameter ('node server.js other_config.json').

An example config file.
```javascript
{
  "port" : 8080,
  "/nodevent" : { 
    "redis" : {"port" :6379 ,"host" : "localhost"}
  }
}
```
The above config us what is used by default if no config file is available.  It tells NoDevent to listen on port 8080, and have one namespace.  Each Namespace listens to a distinct redis instance for events.  Each namespace may also provide a secret for security.

You may provide a top-level ssl : {key : 'keyfile', cert: 'certfile'} if you want to use ssl.

A config with every option being used
```javascript
{
  "port" : 443,
  "ssl" : {
    "key" : "keyfilename",
    "cert" : "certfilename"
  }
  "/nodevent" : { 
    "redis" : {"port" :6379 ,"host" : "localhost"}
  }
  "/other_namespace" : { 
    "redis" : {"port" :6379 ,"host" : "redis.internal"},
    "secret" : "lajf0q983j4laidsnvqo84jfoqijflkjafds"
  }
}
```

### Security
It is possible to secure access to rooms with a secret key.  If a secret string has been provided for a room that room is secured and any client that wishes to join must provide a valid key.

A valid key is generated by:
```javascript
sha256(room + ts + secret)
```
where 'room' is the name of the room, ts is the experation time of the key, and secret is the configured secret.


### Upstart

You can use the following upstart script to start NoDevent whenever your system starts

```
description "nodevent"

start on (filesystem and net-device-up IFACE=lo)
stop on shutdown

respawn
respawn limit 99 5

script
    exec /usr/local/bin/npm start -g nodevent
end script
```

## Web

The web (javscript) component of NoDevent is for receiving events.  

### Connecting

```html
<script src='//localhost:8080/api/nodevent' type='text/javascript'></script
```

The above loads the required js, and initiates a socket.io connection to the '/nodevent' namespace.  After this script tag, the NoDevent system is ready to go.

### NoDevent object

```javascript
window.NoDevent
```

You do not need to wait to use this object.  It will be there after the script tag has executed and it's ready to go.

#### Events
To get notified of events happening to the socket connection used by NoDevent you may listen for events directly on the window.NoDevent object:

```javascript
// Called when a connection is established to the server
NoDevent.on('connect', function(){});

// Called when disconnected from the server
NoDevent.on('disconnect', function(){});
```

For more events, see https://github.com/LearnBoost/socket.io-client/#events

You can also ask NoDevent directly if it is connected with the connected() method.

```javascript
Nodevent.connected()
```

####  Rooms

To get events you must first get a room.

```javascript
var room = NoDevent.room('the_room')
```

The room object returned is where you manage all interactions with a room

##### Methods and Properties

Setting the key for a room (see security section above)
```javascript
room.setKey(key)
```

Joining a room:
```javascript
room.join(function(err){});
```
Initiates a request to join a room.  If a callback is provided it will be called a single time, with an err if the join failed.

Leaving a room
```javascript
room.leave(function(err){});
```
Initiates a request to leave a room.  If a callback is provided it will be called a single time, with an err if leaving the room failed.

If all you have is the room object you can get the room's name
```javascript
room.id;
```

##### Events

Every room object is an event emitter with two special events:

```javascript
room.on('join', function(err){})
```
This event is emitted every time a join to the room happens, or is attempted.  The err object will exist if there was an error joining the room

```javascript
room.on('leave', function(err){})
```
This event is emitted every time you leave the room.  The err object will exist if the leave was unexpected and indicate what happened.

To listen to an event that you sent from a backend process simply wait for the named event
```javascript
room.on('the_event', function(data){})
```

The data variable will have whatever was passed (if anything).

## Language specific support

Ruby: http://github.com/cconstantine/NoDevent-gem


## Versioning

I'm using the following versioning scheme:
```
x.y.z
```

A change in the 'x' indictes that there has been a backwards incompatible interface change.  A change in the 'y' indicates some new functionality or significant bug fix, and a change in the 'z' indictes a minor bug fix.  

Don't attempt to get a guage on how mature this system is by the version numbers.  I'm following this pattern so that automatic version matching systems like bundler and npm can update developers' packages and protect against backwards incompatible changes. The rubygem and npm package follow each other in version number on the 'x', and 'y', but not 'z'.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Added some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
