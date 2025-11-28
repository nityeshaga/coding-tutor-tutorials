---
concepts: [Views, ERB, Partials, Layouts, content_for, View Helpers]
description: Deep dive into the V in MVC - how Rails turns your data into HTML. Covers ERB templating, the layout system with yield and content_for, partials for reusable UI, view helpers for clean code, and collection rendering. All demonstrated with real Campfire code showing how 37signals structures a production chat app's views.
understanding_score: null
last_quizzed: null
prerequisites: [references/tutorials/2025-11-25-mvc---the-architecture-that-powers-every-rails-app.md, references/tutorials/2025-11-27-controllers-deep-dive---the-glue-that-holds-rails-together.md]
created: 28-11-2025
last_updated: 28-11-2025
---

# Views, Partials and Layouts - The Art of Rendering HTML

You've followed a request from URL to controller, watched the controller fetch data from models. But that data is just Ruby objects sitting in memory. How does it become the HTML that users actually see?

That's the job of views. And it's not just "loop through an array and print stuff." Rails has an elegant system for organizing view code that prevents the nightmare scenario every web developer has seen: giant template files full of repeated HTML, inline logic, and copy-pasted components.

Think about Campfire - a chat app where every room shows dozens of messages, each with an avatar, timestamp, author name, message body, action buttons, and boost reactions. Without a good view architecture, you'd have the same 50 lines of HTML copy-pasted everywhere a message appears. Change something? Find and update it in 12 different places. Miss one? Enjoy your bug.

Rails solves this with three interlocking pieces: **layouts** (the outer shell that wraps every page), **partials** (reusable chunks of HTML), and **helpers** (Ruby methods that generate HTML). Master these, and you'll write view code that's DRY, maintainable, and actually pleasant to work with.

## ERB: Ruby in Your HTML

Before we get to the architecture, let's understand the basic building block: ERB (Embedded Ruby).

ERB is dead simple. You write HTML, and when you need Ruby, you use special tags:

```erb
<% %>   ← Run Ruby code (no output)
<%= %>  ← Run Ruby code AND output the result
```

That's it. Two tags. Here's how they work:

```erb
<h1>Hello, <%= @user.name %></h1>        <!-- Outputs: <h1>Hello, Jason</h1> -->

<% if @user.admin? %>                     <!-- Runs the if check, outputs nothing -->
  <span class="badge">Admin</span>
<% end %>
```

The `<% %>` tag is for control flow - loops, conditionals, setting variables. The `<%= %>` tag is for when you actually want something to appear in the HTML.

Common beginner mistake: using `<%= %>` for control flow. If you write `<%= if @user.admin? %>`, you'll get weird output because `if` statements return values in Ruby. Use the non-printing tag for logic.

## The Layout: Your Page's Outer Shell

Every page in a Rails app shares common elements: the HTML structure, the `<head>` section with stylesheets and JavaScript, maybe a header and footer. Instead of repeating this in every view, Rails has **layouts**.

Look at Campfire's main layout (`app/views/layouts/application.html.erb:1-69`):

```erb
<!DOCTYPE html>
<html>
  <head>
    <%= page_title_tag %>
    <meta name="viewport" content="width=device-width, initial-scale=1, ...">
    <%= csrf_meta_tags %>
    <%= stylesheet_link_tag :all, "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
    <%= yield :head %>
  </head>

  <body class="<%= body_classes %>">
    <nav id="nav">
      <%= yield :nav %>
    </nav>

    <!-- Flash messages handling here -->

    <main id="main-content">
      <%= yield %>

      <footer id="footer">
        <%= yield :footer %>
      </footer>
    </main>

    <aside id="sidebar">
      <%= yield :sidebar %>
    </aside>

    <%= render "layouts/lightbox" %>
  </body>
</html>
```

The magic word here is **`yield`**. Think of `yield` as a slot - a placeholder that says "put content here."

When you have just `<%= yield %>` (no arguments), that's the **main content slot**. Whatever your individual view template contains gets inserted there.

But look at those other yields: `yield :head`, `yield :nav`, `yield :footer`, `yield :sidebar`. These are **named slots**. They let individual pages inject content into specific parts of the layout.

### content_for: Filling the Slots

How do pages fill those named slots? With `content_for`. Look at `app/views/rooms/show.html.erb:1-24`:

```erb
<% @page_title = room_display_name(@room) %>
<% @body_class = "sidebar" %>
<% content_for :head do %>
  <%= turbo_exempts_page_from_preview %>
  <meta name="current-room-id" content="<%= @room.id %>">
<% end %>

<%= render "rooms/show/nav", room: @room %>

<% content_for :sidebar, sidebar_turbo_frame_tag(src: user_sidebar_path) %>

<%= message_area_tag(@room) do %>
  <!-- Main content here -->
<% end %>

<%= render "rooms/show/composer", room: @room %>
```

See how `content_for :head` adds a meta tag to the `<head>` section? And `content_for :sidebar` fills the sidebar slot? The view is saying "I want this specific stuff to appear in these specific places in the layout."

Here's the mental model:

```
┌─────────────────────────────────────────────────────┐
│  LAYOUT (application.html.erb)                      │
│  ┌─────────────────────────────────────────────┐   │
│  │  <head>                                      │   │
│  │    ... standard stuff ...                    │   │
│  │    [SLOT: :head] ◄─── content_for :head     │   │
│  │  </head>                                     │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌───────────┐  ┌──────────────────────────────┐   │
│  │   <nav>   │  │   <main>                      │   │
│  │  [SLOT:   │  │     [MAIN SLOT] ◄── view     │   │
│  │   :nav]   │  │                   contents   │   │
│  │           │  │   <footer>                    │   │
│  │           │  │     [SLOT: :footer]           │   │
│  │           │  │   </footer>                   │   │
│  └───────────┘  └──────────────────────────────┘   │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │   <aside>                                     │  │
│  │     [SLOT: :sidebar] ◄─── content_for        │  │
│  │                           :sidebar            │  │
│  │   </aside>                                    │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

Why is this brilliant? Because the room page doesn't need to know (or care) about the HTML structure of the entire application. It just says "here's what goes in the nav" and "here's what goes in the sidebar." The layout handles the structure; the view handles the content.

### Instance Variables Control the Layout Too

Notice those lines at the top of rooms/show.html.erb:

```erb
<% @page_title = room_display_name(@room) %>
<% @body_class = "sidebar" %>
```

These set instance variables that the layout uses. Look back at the layout:

```erb
<%= page_title_tag %>              <!-- Uses @page_title -->
<body class="<%= body_classes %>"> <!-- Uses @body_class -->
```

The helpers in `application_helper.rb:2-4` and `application_helper.rb:21-23` read these:

```ruby
def page_title_tag
  tag.title @page_title || "Campfire"
end

def body_classes
  [ @body_class, admin_body_class, account_logo_body_class ].compact.join(" ")
end
```

So each page can set its own title and add CSS classes to the body, all without touching the layout.

## Partials: Reusable Chunks of HTML

A partial is exactly what it sounds like: a partial view. A reusable chunk of HTML that you can render from anywhere.

The naming convention is crucial: **partials start with an underscore**. The file `_message.html.erb` is a partial. When you render it, you drop the underscore:

```erb
<%= render "messages/message" %>
```

Rails knows to look for `app/views/messages/_message.html.erb`.

### Passing Data to Partials

Partials need data. You pass it as local variables:

```erb
<%= render "rooms/show/nav", room: @room %>
```

Inside the partial `_nav.html.erb`, you use `room` (not `@room`) - it's a local variable, not an instance variable.

### The Message Partial: A Real Example

Look at Campfire's message partial (`app/views/messages/_message.html.erb:1-33`):

```erb
<% cache message do %>
  <%= message_tag message do %>
    <h2 class="message__day-separator"><%= local_datetime_tag message.created_at, style: :date %></h2>

    <figure class="avatar message__avatar">
      <%= avatar_tag message.creator %>
    </figure>

    <turbo-frame id="<%= dom_id(message, :edit) %>">
      <div class="message__body">
        <div class="message__body-content">
          <div class="message__meta">
            <h3 class="message__heading">
              <span class="message__author" title="<%= message.creator.title %>">
                <strong data-reply-target="author"><%= message.creator.name %></strong>
              </span>
              <%= link_to message_timestamp(message, class: "message__timestamp"),
                    room_at_message_path(message.room, message), target: "_top",
                    class: "message__permalink" %>
            </h3>
            <%= render "messages/actions", message: message, url: room_at_message_url(message.room, message) %>
          </div>
          <%= render "messages/presentation", message: message %>
          <%= render "messages/boosts/boosts", message: message %>
        </div>
      </div>
    </turbo-frame>
  <% end %>
<% end %>
```

Notice how partials nest other partials? The message partial renders:
- `messages/actions` - the action buttons (reply, copy link, edit)
- `messages/presentation` - the actual message content
- `messages/boosts/boosts` - the emoji reactions

This is the power of partials: compose complex UI from simple pieces. Each partial has one job. Change how actions look? Edit one file. Change how boosts display? Edit one file.

### content_for in Partials

Here's a pattern that blew my mind when I first saw it: partials can use `content_for` too!

Look at `app/views/rooms/show/_nav.html.erb:1-20`:

```erb
<% content_for :nav do %>
  <%= account_logo_tag if Current.account.logo.attached? %>

  <%= tag.span class: "btn btn--reversed btn--faux room--current" do %>
    <h1 class="room__contents txt-medium overflow-ellipsis">
      <%= room_display_name(room) %>
    </h1>
  <% end %>

  <%= link_to_edit_room(room) do %>
    <%= image_tag "menu-dots-horizontal.svg", size: 20, aria: { hidden: "true" } %>
    <span class="for-screen-reader">Settings</span>
  <% end %>

  <%= render "rooms/involvements/bell", room: room %>
<% end %>
```

Wait, the *partial* is filling a layout slot? Yes! When rooms/show.html.erb renders this partial, the partial's content ends up in the layout's `:nav` slot.

This is a brilliant organization pattern. The main view stays clean - it just says "render the nav" - and the nav partial owns its own content, including where it goes in the layout.

### Collection Rendering: The Power Move

Look at this line from rooms/show.html.erb:

```erb
<%= render partial: "messages/message", collection: @messages, cached: true %>
```

This single line renders a partial for **every message** in the collection. Rails automatically:
1. Loops through `@messages`
2. Renders `_message.html.erb` for each one
3. Passes each message as a local variable called `message` (singular of collection name)
4. Caches each rendered partial (the `cached: true` part)

Without collection rendering, you'd write:

```erb
<% @messages.each do |message| %>
  <%= render "messages/message", message: message %>
<% end %>
```

The collection syntax is cleaner AND faster (Rails optimizes it internally). DHH and the 37signals team use collection rendering everywhere. It's a sign of idiomatic Rails.

## View Helpers: Ruby Methods That Generate HTML

Some HTML is awkward to write in ERB. Conditional classes, complex tags, repeated patterns. That's where helpers come in.

Helpers are Ruby modules in `app/helpers/`. Methods defined there are available in all views.

### Example 1: Message Tags

Look at `app/helpers/messages_helper.rb:2-13`:

```ruby
def message_area_tag(room, &)
  tag.div id: "message-area", class: "message-area", contents: true, data: {
    controller: "messages presence drop-target",
    action: [ messages_actions, drop_target_actions, presence_actions ].join(" "),
    messages_first_of_day_class: "message--first-of-day",
    messages_formatted_class: "message--formatted",
    messages_me_class: "message--me",
    messages_mentioned_class: "message--mentioned",
    messages_threaded_class: "message--threaded",
    messages_page_url_value: room_messages_url(room)
  }, &
end
```

This helper creates a `<div>` with tons of data attributes for Stimulus controllers. Writing this directly in ERB would be a mess. Instead, the view just says:

```erb
<%= message_area_tag(@room) do %>
  <!-- content -->
<% end %>
```

Clean, readable, and all the complexity is hidden in Ruby where it's easier to maintain.

### Example 2: The Page Title

From `app/helpers/application_helper.rb:2-4`:

```ruby
def page_title_tag
  tag.title @page_title || "Campfire"
end
```

Simple, but powerful. Every page can set `@page_title` and get a proper title, with a sensible default if they don't.

### Example 3: Message Presentation

From `app/helpers/messages_helper.rb:53-67`:

```ruby
def message_presentation(message)
  case message.content_type
  when "attachment"
    message_attachment_presentation(message)
  when "sound"
    message_sound_presentation(message)
  else
    auto_link h(ContentFilters::TextMessagePresentationFilters.apply(message.body.body)),
              html: { target: "_blank" }
  end
end
```

This helper decides how to render a message based on its type. The partial just calls `<%= message_presentation(message) %>` and the helper figures out whether to show an image, play a sound, or render text.

### When to Extract a Helper

Good candidates for helpers:
- **Complex HTML with lots of attributes** (like message_area_tag)
- **Conditional logic** that would clutter the view
- **Reused across multiple views** (DRY principle)
- **String formatting** for display

Keep in views:
- Simple conditionals (`<% if @user.admin? %>`)
- Basic loops
- Straightforward HTML structure

## The Full Picture

Let's trace how a room page renders in Campfire:

1. **RoomsController#show** sets `@room` and `@messages`
2. Rails looks for `app/views/rooms/show.html.erb`
3. The view sets `@page_title` and `@body_class`
4. It uses `content_for :head` to add meta tags
5. It renders the `rooms/show/nav` partial, which uses `content_for :nav`
6. It renders the `rooms/show/composer` partial, which uses `content_for :footer`
7. It uses `content_for :sidebar` for the sidebar
8. The main content (messages) fills the default `yield`
9. Messages render via collection rendering, each using the `_message` partial
10. Each message partial renders sub-partials for actions, presentation, boosts
11. The layout (`application.html.erb`) wraps everything, with each `yield` slot filled

```
Controller → View → Layout
                ↓
            ┌───────────────────────────────────────┐
            │ content_for :head    → yield :head   │
            │ content_for :nav     → yield :nav    │
            │ content_for :sidebar → yield :sidebar│
            │ content_for :footer  → yield :footer │
            │ (main content)       → yield         │
            └───────────────────────────────────────┘
                        ↓
            ┌───────────────────────────────────────┐
            │ Partials:                             │
            │  └─ rooms/show/nav                    │
            │  └─ rooms/show/composer               │
            │  └─ messages/message (×N)             │
            │       └─ messages/actions             │
            │       └─ messages/presentation        │
            │       └─ messages/boosts/boosts       │
            └───────────────────────────────────────┘
```

## Try It Yourself

Find the `_boost.html.erb` partial in `app/views/messages/boosts/`. Read it and trace:

1. What data does it expect? (What local variables?)
2. Where is it rendered from?
3. Does it use any helpers?
4. How would you add a new piece of data to display (say, a timestamp for when the boost was added)?

Then, look at how Campfire handles the search results page (`app/views/searches/index.html.erb`). Notice how it reuses the message partial? That's the payoff of good partial design - the same component works in totally different contexts.

## Summary

- **ERB** is just HTML with Ruby sprinkled in using `<% %>` and `<%= %>`
- **Layouts** provide the common structure; views fill in the content
- **yield** creates slots in layouts; **content_for** fills them from views (or partials!)
- **Partials** (files starting with `_`) are reusable chunks of HTML
- **Collection rendering** (`render partial:, collection:`) is the idiomatic way to render lists
- **Helpers** extract complex HTML generation into testable Ruby methods
- Instance variables like `@page_title` let views communicate with layouts
- **Partials can use content_for** - this lets them place content anywhere in the layout

The 37signals approach: compose complex pages from simple, focused partials. Each partial does one thing. Helpers handle the messy HTML generation. The result is code that's easy to change because each piece has one job.

---

## Q&A

### Q: How does the application layout get rendered? Which controller renders it?

**A:** No controller explicitly renders the layout - Rails does it automatically via convention over configuration.

**The lookup order:**
1. Controller-specific layout (`layouts/messages.html.erb` for MessagesController)
2. Parent controller's layout (inheritance chain)
3. `layouts/application.html.erb` (the default)

**Overriding options:**
- `layout "admin"` - use a specific layout
- `layout false` - no layout (for HTML fragments)
- `layout false, only: :index` - conditional by action
- `layout :method_name` - dynamic layout based on logic

**When to use multiple layouts:**
- Admin vs user sections (different nav/chrome)
- Public marketing pages vs authenticated app
- Embedded/widget mode (stripped down)
- Email layouts (ActionMailer)
- Turbo/API responses (no layout needed)

**Key insight:** Layouts render *after* views, which is why `content_for` works - by the time the layout processes, all content blocks have been captured.

Campfire example: `MessagesController` uses `layout false, only: :index` because that action returns just message HTML fragments for infinite scroll - no need for the full page shell.

### Q: Wait, layouts render AFTER views? Shouldn't the outer shell render first?

**A:** You're thinking about it visually (outer shell → inner content), which makes sense! But Rails thinks about it **procedurally** (what needs to happen first for the next thing to work).

**The problem:** How can the layout know what to put in `yield :nav` if the view hasn't run yet to capture the `content_for :nav` block? It can't.

**The Letter Analogy:**

Think of it like writing a letter, not building a house:

1. You write the letter content (the view renders)
2. You put it in an envelope (the layout wraps it)
3. The envelope has windows showing the address (yield slots showing content_for)

The envelope doesn't exist until you have something to put in it.

**The actual sequence:**
```
Controller runs
    ↓
View renders (rooms/show.html.erb)
    ↓
During view rendering:
  - Main content captured
  - content_for :nav captured
  - content_for :head captured
  - content_for :sidebar captured
    ↓
Layout renders (application.html.erb)
    ↓
Layout calls yield → inserts captured content
Layout calls yield :nav → inserts nav content
    ↓
Final HTML assembled and sent
```

**Why this is clever:** If layouts rendered first, they'd need to somehow pause, wait for the view to render each section, then continue. Messy. Instead, Rails says: "View, you go first. Capture everything. When you're done, I'll slot it all into the right places in the layout."

It's like a film editor - you don't shoot scenes in movie order. You shoot all scenes (view captures content), then the editor assembles them into the final structure (layout places content in slots).

The "outer shell" metaphor is about the final HTML structure, not the rendering order.

## Quiz History

[Quiz sessions will be recorded here after the learner is quizzed on this topic]
