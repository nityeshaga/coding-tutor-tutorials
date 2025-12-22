---
concepts: [belongs_to, has_many, validates, scope, before_action, callbacks, ActiveSupport::Concern]
source_repo: fizzy
description: The capstone tutorial that brings together Ruby 101, Object Model, and Metaprogramming to fully explain Rails' "magic." We trace through exactly what happens when you write belongs_to, has_many, scope, validates, and callbacks - seeing the Ruby code that powers each feature.
understanding_score: null
last_quizzed: null
prerequisites: [2025-12-22-ruby-101---the-language-beneath-rails.md, 2025-12-22-ruby's-object-model---how-methods-get-found.md, 2025-12-22-ruby-metaprogramming---code-that-writes-code.md]
created: 22-12-2025
last_updated: 22-12-2025
---

# Rails Magic Demystified: How It Actually Works

You now know:
- **Ruby vs Rails** - what's language, what's framework
- **Object Model** - how methods are found (ancestor chain, eigenclass)
- **Metaprogramming** - how methods are created at runtime

Time to put it all together. We're going to take Fizzy's `Card` model and trace through **exactly** what happens when Rails processes each "magical" line.

After this tutorial, you'll never look at a Rails model the same way. The magic becomes machinery - elegant, predictable Ruby machinery.

---

## The Card Model We're Dissecting

Here's Fizzy's `Card` model - 98 lines that demonstrate almost every Rails "magic" feature:

```ruby
class Card < ApplicationRecord
  include Assignable, Attachments, Broadcastable, Closeable, Colored, Entropic, Eventable,
    Exportable, Golden, Mentions, Multistep, Pinnable, Postponable, Promptable,
    Readable, Searchable, Stallable, Statuses, Taggable, Triageable, Watchable

  belongs_to :account, default: -> { board.account }
  belongs_to :board
  belongs_to :creator, class_name: "User", default: -> { Current.user }

  has_many :comments, dependent: :destroy
  has_one_attached :image, dependent: :purge_later

  has_rich_text :description

  before_save :set_default_title, if: :published?
  before_create :assign_number

  after_save   -> { board.touch }, if: :published?
  after_touch  -> { board.touch }, if: :published?
  after_update :handle_board_change, if: :saved_change_to_board_id?

  scope :reverse_chronologically, -> { order created_at: :desc, id: :desc }
  scope :chronologically,         -> { order created_at: :asc,  id: :asc  }

  delegate :accessible_to?, to: :board

  def filled?
    title.present? || description.present?
  end
end
```

Let's trace through each piece.

---

## Part 1: Class Definition and Inheritance

```ruby
class Card < ApplicationRecord
```

**What Ruby does:**

1. Creates a new class named `Card`
2. Sets `ApplicationRecord` as its parent in the inheritance chain
3. The `<` operator adds `ApplicationRecord` to `Card.ancestors`

**The ancestor chain:**

```ruby
Card.ancestors
# => [Card, ...(included modules)..., ApplicationRecord,
#     ActiveRecord::Base, ActiveModel::Validations,
#     ActiveRecord::Callbacks, ..., Object, Kernel, BasicObject]
```

Every method from every ancestor is now available to `Card` instances. That's why `card.save`, `card.valid?`, `card.errors` all work - they're inherited from `ActiveRecord::Base`.

**Ruby features used:** Inheritance (`<`), method lookup chain

---

## Part 2: Module Inclusion

```ruby
include Assignable, Attachments, Broadcastable, Closeable, ...
```

**What Ruby does:**

For each module in the list, Ruby:
1. Inserts it into the ancestor chain (after `Card`, before `ApplicationRecord`)
2. Runs the module's `included` hook if it exists

**The insertion order matters:**

```ruby
# When you write:
include Assignable, Attachments, Broadcastable

# Ruby processes left to right, but each insert goes right after Card:
# Step 1: [Card, Assignable, ApplicationRecord, ...]
# Step 2: [Card, Attachments, Assignable, ApplicationRecord, ...]
# Step 3: [Card, Broadcastable, Attachments, Assignable, ApplicationRecord, ...]

# So the LAST module is searched FIRST!
```

**What each module can do in its `included` block:**

```ruby
module Eventable
  extend ActiveSupport::Concern

  included do
    # This code runs in Card's context when Card includes Eventable
    has_many :events, as: :eventable, dependent: :destroy
  end

  def track_event(action, ...)
    # This becomes an instance method on Card
  end
end
```

When `Card` says `include Eventable`:
1. Ruby adds `Eventable` to `Card.ancestors`
2. Rails' `ActiveSupport::Concern` triggers the `included` block
3. That block runs `has_many :events` inside `Card`'s context
4. `track_event` becomes available as `card.track_event`

**Ruby features used:** `include`, ancestor chain, `included` hook, `class_eval` (inside Concern)

---

## Part 3: `belongs_to` - The Association Deep Dive

```ruby
belongs_to :board
belongs_to :creator, class_name: "User", default: -> { Current.user }
```

This is where it gets interesting. Let's trace through exactly what happens.

### Step 1: `belongs_to` is a Class Method Call

When Ruby loads the `Card` class, it executes each line. `belongs_to :board` is just a method call:

```ruby
# This:
belongs_to :board

# Is the same as:
self.belongs_to(:board)

# Where self is Card (the class, not an instance)
```

### Step 2: Rails Stores the Association Configuration

Inside `ActiveRecord::Associations`:

```ruby
def belongs_to(name, scope = nil, **options)
  # Store this association's configuration
  reflection = create_reflection(:belongs_to, name, scope, options, self)

  # Generate the methods
  Associations::Builder::BelongsTo.build(self, reflection)
end
```

The `reflection` is a configuration object that stores:
- Association name: `:board`
- Association type: `:belongs_to`
- Options: `{ class_name: "Board", foreign_key: "board_id" }`

### Step 3: Methods Are Generated

`Associations::Builder::BelongsTo.build` uses `define_method` to create:

```ruby
# Getter - card.board
define_method(:board) do
  association(:board).reader
end

# Setter - card.board = some_board
define_method(:board=) do |value|
  association(:board).writer(value)
end

# The _id getter/setter are also created
define_method(:board_id) do
  read_attribute(:board_id)
end

define_method(:board_id=) do |value|
  write_attribute(:board_id, value)
end
```

### Step 4: Build/Create Methods

```ruby
# card.build_board(name: "My Board")
define_method(:build_board) do |attributes = {}|
  association(:board).build(attributes)
end

# card.create_board(name: "My Board")
define_method(:create_board) do |attributes = {}|
  association(:board).create(attributes)
end
```

### The `class_name` Option

```ruby
belongs_to :creator, class_name: "User"
```

Without `class_name`, Rails would look for a `Creator` model. The option tells Rails:
- Association name: `:creator`
- But the actual class is: `User`
- Foreign key: `creator_id` (still derived from association name)

### The `default` Option

```ruby
belongs_to :creator, class_name: "User", default: -> { Current.user }
```

The `default` lambda is stored in the reflection. When you create a card without setting `creator`:

```ruby
card = Card.new(title: "Test")
card.creator  # => Current.user (the lambda is called!)
```

Rails calls the lambda lazily when the association is accessed for the first time.

### What You End Up With

After `belongs_to :board`, your `Card` class has these new methods:

| Method | What it does |
|--------|-------------|
| `card.board` | Fetches the associated Board |
| `card.board=` | Sets the associated Board |
| `card.board_id` | Returns the foreign key value |
| `card.board_id=` | Sets the foreign key |
| `card.build_board` | Instantiates a new Board (unsaved) |
| `card.create_board` | Creates and saves a new Board |
| `card.reload_board` | Reloads the association from DB |

**7 methods generated from one line of code.**

**Ruby features used:** `define_method`, `send`, lambdas, class-level method calls

---

## Part 4: `has_many` - The Collection Association

```ruby
has_many :comments, dependent: :destroy
```

### Similar Pattern, More Methods

Like `belongs_to`, but for collections:

```ruby
def has_many(name, scope = nil, **options)
  reflection = create_reflection(:has_many, name, scope, options, self)
  Associations::Builder::HasMany.build(self, reflection)
end
```

### Generated Methods

```ruby
# Getter - returns a CollectionProxy
define_method(:comments) do
  association(:comments).reader
end

# Setter - replaces the entire collection
define_method(:comments=) do |value|
  association(:comments).writer(value)
end

# Singular IDs accessor
define_method(:comment_ids) do
  association(:comments).ids_reader
end

define_method(:comment_ids=) do |ids|
  association(:comments).ids_writer(ids)
end
```

### The `dependent: :destroy` Option

This registers a callback:

```ruby
# When Card is destroyed, also destroy all comments
before_destroy do
  comments.each(&:destroy)
end
```

Actually, Rails is smarter - it uses `dependent: :destroy` to generate efficient deletion:

```ruby
after_destroy -> { comments.destroy_all }
```

### Collection Methods

The `CollectionProxy` returned by `card.comments` has tons of methods:

```ruby
card.comments.build(body: "...")   # Build unsaved Comment
card.comments.create(body: "...")  # Create and save Comment
card.comments.find(id)             # Find within the collection
card.comments.where(...)           # Scope the collection
card.comments.count                # Count comments
card.comments.empty?               # Check if empty
card.comments.first / .last        # Get first/last
card.comments << new_comment       # Add to collection
card.comments.delete(comment)      # Remove from collection
```

These aren't generated individually - `CollectionProxy` is a class that delegates to an underlying relation.

**Ruby features used:** `define_method`, delegation, callbacks

---

## Part 5: `scope` - Class Method Generation

```ruby
scope :reverse_chronologically, -> { order(created_at: :desc, id: :desc) }
scope :chronologically,         -> { order(created_at: :asc,  id: :asc) }
```

### What `scope` Does

```ruby
def self.scope(name, body)
  # Validate it's a callable (lambda/proc)
  raise ArgumentError unless body.respond_to?(:call)

  # Define a class method with that name
  singleton_class.define_method(name) do |*args|
    # Call the lambda and merge into current scope
    scoping { body.call(*args) }
  end
end
```

### After Processing

```ruby
Card.reverse_chronologically
# => Card.order(created_at: :desc, id: :desc)

Card.chronologically
# => Card.order(created_at: :asc, id: :asc)
```

### Scopes Are Chainable

Because scopes return `ActiveRecord::Relation`, they chain:

```ruby
Card.reverse_chronologically.where(status: "active").limit(10)
```

### Scope with Arguments

```ruby
scope :indexed_by, ->(index) do
  case index
  when "stalled" then stalled
  when "closed" then closed
  else all
  end
end
```

The lambda receives arguments:

```ruby
Card.indexed_by("stalled")  # Calls the lambda with "stalled"
```

**Ruby features used:** `define_singleton_method`, lambdas, `call`

---

## Part 6: Callbacks - `before_save`, `after_create`, etc.

```ruby
before_save :set_default_title, if: :published?
before_create :assign_number

after_save   -> { board.touch }, if: :published?
after_update :handle_board_change, if: :saved_change_to_board_id?
```

### The Callback Registry

Rails maintains a list of callbacks for each hook point:

```ruby
class ActiveRecord::Base
  class_attribute :_save_callbacks, default: []
  class_attribute :_create_callbacks, default: []
  class_attribute :_update_callbacks, default: []
  # ... etc
end
```

### Registering a Callback

```ruby
def self.before_save(*methods, **options, &block)
  set_callback(:save, :before, *methods, **options, &block)
end

def self.set_callback(name, kind, *filters, **options)
  filters.each do |filter|
    _save_callbacks << {
      kind: kind,           # :before, :after, :around
      filter: filter,       # :set_default_title or a lambda
      options: options      # { if: :published? }
    }
  end
end
```

### Callback Execution

When you call `card.save`, Rails runs through callbacks:

```ruby
def save
  run_callbacks(:save) do
    if new_record?
      run_callbacks(:create) { insert_record }
    else
      run_callbacks(:update) { update_record }
    end
  end
end

def run_callbacks(kind)
  callbacks = self.class.public_send("_#{kind}_callbacks")

  # Run :before callbacks
  callbacks.select { |c| c[:kind] == :before }.each do |callback|
    next unless should_run?(callback)  # Check :if/:unless conditions
    run_callback(callback)
  end

  # Run the actual operation
  result = yield

  # Run :after callbacks
  callbacks.select { |c| c[:kind] == :after }.each do |callback|
    next unless should_run?(callback)
    run_callback(callback)
  end

  result
end
```

### The `:if` and `:unless` Conditions

```ruby
before_save :set_default_title, if: :published?
```

Rails checks the condition before running the callback:

```ruby
def should_run?(callback)
  if_condition = callback[:options][:if]
  unless_condition = callback[:options][:unless]

  if if_condition
    # If it's a symbol, call the method
    # If it's a proc, call it
    result = if_condition.is_a?(Symbol) ? send(if_condition) : if_condition.call(self)
    return false unless result
  end

  if unless_condition
    result = unless_condition.is_a?(Symbol) ? send(unless_condition) : unless_condition.call(self)
    return false if result
  end

  true
end
```

### Lambda vs Symbol Callbacks

```ruby
# Symbol - calls instance method
before_save :set_default_title

# Lambda - executes the code directly
after_save -> { board.touch }, if: :published?
```

Both work because `run_callback` handles both:

```ruby
def run_callback(callback)
  filter = callback[:filter]

  if filter.is_a?(Symbol)
    send(filter)  # Call the method by name
  else
    instance_exec(&filter)  # Execute lambda in instance context
  end
end
```

**Ruby features used:** `class_attribute`, arrays, `send`, `instance_exec`, lambdas

---

## Part 7: `delegate` - Simple Forwarding

```ruby
delegate :accessible_to?, to: :board
```

### What It Generates

```ruby
def accessible_to?(*args, &block)
  board.public_send(:accessible_to?, *args, &block)
end
```

### The Implementation

```ruby
def delegate(*methods, to:, prefix: nil, allow_nil: false)
  methods.each do |method|
    # Handle prefix option
    method_name = prefix ? "#{prefix}_#{method}" : method

    define_method(method_name) do |*args, &block|
      target = send(to)  # Get the target object

      if allow_nil && target.nil?
        nil
      else
        target.public_send(method, *args, &block)
      end
    end
  end
end
```

**Ruby features used:** `define_method`, `send`, `public_send`

---

## Part 8: `ActiveSupport::Concern` - The Module Pattern

Most Fizzy modules look like this:

```ruby
module Eventable
  extend ActiveSupport::Concern

  included do
    has_many :events, as: :eventable, dependent: :destroy
  end

  class_methods do
    def some_class_method
      # Available as Card.some_class_method
    end
  end

  def track_event(action, ...)
    # Available as card.track_event
  end
end
```

### How `Concern` Works

```ruby
module ActiveSupport::Concern
  def self.extended(base)
    base.instance_variable_set(:@_dependencies, [])
  end

  def included(base = nil, &block)
    if block_given?
      # Store the block for later
      @_included_block = block
    else
      # Actually being included in a class
      super
    end
  end

  def append_features(base)
    # When this module is included...
    super

    # Run the included block in the including class's context
    if @_included_block
      base.class_eval(&@_included_block)
    end

    # Extend with class methods
    if const_defined?(:ClassMethods)
      base.extend(const_get(:ClassMethods))
    end
  end

  def class_methods(&block)
    # Create a ClassMethods module with the given block
    mod = const_defined?(:ClassMethods) ? const_get(:ClassMethods) : const_set(:ClassMethods, Module.new)
    mod.module_eval(&block)
  end
end
```

### The Three Sections

| Section | What it does | How it works |
|---------|-------------|--------------|
| `included do ... end` | Runs code when module is included | `class_eval` in including class |
| `class_methods do ... end` | Adds class methods | Creates `ClassMethods` module, uses `extend` |
| Regular methods | Instance methods | Normal module inclusion |

**Ruby features used:** `extend`, `class_eval`, `module_eval`, `const_set`, `append_features` hook

---

## Part 9: Attribute Methods - Where `title` and `title=` Come From

You never defined `title`, `title=`, or `title?` - but they exist:

```ruby
card.title         # => "My Card"
card.title = "New" # Works!
card.title?        # => true (present?)
```

### How Rails Generates Attribute Methods

When Rails loads your model, it reads the database schema:

```ruby
# Rails queries: DESCRIBE cards;
# Gets columns: id, title, board_id, created_at, ...

Card.columns.map(&:name)
# => ["id", "title", "board_id", "status", "created_at", "updated_at", ...]
```

For each column, Rails generates methods:

```ruby
Card.columns.each do |column|
  # Getter
  define_method(column.name) do
    read_attribute(column.name)
  end

  # Setter
  define_method("#{column.name}=") do |value|
    write_attribute(column.name, value)
  end

  # Query (boolean check)
  define_method("#{column.name}?") do
    query_attribute(column.name)
  end

  # Dirty tracking
  define_method("#{column.name}_changed?") do
    attribute_changed?(column.name)
  end

  define_method("#{column.name}_was") do
    attribute_was(column.name)
  end

  # ... and more
end
```

### Lazy Generation

Actually, Rails doesn't generate all methods upfront - it uses `method_missing` for initial access, then defines the method for future calls:

```ruby
def method_missing(method_name, *args)
  if self.class.column_names.include?(method_name.to_s.chomp("=").chomp("?"))
    # It's an attribute - define the methods now
    self.class.define_attribute_methods
    # Then retry
    send(method_name, *args)
  else
    super
  end
end
```

This is why the first access might be slightly slower - Rails is generating methods on-demand.

**Ruby features used:** `define_method`, `method_missing`, database introspection

---

## The Complete Picture

Here's everything that happens when Ruby loads `Card`:

```
Ruby loads card.rb
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  class Card < ApplicationRecord                          │
│  ├── Inheritance: Add ApplicationRecord to ancestors     │
│  │                                                       │
│  ├── include Eventable, Watchable, ...                   │
│  │   ├── Insert modules into ancestor chain              │
│  │   └── Run each module's `included` block              │
│  │                                                       │
│  ├── belongs_to :board                                   │
│  │   ├── Store reflection (association config)           │
│  │   └── define_method: board, board=, board_id, etc.    │
│  │                                                       │
│  ├── has_many :comments                                  │
│  │   ├── Store reflection                                │
│  │   └── define_method: comments, comments=, etc.        │
│  │                                                       │
│  ├── scope :chronologically, -> { ... }                  │
│  │   └── define_singleton_method(:chronologically)       │
│  │                                                       │
│  ├── before_save :set_default_title                      │
│  │   └── Add to _save_callbacks array                    │
│  │                                                       │
│  ├── delegate :accessible_to?, to: :board                │
│  │   └── define_method(:accessible_to?)                  │
│  │                                                       │
│  └── def filled? ... end                                 │
│      └── Normal Ruby method definition                   │
│                                                          │
│  (Later, on first attribute access)                      │
│  └── Define attribute methods from DB schema             │
└─────────────────────────────────────────────────────────┘
```

---

## Debugging Rails Magic

Now that you understand how it works, here are tools to inspect it:

```ruby
# See all generated methods for an association
Card.instance_methods.grep(/board/)
# => [:board, :board=, :board_id, :board_id=, :build_board, ...]

# See where a method is defined
card.method(:board).source_location
# => [".../activerecord/lib/active_record/associations/builder/...", 42]

# See the association configuration
Card.reflect_on_association(:board)
# => #<ActiveRecord::Reflection::BelongsToReflection ...>

# See all associations
Card.reflections.keys
# => ["account", "board", "creator", "comments", "image_attachment", ...]

# See all callbacks
Card._save_callbacks.map { |c| [c.kind, c.filter] }
# => [[:before, :set_default_title], [:after, #<Proc...>], ...]

# See the ancestor chain (module lookup order)
Card.ancestors
# => [Card, Watchable, Triageable, ..., ApplicationRecord, ...]

# See all scopes
Card.methods.grep(/chronologically/)
# => [:chronologically, :reverse_chronologically]
```

---

## Summary: The Magic Is Just Ruby

| Rails Feature | Ruby Mechanism | What It Actually Does |
|--------------|----------------|----------------------|
| `belongs_to :x` | `define_method` | Creates 7+ methods for the association |
| `has_many :x` | `define_method` | Creates collection methods |
| `scope :x` | `define_singleton_method` | Creates a class method that returns a relation |
| `validates :x` | Class attributes | Stores validation rules for later execution |
| `before_save` | Class attributes + `send` | Stores callback, runs via `send` at save time |
| `delegate :x` | `define_method` + `public_send` | Creates forwarding method |
| `attr_accessor` | `define_method` | Creates getter and setter |
| `include Module` | Ancestor chain | Inserts module, runs `included` block |
| Attribute methods | `method_missing` + `define_method` | Generated lazily from DB schema |

**There is no magic.** Every Rails feature is built from:
1. **Class methods** that store configuration
2. **`define_method`** to generate instance/class methods
3. **Callbacks/hooks** that run at the right time
4. **`send`** to call methods by dynamic names

---

## You've Made It

You now understand:
- **Ruby fundamentals** - the language itself
- **Object model** - how methods are found
- **Metaprogramming** - how methods are created
- **Rails magic** - how Rails uses all of the above

The next time you write `belongs_to :user`, you'll know that Rails is calling a class method that uses `define_method` to generate `user`, `user=`, `user_id`, `build_user`, etc.

The next time something doesn't work, you can debug it:
```ruby
User.reflect_on_association(:posts)  # Check association config
user.method(:posts).source_location  # Find where it's defined
User._save_callbacks                 # See registered callbacks
```

**You're no longer a Rails user who treats it as magic. You're a Ruby developer who understands how Rails works.**

---

## Q&A

*Questions from our learning sessions will be added here.*

---

## Quiz History

*Quiz sessions will be recorded here.*
