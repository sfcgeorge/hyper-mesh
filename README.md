# Synchromesh ![](logo.jpg?raw=true)

[Synchromesh](https://en.wikipedia.org/wiki/Manual_transmission#Synchromesh) provides multi-client synchronization for [reactive-record.](https://github.com/catprintlabs/reactive-record)

In other words browser 1 creates, updates, or destroys a model, and the changes are broadcast to all other clients.

Add the gem, setup your configuration, and synchromesh does the rest.

## Quick Start Guides

Use one of the following guides if you are in a hurry to get going.

The easiest way to get setup is to use the Pusher-Fake gem.  Get started with this [guide.](docs/pusher_faker_quickstart.md)

If you are already using Pusher follow this [guide.](docs/pusher_quickstart.md)

If you are on Rails 5 already, and want to try ActionCable use this [guide.](docs/action_cable_quickstart.md)

All of the above use websockets.  For ultimate simplicity use Polling as explained [here.](docs/simple_poller_quickstart.md)

## Overview

Synchromesh is built on top of Reactrb and ReactiveRecord.

+ Reactrb is a ruby wrapper on Facebook's React.js library.  As data changes on the client (either from user interactions or external events) Reactrb re-draws whatever parts of the display is needed.
+ ReactiveRecord uses Reactrb to render and then dynamically update your ActiveRecord models on the client.
+ Synchromesh broadcasts any changes to your ActiveRecord models as they are persisted on the server.

A minimal synchromesh configuration consists of a simple initializer file, and at least one *Policy* class that will *authorize* who gets to see what.

The initializer file specifies what transport will be used.  Currently you can use [Pusher](http://pusher.com), ActionCable (if using Rails 5), Pusher-Fake (for development) or a Simple Poller for testing etc.

Synchromesh also adds some features to the `ActiveRecord` `scope` method to optimize expensive scope updates.

## Authorization

Each application defines a number of *channels* and *authorization policies* for those channels and the data sent over them.

Policies are defined with *Policy* classes.  These are similar and compatible with [Pundit](https://github.com/elabs/pundit) but
you do not need to use the pundit gem (but can if you want.)

Examples:

```ruby
class ApplicationPolicy
  # define policies for the Application

  # all clients can connect to the Application
  always_allow_connection
end

class ProductionCenterPolicy
  # define policies for the ProductionCenter model

  # any time a ProductionCenter model is updated
  # broadcast the total_jobs_shipped attribute over the
  # application channel (i.e. this is public data anybody can see)
  regulate_broadcast do |policy|
    policy.send_only(:total_jobs_shipped).to(Application)
  end
end

class UserPolicy
  # define policies for the User channel and Model

  # connect a channel for each logged in user
  regulate_instance_connection { self }

  # users can see all but one field of their own data
  regulate_broadcast do |policy|
    policy.send_all_but(:gross_margin_contribution).to(self)
  end
end
```

For complete details see [Authorization Policies](docs/authorization-policies.md)

## Installation

If you do not already have reactrb installed, then use the reactrb-rails-generator gem to setup reactrb, reactive-record and associated gems.

Then add this line to your application's Gemfile:

```ruby
gem 'synchromesh'
```

And then execute:

    $ bundle install

Also you must `require 'synchromesh'` from your client side code.  The easiest way is to
find the `require 'reactive-record'` line (typically in `components.rb`) and add `require 'synchromesh'` directly below it.  

## Configuration

Add an initializer like this:

```ruby
# for rails this would go in: config/initializers/synchromesh.rb
Synchromesh.configuration do |config|
  config.transport = :simple_poller # or :none, action_cable, :pusher - see below)
end
# for a minimal setup you will need to define at least one channel, which you can do
# in the same file as your initializer.
# Normally you would put these policies in the app/policies/ directory
class ApplicationPolicy
  # allow all clients to connect to the Application channel
  regulate_connection { true } # or always_allow_connection for short
  # broadcast all model changes over the Application channel *DANGEROUS*
  regulate_all_broadcasts { |policy| policy.send_all }
end
```

### Action Cable Configuration

If you are on Rails 5 you can use ActionCable out of the box.

```ruby
#config/initializers/synchromesh.rb
Synchromesh.configuration do |config|
  config.transport = :action_cable
end
```

If you have not yet setup action cable all you have to do is include the `action_cable` js file in your assets

```javascript
//application.js
...
//= require action_cable
...
```

The rest of the setup will be handled by Synchromesh.

Synchromesh will not interfere with any ActionCable connections and channels you may have already defined.  

### Pusher Configuration

Add `gem 'pusher'` to your gem file, and add `//= require 'synchromesh/pusher'` to your application.js file.

```ruby
# typically config/initializers/synchromesh.rb
Synchromesh.configuration do |config|
  config.transport = :pusher
  config.opts = {
    app_id: '2xxxx2',
    key:    'dxxxxxxxxxxxxxxxxxx9',
    secret: '2xxxxxxxxxxxxxxxxxx2',
    encrypted: false # optional defaults to true
  }
  config.channel_prefix = 'syncromesh' # or any other string you want
end
```

### Pusher-Fake

You can also use the [Pusher-Fake](https://github.com/tristandunn/pusher-fake) gem while in development.  Setup is a little tricky.  First
add `gem 'pusher-fake'` to the development and/or test section of your gem file. Then setup your config file:

```ruby
# typically config/initializers/synchromesh.rb
# or you can do a similar setup in your tests (see this gem's specs)
require 'pusher'
require 'pusher-fake'
# The app_id, key, and secret need to be assigned directly to Pusher
# so PusherFake will work.
Pusher.app_id = "MY_TEST_ID"      # you use the real or fake values
Pusher.key =    "MY_TEST_KEY"
Pusher.secret = "MY_TEST_SECRET"
# The next line actually starts the pusher-fake server (see the Pusher-Fake readme for details.)
require 'pusher-fake/support/base' # if using pusher with rspec change this to pusher-fake/support/rspec
# now copy over the credentials, and merge with PusherFake's config details
Synchromesh.configuration do |config|
  config.transport = :pusher
  config.channel_prefix = "synchromesh"
  config.opts = {
    app_id: Pusher.app_id,
    key: Pusher.key,
    secret: Pusher.secret
  }.merge(PusherFake.configuration.web_options)
end
```

### Simple Poller Details

Setup your config like this:
```ruby
Synchromesh.configuration do |config|
  config.transport = :simple_poller
  config.channel_prefix = "synchromesh"
  config.opts = {
    seconds_between_poll = 5, # default is 0.5 you may need to increase if testing with Selenium
    seconds_polled_data_will_be_retained = 1.hour  # clears channel data after this time, default is 5 minutes
  }
end
```

## ActiveRecord Scope Enhancement

When the client receives notification that a record has changed Synchromesh finds the set of currently rendered scopes that might be effected, and requests them to be updated from the server.  

To give you control over this process Synchromesh adds some features to the ActiveRecord scope macro.  Note you must use the `scope` macro (and not class methods) for things to work with Synchromesh.

Synchromesh `scope` adds an optional third parameter and an optional block:

```ruby

class Todo < ActiveRecord::Base

  # Standard ActiveRecord form:
  # the proc will be evaluated as normal on the server, and as needed updates
  # will be requested from the clients
  scope :active, -> () { where(completed: true) }
  # In the simple form the scope will be reevaluated if the model that is
  # being scoped changes, and if the scope is currently being used to render data.

  # If the scope joins with other data you will need to specify this by
  # passing an array of the joined models:
  scope :with_recent_comments,
        -> () { joins(:comments).where('created_at >= ?', Time.now-1.week) },
        [Comments]
  # Now with_recent_comments will be re-evaluated whenever Comments or Todo records
  # change.  The array can be the second or third parameter.

  # It is possible to optimize when the scope is re-evaluated by attaching a block to
  # the scope.  If the block returns true, then the scope will be re-evaluated.
  scope :active, -> () { where(completed: true) } do |record|
    (record.completed.nil? && record.destroyed?) || record.previous_changes[:completed]
  end
  # In other words only reevaluate if an "uncompleted" record was destroyed or if
  # the completed attribute has changed.  Note the use of the ActiveRecord
  # previous_changes method.  Also note that the attributes in record are "after"
  # changes are made unless the record is destroyed.

  # For heavily used scopes you can even update the scope manually on the client
  # using the second parameter passed to the block:
  scope :active, -> () { where(completed: true) } do |record, collection|
    if (record.completed.nil? && record.destroyed?) ||
       (record.completed && record.previous_changes[:completed])
      collection.delete(record)
    elsif record.completed && record.previous_changes[:completed]
      collection << record
    end
    nil # return nil so we don't resync the scope from the server
  end

  # The 'joins-array' applies to the block as well.  in other words if no joins
  # array is provided the block will only be called if records for scoped model
  # change.  If an array is provided, then the additional models will be added
  # to the join filter.  However if any empty array is provided all changes will
  # passed.
  scope :scope1, [AnotherModel], -> () {...} do |record|
    # record will be either a Todo, or AnotherModel
  end

  scope :scope2, [], -> () { ... } do |record|
    # any change to any model will be passed to the block
  end

  # The empty join array can also be used to prevent a scope from ever being
  # updated:
  scope :never_synced_scope, [], -> () { ... }

  # Or if you prefer just pass any non-array value of your choice:
  scope :never_synced_scope, :no_sync, -> () {...}

end
```

## Common Errors

- No policy class:  
If you don't define a policy file, nothing will happen because nothing will get connected.  
By default synchromesh will look for a `ApplicationPolicy` class.
- Wrong version of pusher-fake  (pusher-fake/base vs. pusher-fake/rspec)  
See the Pusher-Fake gem repo for details.
- Forgetting to add require pusher in application.js file  
this results in an error like this:
```text
Exception raised while rendering #<TopLevelRailsComponent:0x53e>
    ReferenceError: Pusher is not defined
```  
To resolve make sure you `require 'pusher'` in your application.js file if using pusher.
- No create/update/destroy policies
You must explicitly allow changes to the models to be made by the client. If you don't you will
see 500 responses from the server when you try to update.  To open all access do this in
your application policy: `allow_change(to: :all, on: [:create, :update, :destroy]) { true }`

## Development

Specs run in rspec/capybara/selenium. To run do:

```
bundle exec rspec spec
```

You can run the specs in firefox by adding `DRIVER=ff` (best for debugging.)  You can add `SHOW_LOGS=true` if running in poltergeist (the default) to see what is going on, but ff is a lot better for debug.


## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/reactive-ruby/synchromesh. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](contributor-covenant.org) code of conduct.


## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
