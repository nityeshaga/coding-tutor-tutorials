---
concepts: [Object Model, Method Lookup, Inheritance, Modules, include, extend, prepend, Eigenclass, Class Methods]
source_repo: fizzy
description: Deep dive into Ruby's object model - the system that determines what happens when you call a method. Covers the method lookup chain, inheritance, modules (include/extend/prepend), class methods vs instance methods, and the eigenclass. Essential foundation for understanding how Rails' "magic" actually works.
understanding_score: null
last_quizzed: null
prerequisites: [2025-12-22-ruby-101---the-language-beneath-rails.md]
created: 22-12-2025
last_updated: 22-12-2025  # Q&A: module naming conventions
---

# Ruby's Object Model: How Methods Get Found

When you write `card.save`, how does Ruby know what code to run? The `Card` class doesn't define a `save` method. Neither does `ApplicationRecord`. Yet it works.

When you write `Card.find(1)`, you're calling a method on the *class itself*, not on an instance. How does that work?

When you `include` 20 modules into a class, what happens when two modules define the same method?

This tutorial answers these questions. Understanding Ruby's object model is the key to understanding how Rails builds its "magic" - and it's all pure Ruby.

---

## The Big Idea: Method Lookup Chain

When you call a method on an object, Ruby doesn't just look in that object's class. It follows a specific search path called the **method lookup chain** (or "ancestor chain").

```ruby
card = Card.new
card.save
```

Ruby's search for `save`:

1. Look in `Card` - not there
2. Look in modules included in `Card` - not there (well, depends on the module)
3. Look in `ApplicationRecord` - not there
4. Look in modules included in `ApplicationRecord` - not there
5. Look in `ActiveRecord::Base` - **found it!**

This chain is deterministic. You can actually see it:

```ruby
Card.ancestors
# => [Card, Watchable, Triageable, Taggable, Statuses, Stallable, Searchable,
#     Readable, Promptable, Postponable, Pinnable, Multistep, Mentions, Golden,
#     Exportable, Eventable, Entropic, Colored, Closeable, Broadcastable,
#     Attachments, Assignable, ApplicationRecord, ActiveRecord::Base, ...]
```

Ruby searches this list from left to right. First match wins.

---

## Instance Methods vs Class Methods

This is fundamental, and it confuses a lot of people.

### Instance Methods

Methods you call on *instances* (objects created with `.new`):

```ruby
card = Card.new          # Create an instance
card.save                # Instance method - operates on THIS card
card.filled?             # Instance method - defined in Card class
card.track_event(...)    # Instance method - from Eventable module
```

Instance methods are defined with `def method_name` inside a class:

```ruby
class Card < ApplicationRecord
  def filled?                    # Instance method
    title.present? || description.present?
  end
end
```

### Class Methods

Methods you call on the *class itself*:

```ruby
Card.find(1)             # Class method - finds a card
Card.where(board: board) # Class method - queries cards
Card.create!(title: "X") # Class method - creates a card
```

Class methods are defined differently:

```ruby
class Card < ApplicationRecord
  def self.find_by_number(n)     # Class method (using self.)
    find_by(number: n)
  end
end

# Or equivalently:
class Card < ApplicationRecord
  class << self                   # Opens the eigenclass (more on this later)
    def find_by_number(n)
      find_by(number: n)
    end
  end
end
```

### Why the Distinction Matters

Instance methods have access to `self` (the specific object) and its instance variables:

```ruby
class Card
  def filled?
    # self is the specific card instance
    # Can access @title, @description, etc.
    title.present?
  end
end

card1.filled?  # self = card1
card2.filled?  # self = card2
```

Class methods have `self` as the class:

```ruby
class Card
  def self.count
    # self is Card (the class)
    # No access to any specific card's data
  end
end

Card.count  # self = Card
```

---

## Inheritance: The `<` Operator

You know this syntax:

```ruby
class Card < ApplicationRecord
end
```

This means `Card` inherits from `ApplicationRecord`. But what does "inherit" actually mean?

**Inheritance adds the parent class to the method lookup chain.**

```ruby
class Animal
  def breathe
    "inhale... exhale..."
  end
end

class Dog < Animal
  def bark
    "woof!"
  end
end

dog = Dog.new
dog.bark     # Found in Dog
dog.breathe  # Not in Dog, found in Animal
```

The lookup chain:
```ruby
Dog.ancestors
# => [Dog, Animal, Object, Kernel, BasicObject]
```

### Fizzy's Inheritance Chain

```ruby
Card < ApplicationRecord < ActiveRecord::Base < Object < BasicObject
```

When you call `card.save`:
- Not in `Card`
- Not in `ApplicationRecord`
- Found in `ActiveRecord::Base`

That's why your models have hundreds of methods you didn't write - they're inherited.

---

## Modules: Ruby's Mixins

Inheritance has a limitation: a class can only inherit from ONE parent (single inheritance).

But what if you want to share behavior across unrelated classes? That's what modules are for.

```ruby
module Swimmable
  def swim
    "splash splash"
  end
end

class Fish
  include Swimmable
end

class Duck
  include Swimmable
end

Fish.new.swim  # => "splash splash"
Duck.new.swim  # => "splash splash"
```

`Fish` and `Duck` aren't related by inheritance, but they share the `Swimmable` behavior.

### The `include` Keyword

`include` adds a module's methods as **instance methods**:

```ruby
module Eventable
  def track_event(action, ...)
    # ...
  end
end

class Card
  include Eventable
end

card = Card.new
card.track_event("created")  # Instance method from Eventable
```

**Crucially, `include` inserts the module into the ancestor chain:**

```ruby
# Before include:
Card.ancestors  # => [Card, ApplicationRecord, ...]

# After include Eventable:
Card.ancestors  # => [Card, Eventable, ApplicationRecord, ...]
```

The module sits between the class and its parent. This is how Ruby finds methods from included modules.

### Multiple Includes

Fizzy's `Card` includes 20+ modules:

```ruby
class Card < ApplicationRecord
  include Assignable, Attachments, Broadcastable, Closeable, Colored, Entropic,
    Eventable, Exportable, Golden, Mentions, Multistep, Pinnable, Postponable,
    Promptable, Readable, Searchable, Stallable, Statuses, Taggable, Triageable,
    Watchable
end
```

Each `include` adds to the ancestor chain. **Later includes are searched first:**

```ruby
Card.ancestors
# => [Card, Watchable, Triageable, Taggable, ..., Assignable, ApplicationRecord, ...]
```

`Watchable` (last in the include list) is first in the lookup! This is because each `include` inserts the module right after the class.

Think of it like a stack:
1. `include Assignable` → [Card, Assignable, ...]
2. `include Attachments` → [Card, Attachments, Assignable, ...]
3. ...
4. `include Watchable` → [Card, Watchable, ..., Assignable, ...]

---

## The Three Module Insertion Methods

Ruby gives you three ways to add modules to classes, each inserting at a different point:

### `include` - Instance Methods, After the Class

```ruby
module Greeting
  def greet
    "Hello!"
  end
end

class Person
  include Greeting
end

Person.ancestors  # => [Person, Greeting, Object, ...]
Person.new.greet  # => "Hello!" (instance method)
```

### `extend` - Class Methods

```ruby
module Findable
  def find_by_name(name)
    # ...
  end
end

class Person
  extend Findable
end

Person.find_by_name("Alice")  # Class method!
Person.new.find_by_name("Alice")  # ERROR - not an instance method
```

`extend` adds the module's methods to the **class itself**, not to instances.

### `prepend` - Instance Methods, BEFORE the Class

```ruby
module Logging
  def save
    puts "About to save..."
    super  # Call the original save
    puts "Saved!"
  end
end

class Card < ApplicationRecord
  prepend Logging
end

Card.ancestors  # => [Logging, Card, ApplicationRecord, ...]
```

Notice `Logging` comes *before* `Card`. When you call `card.save`:

1. Ruby finds `save` in `Logging` first
2. `Logging#save` calls `super`
3. `super` continues up the chain to `Card`, then `ApplicationRecord`, etc.

`prepend` is powerful for wrapping existing methods without modifying them.

### Summary Table

| Method | Adds to | Position in Chain | Use Case |
|--------|---------|-------------------|----------|
| `include` | Instances | After class | Share instance behavior |
| `extend` | Class | Class's eigenclass | Share class methods |
| `prepend` | Instances | Before class | Wrap/override methods |

---

## Class Methods in Modules: The `class_methods` Block

Here's a real pattern from Fizzy. Look at `app/controllers/concerns/authentication.rb`:

```ruby
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
    helper_method :authenticated?
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
    end
  end

  private
    def authenticated?
      Current.session.present?
    end
end
```

There are three sections here:

### 1. `included do ... end`

Code that runs when the module is included. Typically used to call class-level methods like `before_action`:

```ruby
included do
  before_action :require_authentication  # Runs in the context of the including class
end
```

### 2. `class_methods do ... end`

Methods that become **class methods** on the including class:

```ruby
class_methods do
  def allow_unauthenticated_access(**options)
    # This becomes SessionsController.allow_unauthenticated_access
  end
end
```

After including `Authentication`:
```ruby
class SessionsController < ApplicationController
  include Authentication
  allow_unauthenticated_access  # Now available as a class method!
end
```

### 3. Regular Methods

These become **instance methods**:

```ruby
def authenticated?
  Current.session.present?
end

# Usage:
controller.authenticated?  # Instance method
```

### Why `extend ActiveSupport::Concern`?

This is Rails' helper that provides `included` and `class_methods` blocks. Without it, you'd write:

```ruby
module Authentication
  def self.included(base)
    base.class_eval do
      before_action :require_authentication
    end
    base.extend(ClassMethods)
  end

  module ClassMethods
    def allow_unauthenticated_access(**options)
      # ...
    end
  end

  def authenticated?
    Current.session.present?
  end
end
```

`ActiveSupport::Concern` is just syntactic sugar. The underlying mechanism is pure Ruby.

---

## The Eigenclass (Singleton Class)

This is where it gets mind-bending, but stay with me - it's the key to understanding Ruby deeply.

Every object in Ruby has a hidden class just for itself called the **eigenclass** (or "singleton class" or "metaclass").

### Why Does This Exist?

Remember: in Ruby, classes are objects too. When you write:

```ruby
class Card
  def self.find(id)
    # ...
  end
end
```

Where does `find` actually live? It can't be in `Card` - that would make it an instance method. It's a method on the `Card` object (the class).

The answer: it lives in `Card`'s eigenclass.

### Accessing the Eigenclass

```ruby
class Card
  class << self  # Opens Card's eigenclass
    def find(id)
      # ...
    end

    def where(conditions)
      # ...
    end
  end
end
```

`class << self` opens the eigenclass. Methods defined inside become methods on that specific object (in this case, the `Card` class).

### Every Object Can Have One

```ruby
card = Card.new

# Give this specific card a unique method:
class << card
  def special_method
    "I'm special!"
  end
end

card.special_method  # => "I'm special!"
Card.new.special_method  # ERROR - other cards don't have this method
```

### The Mental Model

```
┌─────────────────────────────────────────────────────────┐
│                    Object Space                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│   Card (class object)                                    │
│   ├── eigenclass ─── find, where, create (class methods) │
│   └── instance methods ─── save, filled?, etc.          │
│                                                          │
│   card1 (instance)                                       │
│   ├── eigenclass ─── (usually empty)                    │
│   └── looks up methods in Card                          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### `extend` Adds to the Eigenclass

When you write:

```ruby
class Card
  extend SomeModule
end
```

Ruby is actually doing:

```ruby
class Card
  class << self
    include SomeModule
  end
end
```

It includes the module into the eigenclass, making the module's methods available on `Card` itself.

---

## Putting It All Together: Method Lookup in Fizzy

Let's trace what happens when you call `card.track_event("created")`:

```ruby
card = Card.find(1)
card.track_event("created")
```

**Step 1:** Ruby looks for `track_event` in `card`'s eigenclass.
- Not found (we didn't add methods to this specific card)

**Step 2:** Ruby looks in `Card`'s ancestor chain:
```ruby
Card.ancestors
# => [Card, Watchable, Triageable, ..., Eventable, ..., ApplicationRecord, ...]
```

**Step 3:** Ruby searches each ancestor:
- `Card` - no `track_event`
- `Watchable` - no
- `Triageable` - no
- ... (keeps looking)
- `Eventable` - **YES! Found it!**

```ruby
module Eventable
  def track_event(action, creator: Current.user, board: self.board, **particulars)
    if should_track_event?
      board.events.create!(action: "#{eventable_prefix}_#{action}", ...)
    end
  end
end
```

**Step 4:** Ruby executes `Eventable#track_event` with `self` set to `card`.

---

## The `super` Keyword

When you want to call the same method further up the chain:

```ruby
module Logging
  def save
    puts "Saving..."
    super        # Call save from the next ancestor
  end
end

class Card < ApplicationRecord
  prepend Logging
end
```

`super` says "continue the lookup from where we are and call the next `save` you find."

Ancestor chain: `[Logging, Card, ApplicationRecord, ActiveRecord::Base, ...]`

1. `card.save` → finds `save` in `Logging`
2. `Logging#save` calls `super`
3. Ruby continues from `Logging`'s position → finds `save` in `ActiveRecord::Base`
4. Executes `ActiveRecord::Base#save`

### `super` with Arguments

```ruby
super          # Pass all arguments received
super()        # Pass no arguments
super(x, y)    # Pass specific arguments
```

---

## Real Fizzy Example: How `Card` Gets Its Methods

```ruby
class Card < ApplicationRecord
  include Assignable, Attachments, ..., Eventable, ..., Watchable

  def filled?
    title.present? || description.present?
  end
end
```

Where does each method come from?

| Method | Source | How It Got There |
|--------|--------|------------------|
| `card.filled?` | `Card` | Defined directly |
| `card.track_event` | `Eventable` | Module included |
| `card.save` | `ActiveRecord::Base` | Inherited via `ApplicationRecord` |
| `card.title` | `ActiveRecord::Base` | Dynamically generated from DB column |
| `Card.find(1)` | `ActiveRecord::Base` | Class method inherited |
| `Card.where(...)` | `ActiveRecord::Base` | Class method inherited |
| `Card.chronologically` | `Card` | Scope (class method) defined via `scope` |

The ancestor chain makes this traceable:

```ruby
Card.ancestors.find { |a| a.instance_methods(false).include?(:track_event) }
# => Eventable
```

---

## Debugging Method Lookup

Ruby provides tools to understand where methods come from:

```ruby
# What's the ancestor chain?
Card.ancestors

# Where is this method defined?
card.method(:track_event).owner
# => Eventable

# What methods does this specific class define (not inherited)?
Card.instance_methods(false)
# => [:card, :to_param, :move_to, :filled?, ...]

# Does this class respond to a method?
card.respond_to?(:save)
# => true

# Get the Method object
card.method(:save)
# => #<Method: Card(ActiveRecord::Base)#save>
```

---

## Summary: The Object Model Rules

1. **Everything is an object** - including classes
2. **Method lookup follows the ancestor chain** - searched left to right
3. **Inheritance** adds the parent to the chain
4. **`include`** inserts modules after the class (instance methods)
5. **`extend`** adds to the eigenclass (class methods)
6. **`prepend`** inserts modules before the class (for wrapping)
7. **Every object has an eigenclass** where singleton methods live
8. **`super`** continues the lookup from the current position

---

## Challenge

Look at this Fizzy code:

```ruby
class CardsController < ApplicationController
  include Authentication  # From concerns

  def show
    @card = Card.find(params[:id])
  end
end
```

Questions:
1. When `require_authentication` runs (as a `before_action`), where is that method defined?
2. When you call `Card.find(params[:id])`, trace the lookup - where is `find` defined?
3. The `params` method - where does it come from?

Think about it, then we can discuss in the Q&A!

---

## What's Next

Now you understand how Ruby finds methods. In the next tutorial, we'll see how Ruby can **create methods at runtime** - the metaprogramming features that make Rails' "magic" possible. You'll finally understand how `belongs_to :account` creates the `account` and `account=` methods out of thin air.

---

## Q&A

### Q: What's the naming convention for modules? Is the "-able" suffix standard?

The `-able` suffix is common but it's just one pattern. The convention depends on what the module provides:

**`-able` Suffix - "Can be X'd"**
Used when the module makes objects capable of something:
```ruby
Searchable    # Can be searched
Taggable      # Can be tagged
Watchable     # Can be watched
Exportable    # Can be exported
```

**No Suffix / Noun - The system itself**
When the module represents a subsystem:
```ruby
Authentication  # Handles authentication
Authorization   # Handles authorization
Attachments     # Has attachments
Mentions        # Has mentions
```

**Adjective - A state or quality**
```ruby
Colored   # Has color
Golden    # Can be golden
Entropic  # Has entropy behavior
```

**Fizzy mixes all three** in the Card model:
```ruby
include Assignable,      # -able: can be assigned
        Attachments,     # noun: has attachments
        Closeable,       # -able: can be closed
        Colored,         # adjective: has color
        Eventable,       # -able: can have events
        Statuses         # noun: has statuses
```

**The guiding principle:** Name it for what it provides. Grants an ability → `-able`. Adds a subsystem → noun. Adds a quality → adjective.

---

## Quiz History

*Quiz sessions will be recorded here.*
