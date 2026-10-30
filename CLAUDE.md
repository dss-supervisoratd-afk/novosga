# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

NovoSGA is a customer queue (ticket/"senha") management system built on Symfony 7.4 / PHP 8.5, Doctrine ORM 3, running on FrankenPHP. It supports MySQL and PostgreSQL. The domain language is Portuguese (e.g. `Atendimento` = service/ticket, `Unidade` = branch/unit, `Servico` = service, `Usuario` = user, `Lotacao` = user↔unit role assignment, `Painel` = display panel, `Fila` = queue, `Triagem` = triage). Translations: en, es, pt_BR (`translations/`). Default branch for this line is `v2.4`.

## Commands

```bash
composer install

# OAuth2 keys are required (tests and API won't boot without them)
mkdir -p config/jwt
openssl genrsa -out config/jwt/private.pem 2048
openssl rsa -in config/jwt/private.pem -pubout -out config/jwt/public.pem

# Database (DATABASE_URL in .env / .env.local; compose.yaml provides postgres + mercure)
bin/console doctrine:migrations:migrate --no-interaction
bin/console doctrine:fixtures:load --no-interaction --purge-exclusions=view_atendimentos --purge-exclusions=view_atendimentos_codificados

# Checks (all run in CI, against both MySQL 8 and PostgreSQL 15)
vendor/bin/phpcs              # PSR-12 + Slevomat unused-uses
vendor/bin/phpstan            # level 6, covers public/ src/ tests/
bin/console lint:yaml config
bin/console lint:yaml translations
vendor/bin/phpunit
vendor/bin/phpunit tests/Service/AtendimentoServiceTest.php
vendor/bin/phpunit --filter testMethodName

# App-specific commands
bin/console novosga:install | novosga:update | novosga:check | novosga:reset
```

Tests are functional/integration tests that hit a real database: run migrations and load fixtures first (the fixture purge must exclude the two DB views, as above). `phpunit.dist.xml` sets `failOnDeprecation`/`failOnNotice`/`failOnWarning`, so any deprecation fails the suite. `tests/TestHelper.php` holds the shared helpers for creating units/services/users, cleaning test data, and issuing OAuth access tokens for API tests.

## Architecture

**Core contracts live outside this repo.** `novosga/core` defines the entity interfaces (`Novosga\Entity\*Interface`), repository interfaces, service interfaces and domain events. This app supplies the concrete implementations, and `config/services.yaml` explicitly aliases each `Novosga\Repository\*Interface` / `Novosga\Service\*Interface` to its `App\...` class. When adding a shared service or repository that bundles must consume, add the alias there.

**Feature modules are separate bundles** (`novosga/attendance-bundle`, `triage-bundle`, `monitor-bundle`, `panel-bundle`, `reports-bundle`, `scheduling-bundle`, `settings-bundle`, `users-bundle`, `customers-bundle`) installed via Composer as `2.4.x-dev` and registered in `config/bundles.php`. Bundles implementing `Novosga\Module\ModuleInterface` are discovered at runtime:
- `src/Loader/RouterLoader.php` (route type `novosga.modules`) imports each module's `Controller/` attributes and prefixes routes with the module key name. This is why `config/routes.yaml` deliberately uses the old `resource: {path, namespace}` style instead of `routing.controllers`.
- `ModuleService` lists installed modules; `Security/Voter/ModuleAccessVoter` gates access per user profile (`Perfil`) and unit.

**Ticket lifecycle** is centralized in `src/Service/AtendimentoService.php` (create, call, start, finish, redirect, transfer, cancel, reset). It dispatches `Novosga\Event\Ticket*Event`s. Side effects hang off those events:
- `WebhookSubscriber` → `App\Message\SendWebhookMessage` → Messenger `async` transport (in-memory in test) → `SendWebhookMessageHandler`/`WebhookDispatcher`. Consumers run via `etc/consumer.sh`.
- `MercureService` publishes real-time updates on topics like `/unidades/{id}/fila`, `/usuarios/{id}/fila`, and panel topics for the display screens.
- Queue ordering is pluggable: `QueueOrderingSubscriber` delegates to `QueueOrderingServiceInterface` (default `DefaultQueueOrderingService`).

**Persistence notes**
- Live tickets are in `atendimentos`; archived ones live in history tables (`historico_atendimentos`, …). `ViewAtendimento` / `ViewAtendimentoCodificado` are read-only entities mapped to DB views (`view_atendimentos*`). Fixture loads must exclude these views from purging.
- `migrations/` has Doctrine migration classes plus `migrations/sql/` with per-engine SQL (`*.mysql.sql` and `*.postgres.sql`). Schema changes need both variants.
- Custom DQL functions and types are in `src/Doctrine/`; soft-delete and timestamp behavior come from the `SoftDeletable*`/`Timestampable*` traits and `EventListener/TimestampableEntityListener`.
- Entities with `*Meta` siblings store key/value metadata through the `*MetadataRepository` classes.

**HTTP surfaces / security** (`config/packages/security.yaml`):
- `/api/*` (and module-scoped `/<module>/api`) — stateless OAuth2 (`league/oauth2-server-bundle`), token at `/api/token`. Controllers in `src/Controller/Api/`, many built on `ApiCrudController`.
- `/admin/*` — `ROLE_ADMIN`, `src/Controller/Admin/`.
- `/painel` — public display panel.
- Everything else — session login via `LoginFormAuthenticator`.
- Frontend is Twig + Symfony UX (Stimulus, Turbo, Icons) under `templates/` and `assets/`.

## Conventions

- Every PHP file starts with `declare(strict_types=1);` and the NovoSGA license header block; classes carry an `@author` docblock. Follow the surrounding style.
- Keep controllers thin; put logic in `src/Service/`.
- Never change schema without a migration that works on both MySQL and PostgreSQL.
