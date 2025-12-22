---
concepts: [ActiveRecord, ORM, Associations, Migrations, Scopes, Callbacks]
source_repo: once-campfire
description: Deep dive into ActiveRecord, Rails' ORM that lets you work with databases using Ruby instead of SQL. Covers what an ORM actually is (and why Next.js doesn't have one), the naming magic, associations, scopes, migrations, and callbacks - all with real Campfire examples.
understanding_score: null
prerequisites: [references/tutorials/2025-11-25-mvc---the-architecture-that-powers-every-rails-app.md]
created: 25-11-2025
last_updated: 25-11-2025
---

# ActiveRecord - How Models Talk to Databases

In the MVC tutorial, you learned that models are the "brain" of your app - where business logic lives. But we glossed over something magical: when you write `User.find(1)`, how does Rails know to talk to the database? How does it know which table? How does it turn a database row into a Ruby object you can call methods on?

The answer is **ActiveRecord** - Rails' ORM. Understanding it is the difference between copying code from tutorials and actually knowing what you're doing.

## What Is an ORM? (And Why Next.js Doesn't Have One)

### The Problem ORMs Solve

Databases speak SQL. Ruby speaks... Ruby. They're completely different languages.

Without an ORM, you'd write code like this:

```ruby
# WITHOUT an ORM - raw SQL everywhere
result = database.execute("SELECT * FROM users WHERE id = 1")
user_data = result.first
user_name = user_data["name"]
user_email = user_data["email"]

# Want to update? More SQL strings:
database.execute("UPDATE users SET name = 'New Name' WHERE id = 1")

# Want related data? Even more SQL:
messages = database.execute("SELECT * FROM messages WHERE creator_id = 1")
```

This is tedious, error-prone, and you're constantly context-switching between Ruby and SQL.

### What an ORM Does

**ORM = Object-Relational Mapper**

It maps:
- **Database tables** → Ruby classes
- **Table rows** → Ruby objects
- **Table columns** → Object attributes

```ruby
# WITH ActiveRecord (Rails' ORM)
user = User.find(1)           # Returns a Ruby object
user.name                     # Just access attributes
user.update(name: "New Name") # No SQL strings!
user.messages                 # Related data, automatically
```

The ORM translates your Ruby into SQL behind the scenes. You think in objects, not tables.

### Why Next.js Doesn't Ship With One

You're right that Next.js doesn't include an ORM - and this is a fundamental difference in philosophy.

**Next.js** is a *React framework*. It handles:
- Routing
- Server-side rendering
- API routes
- Build optimization

It deliberately doesn't have opinions about your database. You bring your own: Prisma, Drizzle, raw SQL, a headless CMS, whatever.

**Rails** is a *full-stack framework*. It handles:
- Routing
- Controllers
- Views
- **AND the database layer (ActiveRecord)**

Rails says: "Here's how you do databases. It's already set up. It follows conventions. Just use it."

This is why Rails developers can be productive so quickly - the decisions are made for you. It's also why the "Rails way" exists: ActiveRecord's conventions shape how you think about data.

```
┌─────────────────────────────────────────────────────────────┐
│                    FRAMEWORK COMPARISON                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   NEXT.JS (React Framework)       RAILS (Full-Stack)        │
│   ─────────────────────────       ──────────────────        │
│   ✓ Routing                       ✓ Routing                 │
│   ✓ SSR/SSG                       ✓ Views (ERB)             │
│   ✓ API Routes                    ✓ Controllers             │
│   ✗ Database (bring your own)     ✓ ActiveRecord (ORM)      │
│   ✗ ORM (Prisma? Drizzle?)        ✓ Migrations              │
│                                   ✓ Background Jobs         │
│                                   ✓ Mailers                 │
│                                   ✓ Asset Pipeline          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## The Naming Magic (Convention Over Configuration)

This is where Rails feels like magic. Look at this model:

```ruby
# app/models/user.rb
class User < ApplicationRecord
end
```

That's it. Two lines. But Rails already knows:

| What Rails Infers | How It Knows |
|-------------------|--------------|
| Table name: `users` | Class `User` → pluralize → `users` |
| Primary key: `id` | Convention (every table has `id`) |
| All columns as attributes | Reads the database schema |
| Timestamps: `created_at`, `updated_at` | Convention |

You don't configure any of this. The naming IS the configuration.

**From Campfire's database** (`db/structure.sql:67`):

```sql
CREATE TABLE IF NOT EXISTS "users" (
  "id" integer PRIMARY KEY AUTOINCREMENT NOT NULL,
  "name" varchar NOT NULL,
  "email_address" varchar DEFAULT NULL,
  "password_digest" varchar DEFAULT NULL,
  "active" boolean DEFAULT 1,
  "bio" text DEFAULT NULL,
  ...
);
```

Because the table is `users` and the class is `User`, ActiveRecord connects them automatically. Every column becomes an attribute:

```ruby
user = User.find(1)
user.name           # From the 'name' column
user.email_address  # From the 'email_address' column
user.active?        # Boolean columns get ? methods for free
user.bio            # From the 'bio' column
```

## CRUD Operations

ActiveRecord makes the four fundamental operations trivial:

### Create

```ruby
# These all work:
user = User.create(name: "Alice", email_address: "alice@example.com")

# Or in two steps:
user = User.new(name: "Alice")
user.save

# With a block:
User.create do |u|
  u.name = "Alice"
  u.email_address = "alice@example.com"
end
```

**SQL generated:** `INSERT INTO users (name, email_address, created_at, updated_at) VALUES ('Alice', 'alice@example.com', '2024-01-15 10:30:00', '2024-01-15 10:30:00')`

### Read

```ruby
User.find(1)              # Find by ID (raises error if not found)
User.find_by(name: "Alice") # Find by any attribute (returns nil if not found)
User.where(active: true)  # Returns a collection
User.first                # First record
User.last                 # Last record
User.all                  # Everything (careful with large tables!)
```

### Update

```ruby
user.update(name: "New Name")
# or
user.name = "New Name"
user.save
```

### Delete

```ruby
user.destroy      # Runs callbacks, deletes associated records
user.delete       # Just deletes, no callbacks (use carefully)
```

## Associations: Relationships Between Models

This is where ActiveRecord truly shines. In Campfire, look at how models connect:

### Example 1: User has many rooms through memberships

**Location:** `app/models/user.rb:4-6`

```ruby
class User < ApplicationRecord
  has_many :memberships, dependent: :delete_all
  has_many :rooms, through: :memberships
end
```

**What this does:**

```ruby
user = User.find(1)
user.memberships  # All membership records for this user
user.rooms        # All rooms the user belongs to (skipping through memberships)
```

The `through:` is the magic. The database has:
- `users` table
- `memberships` table (with `user_id` and `room_id`)
- `rooms` table

ActiveRecord joins them automatically. You never write the SQL.

### Example 2: Message belongs to room and creator

**Location:** `app/models/message.rb:4-5`

```ruby
class Message < ApplicationRecord
  belongs_to :room, touch: true
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end
```

**What this demonstrates:**

- `belongs_to :room` - Every message has one room. The `messages` table has a `room_id` column.
- `belongs_to :creator, class_name: "User"` - The column is `creator_id`, but the model is `User` (not `Creator`)
- `touch: true` - When a message is saved, update the room's `updated_at` timestamp
- `default: -> { Current.user }` - Automatically set creator to the current user

```ruby
message = Message.find(1)
message.room      # The Room object
message.creator   # The User who created it
```

### Example 3: Room has many messages

**Location:** `app/models/room.rb:21`

```ruby
class Room < ApplicationRecord
  has_many :messages, dependent: :destroy
end
```

The `dependent: :destroy` means: when a room is deleted, delete all its messages too.

```ruby
room = Room.find(1)
room.messages          # All messages in this room
room.messages.count    # How many messages
room.messages.create(body: "Hello!")  # Create a message IN this room
```

### The Association Map

Here's how Campfire's core models relate:

```
┌──────────┐         ┌─────────────┐         ┌──────────┐
│   User   │────────<│ Membership  │>────────│   Room   │
└──────────┘         └─────────────┘         └──────────┘
     │                                             │
     │ has_many                          has_many │
     ▼                                             ▼
┌──────────┐                               ┌──────────┐
│ Message  │>──────────────────────────────│ Message  │
│(creator) │                               │  (room)  │
└──────────┘                               └──────────┘
```

## Scopes: Reusable Query Fragments

Instead of writing the same `where` clause everywhere, define it once as a scope.

**Location:** `app/models/user.rb:17,23-24`

```ruby
class User < ApplicationRecord
  scope :active, -> { where(active: true) }
  scope :ordered, -> { order("LOWER(name)") }
  scope :filtered_by, ->(query) { where("name like ?", "%#{query}%") }
end
```

Now you can chain them:

```ruby
User.active                        # All active users
User.active.ordered                # Active users, sorted by name
User.active.filtered_by("john")    # Active users with "john" in their name
```

**Why scopes matter:**

1. **Readability**: `User.active` is clearer than `User.where(active: true)`
2. **DRY**: Define once, use everywhere
3. **Chainable**: Combine scopes like building blocks
4. **Testable**: Test the scope in isolation

## Migrations: Evolving Your Database

Your database schema will change over time. Migrations track those changes.

**Location:** `db/migrate/20231220143106_add_bio_to_users.rb`

```ruby
class AddBioToUsers < ActiveRecord::Migration[7.2]
  def change
    change_table :users do |t|
      t.text :bio
    end
  end
end
```

**What this does:** Adds a `bio` column to the `users` table.

Run it with:
```bash
bin/rails db:migrate
```

**Why migrations are powerful:**

1. **Version control for your database** - Every change is tracked
2. **Reversible** - `bin/rails db:rollback` undoes the last migration
3. **Team-friendly** - Everyone runs the same migrations, gets the same schema
4. **No manual SQL** - Rails generates it for you

Common migration methods:
```ruby
add_column :users, :bio, :text
remove_column :users, :bio
rename_column :users, :bio, :biography
add_index :users, :email_address, unique: true
create_table :posts do |t|
  t.string :title
  t.text :body
  t.references :user, foreign_key: true
  t.timestamps
end
```

## Callbacks: Hooks Into the Lifecycle

Callbacks let you run code at specific moments in an object's life.

**Location:** `app/models/user.rb:21`

```ruby
class User < ApplicationRecord
  after_create_commit :grant_membership_to_open_rooms

  private
    def grant_membership_to_open_rooms
      Membership.insert_all(
        Rooms::Open.pluck(:id).collect { |room_id| { room_id: room_id, user_id: id } }
      )
    end
end
```

**What this does:** After a user is created and saved to the database, automatically add them to all open rooms.

**Location:** `app/models/message.rb:12`

```ruby
class Message < ApplicationRecord
  after_create_commit -> { room.receive(self) }
end
```

**What this does:** After a message is created, tell the room to "receive" it (which marks memberships as unread and queues push notifications).

**Common callbacks:**

| Callback | When it runs |
|----------|--------------|
| `before_validation` | Before validation runs |
| `after_validation` | After validation passes |
| `before_save` | Before save (create or update) |
| `after_save` | After save |
| `before_create` | Before first save (new record) |
| `after_create` | After first save |
| `after_create_commit` | After first save AND transaction commits |
| `before_destroy` | Before deletion |
| `after_destroy` | After deletion |

**Warning:** Callbacks can make code hard to follow. Use them for:
- Cross-cutting concerns (like our "add to open rooms" example)
- Things that MUST happen every time

Don't use them for business logic that might vary based on context.

## Try It Yourself

Open a Rails console (`bin/rails console`) and try these:

```ruby
# 1. Find a user and explore their associations
user = User.first
user.rooms
user.memberships
user.messages

# 2. Chain some queries
Room.where(type: "Rooms::Open").count
User.active.ordered.limit(5)

# 3. See the SQL ActiveRecord generates
User.where(active: true).to_sql
# => "SELECT \"users\".* FROM \"users\" WHERE \"users\".\"active\" = 1"

# 4. Explore a complex query
user.rooms.joins(:messages).where(messages: { created_at: 1.day.ago.. })
```

## Summary

- **ORM** (Object-Relational Mapper) translates between Ruby objects and database tables - Rails includes one (ActiveRecord), Next.js doesn't (bring your own)
- **Naming conventions** connect everything: `User` class → `users` table, no configuration needed
- **Associations** (`has_many`, `belongs_to`, `through`) let you navigate between related models with simple method calls
- **Scopes** are reusable query fragments that make your code readable and DRY
- **Migrations** version-control your database schema - every change is tracked and reversible
- **Callbacks** hook into lifecycle events, but use them sparingly

**The key insight:** ActiveRecord lets you think in objects, not tables. You say `user.rooms` and it figures out the JOIN. You say `message.create` and it generates the INSERT. The database becomes an implementation detail.

**Next up:** We'll explore Associations in depth - including polymorphic associations, counter caches, and the n+1 query problem that trips up every Rails developer.

---

## Q&A

### Q: Why doesn't Next.js ship with an ORM like Rails does?

Different philosophies:

**Rails philosophy:** "Convention over configuration." Make decisions for developers so they can move fast. Everyone does databases the same way, code is consistent across projects, onboarding is fast.

**Next.js philosophy:** "Flexibility over prescription." Don't force choices. Let developers pick what works for their needs - maybe they use Prisma, maybe Drizzle, maybe a headless CMS, maybe no database at all.

Neither is "right." Rails is opinionated and productive. Next.js is flexible and lets you choose. The tradeoff is: Rails developers spend less time choosing tools but have less flexibility; Next.js developers have complete freedom but must make more decisions.

When people say "Rails' ORM makes it powerful," they mean: ActiveRecord is so well-integrated that you rarely think about the database. You think in Ruby. The framework handles the translation. In Next.js, you're always aware you're using a separate tool (Prisma, etc.) that doesn't "know" about the rest of your app.

### Q: Why is the class singular (User) but the table plural (users)? Why not both the same?

The answer is surprisingly elegant - it's about making code read like English.

**Classes describe ONE thing**

When you define a class, you're describing what a single instance IS:

```ruby
class User < ApplicationRecord
  # I am describing what ONE user is and can do
  has_many :messages    # "A user has many messages"

  def greet
    "Hello, I am #{name}"  # ONE user greeting
  end
end
```

You'd never say "A Users has many messages" - that's grammatically wrong.

**Tables hold MANY things**

The `users` table is a container for ALL users. It's plural because it holds a collection:

```sql
SELECT * FROM users;  -- Give me all the users
```

You wouldn't say "select from user" - that sounds like you're selecting from one person.

**It Makes Code Read Like English**

This is DHH's magic. Watch how natural this reads:

```ruby
User.find(1)           # "Find user 1"
User.all               # "All users" (from the users table)
User.where(active: true) # "Users where active is true"

user = User.first      # "The first user"
user.messages          # "This user's messages"
user.rooms             # "This user's rooms"
```

Compare to if both were singular:

```ruby
User.find(1)           # Still okay
User.all               # "All user"? Weird.
user.message           # "This user's message"? Just one?
```

Or if both were plural:

```ruby
Users.find(1)          # "Users find 1"? Finding multiple?
users = Users.first    # "users equals Users first"? Confusing.
```

**The Pattern Extends Everywhere**

| Concept | Singular | Plural | Why |
|---------|----------|--------|-----|
| Model class | `User` | - | Describes one thing |
| Table | - | `users` | Holds many things |
| Controller | - | `UsersController` | Handles requests about users (the collection) |
| Routes | - | `/users`, `/users/:id` | RESTful paths to the collection |
| Instance variable (one) | `@user` | - | Refers to one |
| Instance variable (many) | - | `@users` | Refers to many |
| Association (one) | `belongs_to :user` | - | This thing has ONE user |
| Association (many) | - | `has_many :users` | This thing has MANY users |

**Real Example from Campfire**

```ruby
# app/models/message.rb
class Message < ApplicationRecord        # ONE message
  belongs_to :room                       # has ONE room
  belongs_to :creator, class_name: "User" # has ONE creator
end

# app/models/room.rb
class Room < ApplicationRecord           # ONE room
  has_many :messages                     # has MANY messages
  has_many :users, through: :memberships # has MANY users
end
```

Reading this aloud: "A message belongs to a room. A room has many messages."

If it were `has_many :message`, you'd read "A room has many message" - grammatically broken.

**The Philosophy**

DHH (Rails creator) optimized for **developer happiness** and **code readability**. He wanted Rails code to read like a conversation about your domain, not like machine instructions.

This is why Rails has an `Inflector` that knows:
- `"user".pluralize` → `"users"`
- `"users".singularize` → `"user"`
- `"person".pluralize` → `"people"` (it knows irregular plurals!)
- `"octopus".pluralize` → `"octopi"`
