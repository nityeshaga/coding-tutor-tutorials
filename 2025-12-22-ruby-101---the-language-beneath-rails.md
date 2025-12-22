---
concepts: [Ruby, Objects, Classes, Methods, Blocks, Symbols, Instance Variables]
source_repo: fizzy
description: The foundational "what is Ruby?" tutorial for Rails developers who learned by doing. Covers the distinction between Ruby the language and Rails the framework, everything-is-an-object, classes, methods, blocks, symbols, and instance variables - all demonstrated by dissecting real Fizzy code line by line.
understanding_score: null
last_quizzed: null
prerequisites: []
created: 22-12-2025
last_updated: 22-12-2025  # Q&A: Rails class hierarchy, POROs
---

# Ruby 101: The Language Beneath Rails

You've been writing Ruby for months. Every time you type `class User < ApplicationRecord`, every `def show`, every `@card = Card.find(params[:id])` - that's Ruby. But which parts are Ruby the language, and which parts are Rails the framework bolted on top?

This matters more than you might think. When something breaks, knowing whether it's a Ruby problem or a Rails problem tells you where to look. When you read documentation, knowing which docs to check saves hours. And when you write code, understanding the foundation helps you write better code on top of it.

Let's untangle this.

---

## The Relationship: Ruby is the Language, Rails is the Library

Here's the simplest way to think about it:

**Ruby** is a programming language. It has syntax, rules, and built-in features. It exists independently of Rails. People write Ruby without Rails all the time - scripts, CLI tools, other web frameworks (Sinatra, Hanami), even iOS apps (RubyMotion).

**Rails** is a Ruby library (technically, a collection of libraries called "gems"). It's Ruby code that someone else wrote to make building web apps easier. When you `gem install rails`, you're downloading Ruby code that adds new classes, methods, and patterns.

Think of it like this: English is a language. Legal jargon is specialized vocabulary built on top of English. You need to know English to understand legal documents, but "habeas corpus" isn't an English word - it's legal terminology. Similarly, `belongs_to` isn't Ruby - it's Rails vocabulary.

---

## Everything in Ruby is an Object

This is the single most important thing to understand about Ruby, and it's purely a Ruby concept.

In some languages, there's a distinction between "primitive" values (numbers, booleans) and "objects" (complex data structures). Not in Ruby. **Everything is an object.** Numbers, strings, `nil`, `true`, even classes themselves - all objects.

What does this mean practically? Every value has methods you can call on it:

```ruby
# Numbers are objects
42.to_s          # => "42"
42.even?         # => true
42.class         # => Integer

# Strings are objects
"hello".upcase   # => "HELLO"
"hello".length   # => 5
"hello".class    # => String

# nil is an object
nil.nil?         # => true
nil.class        # => NilClass

# true and false are objects
true.class       # => TrueClass
false.to_s       # => "false"
```

This is why you can chain methods in Ruby:

```ruby
"  hello world  ".strip.upcase.split(" ")
# => ["HELLO", "WORLD"]
```

Each method returns an object, and you call the next method on that object. It's objects all the way down.

---

## Classes and Inheritance: Pure Ruby

Let's look at a real file from Fizzy - `app/models/user.rb`:

```ruby
class User < ApplicationRecord
  # ...
end
```

This single line contains both Ruby and Rails. Let me dissect it:

| Part | Ruby or Rails? | What it means |
|------|----------------|---------------|
| `class User` | **Ruby** | Defines a new class called User |
| `<` | **Ruby** | Inheritance operator |
| `ApplicationRecord` | **Rails** | A class that Rails provides |

The concept of classes and inheritance is pure Ruby. Rails just provides useful parent classes to inherit from.

Here's what a pure Ruby class looks like - from Fizzy's `lib/web_push/notification.rb`:

```ruby
class WebPush::Notification
  def initialize(title:, body:, path:, badge:, endpoint:, endpoint_ip:, p256dh_key:, auth_key:)
    @title, @body, @path, @badge = title, body, path, badge
    @endpoint, @endpoint_ip, @p256dh_key, @auth_key = endpoint, endpoint_ip, p256dh_key, auth_key
  end

  def deliver(connection: nil)
    WebPush.payload_send \
      message: encoded_message,
      endpoint: @endpoint, endpoint_ip: @endpoint_ip, p256dh: @p256dh_key, auth: @auth_key,
      vapid: vapid_identification,
      connection: connection,
      urgency: "high"
  end

  private
    def vapid_identification
      { subject: "mailto:support@fizzy.do" }.merge \
        Rails.configuration.x.vapid.symbolize_keys
    end
end
```

Notice: no `< ApplicationRecord`, no `belongs_to`, no ActiveRecord magic. This is a Plain Old Ruby Object (PORO). It's just Ruby. The only Rails thing in there is `Rails.configuration` - everything else is pure language.

### The `initialize` Method

`initialize` is Ruby's constructor - the method that gets called when you create a new object with `.new`:

```ruby
# When you write:
notification = WebPush::Notification.new(title: "Hello", body: "World", ...)

# Ruby automatically calls:
def initialize(title:, body:, ...)
  # Your setup code
end
```

This is **Ruby**, not Rails. Every Ruby class can have an `initialize` method.

---

## Instance Variables: The @ Symbol

See all those `@title`, `@body`, `@endpoint` variables? Those are **instance variables** - pure Ruby.

Instance variables:
- Start with `@`
- Belong to a specific object instance
- Are accessible from any method in that object
- Are NOT accessible from outside the object (by default)

```ruby
class Person
  def initialize(name)
    @name = name  # Store the name in an instance variable
  end

  def greet
    "Hello, I'm #{@name}"  # Access it from another method
  end
end

person = Person.new("Nityesh")
person.greet  # => "Hello, I'm Nityesh"
person.@name  # ERROR - can't access instance variables directly from outside
```

When you see `@card` in a Rails controller, that's just Ruby's instance variable syntax. Rails uses instance variables to pass data from controllers to views, but the `@` itself is Ruby.

### The Difference: @, @@, and $

Ruby has three types of variables beyond local ones:

```ruby
@name    # Instance variable - belongs to one object
@@count  # Class variable - shared by all instances of a class (rarely used)
$global  # Global variable - accessible everywhere (almost never use this)
```

You'll almost exclusively use `@` instance variables. If you see `@@` or `$` in code, something unusual is happening.

---

## Methods: def and end

Method definitions are pure Ruby:

```ruby
def greet(name)
  "Hello, #{name}"
end
```

The syntax:
- `def` starts a method definition
- Method name comes next (lowercase, underscores for multiple words)
- Parentheses for parameters (optional but recommended)
- Method body
- `end` closes the definition

### The Last Line is the Return Value

Ruby automatically returns the last evaluated expression. You don't need `return`:

```ruby
# These are equivalent:
def double(n)
  return n * 2
end

def double(n)
  n * 2
end
```

The second style is idiomatic Ruby. You'll rarely see explicit `return` statements unless you're returning early:

```ruby
def process(item)
  return if item.nil?  # Early return if nil

  item.do_something    # Otherwise, this is the return value
end
```

Look at Fizzy's `User#verified?` method:

```ruby
def verified?
  verified_at.present?
end
```

No `return` - the result of `verified_at.present?` (a boolean) is automatically returned.

### Method Names Can End With ? or !

This is a Ruby convention (not enforced, just cultural):

- `?` methods return booleans: `empty?`, `nil?`, `valid?`, `verified?`
- `!` methods are "dangerous" (modify the object or might raise errors): `save!`, `update!`, `destroy!`

Both are just part of the method name - `verified?` is the full name, not `verified` with a `?` operator.

---

## Symbols vs Strings: :this vs "this"

You see these everywhere in Rails:

```ruby
belongs_to :account
has_many :comments
render json: { status: :ok }
```

What are those colons? They're **symbols** - a Ruby data type.

### What's a Symbol?

A symbol is like a string, but:
- Immutable (can't be changed)
- Only one copy exists in memory (same symbol = same object)
- Faster to compare

```ruby
# Strings - each one is a new object
"hello".object_id  # => 70234567890
"hello".object_id  # => 70234567891 (different!)

# Symbols - same object every time
:hello.object_id   # => 1234567
:hello.object_id   # => 1234567 (same!)
```

### When to Use Which?

**Symbols** for identifiers, keys, names of things:
```ruby
user.role              # Accessing an attribute (method call)
{ status: :active }    # Hash keys
def process(type:)     # Keyword arguments
```

**Strings** for actual text content:
```ruby
user.name = "Nityesh"
puts "Hello, world"
"Please enter your email"
```

The rule of thumb: if it's a label/identifier, use a symbol. If it's data/content, use a string.

### The Rails Connection

Rails heavily uses symbols for configuration and method names:

```ruby
belongs_to :account       # :account is a symbol - the name of the association
validates :email, presence: true  # :email and :presence are symbols
```

But this is just Rails convention using Ruby's symbol feature.

---

## Blocks: The Thing That Makes Ruby Feel Magic

This is where Ruby gets interesting, and where most Rails "magic" lives.

A **block** is a chunk of code you pass to a method. You've used them constantly:

```ruby
# Block with do...end
users.each do |user|
  puts user.name
end

# Block with curly braces (same thing, different syntax)
users.each { |user| puts user.name }
```

The part between `do...end` (or `{ }`) is the block. The `|user|` is the block parameter - a variable that receives values from the method.

### How Blocks Work

When you call `users.each`, the `each` method runs your block once for each user, passing each user into the block:

```ruby
# This is roughly what each does internally:
def each
  index = 0
  while index < self.length
    yield self[index]  # "yield" runs your block with this value
    index += 1
  end
end
```

`yield` is the Ruby keyword that executes the block you passed. This is **pure Ruby** - Rails just uses it heavily.

### Blocks in Fizzy

Look at this line from `Card`:

```ruby
scope :reverse_chronologically, -> { order created_at: :desc, id: :desc }
```

Wait, what's that `->` thing? That's a **lambda** - a block that's been saved into a variable.

---

## Lambdas and Procs: Blocks You Can Save

A block is ephemeral - it exists only when you pass it to a method. But sometimes you want to save a block for later. That's what lambdas and procs are.

```ruby
# A lambda (arrow syntax)
my_lambda = -> { puts "Hello" }
my_lambda.call  # => "Hello"

# A lambda with arguments
double = ->(n) { n * 2 }
double.call(5)  # => 10

# Same thing, older syntax
double = lambda { |n| n * 2 }
```

### The -> Syntax You See Everywhere

In Rails/Fizzy code, you see `->` constantly:

```ruby
# From Card model
belongs_to :creator, class_name: "User", default: -> { Current.user }

scope :reverse_chronologically, -> { order created_at: :desc, id: :desc }

after_save -> { board.touch }, if: :published?
```

Each of these `-> { ... }` is a lambda. Why use a lambda instead of just a value?

**Because lambdas are evaluated when called, not when defined.**

```ruby
# This is evaluated NOW, when the file loads:
default: Current.user  # Might be nil at load time!

# This is evaluated LATER, when a Card is created:
default: -> { Current.user }  # Gets the current user at creation time
```

This is critical for things like `Current.user` which changes based on the request.

### Block vs Lambda vs Proc

Here's the quick breakdown:

| Type | Syntax | Saved? | Use case |
|------|--------|--------|----------|
| Block | `do...end` or `{ }` | No | Pass once to a method |
| Lambda | `-> { }` | Yes | Save for later, strict about arguments |
| Proc | `Proc.new { }` | Yes | Save for later, loose about arguments |

For now, just know that `-> { }` creates a callable chunk of code. You'll see it everywhere in Rails.

---

## Modules: Ruby's Way to Share Code

Look at this from `app/models/concerns/eventable.rb`:

```ruby
module Eventable
  extend ActiveSupport::Concern

  included do
    has_many :events, as: :eventable, dependent: :destroy
  end

  def track_event(action, creator: Current.user, board: self.board, **particulars)
    # ...
  end
end
```

The `module` keyword is **pure Ruby**. Modules are collections of methods that can be mixed into classes:

```ruby
# Pure Ruby module
module Greetable
  def greet
    "Hello, I'm #{name}"
  end
end

class User
  include Greetable  # Mix in the module

  attr_accessor :name
end

user = User.new
user.name = "Nityesh"
user.greet  # => "Hello, I'm Nityesh"
```

The `include` keyword adds the module's methods to the class. This is **Ruby**.

### What Rails Adds: ActiveSupport::Concern

The `extend ActiveSupport::Concern` and `included do` blocks? That's **Rails**.

Rails' Concern adds conveniences for module definition, but the underlying concept of modules and `include` is pure Ruby.

---

## Open Classes: Ruby's Superpower

Here's something wild. In Fizzy's `lib/rails_ext/string.rb`:

```ruby
class String
  def all_emoji?
    self.match?(/\A(\p{Emoji_Presentation}|\p{Extended_Pictographic}|\uFE0F)+\z/u)
  end
end
```

This **adds a method to Ruby's built-in String class**. After this file loads:

```ruby
"hello".all_emoji?  # => false
"🎉🔥".all_emoji?    # => true
```

This is called "open classes" or "monkey patching" - Ruby lets you modify any class, even built-in ones. It's powerful and dangerous. Rails uses it extensively to add convenience methods.

When you call `"hello".blank?` - that's not native Ruby. Rails added the `blank?` method to String (and Object, and NilClass...).

---

## Dissecting a Real Rails Line

Let's put it all together. Here's a line from Fizzy's Card model:

```ruby
belongs_to :account, default: -> { board.account }
```

Breaking it down:

| Part | Ruby or Rails? | Explanation |
|------|----------------|-------------|
| `belongs_to` | **Rails** | A method that ActiveRecord added to your class |
| `:account` | **Ruby** | A symbol - the name of the association |
| `,` | **Ruby** | Argument separator |
| `default:` | **Ruby** | A keyword argument |
| `-> { board.account }` | **Ruby** | A lambda |
| `board` | **Rails** | Calls the `board` method (created by another `belongs_to`) |
| `.account` | **Ruby** | Method call syntax |

So the syntax is Ruby, but `belongs_to` itself is a Rails method that does a ton of work behind the scenes (which we'll cover in the metaprogramming tutorial).

---

## The Quick Reference: Ruby vs Rails

Here's a cheat sheet for the things you see in Rails code:

### Pure Ruby (the language)
- `class`, `module`, `def`, `end`
- `@instance_variables`
- `:symbols`
- `-> { }` lambdas
- `do |x| ... end` blocks
- `if`, `unless`, `case`, `while`
- `.method_calls`
- `< Inheritance`
- `include`, `extend`

### Rails (the framework)
- `belongs_to`, `has_many`, `has_one`
- `validates`, `validate`
- `scope :name, -> { }`
- `before_action`, `after_save`
- `ApplicationRecord`, `ApplicationController`
- `Current.user`, `params`, `session`
- `render`, `redirect_to`
- `Rails.configuration`, `Rails.env`

### How to Tell the Difference

Ask yourself: "Would this work in a plain Ruby file without Rails?"

```ruby
# This would work - it's Ruby:
class Dog
  def bark
    "Woof!"
  end
end

# This would fail without Rails - belongs_to doesn't exist:
class Dog < ApplicationRecord
  belongs_to :owner
end
```

---

## Challenge: Identify Ruby vs Rails

Look at this code from Fizzy's `Card` model and identify each part:

```ruby
class Card < ApplicationRecord
  include Assignable, Attachments, Broadcastable

  belongs_to :board
  has_many :comments, dependent: :destroy

  before_save :set_default_title, if: :published?

  scope :chronologically, -> { order created_at: :asc }

  def filled?
    title.present? || description.present?
  end

  private
    def set_default_title
      self.title = "Untitled" if title.blank?
    end
end
```

For each line, ask: "Is this Ruby syntax, or is this a Rails method/feature?"

Try to answer before looking at the breakdown in the Q&A section when we discuss this!

---

## What's Next

Now you know what Ruby actually is versus what Rails adds on top. In the next tutorial, we'll go deeper into **Ruby's Object Model** - how inheritance works, what happens when you call a method, and why `include` behaves the way it does. That's where you'll start to see how Rails' "magic" actually works under the hood.

For now, whenever you write Rails code, try to notice: "Is this Ruby, or is this Rails?" That mental separation will make you a significantly better Rails developer.

---

## Q&A

### Q: You mentioned ApplicationRecord is a class Rails provides. What are all the other common Rails classes, and what do they give us that Ruby doesn't have?

Yes, ActiveRecord is absolutely a Rails class. Rails is actually a collection of gems, each providing a major base class:

**1. ActiveRecord::Base (your models)**
```ruby
class Card < ApplicationRecord  # ApplicationRecord < ActiveRecord::Base
```
Gives you: Database connection, SQL generation (`find`, `where`), associations (`belongs_to`, `has_many`), validations, callbacks, migrations. Without it, you'd write raw SQL and manually map results to objects.

**2. ActionController::Base (your controllers)**
```ruby
class CardsController < ApplicationController  # < ActionController::Base
```
Gives you: `params`, `session`, `cookies`, `render`, `redirect_to`, `before_action`, CSRF protection. Without it, you'd manually parse HTTP requests and build responses.

**3. ActionView (your views)**
Not inherited, but provides: ERB templating, partials, layouts with `yield`, helpers (`link_to`, `form_with`). Without it, you'd concatenate HTML strings.

**4. ActiveJob::Base (background jobs)**
```ruby
class NotificationJob < ApplicationJob  # < ActiveJob::Base
```
Gives you: `perform_later` for queuing, adapter abstraction (Solid Queue, Sidekiq), retries, argument serialization.

**5. ActionMailer::Base (emails)**
```ruby
class UserMailer < ApplicationMailer  # < ActionMailer::Base
```
Gives you: Email templating with views, `mail()` method, attachments, development previews.

**6. ActionCable (WebSockets)**
```ruby
class ChatChannel < ApplicationCable::Channel
```
Gives you: Real-time bidirectional communication, channel abstraction, Turbo Streams integration.

**7. ActiveStorage (file uploads)**
Adds methods to models: `has_one_attached :image`. Handles cloud storage, image variants, direct uploads.

**8. ActiveSupport (the sneaky one)**
Doesn't give you a class - instead **modifies Ruby's built-in classes**:
```ruby
"hello".blank?           # Added to String (not native Ruby!)
nil.blank?               # Added to NilClass
1.day.ago                # Added to Integer
"user".pluralize         # Added to String
{ a: 1 }.symbolize_keys  # Added to Hash
```

**The naming pattern:**
- **Active___** = works with data/state (ActiveRecord, ActiveJob, ActiveStorage, ActiveSupport)
- **Action___** = handles HTTP actions (ActionController, ActionView, ActionMailer, ActionCable)

**The inheritance pattern:**
Each gem provides a base class → Your app creates `Application___` (shared behavior) → Your classes inherit from that.

```
Your Class → ApplicationRecord → ActiveRecord::Base → Ruby
```

**The key insight:** Rails is Ruby code that does the boring infrastructure work so you can focus on your app's logic.

---

### Q: What's a PORO? Why do Rails devs talk about "Plain Old Ruby Objects" like they're special?

PORO stands for "Plain Old Ruby Object" - and the name tells you what's special: **it's just Ruby, no Rails inheritance.**

In Rails, almost every class inherits from something:
```ruby
class Card < ApplicationRecord          # Inherits ActiveRecord
class CardsController < ApplicationController  # Inherits ActionController
class NotificationJob < ApplicationJob  # Inherits ActiveJob
```

A PORO inherits from nothing:
```ruby
class TaxCalculator
  def initialize(order)
    @order = order
  end

  def calculate
    @order.subtotal * tax_rate
  end

  private
    def tax_rate
      0.08
    end
end
```

No magic. No database. No callbacks. Just Ruby.

**Why it matters:** Knowing when NOT to use Rails' classes is a sign of maturity. Beginners stuff everything into models and controllers until they become 500-line monsters. Experienced devs ask: "Does this need ActiveRecord? Does it need callbacks?" If no, it should be a PORO.

**Common PORO patterns:**

1. **Service Objects** - encapsulate a business operation:
```ruby
class CardMover
  def initialize(card, new_board)
    @card, @new_board = card, new_board
  end

  def move
    @card.transaction do
      @card.update!(board: @new_board)
      notify_watchers
    end
  end
end
```

2. **Query Objects** - encapsulate complex queries:
```ruby
class StalledCardsQuery
  def initialize(board)
    @board = board
  end

  def call
    @board.cards.where("last_active_at < ?", 7.days.ago)
  end
end
```

3. **Value Objects** - represent concepts defined by value:
```ruby
class Money
  def initialize(cents)
    @cents = cents
  end

  def dollars
    @cents / 100.0
  end
end
```

**Real Fizzy example:** `lib/web_push/notification.rb` is a PORO - it takes data and sends a push notification. No database, no ActiveRecord, just Ruby.

**When to reach for a PORO:**
- Logic doesn't need the database
- Logic is called from multiple places
- Your model file is getting huge
- An operation spans multiple models

**The 37signals perspective:** They're somewhat skeptical of heavy PORO usage - preferring concerns for sharing model behavior and keeping logic in models when it relates to that model's data. POROs are for truly standalone operations like `WebPush::Notification`.

**Bottom line:** "PORO" isn't a fancy pattern - it's developers reminding themselves: "You don't have to use Rails for everything. Sometimes plain Ruby is the right tool."

---

## Quiz History

*Quiz sessions will be recorded here.*
