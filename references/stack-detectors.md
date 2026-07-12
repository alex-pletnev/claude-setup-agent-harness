# Stack detectors

Правила определения стека, команды тестов/сборки, эвристики профиля.

## Определение по манифесту

Приоритет — снизу вверх (более специфичный побеждает):

| Файл | Стек | build | test |
|------|------|-------|------|
| `package.json` | Node.js/JS/TS | `npm install` / `pnpm install` / `yarn` | `npm test` / `pnpm test` / `yarn test` (или `test` из scripts) |
| `pyproject.toml` (poetry/PDM/uv) | Python | `poetry install` / `pip install -e .` | `pytest` / `python -m unittest` |
| `requirements.txt` | Python (без POM-инструмента) | `pip install -r requirements.txt` | `pytest` |
| `Cargo.toml` | Rust | `cargo build` | `cargo test` |
| `go.mod` | Go | `go build ./...` | `go test ./...` |
| `build.gradle.kts` / `build.gradle` | JVM (Gradle) — Kotlin/Java/Groovy/Scala | `./gradlew build` | `./gradlew test` |
| `pom.xml` | JVM (Maven) — Java/Kotlin | `mvn compile` | `mvn test` |
| `mix.exs` | Elixir | `mix compile` | `mix test` |
| `Gemfile` | Ruby | `bundle install` | `bundle exec rspec` / `rake test` |
| `composer.json` | PHP | `composer install` | `vendor/bin/phpunit` |
| `pubspec.yaml` | Dart / Flutter | `flutter pub get` / `dart pub get` | `flutter test` / `dart test` |
| `Package.swift` | Swift | `swift build` | `swift test` |
| `*.csproj` / `*.sln` | .NET / C# | `dotnet build` | `dotnet test` |

## Уточнения по манифесту

### Node.js
- `tsconfig.json` — TypeScript.
- `next.config.*` — Next.js (frontend).
- `nest-cli.json` / `@nestjs/*` в deps — NestJS (backend).
- `express` в deps — Express-service.
- `vite.config.*` — Vite (обычно frontend).
- `astro.config.*` — Astro.
- `remix.config.*` — Remix.
- Если есть `pages/` или `app/` рядом с `next.config.*` — Next.js.
- Тестовый фреймворк — по deps: `jest`, `vitest`, `mocha`, `@playwright/test`.

### Python
- Django: `django` в deps, `manage.py` в корне → `python manage.py test` вместо голого pytest.
- FastAPI: `fastapi` в deps → REST-backend.
- Flask: `flask` в deps.
- Poetry: `poetry install`, `poetry run pytest`.
- PDM: `pdm install`, `pdm run pytest`.
- uv: `uv sync`, `uv run pytest`.
- Иначе venv + pip.

### JVM
- Spring Boot: `org.springframework.boot` в `build.gradle*` или `pom.xml`.
- Kotlin: `kotlin("jvm")` в Gradle или `kotlin-stdlib` в Maven.
- `bootRun`, `bootJar` доступны при Spring Boot.

### Go
- Веб-фреймворки в go.mod: `gin`, `echo`, `fiber`, `chi`.
- Стандартный Go project layout: `cmd/`, `internal/`, `pkg/`.

### Rust
- `[[bin]]` — binary crate (CLI/service).
- `[lib]` — библиотека.
- Веб: `actix-web`, `axum`, `rocket`, `warp` в deps.
- Async: `tokio`, `async-std` в deps.

### .NET
- `TargetFramework` в `.csproj` определяет версию.
- Solution vs project — если `.sln`, обрабатывать все проекты.

## Профиль проекта

Определить, что за приложение (эвристики + артефакты):

| Профиль | Признаки | Что это значит для docs |
|---------|----------|-------------------------|
| **Backend service (REST)** | HTTP-framework, routes/controllers, порт в конфиге | `docs/05-api.md` — обязательно |
| **Backend service (GraphQL)** | Схемы `.graphql`/`.gql`, resolver'ы | `docs/05-api.md` под GraphQL |
| **Backend service (RPC/gRPC)** | `.proto` файлы, gRPC-стабы | `docs/05-api.md` под RPC |
| **Frontend SPA** | React/Vue/Svelte/Solid без backend в том же репо | `docs/05-api.md` — только внешние API, которые consuming; `docs/07-ui-components.md` вместо DTO |
| **Frontend SSR** | Next/Nuxt/SvelteKit/Remix | оба API и UI-разделы |
| **CLI** | Одна точка входа с argument parser | `docs/05-cli.md` (команды и флаги) |
| **Библиотека** | Нет entry point приложения, конфиг публикации | `docs/05-public-api.md` (экспортируемые модули/функции), нет `docs/10-configuration.md` |
| **Инфраструктура** | Terraform/Ansible/K8s — основа | `docs/02-domain-model.md` → `docs/02-resources.md`, `docs/05-` может отсутствовать |
| **Monorepo** | Workspace-файл (pnpm-workspace, cargo workspace, go.work, nx.json) | Каждый пакет — свой sub-`docs/`; глобальный `docs/00-monorepo.md` с картой |

## Асинхронные интерфейсы

Проверить наличие для `docs/06-async-api.md`:
- WebSocket (`socket.io`, `ws`, `starscream`, `tokio-tungstenite`, `spring-boot-starter-websocket`).
- Message queues (`kafka`, `rabbitmq`, `redis-streams`, `nats`, `sqs`, `pubsub`).
- SSE (`text/event-stream`, `EventSource`).
- gRPC streaming.

Если ничего нет — не создавать этот файл.

## Персистентность

Для `docs/02-domain-model.md`:
- SQL: PostgreSQL/MySQL/SQLite/MSSQL/Oracle — через driver или ORM.
- ORM: JPA/Hibernate, SQLAlchemy, Prisma, TypeORM, Sequelize, Diesel, GORM, Entity Framework.
- NoSQL: MongoDB, DynamoDB, Cassandra, Redis (как БД).
- Файловая: JSON/YAML-based (упомянуть, но раздел может быть очень коротким).
- Ничего — раздел не создавать.

## Аутентификация

Для `docs/08-security.md`:
- JWT: `jjwt`, `jsonwebtoken`, `PyJWT`, `python-jose`, `jsonwebtokens` (rust).
- OAuth: `passport`, `authlib`, `omniauth`.
- Session-based: Spring Security session, Rails devise.
- API keys: middleware, header-check.
- Нет auth — секцию упомянуть коротко или пропустить.

## CI/CD

Для `docs/10-configuration.md`:
- `.github/workflows/` — GitHub Actions.
- `.gitlab-ci.yml` — GitLab CI.
- `.circleci/config.yml` — CircleCI.
- `Jenkinsfile` — Jenkins.
- `.buildkite/` — Buildkite.

Кратко упомянуть — какая система, какие workflow'ы (по именам).

## Контейнеризация

- `Dockerfile` — упомянуть.
- `compose.yaml` / `docker-compose.yml` — упомянуть, включая сервисы.
- `k8s/`, `deployment.yaml` — Kubernetes.
- `Chart.yaml` — Helm.

## Language config

- `.editorconfig` — форматирование.
- `.prettierrc*`, `.eslintrc*`, `.rubocop.yml`, `ruff.toml`, `pyproject.toml [tool.ruff]`, `.rustfmt.toml`, `.gofmt` (imply) — linters.
- Кратко перечислить в отчёте фазы 1, использовать при написании CLAUDE.md.

## Совсем неопределимый стек

Если ничего из выше не сработало:
- Показать `ls` корня, попросить пользователя коротко описать проект.
- Использовать минимальный набор docs: `README`, `01-overview`, `11-known-gaps`.
- `TEST_COMMAND` / `BUILD_COMMAND` оставить как `_TODO_` в CLAUDE.md.
