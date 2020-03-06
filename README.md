# Devise_JWT_Warden

![lock](./logo/lock.png)

## Devise with a Rails API

### Objectives

- Build a easy to use full featured option to handle User Auth with a Rails API

`$ rails new devise_auth_app --api`

Once every thing is installed move into the `cd devise_auth_app` directory

lets install the required gems

```Ruby
gem 'devise'
gem 'jwt'

# we also need to uncomment
gem 'rack-cors'
```

and now run`$ bundle install`

---

Now that we have the required gems in our app, lets run the devise installer

`$ rails g devise:install`

when running this you will get this output in the console, don't worry about any of this now

```
===============================================================================

Some setup you must do manually if you haven't yet:

  1. Ensure you have defined default url options in your environments files. Here
     is an example of default_url_options appropriate for a development environment
     in config/environments/development.rb:

       config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }

     In production, :host should be set to the actual host of your application.

  2. Ensure you have defined root_url to *something* in your config/routes.rb.
     For example:

       root to: "home#index"

  3. Ensure you have flash messages in app/views/layouts/application.html.erb.
     For example:

       <p class="notice"><%= notice %></p>
       <p class="alert"><%= alert %></p>

  4. You can copy Devise views (for customization) to your app by running:

       rails g devise:views

===============================================================================
```

---

### Build out the user model

One awesome feature of Devise is that it builds a full User model for you out of the box and can also be edited easily if needed. In only one line!

`$ rails g devise User`

open your `db/migrations/` folder and you will see the user migration that was generated by devise.

```ruby
# frozen_string_literal: true

class DeviseCreateUsers < ActiveRecord::Migration[6.0]
  def change
    create_table :users do |t|
      ## Database authenticatable
      t.string :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""

      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Rememberable
      t.datetime :remember_created_at

      ## Trackable
      # t.integer  :sign_in_count, default: 0, null: false
      # t.datetime :current_sign_in_at
      # t.datetime :last_sign_in_at
      # t.string   :current_sign_in_ip
      # t.string   :last_sign_in_ip

      ## Confirmable
      # t.string   :confirmation_token
      # t.datetime :confirmed_at
      # t.datetime :confirmation_sent_at
      # t.string   :unconfirmed_email # Only if using reconfirmable

      ## Lockable
      # t.integer  :failed_attempts, default: 0, null: false # Only if lock strategy is :failed_attempts
      # t.string   :unlock_token # Only if unlock strategy is :email or :both
      # t.datetime :locked_at


      t.timestamps null: false
    end

    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    # add_index :users, :confirmation_token,   unique: true
    # add_index :users, :unlock_token,         unique: true
  end
end
```

for now lets leave this alone, if you want to add more values just check out the Devise documentation.

`$ rails db:migrate`

---

At this point we need to set up our API with rack-cors and and we need to build our Devise Strategy to use Warden and JWT

`./config/initializers/cors.rb`

```ruby
# Be sure to restart your server when you modify this file.

# Avoid CORS issues when API is called from the frontend app.
# Handle Cross-Origin Resource Sharing (CORS) in order to accept cross-origin AJAX requests.

# Read more: https://github.com/cyu/rack-cors

Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'

    resource '*',
             headers: :any,
             methods: %i[get post put patch delete options head]
  end
end
```

`./config/initializers/devise.rb`

```ruby

## around line 268 ##

# ==> Warden configuration
  # If you want to use other strategies, that are not supported by Devise, or
  # change the failure app, you can configure them inside the config.warden block.
  #
  config.warden do |manager|
    # manager.intercept_401 = false
    manager.strategies.add :jwt, Devise::Strategies::JWT
    manager.default_strategies(scope: :user).unshift :jwt
  end
```

And at the end of the file we can place this code

```ruby
### Taking from GoRails.com ###
 module Devise
  module Strategies
    class JWT < Base
      def valid?
        request.headers['Authorization'].present?
      end

      def authenticate!
        token = request.headers.fetch('Authorization', '').split(' ').last
        payload = JsonWebToken.decode(token)
        success! User.find(payload['sub'])
      rescue ::JWT::ExpiredSignature
        fail! 'Auth token has expired'
      rescue ::JWT::DecodeError
        fail! 'Auth token is invalid'
      end
    end
  end
end
end
```

that is a the configuration we have to handle now we need to add a few controllers and end points.

---

we can start `$ rails s` and make sure there is no errors

lets configure the routes that the basic api will have

`./config/routes.rb`

```ruby
Rails.application.routes.draw do
  devise_for :users, controllers: { registrations: 'registrations' }
  namespace :api do
    namespace :v1 do
      get 'post/index'
      post :auth, to: 'authentication#create'
      get  '/auth' => 'authentication#fetch'
    end

    namespace :v2 do
      # Things yet to come
    end
  end
end
```

The line `devise_for` build a bunch of devise end points most of which we will not use but reference the Devise docs for more information.

And name spacing allows allows us to version our API without errors.

Now to configure `./app/controllers/application_controller.rb`

```ruby
class ApplicationController < ActionController::Base
  skip_before_action :verify_authenticity_token
end
```

We need to modify this class to inherit from ActionController::Base instead of ::API due to Devise
and since we did that Rails now expects all requests to come from special Rails forms the have built in Auth tokens, so we need to skip that check app wide
