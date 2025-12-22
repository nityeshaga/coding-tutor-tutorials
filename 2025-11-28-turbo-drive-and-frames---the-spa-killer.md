---
concepts: [Turbo Drive, Turbo Frames, Hotwire, SPA-like Navigation, Lazy Loading]
source_repo: once-campfire
description: Introduction to Hotwire's Turbo - the technology that lets Rails apps feel like SPAs without writing JavaScript frameworks. Covers how Turbo Drive automatically makes navigation fast, and how Turbo Frames enable surgical updates to parts of a page. All demonstrated through real Campfire examples including message editing, reactions, and the sidebar.
understanding_score: null
last_quizzed: null
prerequisites: [references/tutorials/2025-11-28-views,-partials-and-layouts---the-art-of-rendering-html.md]
created: 28-11-2025
last_updated: 28-11-2025  # Q&A: server vs client, dom_id construction
---

# Turbo Drive and Frames - The SPA Killer

Open Campfire in your browser. Click between rooms. Notice something? The page doesn't do that classic "flash white and reload" thing. It feels... smooth. Like a React app. Now click the edit button on a message. The message transforms into an editor, but the rest of the page stays put. Cancel, and it snaps back.

This isn't JavaScript framework magic. There's no React. No Vue. No 50MB of node_modules. This is **Turbo** - and it's the technology that DHH and 37signals believe will kill the SPA as we know it.

Here's the radical idea: what if you could get 90% of the SPA experience while keeping 100% of your logic on the server, in Ruby? That's what Turbo delivers.

## The Problem: The SPA Tax

Let's rewind to understand why this matters.

In traditional web apps, every link click triggers a full page reload:

```
User clicks link
    ↓
Browser throws away EVERYTHING
    ↓
Request goes to server
    ↓
Server renders ENTIRE page
    ↓
Browser downloads EVERYTHING again
    ↓
CSS re-parses, JS re-executes
    ↓
Page finally appears (300-500ms later)
```

This felt slow. So the industry invented SPAs (Single Page Applications):

```
User clicks link
    ↓
JavaScript intercepts click
    ↓
JS fetches just the data needed (JSON)
    ↓
JS updates just the part of the DOM that changed
    ↓
Page updates instantly (~50ms)
```

Fast! But here's what you pay for that speed:

1. **Duplicate logic** - Your routing, validation, and business logic now lives in both Ruby AND JavaScript
2. **API layer** - You need to build JSON APIs for everything
3. **State management** - Redux, Vuex, or some other state nightmare
4. **Build complexity** - Webpack, Babel, TypeScript configs
5. **Two mental models** - Backend thinking + frontend thinking
6. **Hiring** - Need developers fluent in both worlds

DHH looked at this and said: "This is insane. We're paying an enormous complexity tax for... faster navigation?"

## Turbo Drive: Automatic SPA-Like Navigation

Turbo's first trick is **Drive**. It intercepts every link click and form submission, fetches the new page in the background, and swaps just the `<body>` content. The `<head>` stays put (your CSS and JS don't re-parse).

Here's the beautiful part: **you don't write any code**. Just add Turbo to your app, and navigation becomes instant.

Look at `app/javascript/application.js`:

```javascript
import "@hotwired/turbo-rails"
```

That's it. One import. Every link in Campfire now works like an SPA.

When you click a link:

```
1. Turbo intercepts the click
2. Fetches the new page via AJAX
3. Extracts the <body> from the response
4. Replaces the current <body> with the new one
5. Updates the URL in the browser history
6. Merges any new <head> elements
```

The page appears to load instantly because:
- CSS doesn't re-parse (already loaded)
- JavaScript doesn't re-execute (already running)
- Browser doesn't throw away the DOM and rebuild (just swaps the body)

### The Prefetch Bonus

Look at line 17 of `app/views/layouts/application.html.erb`:

```erb
<%= tag.meta name: "turbo-prefetch", content: "true" %>
```

This tells Turbo to prefetch links when you hover over them. By the time you click, the page is already loaded. It's feels like the future.

## Turbo Frames: Surgical Updates

Drive makes navigation fast. But what about updating just *part* of a page?

This is where **Turbo Frames** come in. Think of them as iframes that don't suck.

### The Mental Model

A Turbo Frame is a section of your page with an ID. When a link or form inside that frame triggers a request, Turbo:

1. Makes the request
2. Finds a frame with the **same ID** in the response
3. Replaces **only that frame** on the page
4. Everything else stays exactly as it was

```
┌─────────────────────────────────────────┐
│  Page Header                            │
├─────────────────────────────────────────┤
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  TURBO FRAME id="message_42_edit" │  │
│  │                                   │  │
│  │  This part can update             │  │
│  │  independently!                   │  │
│  │                                   │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  TURBO FRAME id="message_43_edit" │  │
│  │  So can this one!                 │  │
│  └───────────────────────────────────┘  │
│                                         │
│  Footer                                 │
└─────────────────────────────────────────┘
```

### Example 1: Message Editing

This is the clearest example in Campfire. Open `app/views/messages/_message.html.erb`:

```erb
<turbo-frame id="<%= dom_id(message, :edit) %>">
  <div class="message__body">
    <div class="message__body-content">
      <!-- Message content displays here -->
      <%= render "messages/presentation", message: message %>
      <%= render "messages/boosts/boosts", message: message %>
    </div>
  </div>
</turbo-frame>
```

Each message is wrapped in a Turbo Frame with a unique ID like `edit_message_42`.

Now look at `app/views/messages/edit.html.erb`:

```erb
<turbo-frame id="<%= dom_id(@message, :edit) %>">
  <div class="message__body position-relative">
    <div class="message__body-content message__body-content--editing gap">
      <!-- Edit form with rich text editor -->
      <%= form_with model: @message, url: room_message_path(@room, @message) do |form| %>
        <%= form.rich_text_area :body %>
        <!-- Save and delete buttons -->
      <% end %>
    </div>
  </div>
</turbo-frame>
```

Same ID. When you click "Edit" on a message:

1. Browser requests `GET /rooms/1/messages/42/edit`
2. Rails renders the edit view
3. Turbo finds `<turbo-frame id="edit_message_42">` in the response
4. Turbo replaces *just that frame* on the current page
5. The message transforms into an editor, everything else untouched

When you cancel or save, the same magic happens in reverse - the frame gets replaced with the display view.

**No page reload. No JavaScript state management. No DOM manipulation code.** Just matching IDs.

### Example 2: The Boost (Reaction) System

Campfire's "boost" feature (those little reaction emoji) is a beautiful example of nested frames. Look at `app/views/messages/boosts/_boosts.html.erb`:

```erb
<%= turbo_frame_tag message, :boosting do %>
  <div class="boosts flex flex-wrap align-center gap">
    <!-- Existing boosts render here -->
    <div id="<%= dom_id(message, :boosts) %>">
      <%= render partial: "messages/boosts/boost", collection: message.boosts.ordered %>
    </div>

    <!-- The "Add boost" button is ALSO a frame -->
    <%= turbo_frame_tag message, :new_boost do %>
      <%= link_to new_message_boost_path(message), class: "boost__action" do %>
        <%= image_tag "boost.svg" %>
      <% end %>
    <% end %>
  </div>
<% end %>
```

There are two nested frames here:
- `boosting_message_42` - The outer container for all boost UI
- `new_boost_message_42` - Just the "add boost" button

When you click the add button, only the inner `new_boost` frame gets replaced with the input form from `app/views/messages/boosts/new.html.erb`:

```erb
<%= turbo_frame_tag dom_id(@message, :new_boost) do %>
  <div class="boost flex-inline">
    <%= form_with model: [ @message, Boost.new ] do |form| %>
      <%= form.text_field :content, autofocus: true, maxlength: 16 %>
      <!-- Submit and cancel buttons -->
    <% end %>
  </div>
<% end %>
```

The button becomes an input. Type your emoji, submit, and it snaps back to a button while the new boost appears in the list.

This is surgical UI updates without writing a single line of JavaScript.

### Example 3: The Persistent Sidebar

Here's a clever pattern. Look at `app/helpers/users/sidebar_helper.rb`:

```ruby
def sidebar_turbo_frame_tag(src: nil, &)
  turbo_frame_tag :user_sidebar, src: src, target: "_top", data: {
    turbo_permanent: true,
    # ... controllers and actions
  }, &
end
```

Two important things here:

**`data: { turbo_permanent: true }`** - This tells Turbo: "Never replace this frame during Drive navigation." The sidebar persists across page loads. When you click from Room A to Room B, the sidebar doesn't flicker or re-render. It just... stays.

**`src: user_sidebar_path`** - This is lazy loading. The sidebar frame doesn't have content initially - it has a URL. Turbo automatically fetches that URL and fills in the frame. This means:
- The main page loads instantly
- The sidebar loads in parallel
- If sidebar loading is slow, it doesn't block the main content

Look at how it's used in `app/views/rooms/show.html.erb`:

```erb
<% content_for :sidebar, sidebar_turbo_frame_tag(src: user_sidebar_path) %>
```

And the welcome page uses the exact same pattern:

```erb
<% content_for :sidebar, sidebar_turbo_frame_tag(src: user_sidebar_path) %>
```

One sidebar component, lazy-loaded, persistent across navigation. Zero JavaScript.

### Example 4: Direct Message Creation

Open `app/views/rooms/directs/new.html.erb`:

```erb
<turbo-frame id="direct_rooms_control" target="_top">
  <div class="directs directs--new flex flex-column gap">
    <%= form_with model: @room do |form| %>
      <!-- User selection autocomplete -->
      <!-- Cancel link -->
      <%= link_to user_sidebar_path, data: { turbo_frame: "user_sidebar" } %>
    <% end %>
  </div>
</turbo-frame>
```

Notice `target="_top"` - this tells Turbo: "When a form in this frame submits successfully, break out of the frame and do a full Drive navigation."

The user selection happens in the frame (surgical update), but once you create the DM, you navigate to the new room (full page swap).

Also notice the cancel link: `data: { turbo_frame: "user_sidebar" }`. This overrides the default frame targeting. Clicking cancel doesn't replace `direct_rooms_control` - it replaces the `user_sidebar` frame instead, effectively refreshing the sidebar back to its normal state.

## The Frame Targeting Rules

Understanding how Turbo decides which frame to update:

1. **Default**: Links/forms inside a frame target that same frame
2. **`target="_top"`**: Break out and do a full Drive navigation
3. **`data-turbo-frame="some_id"`**: Target a specific frame by ID
4. **`target="_parent"`**: Target the parent frame (for nested frames)

```erb
<!-- This targets its containing frame -->
<%= link_to "Edit", edit_path %>

<!-- This does full page navigation -->
<%= link_to "Home", root_path, target: "_top" %>

<!-- This targets a specific frame elsewhere on the page -->
<%= link_to "Refresh sidebar", sidebar_path, data: { turbo_frame: "sidebar" } %>
```

## Opting Out

Sometimes you don't want Turbo's magic. Maybe a link goes to an external site, or you want a true full reload.

```erb
<!-- Disable Turbo for this link -->
<%= link_to "External", "https://google.com", data: { turbo: false } %>

<!-- Disable Turbo for this form -->
<%= form_with url: path, data: { turbo: false } do |f| %>
```

In the layout, line 23 shows another pattern:

```erb
<%= stylesheet_link_tag :all, "data-turbo-track": "reload" %>
```

`turbo-track="reload"` means: "If this asset changes, do a full page reload." This ensures that when you deploy new CSS, users don't see stale styles.

## Try It Yourself

Here's an exercise to cement this:

1. Open `app/views/users/profiles/_membership.html.erb` and find the Turbo Frame
2. Trace the flow: what happens when a user updates their notification settings?
3. Find the corresponding controller action and view that provides the replacement content
4. Now look at `app/views/rooms/involvements/_bell.html.erb` - it uses a similar pattern

Can you explain how the notification bell updates without a page reload?

## Summary

- **Turbo Drive** gives you SPA-like navigation for free - just import it and every link becomes fast
- **Turbo Frames** let you update parts of a page by matching frame IDs between request and response
- The pattern is: wrap content in `<turbo-frame id="something">`, make requests that return frames with the same ID, Turbo handles the swap
- Use `target="_top"` to break out of frames for full navigation
- Use `data-turbo-frame="id"` to target a specific frame from anywhere
- Use `data: { turbo_permanent: true }` for elements that should persist across navigations
- Use `src: url` for lazy-loading frame content

The philosophy: **Keep your logic on the server. Let HTML do the work. Write less JavaScript.**

This is why DHH calls it the "SPA killer." You get the UX of a modern app while keeping the simplicity of traditional Rails.

---

## Q&A

### Q: Where does Turbo actually run? Server or client?

**Question:** In the Turbo Frames example, when clicking the edit button on a message, you said Rails renders the edit view. So where is Turbo running to find the frame ID and replace that particular frame? Is that happening on the server or client side? If it's client-side, the server is sending the full HTML payload and extraction happens on the client - is that really making it faster?

**Answer:** Turbo runs entirely in the browser (client-side JavaScript). Here's the actual flow:

```
1. You click "Edit" inside a Turbo Frame
2. Turbo (JS in browser) intercepts the click
3. Turbo makes a fetch() request to /rooms/1/messages/42/edit
4. Server renders HTML and sends it back
5. Turbo (JS in browser) parses the response HTML
6. Turbo finds <turbo-frame id="edit_message_42"> in the response
7. Turbo extracts that frame and swaps it into the DOM
8. Everything else on the page stays untouched
```

**Why it's still faster even with full HTML responses:**

| What Happens | Full Page Reload | Turbo Frame |
|--------------|-----------------|-------------|
| Server renders HTML | Yes | Yes |
| Network transfers HTML | Yes | Yes |
| Browser parses HTML | Full page | Just to find frame |
| Browser tears down DOM | Everything | Nothing |
| Browser builds new DOM | Everything | Just the frame |
| CSS recalculates | Everything | Just the frame |
| JS reinitializes | All scripts | Nothing |
| Images reload | All of them | None |
| Scroll position | Lost | Preserved |
| Input focus | Lost | Can be preserved |

The expensive part isn't the network - it's the browser rebuilding the entire DOM and re-running all JavaScript.

**Optional optimization:** When Turbo makes a request from inside a frame, it sends a `Turbo-Frame: edit_message_42` header. Rails provides a `turbo_frame_request?` helper to detect this:

```ruby
# In controller - skip layout for frame requests
def edit
  render layout: false if turbo_frame_request?
end
```

Or in views:
```erb
<% if turbo_frame_request? %>
  <!-- Just the frame content -->
<% else %>
  <!-- Full page with navigation etc -->
<% end %>
```

**But Campfire doesn't bother with this optimization.** They render full pages and let client-side Turbo extract the frame. For their use case, the full HTML response is small enough that the optimization isn't worth the added complexity.

This is very DHH - **don't optimize until you need to**. The simple approach works. Ship it.

### Q: How are Turbo Frame IDs constructed?

**Question:** In the examples, Turbo Frame IDs are being constructed in different ways - like `turbo_frame_tag message, :new_boost` translating to `new_boost_message_42`, or `turbo_frame_tag dom_id(@message, :new_boost)` giving the same result. For Turbo Frames to work, the IDs have to match exactly. How are these IDs actually constructed?

**Answer:** This is about Rails' `dom_id` helper - a convention that's easy to miss but crucial to understand.

#### The `dom_id` Helper

Rails has a helper called `dom_id` that generates unique, predictable HTML IDs for ActiveRecord objects. It follows a simple formula:

```ruby
dom_id(record)           # → "#{model_name}_#{id}"
dom_id(record, prefix)   # → "#{prefix}_#{model_name}_#{id}"
```

So for a `Message` with `id: 42`:

```ruby
dom_id(message)              # → "message_42"
dom_id(message, :edit)       # → "edit_message_42"
dom_id(message, :new_boost)  # → "new_boost_message_42"
dom_id(message, :boosting)   # → "boosting_message_42"
```

#### How `turbo_frame_tag` Uses It

The key insight: **`turbo_frame_tag` is smart**. It accepts multiple argument patterns and figures out what you mean:

```ruby
# Pattern 1: Just a symbol → uses the symbol as the ID
turbo_frame_tag :user_sidebar
# → <turbo-frame id="user_sidebar">

# Pattern 2: A record → calls dom_id(record)
turbo_frame_tag message
# → <turbo-frame id="message_42">

# Pattern 3: A record + prefix → calls dom_id(record, prefix)
turbo_frame_tag message, :new_boost
# → <turbo-frame id="new_boost_message_42">

# Pattern 4: Explicit dom_id call → same result as Pattern 3
turbo_frame_tag dom_id(message, :new_boost)
# → <turbo-frame id="new_boost_message_42">
```

**Patterns 3 and 4 are identical!** `turbo_frame_tag` internally calls `dom_id` when you pass a record, so you don't need to do it yourself.

#### Why This Matters

The whole system works because IDs are **predictable and consistent**. On the display page:

```erb
# app/views/messages/_message.html.erb
<turbo-frame id="<%= dom_id(message, :edit) %>">  <!-- "edit_message_42" -->
```

On the edit page:

```erb
# app/views/messages/edit.html.erb
<turbo-frame id="<%= dom_id(@message, :edit) %>">  <!-- "edit_message_42" -->
```

Same message object → same ID → Turbo knows what to swap.

#### Real Examples from Campfire

Here's how IDs are constructed across the codebase:

| Code | Generated ID | Used For |
|------|-------------|----------|
| `dom_id(message)` | `message_42` | The message element itself |
| `dom_id(message, :edit)` | `edit_message_42` | Message editing frame |
| `dom_id(message, :boosting)` | `boosting_message_42` | Container for all boost UI |
| `dom_id(message, :new_boost)` | `new_boost_message_42` | The "add boost" button/form |
| `dom_id(message, :boosts)` | `boosts_message_42` | List of existing boosts |
| `dom_id(message, :presentation)` | `presentation_message_42` | Message content display |
| `dom_id(message, :delete_form)` | `delete_form_message_42` | Hidden delete form |
| `dom_id(room, :involvement)` | `involvement_room_5` | Notification bell for a room |
| `dom_id(room, :messages)` | `messages_room_5` | Container for room's messages |
| `dom_id(room, :list)` | `list_room_5` | Room in sidebar list |
| `dom_id(boost)` | `boost_123` | Individual boost element |

#### The Convention

The prefix describes *what role* this element plays:
- `:edit` - for editing UI
- `:new_something` - for creation forms
- `:list` - for list item representations
- `:presentation` - for display-only content
- `:form`, `:delete_form` - for form elements

This lets you have multiple DOM elements for the same record, each with a distinct purpose.

#### For New Records

What about records that haven't been saved yet (no ID)?

```ruby
message = Message.new  # id is nil

dom_id(message)        # → "new_message"
dom_id(message, :form) # → "form_new_message"
```

Rails handles this gracefully with the `new_` prefix.

## Quiz History

[Quiz sessions will be recorded here after the learner is quizzed on this topic]
