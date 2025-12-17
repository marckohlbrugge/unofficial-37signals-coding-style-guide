# AI/LLM Integration Patterns

> Patterns from 37signals for integrating AI/LLM features into Rails apps.

---

## Command Pattern with STI (#460, #464, #466)

Use Single Table Inheritance for command objects:

```ruby
# Base command class
class Command < ApplicationRecord
  belongs_to :user

  def execute
  end

  def undo
  end

  def undo!
    transaction do
      undo
      destroy
    end
  end

  def undoable?
    false
  end

  def needs_confirmation?
    false
  end

  private
    def redirect_to(...)
      Command::Result::Redirection.new(...)
    end
end

# Specific command using STI
class Command::Assign < Command
  include Command::Cards

  store_accessor :data, :assignee_ids, :toggled_assignees_by_card

  validates_presence_of :assignee_ids

  def execute
    toggled_assignees_by_card = {}

    transaction do
      cards.find_each do |card|
        # Track changes for undo
      end
      update! toggled_assignees_by_card: toggled_assignees_by_card
    end
  end

  def undo
    transaction do
      toggled_assignees_by_card.each do |card_id, assignee_ids|
        # Reverse the changes
      end
    end
  end
end
```

**Key insight**: Commands save themselves AFTER execution, not before. Validates during parsing but only persists successful executions.

## Context Objects for Parsing (#460)

Extract URL context for command awareness:

```ruby
class Command::Parser::Context
  attr_reader :user

  def initialize(user, url:)
    @user = user
    extract_url_components(url)
  end

  def cards
    if controller == "cards" && action == "show"
      user.accessible_cards.where id: params[:id]
    elsif controller == "cards" && action == "index"
      filter.cards
    end
  end

  private
    def extract_url_components(url)
      uri = URI.parse(url || "")
      route = Rails.application.routes.recognize_path(uri.path)
      @controller = route[:controller]
      @action = route[:action]
      @params = ActionController::Parameters.new(
        Rack::Utils.parse_nested_query(uri.query)
          .merge(route.except(:controller, :action))
      )
    end
end
```

Use `Rails.application.routes.recognize_path` to extract controller/action/params from URL strings.

## Cost Tracking in Microcents (#978)

Track LLM costs precisely using microcents (1/1,000,000 of a dollar):

```ruby
def summarize
  response = chat.ask("Summarize...")
  [response.content, calculate_cost_in_microcents(response)]
end

# Usage
summary, cost = Event::Summarizer.new(events).summarize

Event::ActivitySummary.create!(
  content: summary,
  cost_in_microcents: cost
)
```

Naming: Use `_in_` particle for unit clarity: `cost_in_microcents` not `cost_microcents`.

## Result Objects for Responses (#460, #857)

Use lightweight result objects instead of coupling to HTTP:

```ruby
Command::Result::Redirection = Struct.new(:url)
Command::Result::ShowModal = Struct.new(:turbo_frame, :url)

# Controller pattern matching
def respond_with_execution_result(result)
  case result
  when Command::Result::Redirection
    redirect_to result.url
  when Command::Result::ShowModal
    render json: { turbo_frame: result.turbo_frame, url: result.url },
           status: :accepted
  else
    redirect_back_or_to root_path
  end
end
```

## Tool Pattern for LLM Function Calling (#857)

```ruby
class Ai::Tool < RubyLLM::Tool
  include Rails.application.routes.url_helpers

  private
    def paginated_response(records, page:, ordered_by:, per_page: nil, &block)
      page = GearedPagination::Recordset.new(records, ordered_by: ordered_by, per_page: per_page).page(page)

      response = ["There are #{page.recordset.records_count} records in total."]

      if page.only? || page.last?
        response << "This is the last page of results."
      else
        response << "This is one page of results."
        response << "To see more, use this cursor for the next page:"
        response << "```"
        response << page.next_param
        response << "```"
      end

      response << nil
      response << "Records:"
      response << records_to_json(page.records, &block)

      response.join("\n")
    end
end

# Specific tool with DSL
class Ai::ListCardsTool < Ai::Tool
  description <<-MD
    Lists all cards accessible by the current user.
    The response is paginated.
  MD

  param :page,
    type: :string,
    desc: "Which page to return",
    required: false

  attr_reader :user

  def initialize(user:)
    @user = user
  end

  def execute(**params)
    cards = Card.where(collection: user.collections)
    paginated_response(cards, page: params[:page], ordered_by: { id: :desc })
  end
end
```

**Key insight**: User-scoped tools prevent data leakage.

## Confirmation Pattern for Bulk Operations (#464)

```ruby
# Controller
def create
  command = parse_command(params[:command])

  if command.valid?
    if confirmed?(command)
      command.save!
      result = command.execute
      respond_with_execution_result(result)
    else
      render plain: command.title, status: :conflict
    end
  else
    head :unprocessable_entity
  end
end

def confirmed?(command)
  !command.needs_confirmation? || params[:confirmed].present?
end

# Command concern
module Command::Cards
  def needs_confirmation?
    cards.many?  # Auto-confirm single item, confirm bulk
  end
end
```

Uses HTTP 409 Conflict for "needs confirmation" state - stateless, no server-side session needed.

## Filter Registry Pattern (#857)

```ruby
class Ai::Tool::Filter
  def self.filters
    @filters ||= {}
  end

  def self.register_filters(filters_hash)
    filters_hash.each { |name, method| register_filter(name, method) }
  end

  def filter
    filters.reduce(scope) do |current_scope, (filter_name, value)|
      next current_scope unless value.present?
      next current_scope unless method_name = self.class.filters[filter_name]
      send(method_name, current_scope)
    end
  end
end

# Usage
class CardFilter < Ai::Tool::Filter
  register_filters(
    query: :apply_search,
    golden: :apply_golden_filter,
    created_after: :apply_created_after_filter
  )
end
```

## Order Clause Parser (#857)

Safely parse user-provided sort strings:

```ruby
class OrderClause
  ALLOWED_DIRECTIONS = %i[asc desc].freeze

  def self.parse(value, defaults: nil, permitted_columns: nil)
    new(nil, defaults: defaults, permitted_columns: permitted_columns).tap do |order_clause|
      if value
        value.split(",").each do |clause|
          column, direction = clause.split(" ", 2).map(&:strip)
          order_clause.add(column, direction)
        end
      end
    end
  end

  def add(column, direction)
    raise ArgumentError unless ALLOWED_DIRECTIONS.include?(direction.downcase.to_sym)
    raise ArgumentError unless permitted?(column)
    @order[column] = direction.downcase.to_sym
  end
end

# Usage
ordered_by = OrderClause.parse(
  params[:ordered_by],
  defaults: { id: :desc },
  permitted_columns: %w[id created_at last_active_at]
)
```

Security: whitelist permitted columns, validate direction.

## Code Review Culture

1. **Ship experimental features early** for team feedback
2. **Acknowledge technical debt** in PR descriptions
3. **Small improvements are worth doing** - rename across entire codebase for readability
4. **Provide review guidance** - tell reviewers where to start
