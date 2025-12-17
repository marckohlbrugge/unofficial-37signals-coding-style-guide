# Caching Patterns

> HTTP caching and fragment caching lessons from 37signals.

---

## Cache Invalidation

### Touch Chains
```ruby
belongs_to :card, touch: true
belongs_to :board, touch: true
```
Changes to children automatically update parent timestamps.

### Composite Cache Keys
```ruby
# Bad - same cache for different contexts
cache card

# Good - includes rendering context
cache [card, previewing_card?]
cache [card, Current.user.id]  # if user-specific
```

### Include What Affects Output
- Timezone affects rendered times
- User ID affects personalized content
- Filter state affects what's shown

## HTTP Caching (ETags)

### Basic Pattern
```ruby
def show
  fresh_when etag: @card
end
```

### Don't HTTP Cache Forms
CSRF tokens get stale → 422 errors on submit (#1607)

Remove `fresh_when` from pages with forms.

### Public Caching
- Safe for read-only public pages
- 30 seconds is reasonable (#1377)
- Use concern to DRY up cache headers

## Fragment Caching Gotchas

### Lazy-Loaded Content
Convert expensive menus to turbo frames that load on demand (#1089)

### Context-Dependent Caching
```ruby
# Include workflow in cache key - stages change
cache [card, card.collection.workflow]
```

### User-Specific Content
Move to client-side JavaScript if it breaks caching:
- "You commented..." → check via JS
- Delete buttons → show/hide via JS based on meta tag

## Extract Dynamic Content to Turbo Frames (#317)

When part of a cached fragment needs updates:
```erb
<% cache [bubble, bucket] do %>
  <%= turbo_frame_tag bubble, :assignment do %>
    <!-- Dynamic dropdown -->
  <% end %>
<% end %>
```

## Include Rendering Context (#340)

```ruby
# Bad
cache bubble

# Good
cache [bubble, previewing_card?]
```

## Touch Chains for Dependencies (#566)

```ruby
class Workflow::Stage < ApplicationRecord
  belongs_to :workflow, touch: true
end

# View
cache [card, card.collection.workflow]
```

## Domain Models for Cache Keys (#1132)

For complex views, create dedicated cache key objects:
```ruby
class Cards::Columns
  def cache_key
    ActiveSupport::Cache.expand_cache_key([
      considering, on_deck, doing, closed,
      Workflow.all, user_filtering
    ])
  end
end
```
