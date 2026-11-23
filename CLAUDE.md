# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

This is a freshly generated Rails 8.1 app (Ruby 4.0.10, see `.ruby-version`) intended to become a CRM. There are no domain models, controllers, or routes yet beyond the `/up` health check, and no root route is defined. The README is still the Rails placeholder.

## Commands

- Setup (install gems, prepare DB, start server): `bin/setup`. Use `bin/setup --skip-server` to skip starting the server.
- Dev server: `bin/dev`, which just runs `bin/rails server`. No JS/CSS build step is needed.
- Full local CI (setup, rubocop, bundler-audit, importmap audit, brakeman, tests, seed replant): `bin/ci`, defined in `config/ci.rb`
- Tests: `bin/rails test`
- Single file: `bin/rails test test/models/foo_test.rb`
- Single test by line: `bin/rails test test/models/foo_test.rb:42`
- System tests (Capybara + Selenium; these are not part of `bin/ci`): `bin/rails test:system`
- Lint: `bin/rubocop` (add `-a` to autocorrect). The style is `rubocop-rails-omakase`, which wants spaces inside array brackets, e.g. `[ a, b ]`.
- Security scans: `bin/brakeman`, `bin/bundler-audit`, `bin/importmap audit`
- Seeds must stay idempotent, because CI runs `RAILS_ENV=test bin/rails db:seed:replant`.

CI runs on CircleCI (`.circleci/config.yml`), not GitHub Actions. It has four parallel jobs on `cimg/base:stable`: `lint`, `security` (brakeman, bundler-audit, importmap audit), `test` (unit tests plus seed replant), and `system-test` (Chrome via the `setup-browser-tools` function). Every job installs Ruby from `.ruby-version` and gems with the `setup-ruby` function, which also caches them. A commented-out Smarter Testing block in the `test` job explains how to enable it later. Run `circleci config validate` after editing the config.

## Architecture

- **Stack**: Rails "omakase" with no Node toolchain. It uses Propshaft for assets, importmap-rails for JS (pins go in `config/importmap.rb`), and Hotwire (Turbo + Stimulus). Stimulus controllers live in `app/javascript/controllers/` and are auto-registered by `index.js`. Jbuilder is available for JSON views.
- **Database**: SQLite everywhere, with files under `storage/`. In production there are four databases (see `config/database.yml`): `primary`, plus `cache`, `queue`, and `cable` for Solid Cache, Solid Queue, and Solid Cable. Those three use their own schema files (`db/cache_schema.rb`, `db/queue_schema.rb`, `db/cable_schema.rb`) and migration paths (`db/*_migrate`). App migrations go in `db/migrate` as normal.
- **Background jobs**: Active Job runs on Solid Queue in production. Run workers with `bin/jobs`, or inside Puma by setting `SOLID_QUEUE_IN_PUMA`. Recurring jobs are configured in `config/recurring.yml`.
- **Auth**: none yet. `bcrypt` is commented out in the Gemfile, so enable it if you add `has_secure_password` or the Rails auth generator.
- **Tests**: Minitest with fixtures (`fixtures :all`). Tests run in parallel across processors (`test/test_helper.rb`).
- **Deployment**: Kamal (`config/deploy.yml`, `.kamal/secrets`) builds the Docker image from `Dockerfile`, and Thruster fronts Puma. Production DB files live on the persistent `storage/` volume.
