---
concepts: [MVC, Request-Response Cycle, Convention Over Configuration]
source_repo: once-campfire
description: Your first Rails tutorial exploring MVC architecture by tracing a real request through Campfire - from URL to rendered page. Covers how routes, controllers, models, and views work together, and why Rails organizes code the way it does.
understanding_score: 7
prerequisites: []
created: 25-11-2025
last_updated: 25-11-2025
last_quizzed: 25-11-2025
---

# MVC - The Architecture That Powers Every Rails App

Imagine you're dropped into a Rails codebase with 200 files. Overwhelming, right? But here's the secret senior Rails engineers know: **every single feature follows the same pattern**. Learn the pattern once, and you can navigate any Rails app on Earth.

That pattern is MVC: Model-View-Controller.

By the end of this tutorial, you'll be able to pick any URL in Campfire, trace exactly which files handle it, and know exactly where to add new code. That's the superpower we're building today.

## The Problem

You want to visit a chat room in Campfire. You type `http://localhost:3000/rooms/1` in your browser.

Now what?

Somewhere in this codebase, code needs to:
1. Figure out you want to see room #1
2. Check if you're allowed to see it
3. Fetch the room from the database
4. Fetch all the messages in that room
5. Build an HTML page showing everything
6. Send it back to your browser

That's a lot of responsibilities. If you threw all that code in one file, it would become an unmaintainable mess. MVC is how Rails keeps things organized.

## Key Concepts

### The Three Amigos

Think of MVC like a restaurant:

```
┌─────────────────────────────────────────────────────────────────┐
│                         RESTAURANT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CUSTOMER ──────► WAITER ──────► KITCHEN ──────► INGREDIENTS   │
│   (Browser)      (Controller)     (Model)         (Database)    │
│                       │                                          │
│                       │                                          │
│                       ▼                                          │
│                   PLATING                                        │
│                   (View)                                         │
│                       │                                          │
│                       ▼                                          │
│                  CUSTOMER                                        │
│                  (Browser)                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

- **Model** = The Kitchen. Knows the recipes (business logic), works with ingredients (database). Doesn't care who ordered or how it's served.
- **View** = Plating. Makes things pretty for the customer. Doesn't cook, just presents.
- **Controller** = The Waiter. Takes orders, talks to kitchen, delivers plates. Coordinates everything but doesn't cook or plate.

### The Request Lifecycle

Here's exactly what happens when you visit `/rooms/1`:

```
┌──────────┐     ┌──────────┐     ┌────────────┐     ┌───────┐     ┌──────┐
│ Browser  │────►│  Router  │────►│ Controller │────►│ Model │────►│  DB  │
│          │     │          │     │            │◄────│       │◄────│      │
└──────────┘     └──────────┘     └────────────┘     └───────┘     └──────┘
                                        │
                                        ▼
                                   ┌──────────┐
                                   │   View   │
                                   └──────────┘
                                        │
                                        ▼
                                   ┌──────────┐
                                   │ Browser  │
                                   └──────────┘
```

Let's trace it through Campfire's actual code.

## Examples from Codebase

### Step 1: The Router - Traffic Cop

**Location:** `config/routes.rb:61`

```ruby
resources :rooms do
  resources :messages
  # ...
end
```

**What this does:** `resources :rooms` is Rails magic that creates 7 routes automatically:

| HTTP Verb | Path | Controller#Action | Purpose |
|-----------|------|-------------------|---------|
| GET | /rooms | rooms#index | List all rooms |
| GET | /rooms/:id | rooms#show | **Show one room** ← We're here! |
| GET | /rooms/new | rooms#new | Form to create room |
| POST | /rooms | rooms#create | Save new room |
| GET | /rooms/:id/edit | rooms#edit | Form to edit room |
| PATCH | /rooms/:id | rooms#update | Save room changes |
| DELETE | /rooms/:id | rooms#destroy | Delete room |

So `/rooms/1` → `RoomsController#show` with `params[:id] = "1"`

**The convention:** Rails doesn't need you to spell this out. It *assumes* that `resources :rooms` means there's a `RoomsController` with these action names. Convention over configuration.

---

### Step 2: The Controller - The Coordinator

**Location:** `app/controllers/rooms_controller.rb:10-12`

```ruby
class RoomsController < ApplicationController
  before_action :set_room, only: %i[ show destroy ]

  def show
    @messages = find_messages
  end

  private
    def set_room
      if room = Current.user.rooms.find_by(id: params[:room_id] || params[:id])
        @room = room
      else
        redirect_to root_url, alert: "Room not found or inaccessible"
      end
    end

    def find_messages
      messages = @room.messages.with_creator

      if show_first_message = messages.find_by(id: params[:message_id])
        @messages = messages.page_around(show_first_message)
      else
        @messages = messages.last_page
      end
    end
end
```

**What this does:**

1. `before_action :set_room` runs `set_room` before `show` executes
2. `set_room` finds the room from the database (using the Model) and stores it in `@room`
3. `show` fetches the messages and stores them in `@messages`
4. Instance variables (`@room`, `@messages`) are automatically available in the view

**Notice what the controller does NOT do:**
- It doesn't know how to render HTML
- It doesn't know what a room's fields are
- It doesn't contain business logic about rooms

It just coordinates: get data, hand it to the view.

---

### Step 3: The Model - The Brain

**Location:** `app/models/room.rb:1-30`

```ruby
class Room < ApplicationRecord
  has_many :memberships, dependent: :delete_all
  has_many :users, through: :memberships
  has_many :messages, dependent: :destroy

  belongs_to :creator, class_name: "User", default: -> { Current.user }

  scope :opens,   -> { where(type: "Rooms::Open") }
  scope :closeds, -> { where(type: "Rooms::Closed") }
  scope :directs, -> { where(type: "Rooms::Direct") }

  def open?
    is_a?(Rooms::Open)
  end

  def direct?
    is_a?(Rooms::Direct)
  end

  def receive(message)
    unread_memberships(message)
    push_later(message)
  end
end
```

**What this does:**

The model defines:
- **Relationships**: A room `has_many :messages`, `belongs_to :creator`
- **Scopes**: Easy ways to query rooms (`Room.opens`, `Room.directs`)
- **Business logic**: What happens when a room receives a message

**Notice what the model does NOT do:**
- It doesn't know about HTTP requests
- It doesn't know about HTML
- It doesn't know about controllers

It just knows what a Room IS and what it can DO.

**The convention:** Model file is `app/models/room.rb`, class is `Room`, table is `rooms`. Rails figures out the connections from the naming.

---

### Step 4: The View - The Presenter

**Location:** `app/views/rooms/show.html.erb`

```erb
<% @page_title = room_display_name(@room) %>
<% @body_class = "sidebar" %>

<%= render "rooms/show/nav", room: @room %>

<%= message_area_tag(@room) do %>
  <%= messages_tag(@room) do %>
    <%= render "rooms/show/invitation", room: @room %>
    <%= render partial: "messages/message", collection: @messages, cached: true %>
  <% end %>

  <%= turbo_stream_from @room, :messages %>
  <%= button_to_jump_to_newest_message %>
<% end %>

<%= render "rooms/show/composer", room: @room %>
```

**What this does:**

- Uses `@room` and `@messages` (set by the controller)
- Mixes HTML with Ruby (`<%= %>` for output, `<% %>` for logic)
- `render partial:` breaks complex views into smaller pieces

**The convention:** Controller action `rooms#show` automatically renders `app/views/rooms/show.html.erb`. No configuration needed.

---

## Convention Over Configuration: The Pattern

Notice how everything connects without explicit wiring:

| Component | Convention | File Location |
|-----------|------------|---------------|
| Route | `resources :rooms` | config/routes.rb |
| Controller | `RoomsController` | app/controllers/rooms_controller.rb |
| Model | `Room` | app/models/room.rb |
| View | `show` action | app/views/rooms/show.html.erb |
| Database | `rooms` table | db/schema.rb |

**This is the superpower.** A senior Rails developer can join any project and immediately know where to look. Authentication code? `app/models/user.rb` and `app/controllers/sessions_controller.rb`. Message sending? `app/models/message.rb` and `app/controllers/messages_controller.rb`.

## Try It Yourself

**Challenge:** If you wanted to add a "room description" field to Campfire, which files would you touch and in what order?

Think about it before reading the answer...

<details>
<summary>Click for answer</summary>

1. **Database migration** (create the column): `rails generate migration AddDescriptionToRooms description:text`
2. **Model** (if you need validations): `app/models/room.rb` - maybe add `validates :description, length: { maximum: 500 }`
3. **Controller** (permit the parameter): `app/controllers/rooms_controller.rb` - update `room_params` to permit `:description`
4. **View** (display it): `app/views/rooms/show.html.erb` or a partial - add `<p><%= @room.description %></p>`
5. **Form** (allow editing): Whatever form creates/edits rooms - add a text area for description

The order matters: you can't use a column that doesn't exist in the database!

</details>

## Summary

- **MVC separates concerns**: Models handle data/logic, Controllers coordinate, Views render HTML
- **Convention over configuration**: Rails assumes file locations based on naming - learn the pattern once, navigate any Rails app
- **The request flow**: Browser → Router → Controller → Model → Database → Model → Controller → View → Browser
- **Instance variables bridge controller and view**: Set `@room` in controller, use `@room` in view
- **Everything has a place**: When you're not sure where code goes, ask "Is this data logic (Model), coordination (Controller), or presentation (View)?"

**Next up:** We'll explore ActiveRecord - the magic that makes `Room.find(1)` actually query the database and return a Ruby object. That's where the real Rails superpowers begin.

---

## Q&A

### Q: What does "business logic" actually mean?

**A:** Business logic = the rules that make your app YOUR app. It's called "business" logic because it comes from "business requirements" - but a clearer name is **domain rules**.

**Example from Campfire** (`app/models/room.rb:68-69`):
```ruby
def unread_memberships(message)
  memberships.visible.disconnected.where.not(user: message.creator).update_all(unread_at: message.created_at)
end
```

This encodes the rule: "When a message arrives, mark the room as unread for everyone except the sender." That's not a Rails thing or a database thing - it's a *Campfire* thing.

**The test:** If you rebuilt this app in Python/Django, what rules would still apply? Those are your business logic. "Only admins can delete rooms" would still apply. "Use `before_action` callbacks" would not - that's Rails-specific infrastructure.

**Why models?** Models represent your domain (rooms, messages, users). The rules about those things belong with those things. Controllers are traffic cops, views are painters, models are where your app's *brain* lives.

### Q: What are the actual boundaries of business logic? Why can't it go in controllers or views?

**The Boundary Test**

Here's a simple rule for identifying business logic:

**Business logic = any decision your app makes that a non-programmer stakeholder would care about.**

If you told your boss "I'm changing how we decide who gets notified," they'd care. That's business logic.

If you told them "I'm changing which database column stores the timestamp," they'd say "I don't care, just make it work." That's infrastructure.

**What It Looks Like in the Wrong Place**

Let's see the **same feature** implemented three ways - the "mark room as unread" feature from Campfire.

**Bad: Business Logic in the Controller**

```ruby
# app/controllers/messages_controller.rb
def create
  @message = @room.messages.create!(message_params)

  # Business logic BURIED in the controller
  @room.memberships
       .visible
       .disconnected
       .where.not(user: @message.creator)
       .update_all(unread_at: @message.created_at)

  # More business logic - push notifications
  @room.memberships.involved.where.not(user: @message.creator).each do |membership|
    PushNotificationJob.perform_later(membership.user, @message)
  end
end
```

**Bad: Business Logic in the View**

```erb
<%# app/views/rooms/show.html.erb %>

<% # Calculating who should see the "unread" badge - IN THE VIEW! %>
<% @room.memberships.each do |membership| %>
  <% if membership.visible? && membership.disconnected? && membership.user != current_user %>
    <% membership.update(unread_at: Time.current) %>
  <% end %>
<% end %>

<%# This is horrifying but I've seen it in real codebases %>
```

**Good: Business Logic in the Model**

```ruby
# app/models/room.rb
def receive(message)
  unread_memberships(message)
  push_later(message)
end

private
  def unread_memberships(message)
    memberships.visible.disconnected.where.not(user: message.creator)
               .update_all(unread_at: message.created_at)
  end

  def push_later(message)
    Room::PushMessageJob.perform_later(self, message)
  end
```

```ruby
# app/controllers/messages_controller.rb
def create
  @message = @room.messages.create!(message_params)
  @room.receive(@message)  # That's it. The controller doesn't know HOW.
end
```

**Why the Wrong Places Are Wrong**

*Problem 1: Duplication*

What if messages can also be created by:
- The web form (MessagesController)
- A bot API (Bots::MessagesController)
- An import script (rake task)
- A test

With logic in the controller, you'd copy-paste that code 4 times. When the rule changes ("actually, don't notify people who have Do Not Disturb on"), you'd have to find and update 4 places.

With logic in the model, there's ONE place: `room.receive(message)`. Change it once, everything updates.

*Problem 2: Testing*

```ruby
# Testing business logic in a controller - PAINFUL
test "creating a message marks room as unread for others" do
  post messages_url, params: { message: { body: "hello" } }
  # Now I need to set up HTTP requests, sessions, authentication...
  # Just to test a business rule??
end

# Testing business logic in a model - EASY
test "receiving a message marks room as unread for others" do
  room.receive(message)
  assert membership.reload.unread_at.present?
  # No HTTP, no sessions, no controllers. Just the rule.
end
```

*Problem 3: Reading the Code*

When a new developer joins and asks "how does the unread system work?", where do they look?

- If it's in controllers: they have to search through every controller that might create messages
- If it's in models: they open `room.rb`, see `receive`, done

**Boundary Examples**

| Scenario | Business Logic? | Why |
|----------|-----------------|-----|
| "Rooms must have a name" | Yes | Domain rule - a nameless room doesn't make sense |
| "Names must be ≤ 50 chars" | Yes | Product decision someone made |
| "Store name in VARCHAR column" | No | Infrastructure - could be TEXT, doesn't matter to users |
| "Only room creator can delete it" | Yes | Authorization rule - stakeholders care |
| "Use `before_action` for auth check" | No | Rails mechanism - could use different pattern |
| "Display dates as 'Jan 5'" | No | Presentation - the underlying date is the same |
| "Messages older than 30 days auto-delete" | Yes | Policy someone decided |
| "Use background job for deletion" | No | Implementation detail |

**The Real Campfire Pattern**

Look at how Campfire actually does it (`app/models/room.rb:46-49`):

```ruby
def receive(message)
  unread_memberships(message)
  push_later(message)
end
```

The controller just says `@room.receive(@message)`. It doesn't know or care about:
- Which memberships get marked unread
- How push notifications work
- What "disconnected" means

The controller's job is: "a message was created, tell the room." The model's job is: "I know what to do when I receive a message."

**The key insight: business logic is about WHAT happens, not HOW it's implemented or WHERE it's displayed.**

---

## Quiz History

### Quiz - 25-11-2025

**Q1:** Why does `room.receive(message)` live in the Model instead of the Controller?
**A:** Correctly identified it as business logic using the "stakeholder test" - if a PM would care about the rule, it's business logic and belongs in models.

**Q2:** List the file paths for a new "Bookmarks" feature.
**A:** Got model (`bookmark.rb`) and controller (`bookmarks_controller.rb`) correct. Initially confused about views - thought it was a single file rather than a directory with action-named files (`app/views/bookmarks/*.html.erb`).

**Q3:** Order the request lifecycle steps for visiting `/rooms/5`.
**A:** Perfect - correctly ordered: Browser → Router → Controller → Model/DB → View → Browser.

**Score:** 7/10 - Strong conceptual understanding of MVC separation and request flow. Gap in precise view file conventions.
