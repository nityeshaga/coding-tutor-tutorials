---
concepts: [Controllers, before_action, Strong Parameters, render vs redirect, Concerns]
source_repo: once-campfire
description: A deep dive into Rails controllers - the orchestrators that receive requests and coordinate responses. Covers before_action filters (how Campfire ensures you're logged in), strong parameters (protecting against mass assignment attacks), the crucial difference between render and redirect_to, and how concerns let you share controller behavior. Uses real Campfire code throughout.
understanding_score: null
last_quizzed: null
prerequisites: [references/tutorials/2025-11-25-mvc---the-architecture-that-powers-every-rails-app.md, references/tutorials/2025-11-25-routing---how-urls-become-code.md]
created: 27-11-2025
last_updated: 27-11-2025
---

# Controllers Deep Dive - The Glue That Holds Rails Together

In your MVC tutorial, you learned that controllers are the "traffic cops" - they receive requests and decide what to do. But that was the 10,000-foot view. Now we're going to get into the mechanics of how controllers actually work. This is where the rubber meets the road.

Think about it: when someone visits Campfire, the controller has to:
1. Check if they're logged in (and kick them out if not)
2. Figure out which room they're trying to access
3. Make sure they're allowed to access that room
4. Get the messages to display
5. Decide how to send everything back

That's a lot of orchestration. Rails gives you elegant tools for each piece. Let's break them down.

## The Problem: Repetitive Security Checks

Imagine you're building Campfire without knowing about `before_action`. Every single action would need to start with:

```ruby
class MessagesController < ApplicationController
  def index
    # Check if logged in... again
    return redirect_to login_path unless current_user
    # Find the room... again
    @room = Room.find(params[:room_id])
    # Check access... again
    return head :forbidden unless @room.accessible_by?(current_user)

    # FINALLY do the actual work
    @messages = @room.messages
  end

  def create
    # Check if logged in... again
    return redirect_to login_path unless current_user
    # Find the room... again
    @room = Room.find(params[:room_id])
    # Check access... again
    return head :forbidden unless @room.accessible_by?(current_user)

    # FINALLY do the actual work
    @room.messages.create!(message_params)
  end

  # And on and on for every action...
end
```

This is a nightmare. The same checks repeated everywhere. Miss one and you have a security hole. Change the logic and you have to update 20 places.

Rails' answer? **Filters** - code that runs before (or after) your actions.

## Key Concept 1: before_action

`before_action` is a method that says "run this code before executing any action in this controller."

```
┌─────────────────────────────────────────────────────────────────┐
│                       Request Lifecycle                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Request comes in                                                │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────────┐                                            │
│  │  before_action  │◄── Runs FIRST. Can halt the request        │
│  │  (filters)      │    entirely if something's wrong           │
│  └────────┬────────┘                                            │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │  Your Action    │◄── Only runs if filters pass               │
│  │  (index, show)  │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │  after_action   │◄── Runs after (less common)                │
│  │  (cleanup)      │                                            │
│  └─────────────────┘                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

The mental model: **before_action is a bouncer at a club**. Before you get into the party (your action), the bouncer checks your ID, checks the guest list, makes sure you're dressed appropriately. If any check fails, you don't get in.

### Example 1: Campfire's Authentication Bouncer

**Location:** `app/controllers/concerns/authentication.rb:5-11`

```ruby
included do
  before_action :require_authentication
  before_action :deny_bots
  helper_method :signed_in?

  protect_from_forgery with: :exception, unless: -> { authenticated_by.bot_key? }
end
```

**What's happening:** Every controller that includes this concern (which is all of them, via ApplicationController) will run `require_authentication` before every action. No exceptions by default.

The actual check is at line 33-35:

```ruby
def require_authentication
  restore_authentication || bot_authentication || request_authentication
end
```

This tries three things in order:
1. Restore from session cookie (you're already logged in)
2. Check for a bot key (API access)
3. If neither works, redirect to login

The beauty? Every controller gets this for free. You can't accidentally forget to check authentication.

### Example 2: Selective Filtering with :only and :except

**Location:** `app/controllers/messages_controller.rb:4-6`

```ruby
before_action :set_room, except: :create
before_action :set_message, only: %i[ show edit update destroy ]
before_action :ensure_can_administer, only: %i[ edit update destroy ]
```

**What this demonstrates:** You don't always want filters to run on every action.

- `except: :create` - Run on all actions EXCEPT create
- `only: %i[ show edit update destroy ]` - ONLY run on these specific actions

Why would `set_room` skip `create`? Look at line 21-22:

```ruby
def create
  set_room  # Called manually here
  @message = @room.messages.create_with_attachment!(message_params)
```

The `create` action handles `set_room` differently (inside a rescue block to catch errors gracefully). This is a common pattern: use filters for the common case, handle edge cases explicitly.

### Example 3: Skipping Inherited Filters

**Location:** `app/controllers/sessions_controller.rb:2`

```ruby
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[ new create ]
```

This is interesting. The login page can't require you to be logged in (chicken and egg problem). So `SessionsController` *skips* the authentication filter for `new` and `create`.

That `allow_unauthenticated_access` is defined in the Authentication concern at line 14-16:

```ruby
def allow_unauthenticated_access(**options)
  skip_before_action :require_authentication, **options
end
```

It's just a pretty wrapper around `skip_before_action`. This is how Rails code reads like English.

## Key Concept 2: Strong Parameters

Here's a terrifying scenario. You have a User model:

```ruby
class User < ApplicationRecord
  # has columns: name, email, admin (boolean)
end
```

And a naive update action:

```ruby
def update
  @user.update(params[:user])
end
```

A malicious user could send: `{ user: { name: "Bob", admin: true } }`

And boom, they've made themselves an admin. This is called **mass assignment vulnerability** and it's destroyed real companies.

Rails' solution: **Strong Parameters**. You must explicitly whitelist which parameters are allowed.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Strong Parameters Flow                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Raw params from request                                        │
│   { user: { name: "Bob", admin: true, email: "..." } }          │
│                     │                                            │
│                     ▼                                            │
│   ┌─────────────────────────────────────────────────────┐       │
│   │  params.require(:user).permit(:name, :email)        │       │
│   └─────────────────────────────────────────────────────┘       │
│                     │                                            │
│                     ▼                                            │
│   Sanitized params (safe to use)                                │
│   { name: "Bob", email: "..." }                                 │
│                                                                  │
│   ❌ 'admin' silently dropped - it wasn't permitted             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Example: Message Parameters in Campfire

**Location:** `app/controllers/messages_controller.rb:70-72`

```ruby
def message_params
  params.require(:message).permit(:body, :attachment, :client_message_id)
end
```

Breaking this down:
- `params.require(:message)` - The params MUST have a `:message` key, or raise an error
- `.permit(:body, :attachment, :client_message_id)` - Only these three fields are allowed through

What gets blocked? If someone tried to pass `creator_id: 999` to impersonate another user, it would be silently dropped. The database column might exist, but the controller won't let it through.

### Example: Profile Update with .compact

**Location:** `app/controllers/users/profiles_controller.rb:19-21`

```ruby
def user_params
  params.require(:user).permit(:name, :avatar, :email_address, :password, :bio).compact
end
```

Notice the `.compact` at the end. This removes any `nil` values. Why? If a user submits a form without changing their password, we don't want to set `password: nil` and break their account. `.compact` strips out empty fields.

## Key Concept 3: render vs redirect_to

This is one of the most confusing things for new Rails developers. Both end a controller action, but they're completely different.

```
┌─────────────────────────────────────────────────────────────────┐
│                render vs redirect_to                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RENDER                           REDIRECT_TO                    │
│  ──────                           ───────────                    │
│  Same request                     New request                    │
│  Shows a template                 Tells browser to go elsewhere │
│  Instance variables preserved     Instance variables LOST        │
│  URL stays the same              URL changes                     │
│                                                                  │
│  Use for:                         Use for:                       │
│  - Showing forms with errors      - After successful save       │
│  - Displaying content             - Changing the URL             │
│  - Error pages                    - Post-create/update/delete    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**The mental model:**
- `render` is "show this template right now" - like turning to a page in a book
- `redirect_to` is "tell the browser to make a whole new request" - like handing someone a different book

### Example: Redirect After Success

**Location:** `app/controllers/users/profiles_controller.rb:9-12`

```ruby
def update
  @user.update user_params
  redirect_to user_profile_url, notice: update_notice
end
```

After successfully updating, we redirect. Why not render?

1. **URL changes** - The user now sees `/profile` in their browser, not `/profile/update`
2. **Refresh safety** - If they hit refresh, they reload the profile page, not re-submit the form
3. **Flash message** - The `notice:` becomes a flash message that shows on the NEXT page

### Example: Render on Failure

**Location:** `app/controllers/sessions_controller.rb:10-16`

```ruby
def create
  if user = User.active.authenticate_by(email_address: params[:email_address], password: params[:password])
    start_new_session_for user
    redirect_to post_authenticating_url
  else
    render_rejection :unauthorized
  end
end
```

Where `render_rejection` does:

```ruby
def render_rejection(status)
  flash.now[:alert] = "Too many requests or unauthorized."
  render :new, status: status
end
```

On failed login:
- `render :new` - Show the login form again (same request, user sees their entered email)
- `flash.now` - Show error message NOW, not on next request
- `status: :unauthorized` - Send 401 HTTP status code

The URL stays at `/session` - we're just showing the form again with an error.

### The head Method

Sometimes you don't need to render anything - just send back a status code.

**Location:** `app/controllers/messages_controller.rb:53-55`

```ruby
def ensure_can_administer
  head :forbidden unless Current.user.can_administer?(@message)
end
```

`head :forbidden` sends back a 403 status with an empty body. No template needed. This is common for:
- Authorization failures
- API responses
- When JavaScript just needs a "yes/no" answer

## Key Concept 4: Concerns (Sharing Controller Code)

You've seen `Authentication` included in ApplicationController. How does that work?

**Concerns** are modules that package up reusable controller behavior. They use `ActiveSupport::Concern` to make the syntax cleaner.

### Example: The RoomScoped Concern

**Location:** `app/controllers/concerns/room_scoped.rb`

```ruby
module RoomScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_room
  end

  private
    def set_room
      @membership = Current.user.memberships.find_by!(room_id: params[:room_id])
      @room = @membership.room
    end
end
```

Any controller that does `include RoomScoped` automatically gets:
1. A `before_action` that sets `@room`
2. The `set_room` method

**Location:** `app/controllers/messages_controller.rb:2`

```ruby
class MessagesController < ApplicationController
  include ActiveStorage::SetCurrent, RoomScoped
```

Now `MessagesController` gets room-finding for free. Notice it still declares its own `before_action :set_room, except: :create` - that overrides the concern's default.

### ApplicationController: The Root of It All

**Location:** `app/controllers/application_controller.rb`

```ruby
class ApplicationController < ActionController::Base
  include AllowBrowser, Authentication, Authorization, SetCurrentRequest, SetPlatform, TrackedRoomVisit, VersionHeaders
  include Turbo::Streams::Broadcasts, Turbo::Streams::StreamName
end
```

This is where Campfire's controller inheritance starts. Every controller inherits from this, getting:
- `Authentication` - login required
- `Authorization` - permission checks
- `SetCurrentRequest` - sets up `Current.user` etc.
- Plus several more behaviors

This is the power of concerns: layer them to build up functionality.

## Try It Yourself

Here's a practical exercise:

1. Open `app/controllers/rooms/opens_controller.rb`
2. Trace the `before_action` chain - which filters run before `show`?
3. What happens if `set_room` fails to find a room?
4. Find where `remember_last_room_visited` is defined (hint: it's a concern)

Bonus: Add some `puts` statements to the `before_action` methods and watch the order they run when you visit a room.

## Summary

- **before_action** filters run before your actions - use them for authentication, authorization, and setting up instance variables. Use `:only` and `:except` to control which actions they apply to.

- **Strong Parameters** protect against mass assignment attacks. Always use `params.require(:model).permit(:field1, :field2)` - never pass raw params to create/update.

- **render** shows a template in the same request (URL stays same). Use for error states and displaying content.

- **redirect_to** tells the browser to make a new request (URL changes). Use after successful creates/updates/deletes.

- **Concerns** let you extract reusable controller behavior into modules. Use `ActiveSupport::Concern` and the `included do` block.

Next up: **Hotwire (Turbo + Stimulus)** - where controllers get even more interesting. You'll learn how controllers can respond with more than just HTML, and how DHH's team built Campfire without writing much JavaScript.

---

## Q&A

[Questions and answers will be added here as the learner asks them during the tutorial]

## Quiz History

[Quiz sessions will be recorded here after the learner is quizzed on this topic]
