---
concepts: [Concerns, ActiveSupport::Concern, included, class_methods, Model Organization]
source_repo: fizzy
description: Deep dive into Rails concerns - the pattern 37signals uses to keep models manageable. Learn what concerns are, when to extract them, and how the included/class_methods blocks work. Uses real examples from Fizzy showing how a Card model can include 20+ concerns without becoming chaos.
understanding_score: null
last_quizzed: null
prerequisites: [2025-11-25-activerecord---how-models-talk-to-databases.md, 2025-11-29-activerecord-associations.md]
created: 09-12-2025
last_updated: 09-12-2025
---

# Concerns: Breaking Up God Models the 37signals Way

You know that feeling when you open a model file and it's 800 lines long? You scroll... and scroll... and somewhere around line 400 you've completely forgotten what you came looking for. That's what we call a "God Model" - a model that knows too much and does too much.

Here's the thing: a `User` model legitimately needs a lot of behavior. Authentication, profile management, settings, notifications, permissions... it all *belongs* to User. The problem isn't that User has responsibilities - it's that jamming everything into one file makes it unreadable.

Enter **concerns**.

## The Problem Concerns Solve

Look at Fizzy's `Card` model. A card in a kanban board needs to:

- Be closeable (marked as done)
- Be postponable (moved to "not now")
- Be taggable (have labels)
- Be watchable (users can subscribe to updates)
- Be searchable (show up in search results)
- Be assignable (have people working on it)
- Track events (record history)
- Handle colors, steps, mentions, pins...

If all that code lived in one file, you'd have a 2000+ line nightmare. Instead, here's the *entire* `Card` model:

```ruby
# app/models/card.rb
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

  # ... about 50 more lines of core card logic
end
```

That's it. The full file is under 100 lines. All that behavior - closeable, postponable, taggable, etc. - lives in separate concern files that are `include`d here.

## What Is a Concern, Really?

A concern is just a Ruby module with some Rails superpowers. Let's look at the simplest possible example first.

Without any Rails magic, you could organize code with plain modules:

```ruby
# Plain Ruby module
module Taggable
  def tagged_with?(tag)
    tags.include?(tag)
  end
end

class Card < ApplicationRecord
  include Taggable
end
```

This works! When you `include` a module, its methods become instance methods on your class. But here's where plain modules fall short...

**What if you need to:**
- Define associations (`has_many :tags`)?
- Add scopes (`scope :tagged_with, ->...`)?
- Set up callbacks (`after_create :do_something`)?

These are **class-level** declarations in Rails. Plain `include` only gives you instance methods. You *could* use Ruby's `self.included` hook, but it gets ugly fast.

That's why Rails gives us `ActiveSupport::Concern`.

## The Anatomy of a Concern

Here's Fizzy's actual `Card::Taggable` concern:

```ruby
# app/models/card/taggable.rb
module Card::Taggable
  extend ActiveSupport::Concern

  included do
    has_many :taggings, dependent: :destroy
    has_many :tags, through: :taggings

    scope :tagged_with, ->(tags) { joins(:taggings).where(taggings: { tag: tags }) }
  end

  def toggle_tag_with(title)
    tag = account.tags.find_or_create_by!(title: title)

    transaction do
      if tagged_with?(tag)
        taggings.destroy_by tag: tag
      else
        taggings.create tag: tag
      end
    end
  end

  def tagged_with?(tag)
    tags.include? tag
  end
end
```

Let's break this down piece by piece.

### `extend ActiveSupport::Concern`

This line gives your module the special powers. Without it, you have a plain Ruby module. With it, you get access to the `included` block and `class_methods` block.

### The `included` Block

```ruby
included do
  has_many :taggings, dependent: :destroy
  has_many :tags, through: :taggings

  scope :tagged_with, ->(tags) { joins(:taggings).where(taggings: { tag: tags }) }
end
```

Everything inside `included do ... end` runs in the context of the class that includes this concern. It's as if you wrote those lines directly in your `Card` model.

This is the magic sauce. When `Card` does `include Taggable`:
1. The `included` block runs, adding associations and scopes to `Card`
2. The instance methods (`toggle_tag_with`, `tagged_with?`) become available on card instances

### Instance Methods (Outside `included`)

```ruby
def toggle_tag_with(title)
  # ...
end

def tagged_with?(tag)
  tags.include? tag
end
```

Methods defined directly in the module (outside any block) become instance methods. You call them on individual records: `card.toggle_tag_with("urgent")`.

## A More Complex Example: Closeable

Let's look at `Card::Closeable`, which is richer:

```ruby
# app/models/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy

    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }

    scope :recently_closed_first, -> { closed.order("closures.created_at": :desc) }
    scope :closed_at_window, ->(window) { closed.where("closures.created_at": window) }
    scope :closed_by, ->(users) { closed.where("closures.user_id": Array(users)) }
  end

  def closed?
    closure.present?
  end

  def open?
    !closed?
  end

  def closed_by
    closure&.user
  end

  def closed_at
    closure&.created_at
  end

  def close(user: Current.user)
    unless closed?
      transaction do
        create_closure! user: user
        track_event :closed, creator: user
      end
    end
  end

  def reopen(user: Current.user)
    if closed?
      transaction do
        closure&.destroy
        track_event :reopened, creator: user
      end
    end
  end
end
```

Notice the pattern here:

1. **Association** (`has_one :closure`) - The "closed" state is stored in a separate `closures` table
2. **Scopes** for querying (`closed`, `open`, `closed_by`)
3. **Predicate methods** (`closed?`, `open?`) for checking state
4. **Accessor methods** (`closed_by`, `closed_at`) for getting related data
5. **Action methods** (`close`, `reopen`) for changing state

This is a beautifully self-contained unit of behavior. Everything about "is this card closed?" lives in one 48-line file.

### The Brilliant Pattern: State as a Record

Notice something clever? Instead of adding a `closed_at` column to `cards`, Fizzy creates a separate `Closure` record. Why?

1. **Clear semantics**: A card is closed if and only if a `closure` record exists
2. **Rich metadata**: The closure can store who closed it, when, maybe even why
3. **Clean queries**: `where.missing(:closure)` is elegant
4. **History**: If you wanted to track "closed/reopened/closed again", you could

You'll see this pattern repeated in `Card::Postponable` (has_one `:not_now`), `Card::Golden` (has_one `:goldness`), etc.

## When Concerns Call Each Other

Look at `close` again:

```ruby
def close(user: Current.user)
  unless closed?
    transaction do
      create_closure! user: user
      track_event :closed, creator: user  # <- This!
    end
  end
end
```

Where does `track_event` come from? It's from the `Eventable` concern! Since `Card` includes both `Closeable` and `Eventable`, methods from one concern can call methods from another.

This is powerful but requires care. The `Closeable` concern assumes that whatever class includes it *also* includes `Eventable`. This is an implicit dependency.

In Fizzy, this is fine because `Card` carefully includes all the concerns that need each other. But it's worth being aware: concerns aren't fully independent when they call each other's methods.

## Shared vs. Model-Specific Concerns

Fizzy organizes concerns in two places:

**`app/models/concerns/`** - Shared concerns used by multiple models:
```
app/models/concerns/
├── attachments.rb      # Used by Card and Comment
├── eventable.rb        # Used by Card and Comment
├── filterable.rb
├── mentions.rb         # Used by Card and Comment
├── notifiable.rb
└── searchable.rb       # Used by Card and Comment
```

**`app/models/card/`** - Concerns specific to Card:
```
app/models/card/
├── closeable.rb
├── entropic.rb
├── golden.rb
├── postponable.rb
├── statuses.rb
├── taggable.rb
├── watchable.rb
└── ... (many more)
```

The naming convention matters:
- `Eventable` (in `concerns/`) - shared, generic name
- `Card::Closeable` (in `card/`) - namespaced under Card, specific to cards

When you `include Closeable` inside `Card`, Ruby finds `Card::Closeable` first (because it looks in the current namespace). That's why the namespacing works.

## The `class_methods` Block

Sometimes you need to add class methods, not just instance methods. That's what `class_methods` is for:

```ruby
# app/models/card/entropic.rb
module Card::Entropic
  extend ActiveSupport::Concern

  included do
    scope :due_to_be_postponed, -> { ... }
    scope :postponing_soon, -> { ... }
  end

  class_methods do
    def auto_postpone_all_due
      due_to_be_postponed.find_each do |card|
        card.auto_postpone(user: card.account.system_user)
      end
    end
  end

  def entropy
    Card::Entropy.for(self)
  end

  def entropic?
    entropy.present?
  end
end
```

The `class_methods` block adds methods you call on the class itself:

```ruby
Card.auto_postpone_all_due  # Class method from class_methods block
card.entropic?              # Instance method defined directly
```

Think of it this way:
- Instance methods = called on a single record (`card.close`)
- Class methods = called on the model itself (`Card.auto_postpone_all_due`)

Scopes are technically class methods too, but the `scope` DSL is cleaner than writing them in `class_methods`.

## A Shared Concern: Searchable

Let's look at a concern designed to be shared across models:

```ruby
# app/models/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  included do
    after_create_commit :create_in_search_index
    after_update_commit :update_in_search_index
    after_destroy_commit :remove_from_search_index
  end

  def reindex
    update_in_search_index
  end

  private
    def create_in_search_index
      search_record_class.create!(search_record_attributes)
    end

    def update_in_search_index
      search_record_class.upsert!(search_record_attributes)
    end

    def remove_from_search_index
      search_record_class.find_by(searchable_type: self.class.name, searchable_id: id)&.destroy
    end

    def search_record_attributes
      {
        account_id: account_id,
        searchable_type: self.class.name,
        searchable_id: id,
        card_id: search_card_id,
        board_id: search_board_id,
        title: search_title,
        content: search_content,
        created_at: created_at
      }
    end

    def search_record_class
      Search::Record.for(account_id)
    end

  # Models must implement these methods:
  # - account_id: returns the account id
  # - search_title: returns title string or nil
  # - search_content: returns content string
  # - search_card_id: returns the card id
  # - search_board_id: returns the board id
end
```

This concern sets up callbacks that automatically sync records to a search index. But notice - it calls methods like `search_title` and `search_content` that it doesn't define!

This is a **contract**: any model that includes `Searchable` must implement those methods. The concern provides the machinery; the model provides the specifics.

Here's how `Card` fulfills that contract (through `Card::Searchable`):

```ruby
# app/models/card/searchable.rb
module Card::Searchable
  extend ActiveSupport::Concern
  include ::Searchable  # Include the shared concern

  private
    def search_title
      title
    end

    def search_content
      description.body.to_plain_text
    end

    def search_card_id
      id
    end

    def search_board_id
      board_id
    end
end
```

See what happened? `Card::Searchable` includes the shared `Searchable` and then implements the required methods. When `Card` includes `Card::Searchable`, it gets the full package.

This layering is elegant:
1. **Shared concern** (`Searchable`) = the mechanism
2. **Model-specific concern** (`Card::Searchable`) = the data mapping

## When to Extract a Concern

Not everything should be a concern. Here's my mental checklist:

**Good candidates for concerns:**

1. **Cohesive behavior** - The methods clearly belong together (everything about closing, everything about tagging)
2. **Reusability** - Multiple models need the same behavior (searchable, eventable)
3. **Testability** - You want to test this behavior in isolation
4. **Readability** - Extracting it makes the main model easier to scan

**Bad candidates:**

1. **One method** - Don't create a concern for a single method. That's over-engineering.
2. **Unrelated methods** - A concern called `Helpers` or `Utilities` with random methods is a code smell
3. **Heavy dependencies** - If your concern needs to know intimate details of the main model, it probably shouldn't be separate

**The 37signals rule of thumb:** If you find yourself scrolling past code to find what you need, it might be time to extract a concern.

## The File Organization Convention

Fizzy uses a clean pattern for organizing concerns:

```
app/models/
├── card.rb                      # The main model (just includes + core logic)
├── card/
│   ├── closeable.rb            # Card::Closeable
│   ├── postponable.rb          # Card::Postponable
│   ├── taggable.rb             # Card::Taggable
│   └── ...
├── concerns/
│   ├── eventable.rb            # Eventable (shared)
│   └── searchable.rb           # Searchable (shared)
```

This maps directly to how you think about it:
- "How does closing work?" → `app/models/card/closeable.rb`
- "How does search work across models?" → `app/models/concerns/searchable.rb`

## Common Gotchas

### 1. Concern Load Order

If concern A depends on concern B, make sure B is included first:

```ruby
class Card < ApplicationRecord
  include Eventable    # Must come before Closeable
  include Closeable    # Uses track_event from Eventable
end
```

In practice, Rails usually figures this out, but explicit ordering prevents surprises.

### 2. Method Conflicts

If two concerns define the same method, the last one included wins. This is standard Ruby module behavior, but it can cause subtle bugs.

### 3. The "Concerns Are Just Mixins" Trap

Don't treat concerns as a dumping ground. "I'll just throw this in a concern" without thinking about cohesion leads to spaghetti code distributed across files instead of in one file. That's worse, not better.

## Your Mental Model

Think of concerns like this:

> A **model** is the main character. **Concerns** are the roles that character plays.

A `Card` plays the role of something closeable, something taggable, something searchable. Each role has its own script (the concern file), but when the show runs, they all come together as one unified `Card`.

The beauty is that you can:
- Read the main model to understand what roles it plays
- Dive into a specific concern to understand how that role works
- Test each role independently

## Challenge: Trace a Feature Through Concerns

Here's something to try in the Fizzy codebase:

1. Open `app/models/card.rb`
2. Pick any concern from the `include` list (maybe `Postponable`)
3. Find the concern file and read through it
4. Trace how the concern's methods would be called when a user postpones a card
5. Notice how it interacts with other concerns (does it call `track_event`? Does it depend on `Closeable`?)

Understanding how concerns collaborate is what separates "I know the syntax" from "I understand the architecture."

---

## Q&A

(Questions from our discussions will be added here)

## Quiz History

(Quiz sessions will be recorded here)
