---
concepts: [Turbo Streams, WebSockets, ActionCable, Broadcasting, Real-time Updates]
source_repo: once-campfire
description: Deep dive into Turbo Streams - how Rails apps push real-time updates to browsers without writing JavaScript. Covers the two delivery mechanisms (HTTP responses and WebSocket broadcasts), the stream actions (append, prepend, replace, remove), and how to subscribe to updates. All demonstrated through Campfire's chat messaging system.
understanding_score: null
last_quizzed: null
prerequisites: [2025-11-28-turbo-drive-and-frames---the-spa-killer.md]
created: 12-12-2025
last_updated: 12-12-2025  # Q&A: foundations, AJAX, HTML-over-wire, two-pipes abstraction
---

# Turbo Streams - Real-Time Without the Complexity

You're in a Campfire chat room. Someone else sends a message. It just... appears. No refresh. No polling. You didn't click anything. The message materialized in your browser as if by magic.

How does that work? Let's build up to it step by step.

## First: What Normally Happens When You Submit a Form?

Before we understand Turbo Streams, let's be crystal clear about what normally happens in Rails.

You're in a chat room. You type "Hello" and click Send. That's a form submission:

```
Your Browser                           Server
     │                                    │
     │  POST /rooms/1/messages            │
     │  (form data: body="Hello")         │
     ├───────────────────────────────────>│
     │                                    │
     │                        Creates the message in database
     │                                    │
     │         ??? What comes back ???    │
     │<───────────────────────────────────┤
     │                                    │
```

In traditional Rails, the server responds with one of two things:

**Option A: A redirect** (most common)
```ruby
def create
  @message = @room.messages.create!(message_params)
  redirect_to @room  # Send a 302 redirect
end
```
The browser sees the redirect, makes a NEW GET request to `/rooms/1`, and the entire page reloads. Your message is there, but everything refreshed.

**Option B: Render HTML**
```ruby
def create
  @message = @room.messages.create!(message_params)
  render :show  # Send back a full HTML page
end
```
The browser receives a complete HTML page and replaces everything on screen.

**Both options replace or reload the entire page.** That's how the web traditionally worked.

## The Problem: You Want Surgical Updates

In a chat app, you don't want the whole page to reload when you send a message. You want:
- Your message to appear at the bottom of the chat
- Everything else to stay exactly as it was
- No page flash, no scroll reset, no disruption

With Turbo Frames (from our last tutorial), you learned one way to do surgical updates: wrap an area in a frame, and only that frame gets replaced.

But Turbo Frames require the user to **click something**. The frame makes a request, gets a response, swaps itself.

What about when you need to:
1. **Add** something new (not replace an existing frame)?
2. Update the page **without the user clicking anything**?

That's where Turbo Streams comes in.

## Turbo Streams: A Third Type of Response

Turbo Streams introduces a new type of server response. Instead of:
- HTML that replaces the whole page
- A redirect that triggers a new request

It's **instructions** that tell the browser exactly what to change:

```ruby
def create
  @message = @room.messages.create!(message_params)
  # Instead of redirect or render, respond with instructions:
  # "Take this HTML and append it to the messages container"
end
```

The server sends back something that looks like this:

```html
<turbo-stream action="append" target="messages_room_5">
  <template>
    <div id="message_42" class="message">
      <strong>You:</strong> Hello
    </div>
  </template>
</turbo-stream>
```

This isn't a page. It's an **instruction**:
- `action="append"` - Add to the end of...
- `target="messages_room_5"` - ...the element with this ID
- `<template>` - Here's the HTML to add

Turbo (the JavaScript running in your browser) receives this, reads the instruction, and executes it. Your message appears at the bottom of the chat. Nothing else changes.

## How Does Rails Know to Send This Format?

When Turbo is loaded in your browser, it modifies form submissions to include a special header:

```
Accept: text/vnd.turbo-stream.html
```

This tells Rails: "I understand Turbo Streams. You can send me instructions, not just HTML."

Rails sees this header and looks for a `.turbo_stream.erb` template:

```
app/views/messages/
├── create.html.erb          ← Regular HTML (fallback)
├── create.turbo_stream.erb  ← Turbo Stream instructions ✓
```

If the Turbo Stream template exists, Rails uses it. If not, it falls back to HTML.

## Connecting to Turbo Frames

Remember from the last tutorial: Turbo Frames let you **replace** part of a page when the user clicks something.

Turbo Streams goes further:
- Can **append, prepend, remove** - not just replace
- Can update **any element by ID** - not just matching frames
- Works for form responses AND (as we'll see) real-time pushes

Think of it this way:
- **Turbo Frames** = "When this area needs to change, swap in new content"
- **Turbo Streams** = "Here's a specific instruction for what to change in the DOM"

## Now: The Two Delivery Mechanisms

Here's the key insight that unlocks the rest of this tutorial: **Turbo Streams instructions can arrive via two completely different pipes.**

### Pipe 1: HTTP Response (for YOU, the person who did the action)

When YOU send a message, your browser made an HTTP request. The server responds with Turbo Stream instructions:

```
You click Send
     │
     ▼
POST /rooms/1/messages
     │
     ▼
Server creates message, responds with:
<turbo-stream action="append" target="messages">
  <template>Your message HTML</template>
</turbo-stream>
     │
     ▼
YOUR browser executes the instruction
Your message appears
```

This is the `.turbo_stream.erb` response we just discussed.

### Pipe 2: WebSocket (for EVERYONE ELSE in the room)

But what about the other people in the chat room? They didn't click anything. They didn't make a request. How do they see your message?

This is where **WebSockets** come in.

Normal HTTP is like sending letters:
- You send a request (letter to server)
- Server sends a response (letter back)
- Connection closes
- If you want updates, you have to send another letter

WebSockets are like a phone call:
- You establish a connection (call the server)
- The line stays open
- Either side can talk at any time
- Server can PUSH messages to you without you asking

When you load a Campfire chat room, your browser opens a WebSocket connection and says "I want to hear about new messages in this room."

When someone sends a message, the server:
1. Responds to THEM via HTTP (Pipe 1)
2. PUSHES to EVERYONE ELSE via WebSocket (Pipe 2)

Both pipes carry the same Turbo Stream instructions. Same format, different delivery method.

```
You send "Hello"
        │
        ▼
┌─────────────────────────────────────────────────────┐
│                    SERVER                            │
│                                                      │
│  Creates message in database                         │
│         │                                            │
│         ├──────────────────┬────────────────────┐   │
│         │                  │                    │   │
│         ▼                  ▼                    ▼   │
│   HTTP Response      WebSocket Push       WebSocket Push
│   (to you)           (to Alice)           (to Bob)  │
│                                                      │
└─────────────────────────────────────────────────────┘
        │                  │                    │
        ▼                  ▼                    ▼
   Your browser       Alice's browser      Bob's browser

   All execute: turbo_stream.append "Hello" to messages
```

**Same instruction format. Two different pipes.** This is the elegance of Turbo Streams.

## Stream Actions: The Vocabulary

Turbo Streams speaks a simple language of DOM operations:

| Action | What it does |
|--------|-------------|
| `append` | Add HTML to the END of a container |
| `prepend` | Add HTML to the BEGINNING of a container |
| `replace` | Swap out an element entirely |
| `update` | Replace just the CONTENTS of an element |
| `remove` | Delete an element |
| `before` | Insert HTML before an element |
| `after` | Insert HTML after an element |

Each action needs to know WHERE to operate. That's the `target` - a DOM element ID.

## The Anatomy of a Turbo Stream

Under the hood, a Turbo Stream is just HTML with special tags:

```html
<turbo-stream action="append" target="messages_room_5">
  <template>
    <div id="message_42" class="message">
      Hello from the other side!
    </div>
  </template>
</turbo-stream>
```

This says: "Append the contents of `<template>` to the element with id `messages_room_5`."

When Turbo (running in the browser) receives this, it:
1. Finds the element with id `messages_room_5`
2. Takes the HTML inside `<template>`
3. Appends it to that element

That's it. No custom JavaScript. Turbo does the DOM manipulation for you.

## Real Example 1: Sending a Message

Let's trace what happens when you send a message in Campfire.

### Step 1: The Controller Creates the Message

Look at `app/controllers/messages_controller.rb`:

```ruby
def create
  set_room
  @message = @room.messages.create_with_attachment!(message_params)

  @message.broadcast_create  # ← This is the magic line
  deliver_webhooks_to_bots
end
```

Notice there's no explicit `render` call. Rails will automatically look for a template matching the format. For Turbo requests, that's `create.turbo_stream.erb`.

### Step 2: Your Browser Gets a Turbo Stream Response

Look at `app/views/messages/create.turbo_stream.erb`:

```erb
<%= turbo_stream.append dom_id(@message.room, :messages), @message %>
```

One line. Let's unpack it:

- `turbo_stream.append` - The action (add to end)
- `dom_id(@message.room, :messages)` - The target (`messages_room_5`)
- `@message` - Rails automatically renders `_message.html.erb` partial

This generates:

```html
<turbo-stream action="append" target="messages_room_5">
  <template>
    <!-- The full message HTML from _message.html.erb -->
  </template>
</turbo-stream>
```

Your browser receives this, and the message appears at the bottom of your chat.

### Step 3: Everyone Else Gets a WebSocket Broadcast

But you're not the only one in the room. Look at `broadcast_create` in `app/models/message/broadcasts.rb`:

```ruby
module Message::Broadcasts
  def broadcast_create
    broadcast_append_to room, :messages, target: [ room, :messages ]
    ActionCable.server.broadcast("unread_rooms", { roomId: room.id })
  end
end
```

`broadcast_append_to` does the same thing as the HTTP response, but over WebSockets to ALL subscribers.

The arguments:
- `room, :messages` - The channel name (becomes `rooms:5:messages`)
- `target: [ room, :messages ]` - Where to append (becomes `messages_room_5`)

Rails renders the `_message.html.erb` partial, wraps it in a `<turbo-stream>` tag, and pushes it over ActionCable to everyone subscribed to that channel.

### Step 4: Browsers Must Subscribe First

For WebSocket broadcasts to work, browsers need to subscribe. Look at `app/views/rooms/show.html.erb`:

```erb
<%= turbo_stream_from @room, :messages %>
```

This tiny line does a lot:
1. Creates an invisible `<turbo-cable-stream-source>` element
2. Opens a WebSocket connection to ActionCable
3. Subscribes to the `rooms:5:messages` channel
4. When messages arrive on that channel, executes them as Turbo Stream actions

The subscription happens automatically when you load the page. You don't write any JavaScript.

## The Full Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                         SERVER                                   │
│                                                                  │
│  POST /rooms/1/messages                                         │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────────┐                                            │
│  │ MessagesController │                                          │
│  │   create action   │                                          │
│  └────────┬──────────┘                                          │
│           │                                                      │
│           │ Creates @message                                     │
│           │                                                      │
│           ├────────────────────────────┐                        │
│           │                            │                        │
│           ▼                            ▼                        │
│  ┌────────────────────┐    ┌─────────────────────────┐         │
│  │ create.turbo_stream │    │ broadcast_append_to     │         │
│  │ (HTTP Response)     │    │ (WebSocket Broadcast)   │         │
│  └──────────┬─────────┘    └───────────┬─────────────┘         │
│             │                          │                        │
└─────────────┼──────────────────────────┼────────────────────────┘
              │                          │
              │ HTTP                     │ WebSocket
              │                          │ (ActionCable)
              ▼                          ▼
      ┌──────────────┐          ┌──────────────┐
      │  YOUR        │          │  EVERYONE    │
      │  BROWSER     │          │  ELSE        │
      │              │          │              │
      │ turbo_stream │          │ turbo_stream │
      │ .append      │          │ .append      │
      └──────────────┘          └──────────────┘
              │                          │
              └──────────┬───────────────┘
                         │
                         ▼
              Message appears in all browsers!
```

## Real Example 2: Deleting a Message

Deletion is even simpler. Look at the controller:

```ruby
def destroy
  @message.destroy
  @message.broadcast_remove_to @room, :messages
end
```

And `app/views/messages/destroy.turbo_stream.erb`:

```erb
<%= turbo_stream.remove @message %>
```

`turbo_stream.remove` generates:

```html
<turbo-stream action="remove" target="message_42">
</turbo-stream>
```

No template needed - we're just removing, not adding content.

The `broadcast_remove_to` pushes this to all subscribers, and the message disappears from everyone's screen.

## Real Example 3: Updating a Message

When you edit a message, Campfire updates just the presentation part:

```ruby
def update
  @message.update!(message_params)

  @message.broadcast_replace_to @room, :messages,
    target: [ @message, :presentation ],
    partial: "messages/presentation",
    attributes: { maintain_scroll: true }

  redirect_to room_message_url(@room, @message)
end
```

This is more surgical. Instead of replacing the whole message, it targets just `presentation_message_42` - the content area inside the message frame.

The `maintain_scroll: true` attribute tells Turbo not to scroll when replacing. Nice touch.

## Real Example 4: Boost (Reactions)

When someone adds a reaction, look at `app/controllers/messages/boosts_controller.rb`:

```ruby
def create
  @boost = @message.boosts.create!(boost_params)

  broadcast_create
  redirect_to message_boosts_url(@message)
end

private

def broadcast_create
  @boost.broadcast_append_to @boost.message.room, :messages,
    target: "boosts_message_#{@boost.message.client_message_id}",
    partial: "messages/boosts/boost",
    attributes: { maintain_scroll: true }
end
```

Notice the target: `boosts_message_abc123`. This appends the new boost to the boosts container inside a specific message. The reaction appears for everyone without touching any other part of the UI.

## Real Example 5: Sidebar Updates

The sidebar subscribes to TWO channels. Look at `app/views/users/sidebars/show.html.erb`:

```erb
<%= sidebar_turbo_frame_tag do %>
  <%= turbo_stream_from :rooms %>
  <%= turbo_stream_from Current.user, :rooms %>
```

Why two?

1. `:rooms` - Global room events (new public room created, room renamed)
2. `Current.user, :rooms` - User-specific events (you were added to a private room)

When a new room is created, look at `app/controllers/rooms/opens_controller.rb`:

```ruby
def create
  room = Rooms::Open.create_for(room_params, users: Current.user)

  broadcast_create_room(room)
  redirect_to room_url(room)
end

private

def broadcast_create_room(room)
  broadcast_prepend_to :rooms,
    target: :shared_rooms,
    partial: "users/sidebars/rooms/shared",
    locals: { room: room }
end
```

`broadcast_prepend_to :rooms` pushes to everyone subscribed to the `:rooms` channel. The new room appears at the top of everyone's sidebar.

For private rooms, it's different - `app/controllers/rooms/directs_controller.rb`:

```ruby
def broadcast_create_room(room)
  room.memberships.each do |membership|
    membership.broadcast_prepend_to membership.user, :rooms,
      target: :direct_rooms,
      partial: "users/sidebars/rooms/direct"
  end
end
```

This broadcasts to EACH member's personal channel (`user_42:rooms`). Only the people in the DM see it appear.

## The Naming Convention

Channel names follow a pattern:

```ruby
turbo_stream_from @room, :messages
# Subscribes to: rooms:5:messages

turbo_stream_from :rooms
# Subscribes to: rooms

turbo_stream_from Current.user, :rooms
# Subscribes to: users:42:rooms
```

Broadcasting uses the same convention:

```ruby
broadcast_append_to room, :messages
# Broadcasts to: rooms:5:messages

broadcast_prepend_to :rooms
# Broadcasts to: rooms

broadcast_prepend_to user, :rooms
# Broadcasts to: users:42:rooms
```

If you subscribe to `X`, you receive broadcasts to `X`. Simple.

## Common Patterns

### Pattern 1: HTTP Response + Broadcast

The sender gets an HTTP response. Everyone else gets a broadcast.

```ruby
# Controller
def create
  @thing = Thing.create!(thing_params)
  @thing.broadcast_append_to :things
end
```

```erb
<%# create.turbo_stream.erb %>
<%= turbo_stream.append :things, @thing %>
```

### Pattern 2: Broadcast Only

Sometimes you don't need an HTTP response (the action doesn't change the current user's view).

```ruby
def mark_read
  @notification.update!(read: true)
  @notification.broadcast_remove_to Current.user, :notifications
  head :ok
end
```

### Pattern 3: Multiple Streams

One action can trigger multiple broadcasts:

```ruby
def create
  @message = @room.messages.create!(message_params)

  # Append message to room
  @message.broadcast_append_to @room, :messages

  # Update unread indicators in sidebar
  @room.memberships.each do |m|
    m.broadcast_replace_to m.user, :rooms,
      target: dom_id(m.room, :list),
      partial: "sidebar/room"
  end
end
```

## When HTTP Response vs Broadcast?

| Scenario | Use |
|----------|-----|
| Update the UI for the person who made the request | HTTP response (`.turbo_stream.erb`) |
| Update everyone else's UI | WebSocket broadcast (`broadcast_*_to`) |
| Update EVERYONE including requester | Both, or just broadcast if identical |
| Background job finishes | Broadcast only (no HTTP request to respond to) |

## The Mental Model

Think of it like a chat app (fitting, since we're studying one):

- **HTTP Response** = The server replies directly to you
- **WebSocket Broadcast** = The server announces to the whole room

Both use the same language (Turbo Stream actions), just different delivery pipes.

## Try It Yourself

1. Open `app/controllers/messages/boosts_controller.rb`
2. Find the `destroy` action
3. Trace how the boost disappears from everyone's screen:
   - What broadcast method is called?
   - What channel does it broadcast to?
   - What target does it remove?
4. Now open the sidebar in a browser. Create a new room. See it appear.
5. Open `app/controllers/rooms/opens_controller.rb` and trace how that happened.

## Summary

- **Turbo Streams** = DOM manipulation instructions that can travel via HTTP or WebSocket
- **Two delivery mechanisms**: HTTP responses for the requester, broadcasts for everyone else
- **Five core actions**: append, prepend, replace, update, remove (plus before/after)
- **`turbo_stream_from`** subscribes a browser to a channel
- **`broadcast_*_to`** pushes to all subscribers of a channel
- **ActionCable** handles the WebSocket infrastructure (you don't touch it directly)

The philosophy: **Describe what should change. Let Rails figure out how to deliver it.**

You've now seen how Campfire achieves real-time chat without writing any WebSocket JavaScript. The next tutorial covers **Stimulus** - for the JavaScript you DO need to write.

---

## Q&A

### Q: What does "respond with a Turbo Stream instead of HTML" mean?

**Question:** I don't understand what it means when you say the server can respond with a Turbo Stream instead of HTML. Can you connect this with what's actually happening? I don't have the "old days" context - I learned Rails in the AI era.

**Answer:** This question revealed a gap in the tutorial - it assumed knowledge of how the web traditionally worked.

**The foundation:** When you submit a form in Rails, the server traditionally responds with either:
1. A **redirect** (302 response) → browser makes a new request, page reloads
2. **Full HTML** → browser replaces everything on screen

Both result in the entire page refreshing.

**What Turbo Streams changes:** Instead of HTML or a redirect, the server can respond with **instructions**:

```html
<turbo-stream action="append" target="messages_room_5">
  <template>
    <div class="message">Hello</div>
  </template>
</turbo-stream>
```

This isn't a page - it's a command: "Append this HTML to the element with id `messages_room_5`."

Turbo (JavaScript in the browser) receives this, parses the instruction, and executes it. Only that one element changes. Everything else stays put.

**How Rails knows to send this format:** When Turbo is loaded, it adds a header to requests: `Accept: text/vnd.turbo-stream.html`. Rails sees this and looks for `.turbo_stream.erb` templates instead of `.html.erb`.

**Connection to Turbo Frames:**
- Turbo Frames = "When this area changes, swap in new content" (requires clicking)
- Turbo Streams = "Here's an instruction for what to change" (can be pushed anytime)

### Q: Is Turbo Streams related to AJAX?

**Question:** It sounds like Turbo Streams are trying to solve the problem that AJAX solved - surgical updates to the page. Are these two concepts connected?

**Answer:** Yes! You're making exactly the right connection. Turbo Streams is essentially AJAX with the JavaScript abstracted away.

#### The AJAX Era (2005+)

AJAX (Asynchronous JavaScript and XML) was revolutionary when it appeared. Before AJAX, every user action meant a full page reload. AJAX introduced:

1. Make HTTP requests **in the background** (without reloading)
2. Receive data back
3. Update just part of the page

But here's what AJAX required you to write:

```javascript
// 1. Intercept the form submit
document.querySelector('form').addEventListener('submit', (e) => {
  e.preventDefault();

  // 2. Make background request
  fetch('/messages', {
    method: 'POST',
    body: new FormData(e.target)
  })
  .then(response => response.json())  // 3. Parse the JSON response
  .then(data => {
    // 4. Manually build HTML from the data
    const messageHtml = `
      <div class="message">
        <strong>${data.author}:</strong> ${data.body}
      </div>
    `;

    // 5. Find the container and insert
    document.getElementById('messages').insertAdjacentHTML('beforeend', messageHtml);
  });
});
```

You had to write JavaScript to:
- Intercept the action
- Make the request
- Parse the response (usually JSON)
- Build the HTML
- Find the DOM element
- Insert it correctly

**Every feature = more JavaScript.** This is why frontend frameworks exploded - React, Vue, Angular all emerged to manage this complexity.

#### Turbo Streams: AJAX Without the JavaScript

Turbo Streams uses the same underlying mechanism (background HTTP requests via `fetch()`), but:

| AJAX | Turbo Streams |
|------|---------------|
| Server returns JSON data | Server returns HTML with instructions |
| You write JS to parse data | Turbo parses it for you |
| You write JS to build HTML | Server already sends HTML |
| You write JS to update DOM | Turbo executes the instruction |

```ruby
# Server just says what to do:
turbo_stream.append :messages, @message
```

```html
<!-- Browser receives: -->
<turbo-stream action="append" target="messages">
  <template>
    <div class="message">Already-rendered HTML</div>
  </template>
</turbo-stream>
```

Turbo (the library) handles everything else. You write zero JavaScript for the update.

#### The Progression of Web Development

```
1995-2005: Full page reload for everything
    ↓
2005-2015: AJAX - background requests, but tons of JavaScript to handle responses
    ↓
2015-2020: SPA frameworks (React/Vue) - even MORE JavaScript, but organized
    ↓
2020+: Turbo Streams - background requests, server sends instructions, minimal JS
```

#### HTML Over The Wire

This is what DHH means by "HTML over the wire" - and it's literally what **Hotwire** stands for: **H**TML **O**ver **T**he **Wire**.

The insight: Instead of sending JSON over the wire and having JavaScript build HTML on the client... just send the HTML directly. The server already knows how to render HTML beautifully (that's what Rails views do!) - why duplicate that logic in JavaScript?

| JSON Over The Wire (SPA approach) | HTML Over The Wire (Hotwire approach) |
|-----------------------------------|---------------------------------------|
| Server sends: `{"author": "Bob", "body": "Hello"}` | Server sends: `<div class="message"><strong>Bob:</strong> Hello</div>` |
| Client JS builds HTML from JSON | Client just inserts the HTML |
| Two rendering systems (server + client) | One rendering system (server only) |
| Duplicate templates | Single source of truth for views |

Turbo Streams is DHH saying: "AJAX was right about the UX, but wrong about where the complexity should live. Keep the rendering logic on the server."

### Q: Why does Turbo Streams handle both HTTP responses AND WebSocket broadcasts?

**Question:** HTTP and WebSockets are two completely different web technologies. It feels like Turbo Streams is doing two separate jobs - responding to my requests AND pushing updates I didn't ask for. Why are these combined under one name?

**Answer:** You're absolutely right - these ARE two different technologies and two different jobs. The unification is intentional.

#### The Two Different Jobs

| Job 1: HTTP Response | Job 2: WebSocket Broadcast |
|---------------------|---------------------------|
| Respond to MY request | Push updates I didn't ask for |
| I clicked something | Someone else did something |
| `create.turbo_stream.erb` | `broadcast_append_to` |

#### Why They're Unified

The insight: **the instruction format can be the same even if delivery differs**.

Whether this arrives via HTTP or WebSocket:
```html
<turbo-stream action="append" target="messages">
  <template><div>Hello</div></template>
</turbo-stream>
```

Your browser does the same thing. It's like receiving a letter vs a text message - different delivery, same instruction.

Think of it this way:
```
"Turbo Streams" = a FORMAT for DOM instructions

That format can ARRIVE via:
├── HTTP Response (you asked for it)
└── WebSocket Push (you subscribed to it)
```

#### The Rails Philosophy: Abstract Away the Technology

This is a deliberate design choice that follows the same pattern throughout Rails:

| Technology | Abstraction | You think about... | Not about... |
|------------|-------------|-------------------|--------------|
| SQL | ActiveRecord | `User.where(active: true)` | `SELECT * FROM users WHERE...` |
| WebSockets | Turbo Streams | `broadcast_append_to :messages` | Connections, channels, reconnection |
| HTTP caching | Fresh/stale helpers | `fresh_when @message` | Cache headers, ETags |
| Background jobs | ActiveJob | `MessageJob.perform_later` | Redis queues, workers |

A new Rails developer could build a real-time chat app with Turbo Streams and **never know WebSockets exist** - just like they might use ActiveRecord for years without writing raw SQL.

That's the power (and sometimes criticism) of Rails abstractions: they hide complexity. When it works, it's magic. When it breaks, you might not understand what's underneath.

DHH's bet: for 90% of use cases, you don't NEED to understand WebSocket technology. You just need to express: "push this update to everyone in the room." Turbo Streams lets you stay at that level.

#### When Each Delivery Method Is Used

| Situation | Delivery | Why |
|-----------|----------|-----|
| You submit a form | HTTP response | You made a request → you get a response |
| Someone else acts | WebSocket push | You didn't ask → server pushes to you |
| Background job finishes | WebSocket push | No HTTP request to respond to |
| Page first loads | Neither | Regular HTML, then subscribe for future updates |

## Quiz History

[Quiz sessions will be recorded here after the learner is quizzed on this topic]
