# Background Jobs

> Solid Queue patterns from 37signals.

---

## Configuration

### Development
```bash
# Run jobs in Puma process
export SOLID_QUEUE_IN_PUMA=1
```
Simplifies dev - no separate worker process (#469).

### Production
- Match workers to CPU cores (#1290)
- 3 threads for I/O-bound jobs (#1329)

## Stagger Recurring Jobs

Prevent resource spikes by offsetting schedules (#1329):

```yaml
# Bad - all at :00
job_a: every hour at minute 0
job_b: every hour at minute 0

# Good - staggered
job_a: every hour at minute 12
job_b: every hour at minute 50
```

## Transaction Safety

### Enqueue After Commit (#1664)
```ruby
# In initializer
ActiveJob::Base.enqueue_after_transaction_commit = true
```

Prevents jobs from running before the data they need exists.
Fixes `ActiveStorage::FileNotFoundError` on uploads.

## Error Handling

### Transient Errors (#1924)
Retry network errors with polynomial backoff:
- `Net::OpenTimeout`
- `Net::ReadTimeout`
- `Socket::ResolutionError`

### Permanent Failures
Swallow gracefully - don't fail the job:
- Bad email addresses (550)
- Oversized messages (552)

Log to Sentry at info level, not error.

## Maintenance Jobs

### Clean Finished Jobs (#943)
```yaml
clear_finished_jobs:
  command: "SolidQueue::Job.clear_finished_in_batches"
  schedule: every hour at minute 12
```

### Clean Orphaned Data (#494)
- Unused tags (daily)
- Old webhook deliveries (every 4 hours)
- Expired magic links

## Job Patterns

### Shallow Jobs
Jobs just call model methods:
```ruby
class NotifyRecipientsJob < ApplicationJob
  def perform(notifiable)
    notifiable.notify_recipients
  end
end
```

### `_later` and `_now` Convention
```ruby
def mark_as_read_later(user:)
  MarkAsReadJob.perform_later(self, user)
end

def mark_as_read_now(user:)
  readings.find_or_create_by!(user:).touch
end
```

## SMTP Error Handling (#1924)

Create reusable concern:
```ruby
module SmtpDeliveryErrorHandling
  extend ActiveSupport::Concern
  included do
    retry_on Net::OpenTimeout, Net::ReadTimeout,
             wait: :polynomially_longer
    retry_on Net::SMTPServerBusy, wait: :polynomially_longer

    rescue_from Net::SMTPFatalError do |error|
      case error.message
      when /\A550 5\.1\.1/, /\A552 5\.6\.0/
        Sentry.capture_exception error, level: :info
      else
        raise
      end
    end
  end
end
```

Apply to ActionMailer::MailDeliveryJob.

## Continuable Jobs for Resilient Iteration (#1083)

Use `ActiveJob::Continuable` to resume from where you left off after crashes:

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

**Why it matters**: If the job crashes midway through iteration, it resumes from where it left off rather than reprocessing everything. Essential for jobs processing large batches that might timeout.

**Use cases**: Webhooks dispatching, email broadcasts, bulk updates, data migrations.
