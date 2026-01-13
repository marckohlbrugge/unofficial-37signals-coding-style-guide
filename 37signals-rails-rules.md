# 37signals Rails Rules

A comprehensive set of actionable rules for Rails development, extracted from 37signals' patterns.

---

## Table of Contents

1. [Quick Reference](#1-quick-reference)
2. [Development Philosophy](#2-development-philosophy)
3. [Routing](#3-routing)
4. [Controllers](#4-controllers)
5. [Models](#5-models)
6. [Views](#6-views)
7. [Hotwire](#7-hotwire)
8. [Stimulus](#8-stimulus)
9. [CSS](#9-css)
10. [Database](#10-database)
11. [Background Jobs](#11-background-jobs)
12. [Testing](#12-testing)
13. [Security](#13-security)
14. [Multi-Tenancy](#14-multi-tenancy)
15. [What to Avoid](#15-what-to-avoid)
16. [Appendix: Reusable Stimulus Controllers](#appendix-a-reusable-stimulus-controllers)

---

## 1. Quick Reference

**The 37signals Way - Top 10 Principles**

- [ ] Rich domain models over service objects
- [ ] CRUD controllers over custom actions
- [ ] Concerns for horizontal code sharing
- [ ] Records as state (not booleans)
- [ ] Database-backed everything (no Redis)
- [ ] Build it yourself before reaching for gems
- [ ] Ship to learn - prototype quality is valid
- [ ] Vanilla Rails is plenty
- [ ] Fix root causes, not symptoms
- [ ] Explicit over clever

---

## 2. Development Philosophy

### Ship, Validate, Refine

- [ ] Merge "prototype quality" code to validate design before cleanup
- [ ] Don't polish prematurely - real-world usage reveals what matters
- [ ] Features evolve through iterations (accept 2-3 attempts before settling)
- [ ] Real data reveals what local testing cannot

### Fix Root Causes

| Bad (Symptom Fix) | Good (Root Cause Fix) |
|-------------------|----------------------|
| Add retry logic for race conditions | Use `enqueue_after_transaction_commit` |
| Work around CSRF on cached pages | Don't HTTP cache pages with forms |
| Add null checks everywhere | Fix the source of nil values |

### Vanilla Rails Over Abstractions

- [ ] Thin controllers calling rich domain models
- [ ] No service objects unless truly justified
- [ ] Direct ActiveRecord is fine: `@card.comments.create!(params)`
- [ ] When services exist, they're just POROs: `Signup.new(email:).create_identity`

### When to Extract

- [ ] Start in controller, extract when it gets messy
- [ ] Rule of three: duplicate twice before abstracting
- [ ] Don't extract prematurely - wait for real pain
- [ ] Ask "Is this abstraction earning its keep?"

### Code Quality

- [ ] Use positive names (`active` not `not_deleted`)
- [ ] Prefer database constraints over AR validations
- [ ] Use `params.expect` over `params.require.permit` (Rails 7.1+)
- [ ] Use StringInquirer for action predicates: `event.action.completed?`
- [ ] Write-time operations over read-time computations
- [ ] Design caching early - it reveals architectural issues

---

## 3. Routing

### The CRUD Principle

Every action maps to a CRUD verb. When something doesn't fit, create a new resource.

```ruby
# BAD: Custom actions
resources :cards do
  post :close
  post :archive
end

# GOOD: New resources for state changes
resources :cards do
  resource :closure      # POST to close, DELETE to reopen
  resource :goldness     # POST to gild, DELETE to ungild
  resource :pin          # POST to pin, DELETE to unpin
end
```

### Turn Verbs into Nouns

| Action | Resource |
|--------|----------|
| Close a card | `resource :closure` |
| Watch a board | `resource :watching` |
| Pin an item | `resource :pin` |
| Publish a board | `resource :publication` |
| Assign a user | `resources :assignments` |
| Mark as golden | `resource :goldness` |

### Routing Rules

- [ ] Every action maps to a CRUD verb
- [ ] Use `resource` (singular) for one-per-parent resources
- [ ] Use `shallow: true` to avoid deep nesting (3+ levels)
- [ ] Use `scope module:` for grouping without URL prefix
- [ ] Use `namespace` when you want URL prefix
- [ ] Use `resolve` for custom polymorphic URL generation
- [ ] No separate API namespace - just `respond_to`

### Response Codes

| Action | Success Code |
|--------|--------------|
| Create | `201 Created` + `Location` header |
| Update | `204 No Content` |
| Delete | `204 No Content` |

### Controller Organization

```
app/controllers/
├── application_controller.rb
├── cards_controller.rb
├── cards/
│   ├── closures_controller.rb
│   ├── assignments_controller.rb
│   ├── pins_controller.rb
│   └── comments/
│       └── reactions_controller.rb
└── boards/
    └── publications_controller.rb
```

---

## 4. Controllers

### Core Principle

Controllers should be thin orchestrators. Business logic lives in models.

```ruby
# GOOD: Controller just orchestrates
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create
    @card.close  # All logic in model
    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.json { head :no_content }
    end
  end
end

# BAD: Business logic in controller
def create
  @card.transaction do
    @card.create_closure!(user: Current.user)
    @card.events.create!(action: :closed)
    NotificationMailer.card_closed(@card).deliver_later
  end
end
```

### Controller Rules

- [ ] Controllers are thin orchestrators
- [ ] Business logic lives in models
- [ ] Use bang methods (`create!`) - let it crash
- [ ] Authorization: controller checks, model defines permissions
- [ ] Use `respond_to` for multiple formats (turbo_stream, json, html)
- [ ] Return `head :forbidden` for authorization failures

### ApplicationController Pattern

```ruby
class ApplicationController < ActionController::Base
  include Authentication
  include Authorization
  include CurrentRequest, CurrentTimezone
  include TurboFlash

  etag { "v1" }
  allow_browser versions: :modern
end
```

### Controller Concerns

Use concerns for composable, reusable behaviors:

#### Resource Scoping Concerns

```ruby
# app/controllers/concerns/card_scoped.rb
module CardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_card, :set_board
  end

  private
    def set_card
      @card = Current.user.accessible_cards.find_by!(number: params[:card_id])
    end

    def set_board
      @board = @card.board
    end

    def render_card_replacement
      render turbo_stream: turbo_stream.replace(
        [@card, :card_container],
        partial: "cards/container",
        method: :morph,
        locals: { card: @card.reload }
      )
    end
end
```

#### Request Context Concerns

```ruby
# app/controllers/concerns/current_request.rb
module CurrentRequest
  extend ActiveSupport::Concern

  included do
    before_action do
      Current.request_id = request.uuid
      Current.user_agent = request.user_agent
      Current.ip_address = request.ip
    end
  end
end
```

#### Timezone Concern

```ruby
module CurrentTimezone
  extend ActiveSupport::Concern

  included do
    around_action :set_current_timezone
    helper_method :timezone_from_cookie
    etag { timezone_from_cookie }
  end

  private
    def set_current_timezone(&)
      Time.use_zone(timezone_from_cookie, &)
    end

    def timezone_from_cookie
      @timezone_from_cookie ||= ActiveSupport::TimeZone[cookies[:timezone]]
    end
end
```

### Concern Composition Rules

- [ ] Concerns can include other concerns
- [ ] Use `before_action` in `included` block
- [ ] Provide shared private methods (e.g., `render_card_replacement`)
- [ ] Use `helper_method` for view access
- [ ] Add `etag` blocks for HTTP caching

### Authorization Pattern

```ruby
module Authorization
  extend ActiveSupport::Concern

  private
    def ensure_can_administer
      head :forbidden unless Current.user.admin?
    end
end

# In controller
class WebhooksController < ApplicationController
  before_action :ensure_can_administer
end
```

### Rate Limiting

```ruby
class Sessions::MagicLinksController < ApplicationController
  rate_limit to: 10, within: 15.minutes, only: :create,
    with: -> { redirect_to root_path, alert: "Try again later." }
end
```

---

## 5. Models

### Heavy Use of Concerns

```ruby
class Card < ApplicationRecord
  include Assignable, Broadcastable, Closeable, Eventable,
    Pinnable, Searchable, Taggable, Watchable

  belongs_to :account, default: -> { board.account }
  belongs_to :board
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end
```

### Concern Structure

Each concern is self-contained with associations, scopes, and methods:

```ruby
# app/models/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy

    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def closed?
    closure.present?
  end

  def close(user: Current.user)
    unless closed?
      transaction do
        create_closure!(user: user)
        track_event :closed, creator: user
      end
    end
  end

  def reopen(user: Current.user)
    if closed?
      transaction do
        closure&.destroy
        track_event :reopened, creator: user
      end
    end
  end
end
```

### State as Records, Not Booleans

```ruby
# BAD: Boolean column
class Card < ApplicationRecord
  # closed: boolean
  scope :closed, -> { where(closed: true) }
end

# GOOD: Separate record
class Closure < ApplicationRecord
  belongs_to :card, touch: true
  belongs_to :user, optional: true
  # created_at gives you WHEN
  # user gives you WHO
end

class Card < ApplicationRecord
  has_one :closure, dependent: :destroy
  scope :closed, -> { joins(:closure) }
  scope :open, -> { where.missing(:closure) }
end
```

**Why records over booleans:**
- Know WHO made the change
- Know WHEN it happened
- Query history easily
- Add metadata (reason, notes)

### Model Rules

- [ ] Each concern should be 50-150 lines
- [ ] Concerns must be cohesive - related functionality together
- [ ] Don't create concerns just to reduce file size
- [ ] Name concerns for capability: `Closeable`, `Watchable`, `Assignable`
- [ ] State records over booleans (Closure not `closed:boolean`)
- [ ] Use `has_one` for state records
- [ ] Query with `joins` and `where.missing` for state
- [ ] Use `default: -> { }` lambdas for belongs_to defaults
- [ ] Default creator to `Current.user`
- [ ] Default account to parent's account
- [ ] Use `Current` for request context
- [ ] Minimal validations - prefer DB constraints
- [ ] Use bang methods (`create!`) - let it crash
- [ ] Use `touch: true` for cache invalidation chains

### Scope Naming

```ruby
# GOOD - business-focused
scope :active, -> { where.missing(:pop) }
scope :unassigned, -> { where.missing(:assignments) }
scope :golden, -> { joins(:goldness) }

# BAD - SQL-ish
scope :without_pop, -> { ... }
scope :no_assignments, -> { ... }
```

### PORO Patterns

POROs live under model namespaces for related logic:

```ruby
# Presentation logic
class Event::Description
  def initialize(event)
    @event = event
  end

  def to_s
    case event.action
    when "created" then "#{creator_name} created this card"
    when "closed"  then "#{creator_name} closed this card"
    end
  end
end

# View context bundling
class User::Filtering
  attr_reader :user, :filter

  def initialize(user, filter)
    @user = user
    @filter = filter
  end

  def boards
    user.boards.accessible
  end
end
```

### Callbacks - Use Sparingly

- [ ] Only 38 callback occurrences across entire codebase
- [ ] Use callbacks for setup/cleanup, not business logic
- [ ] `before_validation` for setup on create
- [ ] `after_create_commit` for notifications

---

## 6. Views

### Turbo Streams for Partial Updates

```erb
<%# app/views/cards/comments/create.turbo_stream.erb %>
<%= turbo_stream.before [@card, :new_comment],
    partial: "cards/comments/comment",
    locals: { comment: @comment } %>
```

### Morphing for Complex Updates

```erb
<%= turbo_stream.replace dom_id(@card, :card_container),
    partial: "cards/container",
    method: :morph,
    locals: { card: @card.reload } %>
```

### View Rules

- [ ] Use Turbo Streams for partial page updates
- [ ] Use `method: :morph` for complex updates
- [ ] Standard partials over ViewComponents
- [ ] Subscribe to streams with `turbo_stream_from`
- [ ] Prefer locals over instance variables in partials
- [ ] Use Rails `dom_id` helper for IDs

### Fragment Caching

```erb
<%# Basic cache %>
<% cache card do %>
  <%= render "cards/preview", card: card %>
<% end %>

<%# Composite cache keys %>
<% cache [card, Current.user, timezone_from_cookie] do %>
  <%= render "cards/preview", card: card %>
<% end %>

<%# Collection caching %>
<%= render partial: "cards/preview", collection: @cards, cached: true %>
```

### DOM ID Conventions

```erb
<div id="<%= dom_id(card) %>">           <%# card_123 %>
<div id="<%= dom_id(card, :preview) %>"> <%# preview_card_123 %>
<div id="<%= dom_id(card, :comments) %>"><%# comments_card_123 %>
```

### Partial Naming

```
app/views/cards/
├── _card.html.erb           # Single card
├── _preview.html.erb        # Card preview/summary
├── _container.html.erb      # Card with wrapper
├── _form.html.erb           # Card form
└── container/
    └── _content.html.erb    # Nested partial
```

### View Helpers for Stimulus Integration

```ruby
module DialogHelper
  def dialog_tag(id, **options, &block)
    options[:data] ||= {}
    options[:data][:controller] = "dialog"
    options[:data][:action] = "click->dialog#closeOnOutsideClick"
    tag.dialog(id: id, **options, &block)
  end
end
```

---

## 7. Hotwire

### Turbo Morphing Configuration

- [ ] Enable globally: `turbo_refreshes_with method: :morph, scroll: :preserve`
- [ ] Use `data-turbo-permanent` for elements that shouldn't refresh
- [ ] Ensure unique IDs - duplicates break morphing
- [ ] Set `refresh: :morph` on frames with `src`
- [ ] Listen for `turbo:morph-element` to restore client-side state

### Turbo Frames

- [ ] Use `data-turbo-frame="_parent"` to target parent frame
- [ ] Wrap form sections in frames to prevent reset
- [ ] Lazy-load expensive content with `loading: "lazy"`
- [ ] Use `turbo_stream.replace` instead of redirects

### Common Turbo Issues

| Problem | Solution |
|---------|----------|
| Timers not updating after morph | Bind to `turbo:morph-element` |
| Forms resetting on refresh | Wrap in turbo frames |
| Pagination breaking | Ensure unique IDs |
| Flickering on replace | Use `method: :morph` |
| localStorage lost | Restore on `turbo:morph-element` |

### Morphing + Turbo Streams

```ruby
render turbo_stream: turbo_stream.replace(
  [@record, :container],
  partial: "records/container",
  method: :morph
)
```

### State Persistence with localStorage

```javascript
export default class extends Controller {
  static targets = ["input"]
  static values = { key: String }

  connect() {
    this.restoreContent()
  }

  save() {
    localStorage.setItem(this.keyValue, this.inputTarget.value)
  }

  async restoreContent() {
    await nextFrame()
    const saved = localStorage.getItem(this.keyValue)
    if (saved) this.inputTarget.value = saved
  }
}
```

Wire up to restore after morphs:

```erb
data-action="input->local-save#save turbo:morph-element->local-save#restoreContent"
```

### Hotwire Rules

- [ ] Filter chips as plain `<a>` tags, not JS-powered buttons
- [ ] Use POST + Turbo Streams for UI state toggles
- [ ] Mark nested frames as permanent during morphs
- [ ] Use element-level morph events over global for performance
- [ ] Accept flash-of-collapsed-content as acceptable tradeoff

---

## 8. Stimulus

### Philosophy

- **Single-purpose** - One job per controller
- **Configured via values/classes** - No hardcoded strings
- **Event-based communication** - Dispatch events, don't call other controllers

### Stimulus Rules

- [ ] Use Values API over `getAttribute()`: `static values = { url: String }`
- [ ] Use camelCase in JavaScript
- [ ] Always clean up in `disconnect()` - timers, listeners, observers
- [ ] Use `:self` action filter: `click:self->modal#close`
- [ ] Extract shared helpers to modules
- [ ] Dispatch events for controller communication
- [ ] Use targets over CSS selectors

### Timing Helpers

```javascript
// helpers/timing_helpers.js
export function debounce(fn, delay = 1000) {
  let timeoutId = null
  return (...args) => {
    clearTimeout(timeoutId)
    timeoutId = setTimeout(() => fn.apply(this, args), delay)
  }
}

export function nextFrame() {
  return new Promise(requestAnimationFrame)
}
```

### Timer Cleanup Pattern

```javascript
export default class extends Controller {
  #timer

  connect() {
    this.#timer = setInterval(() => this.refresh(), 30_000)
  }

  disconnect() {
    clearInterval(this.#timer)
  }
}
```

### File Organization

```
app/javascript/
├── controllers/
│   ├── application.js
│   ├── auto_submit_controller.js
│   ├── dialog_controller.js
│   └── copy_to_clipboard_controller.js
└── helpers/
    ├── timing_helpers.js
    └── dom_helpers.js
```

---

## 9. CSS

### Philosophy

Native CSS only - no Sass, PostCSS, or Tailwind.

### Cascade Layers

```css
@layer reset, base, layout, components, utilities;

@layer reset {
  *, *::before, *::after { box-sizing: border-box; }
}

@layer components {
  .card { /* component styles */ }
  .btn { /* button styles */ }
}

@layer utilities {
  .hidden { display: none; }
  .flex { display: flex; }
}
```

### OKLCH Color Space

```css
:root {
  --lch-blue-dark: 57.02% 0.1895 260.46;
  --color-link: oklch(var(--lch-blue-dark));
}
```

### Dark Mode via CSS Variables

```css
:root {
  --lch-ink-darkest: 26% 0.05 264;
  --lch-canvas: 100% 0 0;
}

html[data-theme="dark"] {
  --lch-ink-darkest: 96.02% 0.0034 260;
  --lch-canvas: 20% 0.0195 232.58;
}

@media (prefers-color-scheme: dark) {
  html:not([data-theme]) {
    --lch-ink-darkest: 96.02% 0.0034 260;
    --lch-canvas: 20% 0.0195 232.58;
  }
}
```

### Component Naming

```css
.card { }
.card__header { }
.card__body { }
.card--closed { }
```

### CSS Rules

- [ ] Use native CSS only - no Sass, PostCSS, Tailwind
- [ ] Use `@layer` for explicit specificity management
- [ ] Layer order: reset, base, layout, components, utilities
- [ ] Use OKLCH for perceptually uniform colors
- [ ] Dark mode by redefining CSS variable values
- [ ] Use native CSS nesting
- [ ] Components expose customization via CSS variables
- [ ] Use logical properties (`padding-block`, `margin-inline`)
- [ ] Use container queries for component-level responsiveness
- [ ] Minimal breakpoints, mostly fluid with `clamp()`
- [ ] ~60 utility classes total (not Tailwind-style hundreds)

### Modern CSS Features

```css
/* Entry animations */
.dialog {
  @starting-style {
    &[open] { opacity: 0; }
  }
}

/* Parent-aware styling */
.card:has(.card__closed) {
  --card-color: var(--color-complete);
}

/* Dynamic colors */
--card-bg: color-mix(in srgb, var(--card-color) 4%, var(--color-canvas));
```

---

## 10. Database

### UUIDs as Primary Keys

```ruby
create_table :cards, id: :uuid do |t|
  t.references :board, type: :uuid, foreign_key: true
end
```

**Why UUIDs:**
- No ID guessing attacks
- Safe for distributed systems
- Client can generate IDs before insert

### State as Records

```ruby
# State record with attribution
class Closure < ApplicationRecord
  belongs_to :card, touch: true
  belongs_to :creator, class_name: "User"
end
```

### Database-Backed Infrastructure

```ruby
# No Redis - everything uses database
gem "solid_queue"   # Jobs
gem "solid_cache"   # Caching
gem "solid_cable"   # WebSockets
```

### Database Rules

- [ ] Use UUIDs instead of auto-incrementing integers
- [ ] UUIDv7 for time-sortable IDs
- [ ] State records over booleans - capture WHO and WHEN
- [ ] Use Solid Queue/Cache/Cable (no Redis)
- [ ] Multi-tenancy via `account_id` foreign key everywhere
- [ ] Scoped uniqueness: `validates :number, uniqueness: { scope: :account_id }`
- [ ] Hard deletes over soft deletes
- [ ] Use audit logs for history if needed
- [ ] Use counter caches for performance
- [ ] Always index foreign keys
- [ ] Index columns you filter/sort by

---

## 11. Background Jobs

### Configuration

```bash
# Development: Run jobs in Puma process
SOLID_QUEUE_IN_PUMA=1
```

```ruby
# Production: Match workers to CPU cores
# 3 threads for I/O-bound jobs
```

### Transaction Safety

```ruby
# In initializer
ActiveJob::Base.enqueue_after_transaction_commit = true
```

### Stagger Recurring Jobs

```yaml
# BAD - all at :00
job_a: every hour at minute 0
job_b: every hour at minute 0

# GOOD - staggered
job_a: every hour at minute 12
job_b: every hour at minute 50
```

### Error Handling

```ruby
module SmtpDeliveryErrorHandling
  extend ActiveSupport::Concern

  included do
    # Transient - retry
    retry_on Net::OpenTimeout, Net::ReadTimeout,
      wait: :polynomially_longer

    # Permanent - swallow gracefully
    rescue_from Net::SMTPSyntaxError do |error|
      Sentry.capture_exception error, level: :info
    end
  end
end
```

### Shallow Jobs Pattern

```ruby
# Jobs just call model methods
class NotifyRecipientsJob < ApplicationJob
  def perform(notifiable)
    notifiable.notify_recipients
  end
end
```

### `_later` Convention

```ruby
module Notifiable
  extend ActiveSupport::Concern

  included do
    after_create_commit :notify_recipients_later
  end

  def notify_recipients
    Notifier.for(self)&.notify
  end

  private
    def notify_recipients_later
      NotifyRecipientsJob.perform_later(self)
    end
end
```

### Job Rules

- [ ] Run jobs in Puma process in development
- [ ] Match workers to CPU cores in production
- [ ] Stagger recurring job schedules
- [ ] Use `enqueue_after_transaction_commit = true`
- [ ] Retry transient errors with polynomial backoff
- [ ] Swallow permanent failures gracefully (log at info level)
- [ ] Jobs just call model methods (shallow jobs)
- [ ] Use `_later` suffix for async, plain name for sync
- [ ] Use `ActiveJob::Continuable` for large batch jobs

---

## 12. Testing

### Minitest Over RSpec

- [ ] Use Minitest, not RSpec
- [ ] Simpler, less DSL magic
- [ ] Ships with Rails
- [ ] Plain Ruby assertions

### Fixtures Over Factories

```yaml
# test/fixtures/cards.yml
urgent_bug:
  board: engineering
  creator: david
  title: "Fix login bug"
  created_at: <%= 2.days.ago %>
```

```ruby
test "closing a card creates an event" do
  card = cards(:urgent_bug)
  user = users(:david)

  assert_difference "Event.count", 1 do
    card.close(by: user)
  end
end
```

### Test Structure

```ruby
class CardTest < ActiveSupport::TestCase
  setup do
    @card = cards(:urgent_bug)
    @user = users(:david)
  end

  test "closed cards cannot be edited" do
    @card.close(by: @user)
    assert_not @card.editable_by?(@user)
  end
end
```

### Testing Rules

- [ ] Use Minitest, not RSpec
- [ ] Use fixtures, not FactoryBot
- [ ] Use labels for relationships, not IDs
- [ ] Use ERB in fixtures for dynamic values
- [ ] Use `assert_difference` for counting changes
- [ ] Use integration tests for full request/response cycles
- [ ] Use system tests with Capybara for browser testing
- [ ] Use `travel_to` for time-dependent tests
- [ ] Use VCR for external API recording/replay
- [ ] Tests ship with features in the same commit

---

## 13. Security

### XSS Prevention

```ruby
# Always escape before html_safe
"<span>#{h(user_input)}</span>".html_safe
```

### CSRF Protection

- [ ] Don't HTTP cache pages with forms (CSRF tokens get stale)
- [ ] Use `Sec-Fetch-Site` header as additional CSRF check
- [ ] Add `Sec-Fetch-Site` to Vary header

```ruby
def verified_request?
  super || safe_fetch_site?
end

def safe_fetch_site?
  %w[same-origin same-site].include?(
    request.headers["Sec-Fetch-Site"]&.downcase
  )
end
```

### SSRF Protection

```ruby
# Resolve DNS once, pin the IP
resolved_ip = resolve_dns(url)
Net::HTTP.new(host, port, ipaddr: resolved_ip)
```

- [ ] Block private networks (loopback, private, link-local)
- [ ] Block IPv4-mapped IPv6
- [ ] Validate URLs at creation time AND request time

### Content Security Policy

```ruby
config.content_security_policy do |policy|
  policy.script_src :self
  policy.style_src :self, :unsafe_inline
  policy.base_uri :none
  policy.form_action :self
  policy.frame_ancestors :self
end
```

### Rate Limiting

```ruby
class Sessions::MagicLinksController < ApplicationController
  rate_limit to: 10, within: 15.minutes, only: :create,
    with: -> { redirect_to root_path, alert: "Try again later." }
end
```

### Security Rules

- [ ] Always escape before `html_safe`
- [ ] Don't HTTP cache pages with forms
- [ ] Use `Sec-Fetch-Site` as additional CSRF check
- [ ] Pin DNS for webhook requests
- [ ] Block private network ranges
- [ ] Validate URLs at creation AND request time
- [ ] Scope broadcasts by account (multi-tenant)
- [ ] Disconnect deactivated users from ActionCable
- [ ] Configure Content Security Policy
- [ ] Rate limit auth endpoints

---

## 14. Multi-Tenancy

### Path-Based Tenancy

```ruby
# URL: /1234567/boards/123
module AccountSlug
  class Extractor
    def call(env)
      request = ActionDispatch::Request.new(env)

      if request.path_info =~ /\A(\/\d{7,})/
        request.script_name = $1
        request.path_info = $'
        account = Account.find_by(external_id: decode($1))
        Current.with_account(account) { @app.call(env) }
      else
        @app.call(env)
      end
    end
  end
end
```

### ActiveJob Tenant Preservation

```ruby
module TenantedJob
  extend ActiveSupport::Concern

  prepended do
    attr_reader :account
  end

  def initialize(...)
    super
    @account = Current.account
  end

  def serialize
    super.merge({ "account" => @account&.to_gid })
  end

  def perform_now
    if account.present?
      Current.with_account(account) { super }
    else
      super
    end
  end
end
```

### Multi-Tenancy Rules

- [ ] Use path-based tenancy (`/1234567/boards`)
- [ ] Middleware extracts tenant and sets `Current.account`
- [ ] Use `Current.with_account(account) { }` blocks
- [ ] Serialize tenant in jobs via GlobalID
- [ ] Recurring jobs iterate all tenants
- [ ] Scope session cookie to account path
- [ ] Always scope controller lookups through tenant
- [ ] Defense in depth - don't rely solely on middleware

### Always Scope Lookups

```ruby
# BAD
@comment = Comment.find(params[:id])

# GOOD - scope through tenant
@comment = Current.account.comments.find(params[:id])

# BETTER - scope through user's accessible records
@bubble = Current.user.accessible_bubbles.find(params[:id])
```

---

## 15. What to Avoid

### Gems and Patterns to Skip

| Instead of | Use | Why |
|------------|-----|-----|
| Devise | Custom magic links (~150 lines) | Too heavyweight for passwordless |
| Pundit/CanCanCan | Model predicates | Simpler, no separate policy files |
| Service objects | Rich model methods | Fragments domain logic |
| Form objects | Strong params | Rails provides enough |
| Decorators | View helpers | No gem overhead |
| ViewComponent | ERB partials | Simpler mental model |
| GraphQL | REST + Turbo | Unnecessary complexity |
| Sidekiq | Solid Queue | No Redis needed |
| React/Vue | Turbo + Stimulus | Server rendering is simpler |
| Tailwind | Native CSS | Modern CSS is capable |
| RSpec | Minitest | Simpler, less DSL |
| FactoryBot | Fixtures | Faster, deterministic |

### Authorization Without Gems

```ruby
class Card < ApplicationRecord
  def editable_by?(user)
    !closed? && (creator == user || user.admin?)
  end

  def deletable_by?(user)
    user.admin? || creator == user
  end
end

# In controller
def edit
  head :forbidden unless @card.editable_by?(Current.user)
end
```

### The Philosophy

Before adding a dependency, ask:
1. Can vanilla Rails do this?
2. Is the complexity worth the benefit?
3. Will we need to maintain this dependency?
4. Does it make the codebase harder to understand?

---

## Appendix A: Reusable Stimulus Controllers

### Copy-to-Clipboard Controller

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { content: String }
  static classes = [ "success" ]

  async copy(event) {
    event.preventDefault()
    this.element.classList.remove(this.successClass)
    this.element.offsetWidth // Force reflow

    try {
      await navigator.clipboard.writeText(this.contentValue)
      this.element.classList.add(this.successClass)
    } catch {}
  }
}
```

### Auto-Submit Controller

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { delay: { type: Number, default: 300 } }

  connect() {
    this.timeout = null
  }

  submit() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }

  disconnect() {
    clearTimeout(this.timeout)
  }
}
```

### Dialog Controller

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  open() {
    this.element.showModal()
  }

  close() {
    this.element.close()
  }

  closeOnOutsideClick(event) {
    if (event.target === this.element) {
      this.close()
    }
  }
}
```

### Auto-Resize Controller

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { minHeight: { type: Number, default: 0 } }

  connect() {
    this.resize()
  }

  resize() {
    this.element.style.height = "auto"
    const newHeight = Math.max(this.minHeightValue, this.element.scrollHeight)
    this.element.style.height = `${newHeight}px`
  }
}
```

### Beacon Controller

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { url: String }

  connect() {
    if (this.hasUrlValue) {
      navigator.sendBeacon(this.urlValue)
    }
  }
}
```

### Form Reset Controller

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  resetOnSuccess(event) {
    if (event.detail.success) {
      this.element.reset()
    }
  }
}
```

### Element Removal Controller

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  remove() {
    this.element.remove()
  }
}
```

### Toggle Class Controller

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static classes = [ "toggle" ]

  toggle() {
    this.element.classList.toggle(this.toggleClass)
  }

  add() {
    this.element.classList.add(this.toggleClass)
  }

  remove() {
    this.element.classList.remove(this.toggleClass)
  }
}
```

---

## Document Info

- **Source**: Unofficial 37signals Coding Style Guide
- **Extracted from**: 265 PRs in Fizzy codebase
- **Last updated**: January 2025
