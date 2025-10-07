# OTW Archive (Archive of Our Own) - AI Agent Guidelines

## Project Overview

The OTW Archive is a Ruby on Rails 7.2 application powering Archive of Our Own (AO3), a multifandom fanworks archive. This is a complex, mature codebase serving millions of users with fanfiction, fanart, and other transformative works.

## Architecture & Core Concepts

### Domain Model Hierarchy
- **Works**: Central content model with chapters, series, collections
- **Users**: Via pseuds (pen names), can create works, bookmarks, comments
- **Tags**: Complex taxonomy - Archive Warnings, Fandoms, Characters, Relationships, Freeforms
- **Collections**: Groups of works (challenges, gift exchanges, themed collections)
- **Comments/Kudos**: Community interaction on works

### Key Architectural Patterns

**Concerns are Core**: Heavy reliance on `app/models/concerns/` and `lib/` modules:
- `Taggable`: Universal tagging system used across models
- `Searchable`: Elasticsearch integration with `IndexQueue` for async indexing
- `Filterable`: Complex filtering system for works/tags
- `Creatable`: User/pseud attribution patterns
- `WorksOwner`: Shared behavior between Users and Pseuds

**Custom Libraries in `lib/`**:
- `HtmlCleaner`/`CssCleaner`: Security-critical sanitization
- `BookmarkCountCaching`/`WorkChapterCountCaching`: Performance optimizations
- `ChallengeCore`: Gift exchange and prompt meme logic

### Database & External Services
- **Primary**: MySQL with complex foreign key relationships
- **Search**: Elasticsearch for works/tags/users (models auto-enqueue via `Searchable`)
- **Cache**: Redis for sessions, Resque job queues, hit counters
- **Background Jobs**: Resque workers for indexing, notifications, cleanup

## Development Workflows

### Testing Strategy
```bash
# RSpec unit tests
bundle exec rspec spec/models/work_spec.rb

# Cucumber integration tests (organized by feature)
bundle exec cucumber features/works/work_create.feature

# Full test suite (runs in CI)
bundle exec rake
```

### Database Management
```bash
# Use custom rake tasks for data operations
bundle exec rake work:update_word_counts
bundle exec rake tag:fix_meta_tags
bundle exec rake search:index_works
```

### Search Reindexing
Models with `Searchable` auto-queue for reindexing. Manual reindexing:
```ruby
Work.reindex_all   # Queues all works for background reindexing
Tag.reindex_all(:high)  # High priority queue
```

## Code Conventions

### Model Patterns
- Use concerns for shared behavior (see `app/models/concerns/`)
- All user-generated content goes through sanitization (`HtmlCleaner`)
- Association callbacks handle complex denormalization (bookmark counts, etc.)
- Models implement `Searchable` for Elasticsearch integration

### Tagging System
```ruby
# Tags have inheritance: Fandom < Media, Character < Fandom
work.fandom_tags     # Polymorphic through taggings
work.tag_string = "Harry Potter, Hermione Granger, Romance"  # Mass assignment
work.tags.canonical # Only approved/canonical tags
```

### Permission Model
- Role-based authorization with `acts_as_authorized_user`
- Per-object policies in `app/policies/`
- Admin controllers in `app/controllers/admin/`

### Form Handling
- Complex nested forms (works with chapters, tags, relationships)
- Heavy use of `accepts_nested_attributes_for`
- Custom form builders for tag inputs

## Critical Gotchas

1. **Tag Wrangling**: Complex canonical/synonym tag relationships. Changes require tag admin privileges.

2. **Search Indexing**: Never directly manipulate Elasticsearch. Use model callbacks or `IndexQueue.enqueue_ids()`.

3. **User vs Pseud**: Users create content through Pseuds. Always check `work.pseuds.first.user` for ownership.

4. **Security**: All user content must pass through `HtmlCleaner` and `CssCleaner`. Never bypass sanitization.

5. **Background Jobs**: Use Resque for heavy operations. Models with `AsyncWithResque` auto-queue certain operations.

6. **Deployment**: Uses Capistrano. Check `config/deploy.rb` for staging/production configurations.

## Testing Patterns

- **Feature tests**: Use Cucumber scenarios in `features/` organized by major user workflows
- **Model specs**: Focus on business logic, associations, validations
- **Controller specs**: Test authorization, parameter handling, redirects
- **Integration tests**: Database + search integration via `spec/requests/`

## Contributing Context

This is open-source software for a nonprofit. Changes must consider:
- Performance impact on high-traffic site
- Accessibility compliance
- Internationalization (50+ locales supported)
- Content policy and safety implications
- Tag wrangling volunteer workflows

Always reference existing patterns in the codebase rather than introducing new architectures.