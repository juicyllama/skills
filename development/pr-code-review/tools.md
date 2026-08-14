# Static-analysis tool catalogue

The tools War Room can run for a `pr-code-review` pass. Step 2 of [`SKILL.md`](SKILL.md) selects from this list and emits the ids in a JSON array; War Room probes each one, runs the ones it has, and hands the normalised findings back.

**Use the `id` column verbatim.** An id not in this table is dropped with a warning, so a typo silently costs you a whole tool's coverage.

Selection rules live in the skill; this file is the menu, not the method. The short version: select a tool when the PR changed files it analyses, prefer whichever tool the repo has already configured, and do not stack overlapping linters on the same files.

---

## Cross-cutting — consider on almost every PR

What these catch does not care what language it is written in.

| id | Tool | Reviews | Select when |
|---|---|---|---|
| `semgrep` | Semgrep | Semantic pattern matching for bugs and vulnerabilities across ~30 languages. | Almost always. Especially when the diff touches auth, input handling, queries, or network calls. |
| `opengrep` | OpenGrep | Open-source Semgrep fork; same rule model. | The repo standardised on OpenGrep, or Semgrep is unavailable. Do not select both. |
| `trufflehog` | TruffleHog | Verified secret detection in code and history. | Any diff that adds config, env handling, credentials, client setup, or CI. |
| `betterleaks` | Betterleaks | Secret and credential leak detection. | As a second secret opinion when the diff is credential-heavy. Skip if `trufflehog` covers it. |
| `osv-scanner` | OSV Scanner | Known vulnerabilities in dependencies, via lockfiles. | Any lockfile or manifest changed. |
| `trivy` | Trivy | Vulnerabilities and misconfiguration in dependencies, containers, and IaC. | Dockerfiles, Kubernetes manifests, IaC, or lockfiles changed. |
| `checkov` | Checkov | IaC misconfiguration and policy (Terraform, CloudFormation, K8s, Helm, ARM). | Any infrastructure-as-code changed. |
| `presidio` | Microsoft Presidio Analyzer | High-signal PII in changed files. | The diff touches logging, analytics, exports, fixtures, or user-data handling. |
| `ast-grep` | ast-grep | Structural AST search for project-specific anti-patterns. | The repo ships `sgconfig.yml` or its own ast-grep rules. |
| `github-checks` | GitHub Checks | The PR's own CI check runs and their failure output. | Always — a red check is review-relevant context, and a check nobody read is a missed finding. |

## JavaScript / TypeScript

| id | Tool | Reviews | Select when |
|---|---|---|---|
| `eslint` | ESLint | Correctness, bugs, and project rules for JS/TS. | `.eslintrc*` or `eslint.config.*` exists and JS/TS files changed. |
| `biome` | Biome | Fast lint + format for JS/TS/JSON. | `biome.json`/`biome.jsonc` exists and JS/TS files changed. Do not stack with `eslint`. |
| `oxlint` | Oxlint | Very fast JS/TS correctness linting. | `.oxlintrc.json` exists, or as the fallback when no JS linter is configured. |
| `react-doctor` | React Doctor | React-specific anti-patterns: hook misuse, render-loop risks, effect dependencies. | `.jsx`/`.tsx` files changed in a React codebase. |
| `ember-template-lint` | ember-template-lint | Ember Handlebars template correctness and accessibility. | `.hbs` files changed in an Ember app. |

## Python

| id | Tool | Reviews | Select when |
|---|---|---|---|
| `ruff` | Ruff | Fast lint covering most Flake8/pylint rules. | `.py` changed. The default Python choice; prefer it over `flake8`. |
| `flake8` | Flake8 | Style and error checking. | `.flake8`/`setup.cfg` config exists and Ruff does not. |
| `pylint` | Pylint | Deeper static analysis: type inference, design smells. | `.pylintrc` exists, or the diff adds substantial new Python structure. |

## Go / Rust / C / C++ / JVM / Swift

| id | Tool | Reviews | Select when |
|---|---|---|---|
| `golangci-lint` | golangci-lint | Aggregated Go linters. | `.go` changed. |
| `clippy` | Clippy | Rust correctness and idiom lints. | `.rs` changed. |
| `clang` | Clang / clang-tidy | C/C++ correctness, UB, and modernisation. | `.c`/`.cc`/`.cpp`/`.h`/`.hpp` changed. |
| `cppcheck` | Cppcheck | C/C++ bug detection, complementary to clang-tidy. | As above, when deeper bug-hunting is warranted. |
| `pmd` | PMD | Java/Apex/XML bug patterns and complexity. | `.java`/`.cls` changed. |
| `detekt` | detekt | Kotlin static analysis and complexity. | `.kt`/`.kts` changed. |
| `swiftlint` | SwiftLint | Swift style and correctness. | `.swift` changed. |

## PHP / Ruby

| id | Tool | Reviews | Select when |
|---|---|---|---|
| `phpstan` | PHPStan | PHP type-level static analysis. | `.php` changed. The strongest PHP signal — prefer it. |
| `phpcs` | PHP CodeSniffer | PHP coding-standard conformance. | `phpcs.xml*` exists and `.php` changed. |
| `phpmd` | PHPMD | PHP mess detection: complexity, unused code, naming. | `.php` changed and the diff adds substantial structure. |
| `rubocop` | RuboCop | Ruby correctness and style. | `.rb` changed. |
| `brakeman` | Brakeman | Rails security vulnerability scanning. | A Rails app and the diff touches controllers, models, views, or routes. |

## Shell / Lua / PowerShell / Make

| id | Tool | Reviews | Select when |
|---|---|---|---|
| `shellcheck` | ShellCheck | Shell script bugs, quoting, and portability. | `.sh`/`.bash`/`.zsh` changed, or a shell script without an extension. |
| `luacheck` | Luacheck | Lua correctness and globals. | `.lua` changed. |
| `psscriptanalyzer` | PSScriptAnalyzer | PowerShell correctness and best practice. | `.ps1`/`.psm1`/`.psd1` changed. |
| `checkmake` | checkmake | Makefile correctness and convention. | `Makefile`/`*.mk` changed. |

## Data, schema, and API contracts

| id | Tool | Reviews | Select when |
|---|---|---|---|
| `sqlfluff` | SQLFluff | SQL linting and dialect conformance. | `.sql` changed. |
| `squawk` | Squawk | Postgres migration safety — locks, rewrites, destructive DDL. | A **`.sql`** migration changed. **High value: a bad migration is a production outage.** |
| `typeorm-migration` | TypeORM migration safety | The same hazards Squawk catches, read out of the SQL inside a TypeORM `.ts` migration: dropped tables and columns, unbounded DML, indexes without `CONCURRENTLY`, type changes, `NOT NULL` without a default, constraints without `NOT VALID`, renames, and empty `down()` rollbacks. | A TypeORM migration changed. Squawk cannot read `.ts`, so without this a TypeORM codebase gets **no** migration safety analysis at all. Built in — always available, nothing to install. |
| `prisma-lint` | Prisma Lint | Prisma schema conventions and modelling. | `schema.prisma` changed. |
| `buf` | Buf | Protobuf lint and **breaking-change detection**. | `.proto` changed. |
| `oasdiff` | oasdiff | OpenAPI spec diffing and **breaking-change detection**. War Room fetches the base revision of each spec out of git and compares it against the PR's version. | An OpenAPI/Swagger spec changed. A spec that is new in this PR is skipped — there is nothing to break yet. |

## Config, CI, containers, infrastructure

| id | Tool | Reviews | Select when |
|---|---|---|---|
| `actionlint` | actionlint | GitHub Actions workflow correctness, expressions, shell steps. | `.github/workflows/**` changed. |
| `zizmor` | zizmor | GitHub Actions **security** audit: injection, over-broad permissions, untrusted checkout. | `.github/workflows/**` changed. Pair with `actionlint`; they catch different things. |
| `hadolint` | Hadolint | Dockerfile best practice and correctness. | A Dockerfile changed. |
| `yamllint` | YAMLlint | YAML syntax and structure. | `.yml`/`.yaml` changed and no more specific tool covers it. |
| `tflint` | TFLint | Terraform correctness and provider-specific rules. | `.tf` changed. Pair with `checkov` for policy. |
| `dotenv-linter` | dotenv-linter | `.env` file correctness and duplicate/leaked keys. | `.env*` changed. |

## Docs, markup, styles, prose

| id | Tool | Reviews | Select when |
|---|---|---|---|
| `markdownlint` | markdownlint | Markdown structure and consistency. | `.md`, `.markdown`, or `.mdx` changed. Keep its findings as nitpicks. On an MDX site it parses JSX as inline HTML, so MD033 fires constantly unless the repo disables it — prefer `languagetool` there. |
| `languagetool` | LanguageTool | Grammar and spelling in prose. | User-facing docs, README, or release notes changed — including `.mdx`. Nitpicks only. |
| `htmlhint` | HTMLHint | HTML correctness and accessibility basics. | `.html` changed. |
| `stylelint` | Stylelint | CSS/SCSS/Less correctness. | `.css`/`.scss`/`.less` changed. |
| `smarty-lint` | smarty-lint | Smarty template correctness. | `.tpl` changed. |
| `shopify-theme-check` | Shopify Theme Check | Liquid theme correctness and performance. | A Shopify theme's `.liquid` files changed. |

## Specialist

| id | Tool | Reviews | Select when |
|---|---|---|---|
| `regal` | Regal | Rego (OPA) policy linting. | `.rego` changed. |
| `fortitude` | Fortitude | Fortran linting. | `.f90`/`.f` changed. |
| `blinter` | Blinter | Windows batch script linting. | `.bat`/`.cmd` changed. |
| `skillspector` | SkillSpector | Agent skill definition review — structure, tool-use safety, prompt quality. | `SKILL.md` or agent/skill definitions changed. **Relevant to this repository.** |

---

## Where these tools come from

They are **not** War Room dependencies and are deliberately absent from its `package.json`. War Room orchestrates other people's repositories; carrying a Go, Rust, PHP, and Ruby toolchain so it can review a TypeScript PR would be absurd, and most of these are not npm packages at all.

Two tiers, and War Room resolves them in this order:

1. **The repository's own toolchain** — `node_modules/.bin`, `vendor/bin`, `.venv/bin`, `bin/`, searched from the checkout upward so a package inside a monorepo also finds the workspace root's. This is *precedence, not fallback*: a linter that loads the repo's config needs that repo's plugins and the version they were written against. A globally-installed ESLint pointed at a config importing `eslint-plugin-x` from the project's `node_modules` simply errors.
2. **The host's `PATH`** — for the self-contained scanners no project vendors: `semgrep`, `trufflehog`, `osv-scanner`, `trivy`, `shellcheck`, `hadolint`, `actionlint`, `zizmor`, `checkov`, `tflint`, `squawk`, `buf`, `ast-grep`, `yamllint`.

So the config-driven linters (ESLint, Biome, Stylelint, markdownlint, PHPStan, PHPCS, RuboCop, detekt, ember-template-lint, prisma-lint) come from whichever repo is being reviewed, with no action needed as long as it installs its own dev dependencies. The cross-cutting scanners are installed once per machine — `mise.toml` is the natural place to declare them so every operator gets the same set.

Nothing here is required. A tool that is absent is skipped, named in the review's coverage section, and costs only its own analysis — never the review. Selection is about relevance; availability is War Room's problem.

## Notes on cost and overlap

- **Overlap groups** — pick one from each: (`semgrep` | `opengrep`), (`eslint` | `biome` | `oxlint`), (`ruff` | `flake8`), (`trufflehog` | `betterleaks`). Stacking them triples the findings and the bill without triple the coverage.
- **Complement pairs** — deliberately select both: `actionlint` + `zizmor` (correctness vs security), `tflint` + `checkov` (correctness vs policy), `clang` + `cppcheck` (different bug classes), `phpstan` + `phpcs` (types vs standards).
- **The highest-value selections** on a typical PR are the ones nobody thinks of: `squawk` on a migration, `buf`/`oasdiff` on a contract change, `zizmor` on a workflow, `osv-scanner` on a lockfile. A missed breaking change costs far more than a missed lint rule.
