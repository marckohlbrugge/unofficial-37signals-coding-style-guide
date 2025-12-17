# Hotwire Patterns

> Turbo and Stimulus best practices from 37signals.

---

## Turbo Morphing

- Enable globally: `turbo_refreshes_with method: :morph, scroll: :preserve`
- Listen for `turbo:morph-element` to restore client-side state
- Use `data-turbo-permanent` for elements that shouldn't refresh
- Ensure unique IDs - duplicates break morphing (#1327)

## Turbo Frames

- Wrap form sections in frames to prevent reset on partial updates (#696)
- Lazy-load expensive menus via frames (#1089)
- Use `turbo_stream.replace` instead of redirects for in-place updates
- Use `refresh: :morph` on lazy-loaded frames to prevent flicker (#339)

## Common Turbo Issues

| Problem | Solution |
|---------|----------|
| Timers not updating after morph | Bind to `turbo:morph` event (#396) |
| Forms resetting | Wrap in turbo frames (#696) |
| Pagination breaking | Ensure unique IDs (#1327) |
| Flickering on replace | Use `method: :morph` (#416) |
| localStorage lost | Restore on `turbo:morph-element` (#339) |

## Stimulus Best Practices

- Use **Values API** over `getAttribute()` - cleaner, type-coerced
- Use **camelCase** in JavaScript (even for data attributes)
- Always clean up in `disconnect()` - timers, listeners
- Use `:self` action filter to scope events (#1936)
- Extract shared helpers to modules (`date_helpers.js`)

## State Persistence

- localStorage for UI preferences (expanded panels, etc.)
- Accept flash-of-collapsed-content as acceptable tradeoff
- Restore state on `turbo:morph-element` events
- Use `nextFrame()` helper to wait for morph completion (#339)

## Links Over JavaScript

- Filter chips as plain `<a>` tags, not JS-powered buttons (#138)
- Better browser affordances (right-click, cmd+click)
- Simpler, more declarative code
- Let the browser do what browsers do

## Morphing + Turbo Streams (#416)

When replacing content containing Turbo Frames:
```ruby
render turbo_stream: turbo_stream.replace(
  [@card, :container],
  partial: "cards/container",
  method: :morph  # Prevents flickering
)
```

Mark nested frames as permanent:
```erb
<%= turbo_frame_tag card, :watch,
    data: { turbo_permanent: true } %>
```

## Element-Level Morph Events (#490)

Prefer element-specific events over global:
```ruby
# In helper
tag.time data: {
  action: "turbo:morph-element->local-time#refreshTarget"
}
```

More efficient than `turbo:morph@window`.

## Turbo Frames Preserve Form State (#696)

Wrap independent sections in frames:
```erb
<%= turbo_frame_tag @collection, :publication do %>
  <!-- form -->
<% end %>
```

Respond with targeted replacement instead of redirect:
```ruby
render turbo_stream: turbo_stream.replace(
  [@collection, :publication],
  partial: "collections/edit/publication"
)
```

## POST + Turbo Streams for UI State (#1091)

For state toggles (expand/collapse), use POST not GET:
```erb
<%= link_to path, data: { turbo_method: "post" } %>
```

Controller returns stream update instead of redirect.

## Frame Morphing Configuration (#1327)

Set `refresh: :morph` on frames with `src`:
```ruby
turbo_frame_tag(id, src: url, refresh: :morph)
```

Prevents frame removal during page morphs.

## Drag and Drop Patterns

### Architecture: Simple Drag Controller Over Complex Sortable (#607)

For basic D&D between containers, use a focused drag-and-drop controller instead of heavyweight sortable libraries:

**Why:** Simpler to reason about, less client-side state, lets server handle ordering logic.

```javascript
// app/javascript/controllers/drag_and_drop_controller.js
export default class extends Controller {
  static targets = [ "item", "container" ]
  static values = { url: String }
  static classes = [ "draggedItem", "hoverContainer" ]

  async dragStart(event) {
    event.dataTransfer.effectAllowed = "move"
    event.dataTransfer.dropEffect = "move"
    event.dataTransfer.setData("37ui/move", event.target)

    await nextFrame() // Wait for drag to start
    this.dragItem = this.#itemContaining(event.target)
    this.sourceContainer = this.#containerContaining(this.dragItem)
    this.dragItem.classList.add(this.draggedItemClass)
  }

  dragOver(event) {
    event.preventDefault()
    const container = this.#containerContaining(event.target)
    this.#clearContainerHoverClasses()

    if (container && container !== this.sourceContainer) {
      container.classList.add(this.hoverContainerClass)
    }
  }

  async drop(event) {
    const container = this.#containerContaining(event.target)
    if (!container || container === this.sourceContainer) return

    this.wasDropped = true
    await this.#submitDropRequest(this.dragItem, container)
  }

  dragEnd() {
    this.dragItem.classList.remove(this.draggedItemClass)
    this.#clearContainerHoverClasses()

    if (this.wasDropped) {
      this.dragItem.remove() // Optimistic removal
    }

    this.sourceContainer = null
    this.dragItem = null
    this.wasDropped = false
  }
}
```

**Key insights:**
- Custom data type (`37ui/move`) prevents interference with other drags
- Use `await nextFrame()` before applying drag classes (prevents visual glitches)
- Track source container to prevent dropping on self
- Optimistically remove on successful drop

### Server-Side Re-render Strategy (#607)

Let the server handle complex ordering/filtering instead of client-side manipulation:

```ruby
class Cards::DropsController < ApplicationController
  def create
    @card.reconsider # or @card.engage

    # Re-render entire column - server knows correct order
    render turbo_stream: turbo_stream.replace(
      "#{@drop_target}-cards",
      method: :morph,
      partial: "cards/index/engagement/#{@drop_target}",
      locals: page_and_filter.to_h
    )
  end
end
```

**Why:** Server re-rendering the whole column is "good enough" and avoids:
- Duplicating sorting logic in client
- Managing complex client-side state
- Out-of-sync client/server issues

**Trade-off:** Slightly less smooth than pure client-side positioning, but much simpler.

### Conditional Draggable Items (#607)

Make draggability a render-time decision, not global:

```erb
<%# Only cards in certain columns are draggable %>
<%= render partial: "cards/display/preview",
           collection: page.records,
           as: :card,
           locals: { draggable: true } %>
```

```erb
<%# In the partial %>
<% draggable = local_assigns.fetch(:draggable, false) %>
<%= card_article_tag card,
      draggable: draggable,
      data: { id: card.id, drag_and_drop_target: "item" } %>
```

**Why:** Not all instances of a component should be draggable. Context matters.

**Cache consideration:** Include `draggable` in cache key:
```ruby
cache cacheable_preview_parts_for(card, draggable)
```

### Drag Visual Feedback (#607, #209)

Provide clear visual states for drag interactions:

```css
.drag-and-drop__dragged-item {
  box-shadow: none;
  filter: grayscale(1) brightness(0.97);
  opacity: 0.6;
  outline: 2px dashed var(--color-selected-dark);
}

.drag-and-drop__hover-container {
  background-color: var(--color-selected-light);
  border-radius: 0.2em;
  outline: 2px dashed var(--color-selected-dark);
  transition: background-color 200ms;
}
```

**Disable hover states during drag (#209):**
```css
ul:not(.dragging) {
  li {
    @media (hover: hover) {
      &:hover {
        background-color: var(--hover-color);
      }
    }
  }
}
```

**Why:** Prevents janky hover flickering while dragging.

### Custom Drag Image (#207)

Provide helpful drag feedback instead of browser default:

```javascript
configureDrag(event) {
  event.dataTransfer.setDragImage(this.dragImageTarget, 0, 0)
}
```

```erb
<div class="divider-drag-image" data-divider-target="dragImage">
  Drag up or down
</div>
```

```css
.divider-drag-image {
  background-color: var(--color-link);
  border-radius: 1rem;
  color: var(--color-ink-reversed);
  inset: auto 100% 100% auto; /* Position off-screen */
  line-height: 2rem;
  padding: 0 1rem;
  position: fixed;
  text-wrap: nowrap;
}
```

**Why:** Browser default is often ugly/confusing. Custom image provides context.

### Prevent Drag Jank with Overlap Threshold (#209)

When dragging near boundaries, prevent jittery movement:

```javascript
const OVERLAP_THRESHOLD = 0.25

moveDivider(event) {
  if (event.target.nodeName == DIVIDER_ITEM_NODE_NAME) {
    const rect = this.dividerTarget.getBoundingClientRect()
    const distanceToTop = Math.abs(event.clientY - rect.top)
    const distanceToBottom = Math.abs(event.clientY - (rect.top + rect.height))
    const distanceToNearestEdge = Math.min(distanceToTop, distanceToBottom)
    const overlap = distanceToNearestEdge / rect.height

    if (overlap > OVERLAP_THRESHOLD) {
      this.#moveDividerTo(this.#items.indexOf(event.target))
    }
  }
}
```

**Why:** Without threshold, tiny mouse movements cause rapid position changes, creating jank.

**When to use:** Dragging between tightly-packed items with varying heights.

### Accessibility for Drag Handles (#207)

Always provide screen reader context:

```erb
<button class="bubbles-list__handle btn btn--reversed">
  <%= image_tag "drag.svg", aria: { hidden: true }, size: 16 %>
  <span class="for-screen-reader">Drag to change number of bubbles shown above</span>
</button>
```

**Best practices:**
- Make handle a `<button>` with descriptive hidden text
- Mark decorative icon as `aria-hidden`
- Describe the action outcome, not just "drag handle"

### Progressive Installation (#207)

Show drag UI only after JavaScript loads:

```javascript
connect() {
  this.install()
}

install() {
  this.#moveDividerTo(this.startCountValue)
  this.dividerTarget.classList.add(this.installedClass)
}
```

```css
.bubbles-list__divider {
  visibility: hidden; /* Hidden by default */
}

.bubbles-list__divider--installed {
  visibility: visible; /* Show when JS ready */
}
```

**Also restore after morphs:**
```javascript
data: {
  action: "turbo:morph@document->divider#install"
}
```

**Why:** Prevents flash of non-functional UI. Only show when interaction is ready.

### Using @rails/request.js with Turbo (#607)

Make `@rails/request.js` use Turbo's fetch for proper integration:

```javascript
// app/javascript/application.js
window.fetch = Turbo.fetch
```

**Why:** Ensures request library respects Turbo's request interception/error handling.

**When to use:** When using `post()` from `@rails/request.js` in Stimulus controllers.

### Single-Purpose Drop Controllers (#607)

Create focused controllers for specific drop behaviors:

```ruby
# app/controllers/cards/drops_controller.rb
class Cards::DropsController < ApplicationController
  def create
    @card = find_card(params[:dropped_item_id])

    case params[:drop_target]
    when "considering"
      @card.reconsider
    when "doing"
      @card.engage
    end

    render_column_replacement
  end
end
```

**Why not reuse existing controllers?**
- Drop handling has unique response requirements (Turbo Streams)
- Mixing with traditional CRUD muddies responsibility
- Dedicated endpoint makes intent clear

**Route as nested resource:**
```ruby
namespace :cards do
  resources :drops
end
```

### Prevent Nested Drag Conflicts (#607)

Disable dragging on nested interactive elements:

```erb
<%= link_to path, draggable: false, class: "card__link" %>
```

**Why:** Links, buttons inside draggable items can trigger unwanted drags.

**Apply to:** Links, nested drag handles, buttons within draggable containers.

## Stimulus for Cached Fragment Personalization (#124)

Cached partials can't access `Current.user`. Move user-specific styling to client-side:

```javascript
// app/javascript/controllers/created_by_current_user_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["creation"]
  static classes = ["mine"]

  creationTargetConnected(element) {
    if (element.dataset.creatorId == Current.user.id) {
      element.classList.add(this.mineClass)
    }
  }
}

// app/javascript/initializers/current.js
class Current {
  get user() {
    const id = document.head.querySelector('meta[name="current-user-id"]')?.content
    return id ? { id: parseInt(id) } : null
  }
}
window.Current = new Current()
```

```erb
<!-- In application.html.erb -->
<meta name="current-user-id" content="<%= Current.user&.id %>">

<!-- Cached partial uses data attributes, not conditionals -->
<div data-creator-id="<%= comment.creator_id %>"
     data-created-by-current-user-target="creation">
```

**Use cases**: Highlighting own content, showing edit/delete buttons, user-specific badges in cached content.
