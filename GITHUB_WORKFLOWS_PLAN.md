# Plán: i4b-cz/github-workflows

Centrální repozitář s reusable GitHub workflows pro projekty firmy i4b.cz.

## Cíl

Vytvořit sdílené, parametrizované GitHub workflows, které eliminují duplikaci kódu mezi projekty. Každý projekt pak obsahuje pouze minimální "caller" workflow, který volá centrální reusable workflow.

## Změny oproti původnímu návrhu (v2)

- **Konsolidace test workflows**: Místo 4 samostatných workflows (`php-quality`, `php-tests`, `node-quality`, `node-tests`) použijeme 2 kombinované (`php-ci.yml`, `node-ci.yml`) - snižuje duplicitu checkout/install kroků
- **Přidán `claude-code-review.yml`**: Automatické code review pro PR
- **Test scope logic**: Podpora pro spouštění integration testů pouze pro PR do hlavních větví
- **Deploy vylepšení**: Backup, production mode verification, JSON config objekt (obejití limitu 10 inputs)
- **Aktualizace**: `actions/cache@v4`, přidány chybějící features z execution-lite

## Kontext - existující projekty

### 1. CMI (cmi-be)

- **Repo**: `i4b-cz/cmi`
- **Stack**: PHP 8.4 + Symfony 7.x, Vue.js 3 + TypeScript, PrimeVue 4, MariaDB
- **Struktura**: `backend/` (Symfony), `frontend/` (Vue.js)
- **Architektura**: Modulární monolit, multi-tenant SaaS

### 2. Execution-Lite

- **Repo**: `i4b-cz/execution-lite`
- **Stack**: PHP 8.4 + Symfony, Vue.js + TypeScript
- **Struktura**: `backend/`, `frontend/`
- **Existující workflows**:
  - `test.yml` - PHPStan, PHP-CS-Fixer, PHPUnit, Vitest
  - `claude.yml` - Claude Code integration
  - `deploy-staging.yml` - SSH/rsync deploy
  - `deploy-production.yml` - produkční deploy

## Analýza existujících workflows (execution-lite)

### test.yml - co je znovupoužitelné

```yaml
# Backend testy - parametrizovatelné
- Setup PHP (shivammathur/setup-php@v2)
- Composer cache (actions/cache@v3)
- composer install
- Symfony cache warmup
- PHPStan --error-format=github
- PHP-CS-Fixer (composer cs-check)
- Doctrine schema create (pro testy)
- PHPUnit s exclude-group podporou

# Frontend testy - parametrizovatelné
- Setup Node.js (actions/setup-node@v4)
- npm ci
- Vitest --run
```

### deploy-staging.yml - co je projekt-specifické

**Znovupoužitelné:**
- SSH setup (klíč, known_hosts)
- rsync deploy pattern
- Health check pattern

**Projekt-specifické (NELZE sdílet):**
- Generování `.env.prod.local` s konkrétními ENV proměnnými:
  - APP_SECRET, DATABASE_URL
  - CEECR_API_KEY, CEECR_API_SECRET (specifické pro execution-lite)
  - GOPAY_* credentials
  - MAILER_DSN, JWT_PASSPHRASE, atd.
- JWT klíče deploy
- Konkrétní rsync exclude patterns
- Messenger worker restart

### claude.yml - plně znovupoužitelné

```yaml
# Kompletně generické, jen potřebuje CLAUDE_CODE_OAUTH_TOKEN secret
- uses: anthropics/claude-code-action@v1
```

## Navrhovaná struktura repo

```
i4b-cz/github-workflows/
├── .github/
│   └── workflows/
│       ├── php-ci.yml            # PHPStan + PHP-CS-Fixer + PHPUnit (kombinovaný)
│       ├── node-ci.yml           # ESLint + TypeCheck + Vitest/Jest (kombinovaný)
│       ├── claude-code.yml       # Claude Code integration (@claude trigger)
│       ├── claude-code-review.yml # Automatické PR code review
│       └── deploy-ssh.yml        # Generický SSH deploy s backup podporou
│
├── actions/
│   ├── setup-php/
│   │   └── action.yml            # PHP + extensions + composer cache
│   ├── setup-node/
│   │   └── action.yml            # Node + npm cache
│   └── ssh-deploy/
│       └── action.yml            # SSH klíč setup + rsync helper
│
└── README.md
```

## Specifikace reusable workflows

### 1. php-ci.yml

**Účel**: Kompletní PHP CI pipeline - statická analýza + testy v jednom workflow

**Inputs**:
| Input | Type | Default | Popis |
|-------|------|---------|-------|
| `php-version` | string | `'8.4'` | Verze PHP |
| `working-directory` | string | `'./backend'` | Cesta k PHP projektu |
| `php-extensions` | string | `'mbstring, xml, ctype, iconv, intl, pdo_sqlite, dom, filter, gd, json, pdo'` | PHP rozšíření |
| `run-phpstan` | boolean | `true` | Spustit PHPStan |
| `run-cs-fixer` | boolean | `true` | Spustit PHP-CS-Fixer |
| `run-tests` | boolean | `true` | Spustit PHPUnit |
| `test-scope` | string | `'unit'` | `'unit'` nebo `'all'` (unit+integration) |
| `exclude-groups` | string | `'integration'` | PHPUnit exclude groups (použito při scope=unit) |
| `coverage` | boolean | `false` | Generovat coverage report |
| `database` | string | `'sqlite'` | Typ DB pro testy (sqlite/mysql/none) |

**Outputs**:
| Output | Type | Popis |
|--------|------|-------|
| `phpstan-result` | string | `'success'` / `'failure'` / `'skipped'` |
| `cs-fixer-result` | string | `'success'` / `'failure'` / `'skipped'` |
| `tests-result` | string | `'success'` / `'failure'` / `'skipped'` |

**Kroky**:
1. Checkout
2. Setup PHP (shivammathur/setup-php@v2)
   - S xdebug coverage pokud `coverage: true`
3. Composer cache (actions/cache@v4)
4. `composer install --prefer-dist --no-progress --no-interaction`
5. `php bin/console cache:warmup --env=dev` (pro PHPStan)
6. **Conditional**: PHPStan (`vendor/bin/phpstan analyse --error-format=github`)
7. **Conditional**: PHP-CS-Fixer (`composer cs-check`)
8. **Conditional**: Create test database schema
9. **Conditional**: PHPUnit
   - Pokud `test-scope: 'unit'`: `--exclude-group=${{ inputs.exclude-groups }}`
   - Pokud `test-scope: 'all'`: bez exclude
   - S `--testdox --colors=always`
10. **Conditional**: Coverage report (`--coverage-text`)

### 2. node-ci.yml

**Účel**: Kompletní Node.js CI pipeline - lint + typecheck + testy

**Inputs**:
| Input | Type | Default | Popis |
|-------|------|---------|-------|
| `node-version` | string | `'20'` | Verze Node.js |
| `working-directory` | string | `'./frontend'` | Cesta k frontend projektu |
| `package-manager` | string | `'npm'` | `'npm'` nebo `'pnpm'` |
| `run-lint` | boolean | `true` | Spustit ESLint |
| `run-typecheck` | boolean | `true` | Spustit tsc/vue-tsc |
| `run-tests` | boolean | `true` | Spustit testy |
| `test-command` | string | `'npm test -- --run'` | Příkaz pro testy |
| `lint-command` | string | `'npm run lint'` | Příkaz pro lint |
| `typecheck-command` | string | `'npm run type-check'` | Příkaz pro type-check |

**Outputs**:
| Output | Type | Popis |
|--------|------|-------|
| `lint-result` | string | `'success'` / `'failure'` / `'skipped'` |
| `typecheck-result` | string | `'success'` / `'failure'` / `'skipped'` |
| `tests-result` | string | `'success'` / `'failure'` / `'skipped'` |

**Kroky**:
1. Checkout
2. Setup Node.js (actions/setup-node@v4)
   - S cache pro npm/pnpm
3. `npm ci` / `pnpm install --frozen-lockfile`
4. **Conditional**: ESLint
5. **Conditional**: TypeScript check
6. **Conditional**: Testy s `--reporter=verbose`

### 3. claude-code.yml

**Účel**: Claude Code integration pro issues a PR - reaguje na @claude zmínky

**Inputs**:
| Input | Type | Default | Popis |
|-------|------|---------|-------|
| `allowed-tools` | string | `''` | Omezení povolených nástrojů (claude_args) |

**Secrets**:
- `CLAUDE_CODE_OAUTH_TOKEN` (required)

**Triggers** (přednastavené):
- `issue_comment` obsahující `@claude`
- `pull_request_review_comment` obsahující `@claude`
- `pull_request_review` obsahující `@claude`
- `issues` opened/assigned obsahující `@claude`

**Permissions**:
```yaml
permissions:
  contents: read
  pull-requests: read
  issues: read
  id-token: write
  actions: read  # Pro čtení CI výsledků
```

### 4. claude-code-review.yml

**Účel**: Automatické code review pro pull requesty

**Inputs**:
| Input | Type | Default | Popis |
|-------|------|---------|-------|
| `target-branches` | string | `'develop,main'` | Větve pro které se spouští review (comma-separated) |
| `paths-ignore` | string | `'**.md,docs/**'` | Cesty které se ignorují |
| `skip-label` | string | `'skip-review'` | Label nebo text v PR title pro skip |
| `review-prompt` | string | `''` | Custom prompt (použije se default pokud prázdný) |

**Secrets**:
- `CLAUDE_CODE_OAUTH_TOKEN` (required)

**Triggers**:
- `pull_request` na specifikované větve

**Default review prompt**:
```
Analyze the changes in this PR and provide feedback on:
- Code quality and best practices
- Potential bugs or issues
- Performance considerations
- Security concerns
- Missing test coverage
- Adherence to project conventions (see CLAUDE.md)

Format:
🔴 **Critical Issues** - security, data loss bugs
🟡 **Important Suggestions** - performance, missing validation
🟢 **Minor Improvements** - code quality, style
✅ **What Looks Good** - positive feedback
```

**Permissions**:
```yaml
permissions:
  contents: read
  issues: write
  pull-requests: write
  id-token: write
```

### 5. deploy-ssh.yml

**Účel**: Generický SSH deploy pomocí rsync s podporou backup a health checků

**Inputs**:
| Input | Type | Default | Popis |
|-------|------|---------|-------|
| `environment` | string | required | GitHub environment (staging/production) |
| `node-version` | string | `'20'` | Verze Node.js pro frontend build |
| `deploy-config` | string | required | JSON konfigurace (viz níže) |

**Deploy config JSON struktura**:
```json
{
  "backend": {
    "enabled": true,
    "path": "./backend",
    "rsync-exclude": [".git", "node_modules", "vendor", "var/cache", "var/log", "var/pdf", "public/"]
  },
  "frontend": {
    "enabled": true,
    "path": "./frontend",
    "build-command": "npm run build",
    "post-build-commands": ["cp .htaccess.production dist/.htaccess"],
    "deploy-to": "backend/public/"
  },
  "backup": {
    "enabled": false,
    "database": true,
    "commit-hash": true
  },
  "health-check": {
    "enabled": true,
    "url": "/api/config",
    "verify-production-mode": false
  },
  "post-deploy": {
    "composer-install": true,
    "run-migrations": true,
    "clear-cache": true,
    "restart-workers": false,
    "custom-commands": []
  }
}
```

**Secrets**:
- `SSH_KEY` (required)

**Variables** (z GitHub environment):
- `SSH_HOST`, `SSH_USER`, `SSH_PORT`, `REMOTE_PATH`
- `APP_URL` (pro health check a frontend build jako `VITE_API_BASE_URL`)

**Kroky**:
1. Checkout
2. Setup Node.js (pokud frontend enabled)
3. Build frontend (pokud enabled)
   - Vytvoří `.env.production` s `VITE_*` proměnnými
   - Spustí build command
   - Spustí post-build commands
4. Download artifacts (pokud existují - pro `.env.prod.local`, JWT keys)
5. Configure SSH (klíč + known_hosts)
6. **Conditional**: Create backup
   - Database dump (parsuje DATABASE_URL z `.env.prod.local`)
   - Uloží aktuální commit hash
7. Deploy frontend via rsync
8. Deploy backend via rsync
   - Excludes dle konfigurace
   - Zvlášť `public/index.php` a `public/cron/`
9. Deploy config files (`.env.prod.local`)
10. **Conditional**: Post-deploy tasks na serveru
    - `composer install --no-dev --optimize-autoloader`
    - `doctrine:migrations:migrate`
    - `cache:clear && cache:warmup`
    - Custom commands
11. **Conditional**: Health check
    - HTTP check na APP_URL + health-check.url
    - Ověření production mode (pokud enabled)
12. Cleanup SSH klíče

**Poznámka**:
- Generování `.env.prod.local` a JWT klíčů zůstává v projekt-specifickém workflow
- Tyto soubory se předávají jako artifacts mezi jobs
- JSON config umožňuje obejít limit 10 inputs pro reusable workflows

## Příklady použití

### CMI - test.yml

```yaml
name: Tests

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]

jobs:
  # Určení test scope - integration testy jen pro PR do main/develop
  setup:
    runs-on: ubuntu-latest
    outputs:
      test-scope: ${{ steps.scope.outputs.scope }}
    steps:
      - id: scope
        run: |
          if [ "${{ github.event_name }}" == "pull_request" ]; then
            if [ "${{ github.base_ref }}" == "develop" ] || [ "${{ github.base_ref }}" == "main" ]; then
              echo "scope=all" >> $GITHUB_OUTPUT
            else
              echo "scope=unit" >> $GITHUB_OUTPUT
            fi
          else
            echo "scope=unit" >> $GITHUB_OUTPUT
          fi

  php-ci:
    needs: setup
    uses: i4b-cz/github-workflows/.github/workflows/php-ci.yml@v1
    with:
      php-version: '8.4'
      working-directory: ./backend
      test-scope: ${{ needs.setup.outputs.test-scope }}
      coverage: ${{ github.event_name == 'pull_request' }}

  node-ci:
    uses: i4b-cz/github-workflows/.github/workflows/node-ci.yml@v1
    with:
      node-version: '20'
      working-directory: ./frontend

  # Volitelný summary job
  test-summary:
    runs-on: ubuntu-latest
    needs: [php-ci, node-ci]
    if: always()
    steps:
      - name: Check results
        run: |
          echo "## Test Results" >> $GITHUB_STEP_SUMMARY
          echo "- PHP CI: ${{ needs.php-ci.result }}" >> $GITHUB_STEP_SUMMARY
          echo "- Node CI: ${{ needs.node-ci.result }}" >> $GITHUB_STEP_SUMMARY
```

### CMI - deploy-staging.yml

```yaml
name: Deploy to Staging

on:
  push:
    branches: [develop]

jobs:
  # Nejprve projekt-specifická příprava (secrets, JWT klíče)
  prepare:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create .env.prod.local
        run: |
          cat > backend/.env.prod.local << 'EOF'
          APP_SECRET=${{ secrets.APP_SECRET }}
          DATABASE_URL=${{ secrets.DATABASE_URL }}
          # ... další CMI-specifické proměnné
          EOF

      - name: Create JWT keys
        run: |
          mkdir -p backend/config/jwt
          echo "${{ secrets.JWT_PRIVATE_KEY }}" | base64 -d > backend/config/jwt/private.pem
          echo "${{ secrets.JWT_PUBLIC_KEY }}" | base64 -d > backend/config/jwt/public.pem

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: deploy-config
          path: |
            backend/.env.prod.local
            backend/config/jwt/

  # Volání reusable deploy workflow
  deploy:
    needs: prepare
    uses: i4b-cz/github-workflows/.github/workflows/deploy-ssh.yml@v1
    with:
      environment: staging
      deploy-config: |
        {
          "backend": {
            "enabled": true,
            "path": "./backend"
          },
          "frontend": {
            "enabled": true,
            "path": "./frontend",
            "post-build-commands": ["cp .htaccess.production dist/.htaccess"]
          },
          "backup": {
            "enabled": false
          },
          "health-check": {
            "enabled": true,
            "url": "/api/config"
          },
          "post-deploy": {
            "composer-install": true,
            "run-migrations": true,
            "clear-cache": true
          }
        }
    secrets:
      SSH_KEY: ${{ secrets.SSH_KEY }}
```

### CMI - deploy-production.yml

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  prepare:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ... stejné jako staging, jen s production secrets

  deploy:
    needs: prepare
    uses: i4b-cz/github-workflows/.github/workflows/deploy-ssh.yml@v1
    with:
      environment: production
      deploy-config: |
        {
          "backend": { "enabled": true, "path": "./backend" },
          "frontend": { "enabled": true, "path": "./frontend" },
          "backup": {
            "enabled": true,
            "database": true,
            "commit-hash": true
          },
          "health-check": {
            "enabled": true,
            "url": "/api/config",
            "verify-production-mode": true
          },
          "post-deploy": {
            "composer-install": true,
            "run-migrations": true,
            "clear-cache": true,
            "restart-workers": true
          }
        }
    secrets:
      SSH_KEY: ${{ secrets.SSH_KEY }}
```

### CMI - claude.yml

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    uses: i4b-cz/github-workflows/.github/workflows/claude-code.yml@v1
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

### CMI - claude-review.yml

```yaml
name: Claude Code Review

on:
  pull_request:
    branches: [develop]

jobs:
  review:
    uses: i4b-cz/github-workflows/.github/workflows/claude-code-review.yml@v1
    with:
      paths-ignore: '**.md,docs/**'
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

## Implementační postup

### Fáze 1: Základní setup

1. [ ] Vytvořit repo `i4b-cz/github-workflows` na GitHubu
2. [ ] Nastavit jako public (reusable workflows vyžadují public repo nebo GitHub Enterprise)
3. [ ] Vytvořit základní strukturu složek
4. [ ] Přidat CLAUDE.md s pravidly pro AI asistenty

### Fáze 2: Implementace workflows

5. [ ] Implementovat `php-ci.yml` (PHPStan + CS-Fixer + PHPUnit)
6. [ ] Implementovat `node-ci.yml` (Lint + TypeCheck + Tests)
7. [ ] Implementovat `claude-code.yml` (@claude trigger)
8. [ ] Implementovat `claude-code-review.yml` (automatické PR review)
9. [ ] Implementovat `deploy-ssh.yml` (deploy s backup podporou)

### Fáze 3: Composite actions (volitelné)

10. [ ] `actions/setup-php/action.yml` - PHP + extensions + composer cache
11. [ ] `actions/setup-node/action.yml` - Node + npm/pnpm cache
12. [ ] `actions/ssh-deploy/action.yml` - SSH klíč setup + rsync helper

### Fáze 4: Integrace a testování

13. [ ] Vytvořit caller workflows v CMI projektu
14. [ ] Otestovat všechny workflows na CMI
15. [ ] Vytvořit tag `v1` po úspěšném testování
16. [ ] Migrovat execution-lite na centrální workflows

### Fáze 5: Dokumentace

17. [ ] README.md s příklady použití a best practices
18. [ ] CHANGELOG.md pro verzování
19. [ ] Dokumentace všech inputs/outputs pro každý workflow

## Poznámky k implementaci

### Reusable workflow omezení

- Reusable workflows musí být v `.github/workflows/` složce
- Caller workflow může předávat max 10 inputs → řešeno JSON config objektem
- Secrets se musí explicitně předávat (nebo použít `secrets: inherit`)
- Nelze použít `env` kontext v `with` (workaround: použít inputs)
- Outputs z reusable workflow musí být explicitně definovány

### Verzování

Doporučuji používat Git tags pro verzování:
- `@v1` - major verze (stabilní)
- `@v1.2.0` - konkrétní verze
- `@main` - latest (pro vývoj)

### Viditelnost repo

Pro reusable workflows mezi repos ve stejné organizaci:
- **Public repo**: Funguje vždy
- **Internal repo**: Funguje v rámci organizace (GitHub Enterprise)
- **Private repo**: Nefunguje pro cross-repo volání

Doporučení: Vytvořit jako **public** repo.

## Alternativní přístupy (k diskuzi)

### JSON config vs více inputs

**Výhody JSON config**:
- Obchází limit 10 inputs
- Flexibilní struktura
- Snadné rozšiřování

**Nevýhody JSON config**:
- Horší IDE podpora (žádná validace)
- Verbose syntax v YAML
- Složitější parsování v workflow

**Alternativa**: Rozdělit `deploy-ssh.yml` na více menších workflows:
- `deploy-backend.yml`
- `deploy-frontend.yml`
- `post-deploy.yml`

Každý by měl méně inputs a caller workflow by je orchestroval.

### Test scope jako input vs workflow_call trigger

Místo `test-scope` inputu lze použít různé workflow_call eventy:
```yaml
on:
  workflow_call:
  pull_request:
    types: [opened, synchronize]
```

A v caller workflow určit scope podle kontextu.

## Reference

- [GitHub: Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [GitHub: Creating composite actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)
- [GitHub: Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [shivammathur/setup-php](https://github.com/shivammathur/setup-php)
- [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action)
