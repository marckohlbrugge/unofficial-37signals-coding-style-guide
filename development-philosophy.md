# 37signals Development Philosophy

> Core principles observed across 265 PRs in Fizzy.

---

## Ship, Validate, Refine

- Merge "prototype quality" code to validate with real usage before cleanup
- Features evolve through iterations (tenanting: 3 attempts before settling)
- Don't polish prematurely - real-world usage reveals what matters
- PR [#335](https://github.com/basecamp/fizzy/pull/335) merged as "prototype quality" to validate design first

## Fix Root Causes, Not Symptoms

**Bad**: Add retry logic for race conditions
**Good**: Use `enqueue_after_transaction_commit` to prevent the race ([#1664](https://github.com/basecamp/fizzy/pull/1664))

**Bad**: Work around CSRF issues on cached pages
**Good**: Don't HTTP cache pages with forms ([#1607](https://github.com/basecamp/fizzy/pull/1607))

## Vanilla Rails Over Abstractions

- Thin controllers calling rich domain models
- No service objects unless truly justified
- Direct ActiveRecord is fine: `@card.comments.create!(params)`
- When services exist, they're just POROs: `Signup.new(email:).create_identity`

## DHH's Review Patterns

See [dhh.md](dhh.md) for comprehensive review patterns extracted from 100+ PR reviews.

Key themes:
- Questions indirection: "Is this abstraction earning its keep?"
- Pushes for directness - collapsed 6 notifier subclasses into 2 ([#425](https://github.com/basecamp/fizzy/pull/425))
- Prefers explicit over clever (define methods directly vs introspection)
- Removes "anemic" code that adds layers without value
- Write-time operations over read-time computations
- Database constraints over AR validations

## Common Review Themes

- **Naming**: Use positive names (`active` not `not_deleted`, `unpopped`)
- **DB over AR**: Prefer database constraints over ActiveRecord validations
- **Migrations**: Use SQL, avoid model references that break future runs
- **Simplify**: Links over JavaScript when browser affordances suffice ([#138](https://github.com/basecamp/fizzy/pull/138))

## When to Extract

- Start in controller, extract when it gets messy
- Filter logic: controller → model concern → dedicated PORO ([#115](https://github.com/basecamp/fizzy/pull/115), [#116](https://github.com/basecamp/fizzy/pull/116))
- Don't extract prematurely - wait for pain
- Rule of three: duplicate twice before abstracting

## Rails 7.1+ `params.expect` ([#120](https://github.com/basecamp/fizzy/pull/120))

Replace `params.require(:key).permit(...)` with `params.expect(key: [...])`:
- Returns 400 (Bad Request) instead of 500 for bad params
- Cleaner, more explicit syntax

```ruby
# Before
params.require(:user).permit(:name, :email)

# After
params.expect(user: [:name, :email])
```

## StringInquirer for Action Predicates ([#425](https://github.com/basecamp/fizzy/pull/425))

Instead of string comparisons, use StringInquirer:

```ruby
# Bad
if event.action == "completed"

# Good
if event.action.completed?

# Implementation
def action
  self[:action].inquiry
end
```

## Caching Constraints Inform Architecture ([#119](https://github.com/basecamp/fizzy/pull/119))

Design caching early - it reveals architectural issues:
- Can't use `Current.user` in cached partials
- Solution: Push user-specific logic to Stimulus controllers reading from meta tags
- Leave FIXME comments when you discover caching conflicts

## Write-Time vs Read-Time Operations ([#108](https://github.com/basecamp/fizzy/pull/108))

DHH's principle: "All manipulation has to happen when you save, not when you present."
- Use delegated types for heterogeneous collections needing pagination
- Pre-compute roll-ups at write time
- Use `dependent: :delete_all` when no callbacks needed
- Use counter caches instead of manual counting

---

## Jason Fried: Product-Oriented Development

> From PRs [#305](https://github.com/basecamp/fizzy/pull/305), [#131](https://github.com/basecamp/fizzy/pull/131), [#335](https://github.com/basecamp/fizzy/pull/335), [#265](https://github.com/basecamp/fizzy/pull/265), #608

### Perceived Performance Over Technical Performance ([#131](https://github.com/basecamp/fizzy/pull/131))

User perception matters more than server metrics. If it feels slow, it is slow.

> "I'd imagined this as a single form where you'd make all your selections and then 'apply' the filter rather than it updating after every choice. Some shopping websites do that latter and it always feels/is slow."

**Pattern**: Question live-updating UIs. Sometimes a single "Apply" action feels faster than multiple instant updates.

### Prototype Quality is a Valid Shipping Standard ([#335](https://github.com/basecamp/fizzy/pull/335))

Explicitly communicate when code is "prototype quality" - built to validate with real usage:

> "This is an unproven feature built with prototype quality code so I would suggest factoring your appetite accordingly. The goal is to get this onto our production instance as soon as possible so we can vet the design with real work."

Ship with enumerated known issues:
1. Performance concerns ("un-holy things with Bubble collections")
2. Missing features ("no pagination in the new view")
3. Technical debt ("the whole `Bubble` namespace doesn't really make sense")

**Pattern**: Different features deserve different polish. Ship to learn when uncertain.

### Production Truth Over Local Speculation ([#335](https://github.com/basecamp/fizzy/pull/335))

> "It runs slowly on beta which may be simply because the droplet doesn't have sufficient specs. It's quite fast on local dev so this might not be an issue at all on production. It could be that you just merge it as is and there's no problem."

**Pattern**: Real data in production reveals what local testing can't. Ship with monitoring ready.

### Simplify by Removing, Not Just Hiding ([#131](https://github.com/basecamp/fizzy/pull/131))

> "One thing we could try is to not show the chips while the form is open. Then there wouldn't be anything to update live on the page."

**Pattern**: Reduce visible UI states to reduce complexity and edge cases.

### Extend Don't Replace ([#608](https://github.com/basecamp/fizzy/pull/608))

When adding "Create and add another" button, keep original "Create" flow intact:

```ruby
def create
  @card.publish
  redirect_to add_another_param? ? @collection.cards.create! : @card
end

private
  def add_another_param?
    params[:creation_type] == "add_another"
  end
```

**Pattern**: Branch with parameters, don't replace routes or create separate endpoints.

### Leverage Robust Systems Creatively ([#335](https://github.com/basecamp/fizzy/pull/335))

> "The whole thing runs through the `Filter` system. It's quite robust and resilient so I got a lot of mileage out of breaking it out into individual forms to get the various sorting and filtering in place."

**Pattern**: Reuse proven systems for new features, even if "there are certainly better ways" - ship with what works.

### Name Technical Debt Without Blocking ([#335](https://github.com/basecamp/fizzy/pull/335))

> "There is also some mess here that we haven't cleaned up since we moved to the cards design. The whole `Bubble` namespace doesn't really make sense anymore but we haven't done anything about it. I'm only pointing that out because it's probably confusing!"

**Pattern**: Acknowledge confusing code, provide context ("exists in main, too"), don't block features on cleanup.

### Feedback as Product Vision, Not Mandate ([#131](https://github.com/basecamp/fizzy/pull/131))

Share vision, not commands:
- ✅ "I'd imagined this as..."
- ✅ "One thing we could try..."
- ❌ "Change this to use X"
- ❌ "You must do Y"

**Pattern**: Give product context and UX concerns, let implementer figure out how.

---

## Rails Patterns

### Delegated Types for Polymorphism ([#124](https://github.com/basecamp/fizzy/pull/124))

Use `delegated_type` instead of traditional polymorphic associations:

```ruby
class Message < ApplicationRecord
  belongs_to :bubble, touch: true
  delegated_type :messageable, types: %w[Comment EventSummary],
                 inverse_of: :message, dependent: :destroy
end

module Messageable
  extend ActiveSupport::Concern
  included do
    has_one :message, as: :messageable, touch: true
  end
end
```

**Why**: Automatic convenience methods (`message.comment?`, `message.comment`) without manual type checking.

### Store Accessor for JSON Columns ([#113](https://github.com/basecamp/fizzy/pull/113))

Use `store_accessor` for structured JSON storage:

```ruby
class Bucket::View < ApplicationRecord
  store_accessor :filters, :order_by, :status, :assignee_ids, :tag_ids

  validates :order_by, inclusion: { in: ORDERS.keys, allow_nil: true }
end
```

**Why**: Type casting, validation, and cleaner API (`view.order_by` vs `view.filters['order_by']`).

### Normalizes for Data Consistency ([#1083](https://github.com/basecamp/fizzy/pull/1083))

Use `normalizes` to clean data before validation (Rails 7.1+):

```ruby
class Webhook < ApplicationRecord
  serialize :subscribed_actions, type: Array, coder: JSON

  normalizes :subscribed_actions,
    with: ->(value) { Array.wrap(value).map(&:to_s).uniq & PERMITTED_ACTIONS }
end
```

**Why**: Ensures data consistency before validation, no `before_validation` callbacks needed.

### Concern Organization by Responsibility ([#124](https://github.com/basecamp/fizzy/pull/124))

Split models into focused concerns:

```ruby
class Bubble < ApplicationRecord
  include Assignable      # Assignment logic
  include Boostable       # Boost counting
  include Eventable       # Event tracking
  include Poppable        # Archive logic
  include Searchable      # Full-text search
  include Staged          # Workflow stage logic
  include Taggable        # Tag associations
end
```

**Guidelines**:
- Each concern should be 50-150 lines
- Must be cohesive (related functionality together)
- Don't create concerns just to reduce file size

### Scopes Named for Business Concepts ([#124](https://github.com/basecamp/fizzy/pull/124))

```ruby
# Good - business-focused
scope :active, -> { where.missing(:pop) }
scope :unassigned, -> { where.missing(:assignments) }

# Not - SQL-ish
scope :without_pop, -> { ... }
scope :no_assignments, -> { ... }
```

### Transaction Wrapping ([#124](https://github.com/basecamp/fizzy/pull/124))

Wrap related updates for consistency:

```ruby
def toggle_stage(stage)
  transaction do
    update! stage: new_stage
    track_event event, stage_id: stage.id
  end
end
```

**When to use**: Multi-step operations, parent + children records, state transitions.

### Touch Chains for Cache Invalidation ([#124](https://github.com/basecamp/fizzy/pull/124))

```ruby
class Comment < ApplicationRecord
  has_one :message, as: :messageable, touch: true
end

class Message < ApplicationRecord
  belongs_to :bubble, touch: true
end
```

Changes propagate up: comment → message → bubble, invalidating caches automatically.
