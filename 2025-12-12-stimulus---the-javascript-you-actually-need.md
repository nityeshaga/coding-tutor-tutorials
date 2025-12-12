---
concepts: [Stimulus, Controllers, Targets, Actions, Values, Classes]
source_repo: once-campfire
description: The final piece of Hotwire - Stimulus is a minimal JavaScript framework for the JavaScript you actually need to write. Covers the HTML-first philosophy, controllers, targets, actions, values, and classes. All demonstrated through real Campfire examples showing how 37signals adds interactivity without framework complexity.
understanding_score: null
last_quizzed: null
prerequisites: [2025-12-12-turbo-streams---real-time-without-the-complexity.md]
created: 12-12-2025
last_updated: 12-12-2025  # Q&A: local_time_controller and targetConnected callbacks
---

# Stimulus - The JavaScript You Actually Need

You've learned Turbo Drive (instant navigation), Turbo Frames (surgical updates), and Turbo Streams (real-time pushes). Together, they eliminate 90% of the JavaScript you'd write in a traditional SPA.

But what about the other 10%?

- Toggle a menu open/closed
- Preview an image before upload
- Copy text to clipboard
- Auto-dismiss a flash message after an animation
- Show a popup when clicking a button

This is **behavior** that happens purely in the browser - no server needed. You can't avoid JavaScript here. But you can avoid the complexity of React, Vue, or sprawling jQuery files.

Enter **Stimulus**.

## The Philosophy: HTML-First JavaScript

Stimulus flips the traditional JavaScript model on its head.

**Traditional JS approach:**
```javascript
// Find elements by CSS selectors
const buttons = document.querySelectorAll('.toggle-button')
const sidebar = document.getElementById('sidebar')

// Attach event listeners
buttons.forEach(btn => {
  btn.addEventListener('click', () => {
    sidebar.classList.toggle('open')
  })
})
```

Problems:
- JS needs to know about HTML structure (fragile)
- Hard to see what JS is attached to which HTML
- Event listeners can pile up, cause memory leaks
- Doesn't play well with Turbo (DOM changes break listeners)

**Stimulus approach:**
```html
<aside data-controller="toggle-class" data-toggle-class-toggle-class="open">
  ...
</aside>

<button data-action="toggle-class#toggle">Menu</button>
```

```javascript
// toggle_class_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static classes = [ "toggle" ]

  toggle() {
    this.element.classList.toggle(this.toggleClass)
  }
}
```

The HTML **declares** its behavior. The JavaScript **defines** what that behavior means.

Look at the HTML - you can immediately see:
- This element has a `toggle-class` controller
- It has a `toggle` class configured as `open`
- The button calls `toggle-class#toggle` when clicked

**The HTML is the source of truth.** JavaScript just provides the implementation.

## The Building Blocks

Stimulus has five core concepts:

| Concept | What it is | HTML attribute |
|---------|-----------|----------------|
| **Controller** | A JavaScript class that adds behavior | `data-controller="name"` |
| **Action** | An event that triggers a method | `data-action="event->controller#method"` |
| **Target** | An element the controller needs to reference | `data-controller-target="name"` |
| **Value** | Data passed from HTML to JS | `data-controller-name-value="..."` |
| **Class** | CSS class names passed from HTML to JS | `data-controller-name-class="..."` |

Let's see each one through real Campfire code.

## Example 1: The Simplest Controller

Look at `app/javascript/controllers/element_removal_controller.js`:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  remove() {
    this.element.remove()
  }
}
```

That's the entire controller. 7 lines. It has one method: `remove()`, which removes the element from the DOM.

Now look at how it's used in `app/views/layouts/application.html.erb`:

```erb
<div class="flash"
     data-controller="element-removal"
     data-action="animationend->element-removal#remove">
```

Breaking this down:

- `data-controller="element-removal"` - Attach the controller to this element
- `data-action="animationend->element-removal#remove"` - When the `animationend` event fires, call the `remove` method

**What happens:** Flash message appears, CSS animation plays, animation ends, element removes itself. No setTimeout, no manual cleanup. Beautiful.

### The Naming Convention

Notice the file is `element_removal_controller.js` but we reference it as `element-removal`.

Stimulus auto-converts:
- `element_removal_controller.js` → `element-removal` (underscores become hyphens, `_controller.js` is stripped)
- `toggle_class_controller.js` → `toggle-class`
- `upload_preview_controller.js` → `upload-preview`

This is **convention over configuration** - name your files right, and Stimulus finds them automatically.

## Example 2: Classes (Configurable CSS)

Look at `app/javascript/controllers/toggle_class_controller.js`:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static classes = [ "toggle" ]

  toggle() {
    this.element.classList.toggle(this.toggleClass)
  }
}
```

`static classes = [ "toggle" ]` declares that this controller expects a class called "toggle" to be configured in HTML.

In the HTML (`app/views/layouts/application.html.erb`):

```erb
<aside id="sidebar"
       data-controller="toggle-class"
       data-toggle-class-toggle-class="open">
```

- `data-toggle-class-toggle-class="open"` passes the CSS class name `open` to the controller
- In the JS, `this.toggleClass` returns `"open"`

**Why this pattern?** The controller is reusable. You could use it anywhere:

```html
<!-- Toggle 'open' class -->
<div data-controller="toggle-class" data-toggle-class-toggle-class="open">

<!-- Toggle 'expanded' class on something else -->
<div data-controller="toggle-class" data-toggle-class-toggle-class="expanded">

<!-- Toggle 'hidden' class -->
<div data-controller="toggle-class" data-toggle-class-toggle-class="hidden">
```

Same controller, different CSS classes. The HTML configures, the JS implements.

## Example 3: Targets (Finding Elements)

Look at `app/javascript/controllers/upload_preview_controller.js`:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = [ "image", "input" ]

  previewImage() {
    const file = this.inputTarget.files[0]

    if (file) {
      this.imageTarget.src = URL.createObjectURL(file)
      this.imageTarget.onload = () => { URL.revokeObjectURL(this.imageTarget.src) }
    }
  }
}
```

`static targets = [ "image", "input" ]` declares that this controller needs to find two elements.

In the JS:
- `this.inputTarget` - The file input element
- `this.imageTarget` - The image preview element

In the HTML (`app/views/users/profiles/show.html.erb`):

```erb
<div data-controller="upload-preview">
  <label>
    <%= form.file_field :avatar,
          data: {
            upload_preview_target: "input",
            action: "upload-preview#previewImage"
          } %>
  </label>

  <%= image_tag user_avatar,
        data: { upload_preview_target: "image" } %>
</div>
```

- `data-upload-preview-target="input"` marks this as the `input` target
- `data-upload-preview-target="image"` marks this as the `image` target
- `data-action="upload-preview#previewImage"` calls the method when file changes

**What happens:** User selects a file → `previewImage()` runs → reads the file → updates the image src → preview appears instantly, before upload.

### Target Naming Pattern

The pattern for target attributes:
```
data-[controller-name]-target="[target-name]"
```

So for `upload-preview` controller with `image` target:
```
data-upload-preview-target="image"
```

## Example 4: Values (Passing Data)

Look at `app/javascript/controllers/copy_to_clipboard_controller.js`:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { content: String }
  static classes = [ "success" ]

  async copy(event) {
    event.preventDefault()
    this.reset()

    try {
      await navigator.clipboard.writeText(this.contentValue)
      this.element.classList.add(this.successClass)
    } catch {}
  }

  reset() {
    this.element.classList.remove(this.successClass)
  }
}
```

`static values = { content: String }` declares a value with its type.

In the JS:
- `this.contentValue` returns the string passed from HTML

In HTML:
```erb
<button data-controller="copy-to-clipboard"
        data-copy-to-clipboard-content-value="https://example.com/invite/abc123"
        data-copy-to-clipboard-success-class="copied"
        data-action="copy-to-clipboard#copy">
  Copy Link
</button>
```

- `data-copy-to-clipboard-content-value="..."` passes the text to copy
- `data-copy-to-clipboard-success-class="copied"` passes the success CSS class
- Click → copies to clipboard → adds "copied" class for visual feedback

### Value Types

Values can have types:
```javascript
static values = {
  content: String,
  count: Number,
  enabled: Boolean,
  config: Object,
  items: Array
}
```

Stimulus automatically converts the HTML string to the right type.

## Example 5: Actions (Event Binding)

The action syntax is: `event->controller#method`

```html
<!-- Click event (default for buttons) -->
<button data-action="toggle-class#toggle">

<!-- Explicit click event -->
<button data-action="click->toggle-class#toggle">

<!-- Change event (for inputs) -->
<input data-action="change->form#submit">

<!-- Keydown with modifier -->
<input data-action="keydown.esc->form#cancel">

<!-- Animation end event -->
<div data-action="animationend->element-removal#remove">

<!-- Multiple actions -->
<input data-action="change->form#submit keydown.esc->form#cancel">
```

### Shorthand

For common elements, you can omit the event:
- `<button data-action="controller#method">` → assumes `click`
- `<input data-action="controller#method">` → assumes `input`
- `<form data-action="controller#method">` → assumes `submit`

## Example 6: Form Controller (Putting It Together)

Look at `app/javascript/controllers/form_controller.js`:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = [ "cancel" ]

  submit() {
    this.element.requestSubmit()
  }

  cancel() {
    this.cancelTarget?.click()
  }

  preventAttachment(event) {
    event.preventDefault()
  }
}
```

This controller:
- Has a `cancel` target (optional, note the `?.`)
- `submit()` programmatically submits the form
- `cancel()` clicks the cancel button/link
- `preventAttachment()` prevents file attachments

Used in `app/views/messages/edit.html.erb`:

```erb
<%= form_with model: @message, data: {
      controller: "form",
      action: "trix-file-accept->form#preventAttachment keydown.esc->form#cancel keydown.ctrl+enter->form#submit:prevent keydown.meta+enter->form#submit:prevent"
    } do |form| %>

  <%= form.rich_text_area :body %>

  <%= link_to "Cancel", room_message_path(@room, @message),
        data: { form_target: "cancel" },
        hidden: true %>
<% end %>
```

This enables:
- `Ctrl+Enter` or `Cmd+Enter` → submit the form
- `Escape` → cancel (clicks the hidden cancel link)
- Drag a file onto the editor → prevented

All through declarative HTML attributes.

## The Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                          HTML                                    │
│                                                                  │
│  data-controller="..."     ← Attach controller to element       │
│  data-action="..."         ← Bind events to methods             │
│  data-[ctrl]-target="..."  ← Mark elements for JS to find       │
│  data-[ctrl]-[name]-value  ← Pass data to controller            │
│  data-[ctrl]-[name]-class  ← Pass CSS class names               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Controller (JS)                             │
│                                                                  │
│  static targets = [...]    ← Declare what elements we need      │
│  static values = {...}     ← Declare what data we expect        │
│  static classes = [...]    ← Declare what CSS classes we need   │
│                                                                  │
│  this.element              ← The element with data-controller   │
│  this.fooTarget            ← Element with target="foo"          │
│  this.barValue             ← Value from bar-value attribute     │
│  this.bazClass             ← Class from baz-class attribute     │
│                                                                  │
│  methodName() { ... }      ← Called by data-action              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## How Campfire Organizes Controllers

Campfire has 36 Stimulus controllers:

```
app/javascript/controllers/
├── auto_submit_controller.js      # Submit form on input change
├── autocomplete_controller.js     # User mention autocomplete
├── badge_dot_controller.js        # Unread indicators
├── boost_delete_controller.js     # Reaction deletion
├── composer_controller.js         # Message composer
├── copy_to_clipboard_controller.js
├── drop_target_controller.js      # File drag-and-drop
├── element_removal_controller.js  # Remove element from DOM
├── filter_controller.js           # Search filtering
├── form_controller.js             # Form keyboard shortcuts
├── lightbox_controller.js         # Image lightbox
├── local_time_controller.js       # Time formatting
├── maintain_scroll_controller.js  # Preserve scroll position
├── messages_controller.js         # Message list behavior
├── notifications_controller.js    # Push notification permissions
├── popup_controller.js            # Popup menus
├── toggle_class_controller.js     # Toggle CSS classes
├── upload_preview_controller.js   # Image preview before upload
└── ... (18 more)
```

Each controller is:
- **Small** - Usually under 50 lines
- **Focused** - Does one thing well
- **Reusable** - Can be attached to any element
- **Declarative** - HTML shows what it does

## When to Use Stimulus vs Turbo

| Behavior | Use |
|----------|-----|
| Update content from server | Turbo Frames / Streams |
| Toggle visibility / classes | Stimulus |
| Form submission | Turbo (automatic) |
| Keyboard shortcuts | Stimulus |
| Preview before upload | Stimulus |
| Copy to clipboard | Stimulus |
| Real-time updates from others | Turbo Streams |
| Drag and drop | Stimulus |
| Animations | CSS + Stimulus for triggers |

**Rule of thumb:** If it needs server data → Turbo. If it's pure browser behavior → Stimulus.

## Try It Yourself

1. Open `app/views/layouts/application.html.erb`
2. Find the flash message with `data-controller="element-removal"`
3. Trace the flow: What event triggers what method?

4. Now open `app/views/users/sidebars/show.html.erb`
5. Find the `toggle-class` controller on the sidebar toggle button
6. What class does it toggle? Where is that class defined?

7. Create a simple Stimulus controller that logs to console when clicked. See if you can wire it up without reading more docs.

## Summary

- **Stimulus** is for the JavaScript you actually need - pure browser behavior
- **HTML-first**: Declare behavior in HTML, define implementation in JS
- **Five building blocks**: Controllers, Actions, Targets, Values, Classes
- **Naming convention**: `snake_case_controller.js` becomes `kebab-case` in HTML
- **Small controllers**: Each does one thing well
- **Works with Turbo**: Stimulus survives DOM changes because behavior is declared in HTML

The philosophy: **HTML is the source of truth. JavaScript just makes it work.**

This completes the Hotwire trilogy:
- **Turbo Drive** → Fast navigation
- **Turbo Frames** → Partial page updates
- **Turbo Streams** → Real-time pushes
- **Stimulus** → Browser-side behavior

Together, they let you build modern, interactive web apps while keeping most of your code in Ruby on the server.

---

## Q&A

### Q: How does local_time_controller work?

**Question:** What does the local_time controller do and how is it implemented? It seems useful.

**Answer:** This solves a classic web problem: **timezones**.

#### The Problem

Campfire stores all times in UTC on the server. But when you see a message timestamp, you want YOUR timezone:

- Server stores: `2024-12-12T15:30:00Z` (UTC)
- User in New York sees: `10:30 AM`
- User in London sees: `3:30 PM`
- User in Tokyo sees: `12:30 AM` (next day)

The server doesn't know your timezone. Only your browser does.

#### The Solution: Server Sends UTC, Browser Formats Locally

**Step 1: Rails Helper** (`app/helpers/time_helper.rb`)

```ruby
def local_datetime_tag(datetime, style: :time, **attributes)
  tag.time **attributes, datetime: datetime.iso8601, data: { local_time_target: style }
end
```

Generates:
```html
<time datetime="2024-12-12T15:30:00Z" data-local-time-target="time"></time>
```

**Step 2: Stimulus Controller** (`app/javascript/controllers/local_time_controller.js`)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = [ "time", "date", "datetime" ]

  initialize() {
    // Browser's built-in formatters - automatically uses user's locale!
    this.timeFormatter = new Intl.DateTimeFormat(undefined, { timeStyle: "short" })
    this.dateFormatter = new Intl.DateTimeFormat(undefined, { dateStyle: "long" })
    this.dateTimeFormatter = new Intl.DateTimeFormat(undefined, { timeStyle: "short", dateStyle: "short" })
  }

  // Called automatically when a "time" target appears in the DOM
  timeTargetConnected(target) {
    this.#formatTime(this.timeFormatter, target)
  }

  dateTargetConnected(target) {
    this.#formatTime(this.dateFormatter, target)
  }

  datetimeTargetConnected(target) {
    this.#formatTime(this.dateTimeFormatter, target)
  }

  #formatTime(formatter, target) {
    const dt = new Date(target.getAttribute("datetime"))
    target.textContent = formatter.format(dt)
    target.title = this.dateTimeFormatter.format(dt)
  }
}
```

**Step 3: Attach to Body**

```erb
<body data-controller="local-time lightbox">
```

The controller watches the entire page for targets.

#### The Magic: `targetConnected` Callbacks

`timeTargetConnected`, `dateTargetConnected`, etc. are **lifecycle callbacks** - Stimulus calls them automatically when a matching target appears in the DOM.

This is perfect for Turbo! When new messages arrive via Turbo Streams:
1. New `<time data-local-time-target="time">` element appears
2. Stimulus sees it, calls `timeTargetConnected`
3. Browser formats the UTC time to local
4. User sees their local time

No manual initialization needed. It just works.

#### Three Format Styles

| Target | Format | Example |
|--------|--------|---------|
| `time` | Time only | `3:30 PM` |
| `date` | Date only | `December 12, 2024` |
| `datetime` | Both | `12/12/24, 3:30 PM` |

#### Why This Pattern Is Clever

1. **Server stays simple** - Just send UTC, no timezone logic
2. **Browser does the work** - `Intl.DateTimeFormat` handles locale automatically
3. **Works with Turbo** - `targetConnected` fires for dynamically added elements
4. **Accessible** - `datetime` attribute has machine-readable time, `textContent` has human-readable

This is a great example of Stimulus lifecycle callbacks and how they integrate seamlessly with Turbo.

## Quiz History

[Quiz sessions will be recorded here after the learner is quizzed on this topic]
