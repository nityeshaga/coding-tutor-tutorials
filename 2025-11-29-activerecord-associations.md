---
concepts: belongs_to,has_many,has_one,through,foreign_key,dependent,class_name
description: A deep dive into how Rails associations work under the hood - from the SQL they generate to the mental models that help you pick the right one. Uses real examples from the Curated Connections codebase to understand belongs_to, has_many, has_one, through associations, and the options that make them flexible.
understanding_score: null
last_quizzed: null
prerequisites: []
created: 29-11-2025
last_updated: 29-11-2025
---

# ActiveRecord Associations: The Foundation of Everything

Here's a question: why does Rails feel magical compared to other frameworks?

A huge part of the answer is associations. When you write `user.community_members`, Rails doesn't just return data - it builds a SQL query, executes it, instantiates Ruby objects, and hands them back to you. You didn't write a single line of SQL.

But here's the thing about magic: if you don't understand how the trick works, you can't improvise. You end up copy-pasting patterns without knowing why. You write code that works but you couldn't explain *how*.

This tutorial will lift the curtain. By the end, you'll look at any association and immediately understand:
- What SQL it generates
- Which table holds the foreign key
- Why certain options exist
- When to reach for `through:` vs direct associations

This is foundational. Every Rails model you write for the rest of your career will use associations.

## The Problem Associations Solve

Open your `User` model (`app/models/user.rb`). Look at lines 46-71. You've got 15+ associations defined there. Now imagine Rails didn't have associations. How would you get all the community memberships for a user?

```ruby
# Without associations (the painful way)
community_members = CommunityMember.where(user_id: user.id)
```

Not terrible for one query. But what about getting a user's matches? Your `User` model has:

```ruby
has_many :matches_as_user1, class_name: "Match", foreign_key: "user1_id"
has_many :matches_as_user2, class_name: "Match", foreign_key: "user2_id"
```

Without associations, you'd need:

```ruby
# Getting all matches for a user... painful
matches = Match.where(user1_id: user.id).or(Match.where(user2_id: user.id))
```

And you'd write this *every single time* you needed matches. Associations let you define this relationship once and use it everywhere with `user.matches_as_user1`.

But associations do more than save typing. They create a *vocabulary* for your domain. When you read `has_many :community_members`, you immediately understand the relationship. The code documents itself.

## The Golden Rule: Who Holds the Foreign Key?

Before we dive into association types, you need to understand one concept that explains almost everything: **the foreign key**.

A foreign key is a column in one table that points to a row in another table. That's it. It's just an integer column (usually) that says "I belong to that record over there."

**The golden rule:** The table with the foreign key column is the "child" in the relationship.

Look at your schema. `community_members` table has a `user_id` column. That's a foreign key pointing to `users`. So `community_members` is the child, and `users` is the parent.

```
┌─────────────────────┐         ┌─────────────────────────────┐
│       users         │         │     community_members       │
├─────────────────────┤         ├─────────────────────────────┤
│ id                  │◄────────│ user_id (foreign key)       │
│ email               │         │ community_id (foreign key)  │
│ first_name          │         │ short_bio                   │
│ ...                 │         │ ...                         │
└─────────────────────┘         └─────────────────────────────┘
```

This single concept explains:
- Why `CommunityMember` uses `belongs_to :user` (it has the foreign key)
- Why `User` uses `has_many :community_members` (it doesn't have the foreign key, but others point to it)

## The Core Associations

### `belongs_to` - "I have a foreign key pointing to someone else"

When a model has a foreign key column, it uses `belongs_to`.

**From `app/models/community_member.rb:35-36`:**
```ruby
belongs_to :user
belongs_to :community
```

Look at the schema comment at the top of that file - you'll see `user_id` and `community_id` columns. Those are foreign keys. This model *belongs to* a user and a community.

**What Rails gives you:**

```ruby
member = CommunityMember.first

member.user          # Returns the User object (runs: SELECT * FROM users WHERE id = ?)
member.user_id       # Returns just the ID (no database query!)
member.user = user   # Sets the association
member.build_user    # Builds a new User in memory
member.create_user   # Creates and saves a new User
```

**The SQL:**
```sql
-- When you call member.user:
SELECT "users".* FROM "users" WHERE "users"."id" = 123 LIMIT 1
```

**Key insight:** `belongs_to` associations are required by default in Rails 5+. If you try to save a `CommunityMember` without a `user`, it will fail validation. This is usually what you want - a community member that doesn't belong to anyone is meaningless.

### `has_many` - "Other records have foreign keys pointing to me"

The inverse of `belongs_to`. When other records point *to* you, you use `has_many`.

**From `app/models/user.rb:57-58`:**
```ruby
has_many :community_members, dependent: :destroy
has_many :signups, dependent: :destroy
```

The `users` table doesn't have a `community_member_id` column. Instead, `community_members` has a `user_id` column pointing back. Rails figures this out by convention.

**What Rails gives you:**

```ruby
user = User.first

user.community_members           # Returns array of CommunityMember objects
user.community_members.count     # Counts without loading all records
user.community_members.where(is_admin: true)  # Chain queries!
user.community_members.build     # Build a new one in memory
user.community_members.create    # Create and save
user.community_members << member # Add to the collection
```

**The SQL:**
```sql
-- When you call user.community_members:
SELECT "community_members".* FROM "community_members"
WHERE "community_members"."user_id" = 123
```

### `has_one` - "One other record has a foreign key pointing to me"

Like `has_many`, but enforces that there's only one.

**From `app/models/user.rb:48-50`:**
```ruby
has_one :notification_setting, dependent: :destroy
has_one :community, foreign_key: "creator_id", dependent: :destroy
```

The first one follows convention - Rails looks for `user_id` in `notification_settings`.

The second one is interesting. By default, Rails would look for `user_id` in `communities`. But your schema uses `creator_id` instead! So you have to tell Rails: "hey, the foreign key isn't the default, it's `creator_id`."

**What Rails gives you:**

```ruby
user.notification_setting        # Returns one object (or nil)
user.build_notification_setting  # Build in memory
user.create_notification_setting # Create and save
```

### `has_many :through` - "I'm connected through an intermediate table"

This is where it gets powerful. Sometimes the relationship between two models goes *through* a third model.

**From `app/models/user.rb:54-56`:**
```ruby
has_many :program_users, dependent: :destroy
has_many :programs, through: :program_users
```

A User is connected to Programs, but not directly. There's a `program_users` table in between (a "join table") that holds the relationship.

```
┌─────────┐       ┌───────────────┐       ┌──────────┐
│  users  │──────▶│ program_users │◀──────│ programs │
└─────────┘       └───────────────┘       └──────────┘
     │                    │                     │
     │                    ├── user_id           │
     │                    └── program_id        │
     │                                          │
     └──────────── through ────────────────────┘
```

Why not just put `user_id` directly on `programs`? Because it's a many-to-many relationship. A user can join many programs. A program has many users. The join table handles this.

**What Rails gives you:**

```ruby
user.programs          # All programs the user is in
user.program_users     # The join records themselves

# The join table can have extra data!
# Maybe program_users has a `role` column
user.program_users.find_by(program: some_program).role
```

**The SQL:**
```sql
-- When you call user.programs:
SELECT "programs".* FROM "programs"
INNER JOIN "program_users" ON "programs"."id" = "program_users"."program_id"
WHERE "program_users"."user_id" = 123
```

### A Richer Example: CommunityMember and Events

Look at `app/models/community_member.rb:47-51`:

```ruby
has_many :event_hosts, dependent: :destroy
has_many :hosted_events, through: :event_hosts, source: :event

has_many :event_registrations, dependent: :destroy
has_many :registered_events, through: :event_registrations, source: :event
```

This is beautiful. A community member can be related to events in two different ways:
1. As a **host** (through `event_hosts`)
2. As an **attendee** (through `event_registrations`)

The `source: :event` option tells Rails: "the association on the through model is called `event`, not `hosted_event` or `registered_event`."

Without this distinction, you'd have to figure out "is this person hosting or attending?" every time. With these associations, the code is crystal clear:

```ruby
member.hosted_events      # Events they're hosting
member.registered_events  # Events they're attending
```

## Key Options You'll Use

### `dependent:` - What happens when the parent is destroyed?

**From `app/models/user.rb:57`:**
```ruby
has_many :community_members, dependent: :destroy
```

When you delete a user, what happens to their community memberships? Options:
- `:destroy` - Delete associated records AND run their callbacks
- `:delete_all` - Delete associated records directly (no callbacks, faster)
- `:nullify` - Set the foreign key to NULL (record stays, just unlinked)
- `:restrict_with_error` - Prevent deletion if associations exist

You use `:destroy` when the associated records need to do cleanup (like your `CommunityMember` which might need to remove related data).

Look at `app/models/user.rb:67-68`:
```ruby
has_many :access_grants, class_name: "Doorkeeper::AccessGrant",
         foreign_key: :resource_owner_id, dependent: :delete_all
```

Here you use `:delete_all` because access grants don't need callbacks - just nuke them fast.

### `class_name:` - When the class name doesn't match

**From `app/models/user.rb:61-62`:**
```ruby
has_many :matches_as_user1, class_name: "Match", foreign_key: "user1_id"
has_many :matches_as_user2, class_name: "Match", foreign_key: "user2_id"
```

A Match has two users (user1 and user2). You can't just write `has_many :matches` because Rails wouldn't know which foreign key to use. So you create two associations with descriptive names and tell Rails which class and foreign key to use.

### `foreign_key:` - When the column name doesn't follow convention

Convention says the foreign key is `{model_name}_id`. When it's not, specify it:

**From `app/models/user.rb:50`:**
```ruby
has_one :community, foreign_key: "creator_id"
```

The `communities` table has `creator_id`, not `user_id`. Without this option, Rails would look for `user_id` and find nothing.

**From `app/models/community_member.rb:55-56`:**
```ruby
has_many :sent_intro_requests, class_name: "IntroRequest", foreign_key: "sender_id"
has_many :received_intro_requests, class_name: "IntroRequest", foreign_key: "receiver_id"
```

Same pattern as matches - one model, two relationships, distinguished by which foreign key to use.

## The Mental Model

Think of associations as **questions you can ask a record**:

| Association | Question it answers |
|-------------|---------------------|
| `belongs_to :user` | "Who is my user?" |
| `has_many :community_members` | "Who are all my community members?" |
| `has_one :notification_setting` | "What is my notification setting?" |
| `has_many :programs, through: :program_users` | "What programs am I in?" |

When you define an association, you're teaching Rails how to answer that question by writing SQL.

## Try It Yourself

Here's a challenge to cement this:

1. Open `rails console`
2. Find a user: `user = User.first`
3. For each association on User, try calling it and notice the SQL in your console
4. Pay attention to which associations run a simple `WHERE` vs which ones run a `JOIN`

Then try this: Add `.to_sql` to any association to see the SQL without executing it:

```ruby
user.community_members.to_sql
# => "SELECT \"community_members\".* FROM \"community_members\" WHERE \"community_members\".\"user_id\" = 1"

user.programs.to_sql
# => "SELECT \"programs\".* FROM \"programs\" INNER JOIN \"program_users\" ON..."
```

This is how you build intuition. When you see the SQL, the magic becomes understandable.

## Summary

- **Foreign key location determines association type**: The table with the foreign key uses `belongs_to`. The other side uses `has_one` or `has_many`.

- **`through:` creates indirect relationships**: When two models connect via a join table, use `has_many :things, through: :join_table`.

- **Options exist because reality is messy**: `foreign_key:`, `class_name:`, `source:` all exist because naming conventions don't always fit your domain.

- **`dependent:` protects data integrity**: Always think about what should happen to associated records when you delete something.

- **Associations are just SQL generators**: They look magical, but they're just building queries. Understanding the SQL helps you debug and optimize.

You now understand the foundation that every Rails model is built on. Next, we'll explore callbacks - understanding when code runs during an object's lifecycle.

---

## Q&A

[Questions and answers will be added here as you ask them during the tutorial]

## Quiz History

[Quiz sessions will be recorded here after you're quizzed on this topic]
