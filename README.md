# Ahoy Email

First-party email analytics for Rails

:fire: For web and native app analytics, check out [Ahoy](https://github.com/ankane/ahoy)

:bullettrain_side: To manage unsubscribes, check out [Mailkick](https://github.com/ankane/mailkick)

[![Build Status](https://github.com/ankane/ahoy_email/workflows/build/badge.svg?branch=master)](https://github.com/ankane/ahoy_email/actions)

## Installation

Add this line to your application’s Gemfile:

```ruby
gem 'ahoy_email'
```

And run the generator. This creates a model to store messages.

```sh
rails generate ahoy_email:install
rails db:migrate
```

## Getting Started

There are three main features:

- [Message history](#message-history)
- [UTM tagging](#utm-tagging)
- [Open & click analytics](#open--click-analytics)

## Message History

Ahoy Email creates an `Ahoy::Message` record for each email sent by default. You can disable history for a mailer:

```ruby
class CouponMailer < ApplicationMailer
  track message: false # use only/except to limit actions
end
```

Or by default:

```ruby
AhoyEmail.default_options[:message] = false
```

### Users

Ahoy Email records the user a message is sent to - not just the email address. This gives you a history of messages for each user, even if they change addresses.

By default, Ahoy tries `@user` then `params[:user]` then `User.find_by(email: message.to)` to find the user.

You can pass a specific user with:

```ruby
class CouponMailer < ApplicationMailer
  track user: -> { params[:some_user] }
end
```

The user association is [polymorphic](https://railscasts.com/episodes/154-polymorphic-association), so use it with any model.

To get all messages sent to a user, add an association:

```ruby
class User < ApplicationRecord
  has_many :messages, class_name: "Ahoy::Message", as: :user
end
```

And run:

```ruby
user.messages
```

### Extra Attributes

Record extra attributes on the `Ahoy::Message` model.

Create a migration to add extra attributes to the `ahoy_messages` table. For example:

```ruby
class AddCouponIdToAhoyMessages < ActiveRecord::Migration[6.1]
  def change
    add_column :ahoy_messages, :coupon_id, :integer
  end
end
```

Then use:

```ruby
class CouponMailer < ApplicationMailer
  track extra: {coupon_id: 1}
end
```

You can use a proc as well.

```ruby
class CouponMailer < ApplicationMailer
  track extra: -> { {coupon_id: params[:coupon].id} }
end
```

## UTM Tagging

Use UTM tagging to attribute a conversion (like an order) to an email campaign. If you use [Ahoy](https://github.com/ankane/ahoy) for web analytics:

1. Send an email with UTM parameters
2. When a user visits the site, Ahoy will create a visit with the UTM parameters
3. When a user orders, the visit will be associated with the order (if [configured](https://github.com/ankane/ahoy#associated-models))

Add UTM parameters to links with:

```ruby
class CouponMailer < ApplicationMailer
  track utm_params: true # use only/except to limit actions
end
```

The defaults are:

- `utm_medium` - `email`
- `utm_source` - the mailer name like `coupon_mailer`
- `utm_campaign` - the mailer action like `offer`

You can customize them with:

```ruby
class CouponMailer < ApplicationMailer
  track utm_params: true, utm_campaign: -> { "coupon#{params[:coupon].id}" }
end
```

Skip specific links with:

```erb
<%= link_to "Go", some_url, data: {skip_utm_params: true} %>
```

## Open & Click Analytics

While it’s nice to get feedback on the performance of your emails, we discourage the use of open tracking. If you do decide to use open or click tracking, be sure to get consent from your users and consider a short retention period. Check out [this article](https://www.eff.org/deeplinks/2019/01/stop-tracking-my-emails) for more best practices.

### Setup

Create a migration with:

```ruby
class AddTokenToAhoyMessages < ActiveRecord::Migration[6.1]
  def change
    add_column :ahoy_messages, :token, :string
    add_index :ahoy_messages, :token

    # for opens
    add_column :ahoy_messages, :opened_at, :timestamp

    # for clicks
    add_column :ahoy_messages, :clicked_at, :timestamp
  end
end
```

Create an initializer `config/initializers/ahoy_email.rb` with:

```ruby
AhoyEmail.api = true
```

And add to mailers you want to track:

```ruby
class CouponMailer < ApplicationMailer
  track open: true, click: true
end
```

Use only and except to limit actions

```ruby
class CouponMailer < ApplicationMailer
  track click: true, only: [:welcome]
end
```

Or make it conditional

```ruby
class CouponMailer < ApplicationMailer
  track click: -> { params[:user].opted_in? }
end
```

### How It Works

For opens, an invisible pixel is added right before the `</body>` tag in HTML emails. If the recipient has images enabled in their email client, the pixel is loaded and the open time recorded.

For clicks, a redirect is added to links to track clicks in HTML emails.

```
https://chartkick.com
```

becomes

```
https://yoursite.com/ahoy/messages/rAnDoMtOkEn/click?url=https%3A%2F%2Fchartkick.com&signature=...
```

A signature is added to prevent [open redirects](https://www.owasp.org/index.php/Open_redirect).

Skip specific links with:

```erb
<%= link_to "Go", some_url, data: {skip_click: true} %>
```

By default, unsubscribe links are excluded. To change this, use:

```ruby
AhoyEmail.default_options[:unsubscribe_links] = true
```

You can specify the domain to use with:

```ruby
AhoyEmail.default_options[:url_options] = {host: "mydomain.com"}
```

### Events

Subscribe to open and click events by adding to the initializer:

```ruby
class EmailSubscriber
  def open(event)
    # your code
  end

  def click(event)
    # your code
  end
end

AhoyEmail.subscribers << EmailSubscriber.new
```

Here’s an example if you use [Ahoy](https://github.com/ankane/ahoy) to track visits and events:

```ruby
class EmailSubscriber
  def open(event)
    event[:controller].ahoy.track "Email opened", message_id: event[:message].id
  end

  def click(event)
    event[:controller].ahoy.track "Email clicked", message_id: event[:message].id, url: event[:url]
  end
end

AhoyEmail.subscribers << EmailSubscriber.new
```

## Data Protection

We recommend encrypting the `to` field (as well as the `subject` if it’s sensitive). [Lockbox](https://github.com/ankane/lockbox) is great for this. Use [Blind Index](https://github.com/ankane/blind_index) if you need to query by the `to` field.

Create `app/models/ahoy/message.rb` with:

```ruby
class Ahoy::Message < ApplicationRecord
  self.table_name = "ahoy_messages"
  belongs_to :user, polymorphic: true, optional: true

  encrypts :to
  blind_index :to
end
```

## Data Retention

Delete older data with:

```ruby
Ahoy::Message.where("created_at < ?", 1.year.ago).in_batches.delete_all
```

Delete data for a specific user with:

```ruby
Ahoy::Message.where(user_id: 1).in_batches.delete_all
```

## Reference

Set global options

```ruby
AhoyEmail.default_options[:user] = -> { params[:admin] }
```

Use a different model

```ruby
AhoyEmail.message_model = -> { UserMessage }
```

Or fully customize how messages are tracked

```ruby
AhoyEmail.track_method = lambda do |data|
  # your code
end
```

## Mongoid

If you prefer to use Mongoid instead of Active Record, create `app/models/ahoy/message.rb` with:

```ruby
class Ahoy::Message
  include Mongoid::Document

  belongs_to :user, polymorphic: true, optional: true, index: true

  field :to, type: String
  field :mailer, type: String
  field :subject, type: String
  field :sent_at, type: Time
end
```

## History

View the [changelog](https://github.com/ankane/ahoy_email/blob/master/CHANGELOG.md)

## Contributing

Everyone is encouraged to help improve this project. Here are a few ways you can help:

- [Report bugs](https://github.com/ankane/ahoy_email/issues)
- Fix bugs and [submit pull requests](https://github.com/ankane/ahoy_email/pulls)
- Write, clarify, or fix documentation
- Suggest or add new features

To get started with development:

```sh
git clone https://github.com/ankane/ahoy_email.git
cd ahoy_email
bundle install
bundle exec rake test
```
