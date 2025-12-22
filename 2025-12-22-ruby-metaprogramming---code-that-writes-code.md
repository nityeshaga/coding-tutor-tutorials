---
concepts: [Metaprogramming, define_method, method_missing, class_eval, instance_eval, send, respond_to_missing]
source_repo: fizzy
description: Deep dive into Ruby's metaprogramming - the ability to write code that writes code at runtime. Covers define_method, method_missing, class_eval, instance_eval, and send. This is the foundation that powers Rails' "magic" like belongs_to, has_many, validates, and scope.
understanding_score: null
last_quizzed: null
prerequisites: [2025-12-22-ruby-101---the-language-beneath-rails.md, 2025-12-22-ruby's-object-model---how-methods-get-found.md]
created: 22-12-2025
last_updated: 22-12-2025
---

# Ruby Metaprogramming: Code That Writes Code

You've seen Rails "magic" - you write `belongs_to :account` and suddenly your model has `account`, `account=`, `account_id`, `build_account`, and `create_account` methods. Where do they come from?

You write `scope :active, -> { where(active: true) }` and your model gets an `active` class method. How?

You write `validates :email, presence: true` and validation just... works. But where's the code?

The answer is **metaprogramming** - Ruby code that writes other Ruby code at runtime. It's the secret sauce behind Rails' elegant DSL (Domain Specific Language). After this tutorial, Rails will stop feeling like magic and start feeling like "clever Ruby."

---

## What is Metaprogramming?

Regular programming: you write code, it runs.

Metaprogramming: you write code that **generates other code** at runtime.

```ruby
# Regular programming
def greet_alice
  "Hello, Alice!"
end

def greet_bob
  "Hello, Bob!"
end

# Metaprogramming - generate methods dynamically
["alice", "bob", "charlie"].each do |name|
  define_method("greet_#{name}") do
    "Hello, #{name.capitalize}!"
  end
end

# Now greet_alice, greet_bob, greet_charlie all exist!
```

The second approach creates methods at runtime. You didn't write `def greet_charlie` - Ruby did, based on your instructions.

---

## Tool #1: `define_method` - Creating Methods at Runtime

`define_method` creates a method with the name you give it:

```ruby
class Greeter
  define_method(:say_hello) do
    "Hello!"
  end
end

Greeter.new.say_hello  # => "Hello!"
```

This is equivalent to:

```ruby
class Greeter
  def say_hello
    "Hello!"
  end
end
```

So why use `define_method`? Because the method name can be **dynamic**:

```ruby
class Greeter
  ["english", "spanish", "french"].each do |language|
    define_method("greet_in_#{language}") do
      case language
      when "english" then "Hello!"
      when "spanish" then "Hola!"
      when "french"  then "Bonjour!"
      end
    end
  end
end

greeter = Greeter.new
greeter.greet_in_english  # => "Hello!"
greeter.greet_in_spanish  # => "Hola!"
greeter.greet_in_french   # => "Bonjour!"
```

Three methods, created from a loop. This is metaprogramming.

### How `attr_accessor` Works

You've used `attr_accessor` many times:

```ruby
class Person
  attr_accessor :name
end

person = Person.new
person.name = "Alice"  # setter
person.name            # getter => "Alice"
```

`attr_accessor` isn't magic - it's metaprogramming. Here's roughly how it works:

```ruby
class Module
  def my_attr_accessor(attribute)
    # Define the getter
    define_method(attribute) do
      instance_variable_get("@#{attribute}")
    end

    # Define the setter
    define_method("#{attribute}=") do |value|
      instance_variable_set("@#{attribute}", value)
    end
  end
end

class Person
  my_attr_accessor :name
end

person = Person.new
person.name = "Alice"
person.name  # => "Alice"
```

When you call `attr_accessor :name`, Ruby uses `define_method` to create both `name` and `name=` methods. That's it. No magic.

### How Rails' `delegate` Works

Fizzy uses `delegate` extensively:

```ruby
# From app/models/card.rb
delegate :accessible_to?, to: :board

# From app/models/event.rb
delegate :card, to: :eventable
```

`delegate` creates a method that forwards the call to another object. Here's roughly how:

```ruby
class Module
  def delegate(*methods, to:)
    methods.each do |method|
      define_method(method) do |*args, &block|
        target = send(to)  # Get the target object
        target.send(method, *args, &block)  # Forward the call
      end
    end
  end
end
```

When you write `delegate :accessible_to?, to: :board`, Rails creates:

```ruby
def accessible_to?(*args, &block)
  board.accessible_to?(*args, &block)
end
```

One line of configuration, a method generated for you.

---

## Tool #2: `method_missing` - Catching Undefined Methods

When you call a method that doesn't exist, Ruby doesn't immediately crash. It first calls `method_missing`:

```ruby
class Flexible
  def method_missing(method_name, *args)
    "You called #{method_name} with #{args.inspect}"
  end
end

f = Flexible.new
f.anything_at_all(1, 2, 3)
# => "You called anything_at_all with [1, 2, 3]"
```

Any method call that would normally fail instead gets intercepted by `method_missing`.

### The Classic Example: OpenStruct

Ruby's `OpenStruct` lets you set arbitrary attributes:

```ruby
require 'ostruct'

person = OpenStruct.new
person.name = "Alice"
person.age = 30
person.name  # => "Alice"
```

How does it accept any attribute? `method_missing`:

```ruby
class MyOpenStruct
  def initialize
    @attributes = {}
  end

  def method_missing(method_name, *args)
    name = method_name.to_s

    if name.end_with?("=")
      # It's a setter like "name="
      attribute = name.chomp("=")
      @attributes[attribute] = args.first
    else
      # It's a getter like "name"
      @attributes[name]
    end
  end
end

person = MyOpenStruct.new
person.name = "Alice"
person.name  # => "Alice"
person.anything = "works"
person.anything  # => "works"
```

### How Rails Finds Records by Attributes

Ever used `find_by_email` or `find_by_name`?

```ruby
User.find_by_email("alice@example.com")
User.find_by_name_and_age("Alice", 30)
```

These methods don't exist explicitly. Rails uses `method_missing` to catch them:

```ruby
# Simplified version of what ActiveRecord does
def method_missing(method_name, *args)
  if method_name.to_s.start_with?("find_by_")
    attributes = method_name.to_s.sub("find_by_", "").split("_and_")
    conditions = attributes.zip(args).to_h
    where(conditions).first
  else
    super  # Let Ruby raise the normal error
  end
end
```

When you call `User.find_by_email("x")`:
1. Ruby doesn't find `find_by_email` method
2. Calls `method_missing` with `:find_by_email`
3. Rails parses the method name, extracts "email"
4. Converts to `where(email: "x").first`

### Always Pair with `respond_to_missing?`

If you implement `method_missing`, you should also implement `respond_to_missing?`:

```ruby
class Flexible
  def method_missing(method_name, *args)
    if method_name.to_s.start_with?("greet_")
      "Hello!"
    else
      super
    end
  end

  def respond_to_missing?(method_name, include_private = false)
    method_name.to_s.start_with?("greet_") || super
  end
end

f = Flexible.new
f.greet_anyone      # => "Hello!"
f.respond_to?(:greet_anyone)  # => true (thanks to respond_to_missing?)
```

Without `respond_to_missing?`, `respond_to?(:greet_anyone)` would return `false` even though the method works.

---

## Tool #3: `class_eval` and `instance_eval` - Executing Code in Context

These let you execute code as if you were inside a class or object.

### `class_eval` - Add Methods to a Class

```ruby
class Person
end

Person.class_eval do
  def greet
    "Hello!"
  end
end

Person.new.greet  # => "Hello!"
```

This is identical to:

```ruby
class Person
  def greet
    "Hello!"
  end
end
```

But `class_eval` is powerful because the class can be **dynamic**:

```ruby
def add_greeting_to(klass)
  klass.class_eval do
    def greet
      "Hello from #{self.class.name}!"
    end
  end
end

add_greeting_to(User)
add_greeting_to(Card)

User.new.greet  # => "Hello from User!"
Card.new.greet  # => "Hello from Card!"
```

### String Version for Dynamic Code

`class_eval` can take a string, allowing truly dynamic code:

```ruby
class Person
end

attribute_name = "name"

Person.class_eval <<-RUBY
  def #{attribute_name}
    @#{attribute_name}
  end

  def #{attribute_name}=(value)
    @#{attribute_name} = value
  end
RUBY

person = Person.new
person.name = "Alice"
person.name  # => "Alice"
```

The `<<-RUBY ... RUBY` is a heredoc string. The `#{attribute_name}` gets interpolated before the code runs.

### `instance_eval` - Execute in Object's Context

While `class_eval` adds methods to a class, `instance_eval` executes code in a specific object's context:

```ruby
class Person
  def initialize
    @secret = "I like Ruby"
  end
end

person = Person.new

# Normally can't access @secret from outside
person.instance_eval do
  @secret  # => "I like Ruby"
end
```

`instance_eval` is often used for DSL configuration blocks:

```ruby
class Config
  attr_accessor :host, :port

  def configure(&block)
    instance_eval(&block)
  end
end

config = Config.new
config.configure do
  self.host = "localhost"
  self.port = 3000
end

config.host  # => "localhost"
```

---

## Tool #4: `send` - Calling Methods by Name

`send` lets you call a method using its name as a string or symbol:

```ruby
"hello".send(:upcase)    # => "HELLO"
"hello".send("upcase")   # => "HELLO"
```

Why is this useful? When the method name is **dynamic**:

```ruby
class Calculator
  def add(a, b) = a + b
  def subtract(a, b) = a - b
  def multiply(a, b) = a * b
end

calc = Calculator.new
operation = :multiply  # Could come from user input

calc.send(operation, 5, 3)  # => 15
```

### Rails Uses `send` Everywhere

When you write:

```ruby
card.update(title: "New Title", status: "active")
```

Rails iterates through the hash and uses `send` to call each setter:

```ruby
# Simplified version of what happens
def update(attributes)
  attributes.each do |key, value|
    send("#{key}=", value)  # Calls title=, status=, etc.
  end
  save
end
```

### `public_send` - The Safer Version

`send` can call private methods. `public_send` only calls public ones:

```ruby
class Secret
  private

  def hidden
    "secret!"
  end
end

s = Secret.new
s.send(:hidden)        # => "secret!" (works!)
s.public_send(:hidden) # => NoMethodError (respects privacy)
```

Use `public_send` when the method name comes from user input to avoid exposing private methods.

---

## Putting It Together: How `belongs_to` Works

Now let's see how Rails uses ALL of these tools. When you write:

```ruby
class Card < ApplicationRecord
  belongs_to :board
end
```

Here's roughly what happens inside Rails:

```ruby
class ActiveRecord::Base
  def self.belongs_to(association_name, options = {})
    # 1. Define the getter method
    define_method(association_name) do
      # Find the associated record
      association_class = association_name.to_s.classify.constantize
      foreign_key = "#{association_name}_id"
      association_class.find(send(foreign_key))
    end

    # 2. Define the setter method
    define_method("#{association_name}=") do |record|
      foreign_key = "#{association_name}_id"
      send("#{foreign_key}=", record&.id)
    end

    # 3. Define the build method
    define_method("build_#{association_name}") do |attributes = {}|
      association_class = association_name.to_s.classify.constantize
      record = association_class.new(attributes)
      send("#{association_name}=", record)
      record
    end
  end
end
```

When you write `belongs_to :board`, Rails:

1. Uses `define_method` to create `board` (getter)
2. Uses `define_method` to create `board=` (setter)
3. Uses `define_method` to create `build_board`
4. Uses `send` internally to work with dynamic attribute names
5. Uses `classify.constantize` to convert `:board` → `"Board"` → `Board` class

One line of code, multiple methods generated. That's metaprogramming.

---

## How `scope` Works

When you write:

```ruby
class Card < ApplicationRecord
  scope :active, -> { where(status: "active") }
end
```

Rails essentially does:

```ruby
class ActiveRecord::Base
  def self.scope(name, body)
    # Define a class method with that name
    define_singleton_method(name) do |*args|
      # Execute the lambda in the context of the relation
      body.call(*args)
    end
  end
end
```

`define_singleton_method` is like `define_method` but adds to the class's eigenclass (making it a class method).

After `scope :active, -> { where(status: "active") }`:

```ruby
Card.active  # Works! Calls the lambda
```

---

## How `validates` Works

When you write:

```ruby
class Card < ApplicationRecord
  validates :title, presence: true
end
```

Rails stores the validation in a class-level registry:

```ruby
class ActiveRecord::Base
  class_attribute :_validators, default: {}

  def self.validates(*attributes, **options)
    options.each do |validator_type, validator_options|
      # Find the validator class (PresenceValidator, LengthValidator, etc.)
      validator_class = "#{validator_type.to_s.camelize}Validator".constantize

      attributes.each do |attribute|
        _validators[attribute] ||= []
        _validators[attribute] << validator_class.new(validator_options)
      end
    end
  end

  def valid?
    self.class._validators.each do |attribute, validators|
      validators.each do |validator|
        validator.validate(self, attribute)
      end
    end
    errors.empty?
  end
end
```

The key insight: `validates` doesn't validate anything immediately. It **stores instructions** that get executed later when you call `valid?` or `save`.

---

## How `before_action` Works

In controllers:

```ruby
class CardsController < ApplicationController
  before_action :require_authentication

  def show
    @card = Card.find(params[:id])
  end
end
```

Rails stores callbacks and runs them in order:

```ruby
class ActionController::Base
  class_attribute :_before_actions, default: []

  def self.before_action(method_name, **options)
    _before_actions << { method: method_name, options: options }
  end

  def process_action(action_name)
    # Run all before_actions first
    self.class._before_actions.each do |callback|
      send(callback[:method])  # Call the method by name
      return if performed?  # Stop if redirected
    end

    # Then run the actual action
    send(action_name)
  end
end
```

`before_action` doesn't run code - it **registers** a method name to be called later.

---

## The Pattern: DSL Through Metaprogramming

All Rails "magic" follows the same pattern:

1. **You call a class method** (`belongs_to`, `scope`, `validates`, `before_action`)
2. **That method stores configuration** (in class variables, arrays, or hashes)
3. **Methods are generated** (using `define_method`)
4. **Later, the framework reads the configuration** and acts on it

```
Your code                   Rails internals
─────────                   ───────────────
belongs_to :board    →      Store association config
                     →      define_method(:board)
                     →      define_method(:board=)

validates :title     →      Store validation config
                     →      (Later) Loop through validators on save

before_action :auth  →      Store callback config
                     →      (Later) Call method before action
```

---

## Debugging Metaprogrammed Code

When methods are generated dynamically, debugging gets tricky. Here are your tools:

```ruby
# Where is this method defined?
card.method(:board).source_location
# => ["/gems/activerecord/lib/associations.rb", 123]

# What methods does this class have that match a pattern?
Card.instance_methods.grep(/board/)
# => [:board, :board=, :board_id, :board_id=, :build_board, :create_board]

# What's the source code? (requires 'method_source' gem in dev)
puts card.method(:board).source

# List all callbacks
Card._before_action_callbacks.map(&:filter)
```

---

## When to Use Metaprogramming

**Good uses:**
- Building DSLs (like Rails' associations, validations)
- Reducing repetitive boilerplate
- Framework/library code where patterns repeat
- Configuration-driven behavior

**Bad uses:**
- When regular methods would be clearer
- When it makes debugging impossible
- Just to seem clever
- In application code (usually)

The rule: **metaprogramming is for library authors**. As an app developer, you *use* metaprogrammed DSLs (like Rails), but rarely write your own.

---

## Summary

| Tool | What it does | Rails uses it for |
|------|-------------|-------------------|
| `define_method` | Creates methods at runtime | `belongs_to`, `has_many`, `scope`, `delegate` |
| `method_missing` | Catches undefined method calls | `find_by_*`, dynamic finders |
| `class_eval` | Executes code in class context | Adding methods to classes dynamically |
| `instance_eval` | Executes code in object context | DSL configuration blocks |
| `send` | Calls methods by name | Mass assignment, `update`, callbacks |

**The key insight:** Rails isn't magical. It's just Ruby metaprogramming - code that writes code. When you write `belongs_to :board`, you're calling a method that generates other methods. That's it.

---

## Challenge

Now that you understand metaprogramming, look at Fizzy's Card model:

```ruby
class Card < ApplicationRecord
  belongs_to :board
  belongs_to :creator, class_name: "User"

  has_many :comments, dependent: :destroy

  scope :chronologically, -> { order(created_at: :asc) }

  validates :board, presence: true

  before_save :set_default_title, if: :published?

  delegate :accessible_to?, to: :board
end
```

For each line, explain:
1. What methods does it create?
2. Which metaprogramming tool is it likely using?
3. When does the actual code run (at class load time, or later)?

---

## What's Next

You now have the complete foundation:
- **Tutorial 1:** Ruby vs Rails - what's the language, what's the framework
- **Tutorial 2:** Object Model - how methods are found
- **Tutorial 3:** Metaprogramming - how methods are created

In **Tutorial 4**, we'll put it all together: **Rails Magic Demystified**. We'll trace through *actual Rails source code* to see exactly how `belongs_to`, `has_many`, and friends work under the hood.

---

## Q&A

*Questions from our learning sessions will be added here.*

---

## Quiz History

*Quiz sessions will be recorded here.*
