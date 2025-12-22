---
concepts: Active Job,Solid Queue,perform_later,Queues,Recurring Jobs,retry_on
source_repo: fizzy
description: Deep dive into Rails background jobs - the pattern that makes production apps fast and reliable. Covers Active Job (the abstraction), Solid Queue (37signals' database-backed backend), queues, recurring jobs, error handling, and the magic that preserves account context across job execution. All with real Fizzy examples.
understanding_score: null
last_quizzed: null
prerequisites: [2025-12-09-concerns---breaking-up-god-models-the-37signals-way.md]
created: 22-12-2025
last_updated: 22-12-2025
---

# Background Jobs - Work That Happens Later

Here's a scenario: a user posts a comment on a card in Fizzy. What needs to happen?

1. Save the comment to the database
2. Send email notifications to everyone watching that card
3. Send push notifications to mobile devices
4. Trigger webhooks to external services
5. Update the search index
6. Detect if there's an "activity spike" on this card

If you did all of this *during* the HTTP request, the user would wait 3-5 seconds staring at a spinner while emails get sent and webhooks fire. Worse: if the email server is slow, or a webhook times out, their comment might *fail to save entirely*.

This is the fundamental insight behind background jobs: **some work doesn't need to happen right now**. The user cares that their comment appears instantly. They don't care (or even notice) whether the email goes out in 100ms or 30 seconds.

Background jobs let you say "I'll do this later" - after the request has already responded. The result: fast, reliable user experiences, with all the heavy lifting happening behind the scenes.

## The World Before Background Jobs

Before you appreciate the solution, you need to feel the pain of the problem.

Imagine Fizzy without background jobs. Here's what the `Comment` model might look like:

```ruby
# HYPOTHETICAL: The naive approach (DON'T DO THIS)
class Comment < ApplicationRecord
  after_create :send_notifications

  private
    def send_notifications
      watchers.each do |user|
        NotificationMailer.new_comment(self, user).deliver_now  # BLOCKING!
      end

      webhooks.each do |webhook|
        webhook.trigger(self)  # BLOCKING HTTP REQUEST!
      end
    end
end
```

Problems with this approach:

1. **Slow requests**: Every email is a network round-trip. 10 watchers = 10 email sends = maybe 5 seconds of waiting.

2. **Fragile requests**: If the email server is down, `deliver_now` raises an exception. Your comment fails to save because of an unrelated email problem.

3. **No retry logic**: If a webhook times out, it's just... gone. No second chance.

4. **User frustration**: The user sees a loading spinner while you're doing work they don't care about.

5. **Thundering herd**: If 100 users post comments at once, your email server gets hammered with 1000 emails simultaneously.

Background jobs solve all of these problems. Let's see how.

## Active Job: The Abstraction Layer

Rails has many possible "backends" for running background work - Sidekiq (uses Redis), Resque (uses Redis), Delayed Job (uses the database), Solid Queue (uses the database). They all work differently under the hood.

**Active Job** is Rails' abstraction layer. It gives you one consistent interface, regardless of which backend you're using. Think of it like Active Record for jobs - you write Ruby, and Active Job translates it to whatever your queue backend understands.

```
┌─────────────────────────────────────────────────────────┐
│                     Your Code                           │
│         NotifyRecipientsJob.perform_later(event)        │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                     Active Job                          │
│     (Consistent API: perform_later, queue_as, etc.)     │
└─────────────────────────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
     ┌─────────┐      ┌─────────┐      ┌─────────┐
     │ Sidekiq │      │ Resque  │      │  Solid  │
     │ (Redis) │      │ (Redis) │      │  Queue  │
     └─────────┘      └─────────┘      └─────────┘
```

The mental model: Active Job is to job backends what Active Record is to databases. You don't write MySQL queries directly - you write Ruby, and Active Record translates. Same idea here.

## Solid Queue: 37signals' Choice

Fizzy uses **Solid Queue** - a job backend that stores jobs in your regular database (MySQL/PostgreSQL/SQLite) instead of Redis. This is 37signals' official recommendation for most Rails apps.

Why database instead of Redis?

1. **One less dependency**: You already have a database. Why add Redis just for jobs?
2. **Transactional safety**: Jobs are inserted in the same database transaction as your data.
3. **Simpler ops**: One thing to back up, monitor, and scale.
4. **Good enough performance**: For most apps, database-backed queues are plenty fast.

Here's Fizzy's queue configuration at `config/queue.yml`:

```yaml
default: &default
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: [ "default", "solid_queue_recurring", "backend", "webhooks" ]
      threads: 3
      processes: <%= Integer(ENV.fetch("JOB_CONCURRENCY") { Concurrent.physical_processor_count }) %>
      polling_interval: 0.1
```

The `workers` section says: "Spin up worker processes that listen to these queues, each with 3 threads, polling every 0.1 seconds for new jobs."

The number of processes scales with your CPU cores. More cores = more parallel job execution.

## Your First Job: The Anatomy

Let's look at a real job from Fizzy.

**Location:** `app/jobs/notify_recipients_job.rb`

```ruby
class NotifyRecipientsJob < ApplicationJob
  def perform(notifiable)
    notifiable.notify_recipients
  end
end
```

That's the entire file. Let's break it down:

1. **Inherits from `ApplicationJob`**: All jobs extend this base class (which extends `ActiveJob::Base`)

2. **`perform` method**: This is what runs when the job executes. The arguments you pass to `perform_later` arrive here.

3. **`notifiable` argument**: This is a model instance (like an `Event`). Active Job automatically serializes it to a Global ID, stores it in the queue, then re-loads it when the job runs.

Now, where is this job called from? Look at the `Notifiable` concern:

**Location:** `app/models/concerns/notifiable.rb`

```ruby
module Notifiable
  extend ActiveSupport::Concern

  included do
    has_many :notifications, as: :source, dependent: :destroy

    after_create_commit :notify_recipients_later
  end

  private
    def notify_recipients_later
      NotifyRecipientsJob.perform_later self
    end
end
```

See the pattern? When a `Notifiable` record is created (like an `Event`), the callback calls `perform_later` - which queues the job for background execution. The HTTP request finishes immediately. The notifications get sent "later."

## The Magic of `perform_later` vs `perform_now`

Two ways to run a job:

```ruby
# Queue for background execution (non-blocking)
NotifyRecipientsJob.perform_later(event)

# Execute immediately, inline (blocking - use sparingly)
NotifyRecipientsJob.perform_now(event)
```

`perform_later` is what you want 99% of the time. It:
1. Serializes the job arguments
2. Inserts a row into the `solid_queue_jobs` table
3. Returns immediately
4. A worker process picks it up and runs it

`perform_now` runs the job synchronously, blocking the current thread. Useful for testing or rare cases where you *need* the result immediately.

### What Gets Serialized?

When you call `perform_later(event)`, Active Job doesn't serialize the entire `Event` object. That would be fragile - what if the event is updated before the job runs?

Instead, it serializes a **Global ID**: a string like `gid://fizzy/Event/abc123`. When the job executes, Active Job uses `GlobalID::Locator.locate` to re-fetch the record from the database.

This means: the job always works with the *current* state of the record, not a stale snapshot.

## Queues: Prioritizing Work

Not all background work is equally urgent. Fizzy uses multiple queues:

```yaml
queues: [ "default", "solid_queue_recurring", "backend", "webhooks" ]
```

Look at how different jobs specify their queue:

**High priority work (default queue):**
```ruby
class NotifyRecipientsJob < ApplicationJob
  # No queue_as = uses :default
  def perform(notifiable)
    notifiable.notify_recipients
  end
end
```

**Webhook delivery (dedicated queue):**
```ruby
class Webhook::DeliveryJob < ApplicationJob
  queue_as :webhooks

  def perform(delivery)
    delivery.deliver
  end
end
```

**Backend work (lower priority):**
```ruby
class Notification::Bundle::DeliverJob < ApplicationJob
  queue_as :backend

  def perform(bundle)
    bundle.deliver
  end
end
```

Why separate queues? **Isolation and prioritization.**

If webhook deliveries are slow (network issues to external services), you don't want them blocking notification emails. By putting webhooks in their own queue, they can back up without affecting other work.

You could also configure different workers with different priorities:

```yaml
workers:
  - queues: [ "default" ]       # High priority
    threads: 5
  - queues: [ "webhooks" ]      # Can be slower
    threads: 2
```

## The Context Problem: Account Preservation

Here's a subtle but critical problem. Look at this Fizzy code:

```ruby
class Notification::Bundle
  def deliver
    user.in_time_zone do
      Current.with_account(user.account) do   # <-- NEEDS CURRENT ACCOUNT
        processing!
        Notification::BundleMailer.notification(self).deliver if deliverable?
        delivered!
      end
    end
  end
end
```

The mailer needs `Current.account` to be set - for URL generation, permissions, multi-tenancy. But jobs run in separate processes! When a job starts, `Current.account` is `nil`.

How does Fizzy solve this? With a clever initializer.

**Location:** `config/initializers/active_job.rb`

```ruby
module FizzyActiveJobExtensions
  extend ActiveSupport::Concern

  prepended do
    attr_reader :account
    self.enqueue_after_transaction_commit = true
  end

  def initialize(...)
    super
    @account = Current.account  # Capture account when job is queued
  end

  def serialize
    super.merge({ "account" => @account&.to_gid })  # Store in job payload
  end

  def deserialize(job_data)
    super
    if _account = job_data.fetch("account", nil)
      @account = GlobalID::Locator.locate(_account)  # Restore when job runs
    end
  end

  def perform_now
    if account.present?
      Current.with_account(account) { super }  # Wrap execution with account
    else
      super
    end
  end
end

ActiveSupport.on_load(:active_job) do
  prepend FizzyActiveJobExtensions
end
```

This is beautiful. The pattern:

1. **Capture context at queue time**: When `perform_later` is called, `initialize` captures `Current.account`
2. **Serialize the context**: `serialize` adds the account's Global ID to the job payload
3. **Restore context at run time**: `deserialize` reconstructs the account from the GID
4. **Wrap execution**: `perform_now` sets `Current.account` before running the job

The job author doesn't have to think about any of this. Every job automatically runs in the right account context.

Notice `enqueue_after_transaction_commit = true` - this ensures jobs aren't queued until the database transaction commits. Otherwise, the job might run before the data it needs is actually saved!

## Recurring Jobs: Work on a Schedule

Some work needs to happen repeatedly - not triggered by user actions, but by time.

Fizzy configures recurring jobs in `config/recurring.yml`:

```yaml
production: &production
  # Application functionality: notifications and summaries
  deliver_bundled_notifications:
    command: "Notification::Bundle.deliver_all_later"
    schedule: every 30 minutes

  # Application cleanup
  auto_postpone_all_due:
    command: "Card.auto_postpone_all_due"
    schedule: every hour at minute 50

  delete_unused_tags:
    class: DeleteUnusedTagsJob
    schedule: every day at 04:02

  # Operations cleanup
  cleanup_webhook_deliveries:
    command: "Webhook::Delivery.cleanup"
    schedule: every 4 hours at minute 51
```

Two ways to define recurring work:

1. **`command`**: Runs a Ruby expression directly
2. **`class`**: Instantiates and runs a job class

The schedule syntax is human-readable: `every 30 minutes`, `every day at 04:02`, `every hour at minute 50`.

This is how Fizzy's "entropy" system works - cards automatically postpone after inactivity. A recurring job runs every hour, finds stale cards, and moves them to "not now."

No cron needed. No separate scheduler service. Solid Queue handles it all.

## Error Handling: When Things Go Wrong

Network calls fail. Servers go down. Timeouts happen. Good job design handles this gracefully.

**Location:** `app/jobs/concerns/smtp_delivery_error_handling.rb`

```ruby
module SmtpDeliveryErrorHandling
  extend ActiveSupport::Concern

  included do
    # Retry delivery to possibly-unavailable remote mailservers.
    retry_on Net::OpenTimeout, Net::ReadTimeout, Socket::ResolutionError,
             wait: :polynomially_longer

    # Net::SMTPServerBusy is SMTP error code 4xx, a temporary error.
    retry_on Net::SMTPServerBusy, wait: :polynomially_longer

    # SMTP error 5xx - permanent failures
    rescue_from Net::SMTPFatalError do |error|
      case error.message
      when /\A550 5\.1\.1/, /\A552 5\.6\.0/, /\A555 5\.5\.4/
        # Unknown user, message too large, etc - don't retry, just log
        Sentry.capture_exception error, level: :info if Fizzy.saas?
      else
        raise  # Re-raise unexpected errors
      end
    end
  end
end
```

Key concepts:

**`retry_on`**: If this exception is raised, don't fail the job - put it back in the queue for another attempt.

**`wait: :polynomially_longer`**: Wait longer between each retry. First retry after a few seconds, second after more, etc. This is "exponential backoff" - it prevents hammering a struggling service.

**`rescue_from`**: Catch specific errors and decide what to do. Some errors (like "unknown email address") aren't worth retrying - the email will never be deliverable.

The job that uses this concern:

```ruby
class Notification::Bundle::DeliverJob < ApplicationJob
  include SmtpDeliveryErrorHandling

  queue_as :backend

  def perform(bundle)
    bundle.deliver
  end
end
```

Simple job code, robust error handling extracted to a concern. Clean.

## Advanced: Continuable Jobs

For really long-running jobs, what if the job crashes halfway through? You'd have to start over.

Fizzy has a webhook dispatch job that might need to trigger hundreds of webhooks:

**Location:** `app/jobs/event/webhook_dispatch_job.rb`

```ruby
require "active_job/continuable"

class Event::WebhookDispatchJob < ApplicationJob
  include ActiveJob::Continuable

  queue_as :webhooks

  def perform(event)
    step :dispatch do |step|
      Webhook.active.triggered_by(event).find_each(start: step.cursor) do |webhook|
        webhook.trigger(event)
        step.advance! from: webhook.id
      end
    end
  end
end
```

`ActiveJob::Continuable` lets jobs **checkpoint their progress**. The `step.cursor` remembers where we left off. If the job crashes after triggering 50 webhooks, it resumes from webhook 51, not webhook 1.

This is a more advanced pattern - you won't need it often. But it's elegant when you do.

## The Callback Connection: When to Queue Jobs

You've seen `after_create_commit :notify_recipients_later`. Why `after_create_commit` and not `after_create`?

**`after_create`** runs inside the database transaction. If you queue a job here, the job might start running *before the transaction commits*. The job tries to load the record... and it doesn't exist yet!

**`after_create_commit`** runs after the transaction commits. The record is definitely in the database. Safe to queue.

Look at `Card::Stallable`:

```ruby
included do
  before_update :remember_to_detect_activity_spikes
  after_update_commit :detect_activity_spikes_later, if: :should_detect_activity_spikes?
end

private
  def detect_activity_spikes_later
    Card::ActivitySpike::DetectionJob.perform_later(self)
  end
```

Notice: `before_update` to remember state, `after_update_commit` to queue the job. The job is only queued after the update is safely committed.

(Fizzy also uses `enqueue_after_transaction_commit = true` in the job extensions, which provides a second layer of protection.)

## Try It Yourself

Here's an exercise to cement your understanding:

1. Run Fizzy's development server: `bin/dev`
2. Open a Rails console: `bin/rails console`
3. Check the job queue:
   ```ruby
   SolidQueue::Job.pending.count
   SolidQueue::Job.pending.last
   ```
4. Create an event that triggers a job:
   ```ruby
   Current.account = Account.first
   Current.user = Current.account.users.first
   board = Current.account.boards.first
   card = board.cards.first
   card.comments.create!(body: "Test comment")
   ```
5. Watch the pending jobs increase, then get processed
6. Check again:
   ```ruby
   SolidQueue::Job.finished.count
   ```

You just watched the entire lifecycle: model callback → job queued → worker picked it up → job finished.

## Summary

- **Background jobs defer work** that doesn't need to happen during the request. Result: fast responses, reliable processing.

- **Active Job** is Rails' abstraction layer - write once, run on any backend (Sidekiq, Solid Queue, etc.).

- **Solid Queue** stores jobs in your database. One less Redis dependency. 37signals' recommendation.

- **`perform_later`** queues the job; **`perform_now`** runs it inline. Almost always use `perform_later`.

- **Queues** let you prioritize and isolate work. Critical notifications on `:default`, slow webhooks on `:webhooks`.

- **Context preservation** (like `FizzyActiveJobExtensions`) is crucial for multi-tenant apps - jobs need to know which account they're running for.

- **`retry_on`** handles transient failures; **`rescue_from`** handles permanent ones. Design jobs to be resilient.

- **Recurring jobs** (`config/recurring.yml`) replace cron for scheduled work.

- **Always queue from `after_*_commit` callbacks**, never from inside a transaction.

This pattern - fast request, background processing - is what separates amateur Rails apps from production-grade ones. Every serious SaaS you'll build needs this.

---

## Q&A

[Questions and answers will be added here as the learner asks them during the tutorial]

## Quiz History

[Quiz sessions will be recorded here after the learner is quizzed on this topic]
