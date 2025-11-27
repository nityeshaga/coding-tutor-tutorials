---
concepts: [Routing, RESTful Resources, Nested Resources, URL Helpers, Namespaces, Scopes]
description: Deep dive into Rails routing - the traffic cop that turns URLs into controller actions. Learn how Campfire maps URLs like /rooms/5/messages to the right code, when to use resources vs resource, how nested routes work, and the subtle but crucial difference between namespace and scope. All with real examples from this codebase.
understanding_score: null
prerequisites: [references/tutorials/2025-11-25-mvc---the-architecture-that-powers-every-rails-app.md]
created: 25-11-2025
last_updated: 25-11-2025  # Q&A added
---

# Routing - How URLs Become Code

Every time someone types a URL or clicks a link in Campfire, something magical happens: Rails looks at that URL and instantly knows which Ruby method to run. `/rooms/5`? That's `RoomsController#show` with `id: 5`. `/account/users`? That's `Accounts::UsersController#index`.

This isn't magic - it's **routing**. And understanding it deeply is what separates developers who copy-paste route configurations from those who design clean, intuitive URL structures.

## The Big Picture

Think of `config/routes.rb` as a giant lookup table:

```
┌─────────────────────────────────┬────────────────────────────────┐
│           URL Pattern           │         What Runs              │
├─────────────────────────────────┼────────────────────────────────┤
│ GET    /                        │ WelcomeController#show         │
│ GET    /rooms                   │ RoomsController#index          │
│ GET    /rooms/5                 │ RoomsController#show (id: 5)   │
│ POST   /rooms                   │ RoomsController#create         │
│ DELETE /rooms/5                 │ RoomsController#destroy (id: 5)│
│ GET    /rooms/5/messages        │ MessagesController#index       │
│        ...                      │ ...                            │
└─────────────────────────────────┴────────────────────────────────┘
```

When a request comes in, Rails scans this table top-to-bottom until it finds a match. First match wins.

### See the Full Map

You can see Campfire's entire routing table by running:

```bash
bin/rails routes
```

That's a lot of output. To filter it:

```bash
bin/rails routes -g room      # Only routes containing "room"
bin/rails routes -c messages  # Only routes for MessagesController
```

---

## RESTful Resources - The 80% Case

Here's the most important line you'll write in any routes file:

```ruby
resources :rooms
```

This single line generates **7 routes**:

| HTTP Verb | Path | Controller#Action | Used For |
|-----------|------|-------------------|----------|
| GET | /rooms | rooms#index | List all rooms |
| GET | /rooms/new | rooms#new | Form to create room |
| POST | /rooms | rooms#create | Create the room |
| GET | /rooms/:id | rooms#show | Show one room |
| GET | /rooms/:id/edit | rooms#edit | Form to edit room |
| PATCH/PUT | /rooms/:id | rooms#update | Update the room |
| DELETE | /rooms/:id | rooms#destroy | Delete the room |

**Why REST matters**: These 7 actions cover almost everything you'd want to do with any "thing" in your app. Users, messages, rooms, products, invoices - they all follow the same pattern. Once you internalize REST, you can navigate any Rails codebase because you know `POST /things` creates and `DELETE /things/5` destroys.

### Limiting Routes with `only:` and `except:`

Campfire doesn't need all 7 routes for everything. Look at this from `config/routes.rb:87`:

```ruby
resources :searches, only: %i[ index create ] do
  delete :clear, on: :collection
end
```

This says: "I only want `index` and `create` for searches - no show, edit, update, etc."

Why? Because a search isn't something you "show" by ID or "edit" later. You either list recent searches (`index`) or create a new one (`create`). The `only:` constraint keeps your routes clean and your attack surface small.

You'll also see `except:`:

```ruby
resources :users, except: [:destroy]  # Everything BUT delete
```

---

## Singular vs Plural: `resource` vs `resources`

Here's something that trips up every Rails beginner. Look at these two routes in Campfire:

```ruby
resource :session          # singular
resources :users           # plural
```

**`resources :users`** (plural) = "There are many users, each with an ID"
- `/users` → list all
- `/users/5` → show user #5
- `/users/5/edit` → edit user #5

**`resource :session`** (singular) = "There's only ONE of these per user"
- `/session` → show YOUR session
- `/session/new` → login form
- No ID in the URL - there's only one session (yours)

Campfire uses singular resources for things that are unique to the current context:

```ruby
resource :session      # Your login session
resource :account      # The account (single-tenant app)
resource :first_run    # First-time setup (happens once)
resource :profile      # Your profile
resource :sidebar      # Your sidebar state
```

### The Mental Model

Ask yourself: **"Would I ever say 'show me session #5' or 'show me account #12'?"**

- If yes → `resources` (plural, with IDs)
- If no → `resource` (singular, no IDs)

---

## Nested Resources - The Parent-Child Relationship

Messages belong to rooms. The URL should reflect that:

```ruby
resources :rooms do
  resources :messages
end
```

This generates routes like:

| Path | Controller#Action | Params |
|------|-------------------|--------|
| /rooms/5/messages | messages#index | room_id: 5 |
| /rooms/5/messages/42 | messages#show | room_id: 5, id: 42 |
| /rooms/5/messages/new | messages#new | room_id: 5 |

The `room_id` automatically flows into your controller via `params[:room_id]`.

### How the Controller Uses It

Look at how `RoomsController` finds the room (`app/controllers/rooms_controller.rb:22-28`):

```ruby
def set_room
  if room = Current.user.rooms.find_by(id: params[:room_id] || params[:id])
    @room = room
  else
    redirect_to root_url, alert: "Room not found or inaccessible"
  end
end
```

Notice `params[:room_id] || params[:id]` - it handles both nested routes (where room comes from `:room_id`) and direct routes (where it's just `:id`).

### The Golden Rule: Never Nest More Than One Level Deep

You might be tempted to do this:

```ruby
# DON'T DO THIS
resources :accounts do
  resources :rooms do
    resources :messages do
      resources :boosts
    end
  end
end
```

This creates URLs like `/accounts/1/rooms/5/messages/42/boosts/3`. Nightmare.

**The fix**: Only nest one level, then use shallow routes or separate resources:

```ruby
# Campfire's approach
resources :rooms do
  resources :messages  # /rooms/5/messages
end

resources :messages do
  scope module: "messages" do
    resources :boosts  # /messages/42/boosts (not nested under rooms)
  end
end
```

---

## Organizing Controllers: Scope vs Namespace

This is where many Rails developers get confused. Campfire uses both - let's understand why.

### `namespace` - Changes BOTH the URL and controller location

```ruby
namespace :autocompletable do
  resources :users, only: :index
end
```

This creates:
- **URL**: `/autocompletable/users`
- **Controller**: `Autocompletable::UsersController`
- **File**: `app/controllers/autocompletable/users_controller.rb`

Look at the actual controller (`app/controllers/autocompletable/users_controller.rb`):

```ruby
class Autocompletable::UsersController < ApplicationController
  def index
    # Returns users for autocomplete dropdown
  end
end
```

**Use namespace when**: You want a distinct section of your app with its own URL prefix. API endpoints, admin panels, etc.

### `scope module:` - Changes controller location BUT NOT the URL

```ruby
resource :account do
  scope module: "accounts" do
    resources :users
  end
end
```

This creates:
- **URL**: `/account/users` (no "accounts" in URL)
- **Controller**: `Accounts::UsersController`
- **File**: `app/controllers/accounts/users_controller.rb`

**Use scope module when**: You want to organize controllers into folders without cluttering URLs.

### Visual Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                        namespace :admin                         │
│  URL:        /admin/users                                       │
│  Controller: Admin::UsersController                             │
│  File:       app/controllers/admin/users_controller.rb          │
├─────────────────────────────────────────────────────────────────┤
│                   scope module: "admin"                         │
│  URL:        /users          (no /admin prefix!)                │
│  Controller: Admin::UsersController                             │
│  File:       app/controllers/admin/users_controller.rb          │
└─────────────────────────────────────────────────────────────────┘
```

### Campfire's Pattern

Campfire nests related controllers under the parent resource using `scope module:`:

```ruby
resource :account do
  scope module: "accounts" do
    resources :users        # /account/users → Accounts::UsersController
    resources :bots         # /account/bots → Accounts::BotsController
    resource :logo          # /account/logo → Accounts::LogosController
  end
end
```

The URL stays clean (`/account/users`) while controllers are organized in folders (`accounts/users_controller.rb`).

---

## Custom Routes - When REST Isn't Enough

Sometimes RESTful routes don't fit. Campfire has several creative solutions:

### Custom Path with Parameters

```ruby
get "join/:join_code", to: "users#new", as: :join
post "join/:join_code", to: "users#create"
```

This creates `/join/abc123` for joining with an invite code. The `:join_code` becomes `params[:join_code]`.

The `as: :join` gives you a URL helper: `join_path("abc123")` → `/join/abc123`

### The Clever `@` Route

```ruby
resources :rooms do
  get "@:message_id", to: "rooms#show", as: :at_message
end
```

This creates `/rooms/5/@42` - a URL that jumps directly to message #42 in room #5. Look how the controller handles it (`app/controllers/rooms_controller.rb:34-41`):

```ruby
def find_messages
  messages = @room.messages.with_creator

  if show_first_message = messages.find_by(id: params[:message_id])
    @messages = messages.page_around(show_first_message)
  else
    @messages = messages.last_page
  end
end
```

If `params[:message_id]` exists (from the `@:message_id` route), it pages around that message. Clean!

### Direct Routes - Custom URL Helpers with Logic

```ruby
direct :fresh_account_logo do |options|
  route_for :account_logo, v: Current.account&.updated_at&.to_fs(:number), size: options[:size]
end
```

This creates `fresh_account_logo_url(size: "large")` which generates a URL with a cache-busting version parameter. Every time the account updates, the URL changes, forcing browsers to fetch the new logo.

### Collection and Member Routes

```ruby
resources :searches, only: %i[ index create ] do
  delete :clear, on: :collection
end
```

- `on: :collection` → acts on ALL searches: `DELETE /searches/clear`
- `on: :member` → acts on ONE search: `DELETE /searches/5/clear`

---

## URL Helpers - The Payoff

Every route automatically gives you helper methods. Never build URLs by hand:

```ruby
# ❌ Don't do this
"/rooms/#{room.id}/messages/#{message.id}"

# ✅ Do this
room_message_path(room, message)
```

**Why helpers matter:**

1. **They change when routes change** - Rename a route, helpers update everywhere
2. **They handle encoding** - Special characters in params? Handled.
3. **`_path` vs `_url`** - `room_path` gives `/rooms/5`, `room_url` gives `http://localhost:3000/rooms/5`

### Common Helpers from Campfire's Routes

```ruby
root_path                        # /
room_path(@room)                 # /rooms/5
room_messages_path(@room)        # /rooms/5/messages
new_session_path                 # /session/new
edit_account_path                # /account/edit
join_path("abc123")              # /join/abc123
room_at_message_path(@room, 42)  # /rooms/5/@42
```

---

## Try It Yourself

**Challenge**: Add a route for archiving rooms.

1. A room archive should be accessed at `/rooms/5/archive`
2. Visiting it (GET) shows an archive confirmation page
3. Submitting it (POST) actually archives the room
4. The controller should be `Rooms::ArchivesController`

Try adding this to `config/routes.rb`. Think about:
- Should this be nested inside `resources :rooms`?
- Should you use `resource` (singular) or `resources` (plural)?
- How do you get it to use `Rooms::ArchivesController`?

<details>
<summary>Click to see the solution</summary>

```ruby
resources :rooms do
  resource :archive, only: [:show, :create], module: "rooms"
end
```

This creates:
- `GET /rooms/5/archive` → `Rooms::ArchivesController#show`
- `POST /rooms/5/archive` → `Rooms::ArchivesController#create`

Using `resource` (singular) because each room has one archive action. Using `module: "rooms"` to put the controller in the `rooms/` folder.

</details>

---

## Summary

- **Routes map URLs to controller actions** - `config/routes.rb` is your app's traffic cop
- **`resources`** generates 7 RESTful routes; use `only:` to limit them
- **`resource` (singular)** for things without IDs - sessions, profiles, settings
- **Nest resources one level max** - `/rooms/5/messages` is fine, deeper is painful
- **`namespace`** changes URL AND controller path; **`scope module:`** changes only controller path
- **Custom routes** fill gaps REST can't - `get "join/:code"`, `direct :helper_name`
- **Always use URL helpers** - `room_path(@room)` not `"/rooms/#{room.id}"`

Next up: **Hotwire** - where you'll see how these routes connect to Turbo Frames and Streams for real-time updates without writing JavaScript.

---

## Q&A

### Q: Why does Campfire use `/messages/42/boosts` instead of nesting boosts under `/rooms/5/messages/42/boosts`?

**The room is redundant information.** Message #42 already knows which room it belongs to.

Look at how the controller finds the message - it only needs `message_id`:

```ruby
# app/controllers/messages/boosts_controller.rb:25-27
def set_message
  @message = Current.user.reachable_messages.find(params[:message_id])
end
```

If you nested under rooms, you'd have:
- `/rooms/5/messages/42/boosts/3` - but message 42 already belongs to room 5
- You'd have to validate *both* params match
- Someone could craft `/rooms/99/messages/42/boosts/3` - a room/message mismatch

**The rule**: Ask yourself "Does this child resource need the grandparent to function?"

```
rooms → messages → boosts
  ↑         ↑         ↑
grandparent parent   child
```

- **To find a boost**: You need the message. You don't need the room.
- **To render boosts**: `message.boosts.ordered` - room not involved.
- **To broadcast**: Controller gets room via `@boost.message.room` - derived, not passed.

**Compare the approaches:**

| Deeply Nested | Shallow |
|--------------|---------|
| `/rooms/5/messages/42/boosts/3` | `/messages/42/boosts/3` |
| `room_message_boost_path(@room, @message, @boost)` | `message_boost_path(@message, @boost)` |
| Validate room AND message match | Just validate message |
| 3 params to juggle | 2 params |

**The pattern**: Nest one level for the relationship you actually need. Boosts need messages. Messages need rooms. But boosts don't directly need rooms - they get there through the message. This keeps URLs short, controllers simple, and eliminates impossible states.
