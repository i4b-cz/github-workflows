# i4b-cz/github-workflows

Centrální repozitář s reusable GitHub workflows pro projekty firmy i4b.cz.

## Bezpečnostní poznámky

> **⚠️ Důležité:** Tyto workflows jsou navrženy pro použití v rámci organizace i4b-cz.
> Některé inputy (např. `lint-command`, `test-command`, `custom-commands`) jsou přímo
> vykonávány jako shell příkazy. **Nikdy nepředávejte nedůvěryhodný vstup** do těchto polí.

- **Secrets**: Všechny citlivé údaje (SSH klíče, tokeny) jsou předávány jako GitHub Secrets
- **Environment variables**: Proměnné serveru (SSH_HOST, REMOTE_PATH) jsou uloženy v GitHub Environments
- **Command execution**: Příkazy v `deploy-config.post-deploy.custom-commands` jsou vykonávány na vzdáleném serveru

## Dostupné workflows

| Workflow | Popis |
|----------|-------|
| [`php-ci.yml`](#php-ciyml) | PHP CI pipeline - PHPStan, PHP-CS-Fixer, PHPUnit |
| [`node-ci.yml`](#node-ciyml) | Node.js CI pipeline - ESLint, TypeCheck, Vitest/Jest |
| [`claude-code.yml`](#claude-codeyml) | Claude Code integration (@claude trigger) |
| [`claude-code-review.yml`](#claude-code-reviewyml) | Automatické PR code review |
| [`deploy-ssh.yml`](#deploy-sshyml) | SSH deploy via rsync s backup podporou |

---

## php-ci.yml

Kompletní PHP CI pipeline zahrnující statickou analýzu a testy.

### Použití

```yaml
jobs:
  php-ci:
    uses: i4b-cz/github-workflows/.github/workflows/php-ci.yml@v1
    with:
      php-version: '8.4'
      working-directory: ./backend
      test-scope: 'all'  # nebo 'unit'
      coverage: true
```

### Inputs

| Input | Type | Default | Popis |
|-------|------|---------|-------|
| `php-version` | string | `'8.4'` | Verze PHP |
| `working-directory` | string | `'./backend'` | Cesta k PHP projektu |
| `php-extensions` | string | `'mbstring, xml, ...'` | PHP rozšíření |
| `run-phpstan` | boolean | `true` | Spustit PHPStan |
| `run-cs` | boolean | `true` | Spustit code style check |
| `cs-tool` | string | `'cs-fixer'` | Nástroj: `'cs-fixer'` nebo `'phpcs'` |
| `run-tests` | boolean | `true` | Spustit PHPUnit |
| `test-scope` | string | `'unit'` | `'unit'` nebo `'all'` |
| `exclude-groups` | string | `'integration'` | PHPUnit exclude groups |
| `coverage` | boolean | `false` | Generovat coverage |
| `database` | string | `'sqlite'` | Typ DB pro testy |

### Outputs

| Output | Popis |
|--------|-------|
| `phpstan-result` | `success` / `failure` / `skipped` |
| `cs-result` | `success` / `failure` / `skipped` |
| `tests-result` | `success` / `failure` / `skipped` |

---

## node-ci.yml

Kompletní Node.js CI pipeline.

### Použití

```yaml
jobs:
  node-ci:
    uses: i4b-cz/github-workflows/.github/workflows/node-ci.yml@v1
    with:
      node-version: '20'
      working-directory: ./frontend
```

### Inputs

| Input | Type | Default | Popis |
|-------|------|---------|-------|
| `node-version` | string | `'20'` | Verze Node.js |
| `working-directory` | string | `'./frontend'` | Cesta k projektu |
| `package-manager` | string | `'npm'` | `'npm'` nebo `'pnpm'` |
| `run-lint` | boolean | `true` | Spustit ESLint |
| `run-typecheck` | boolean | `true` | Spustit type check |
| `run-tests` | boolean | `true` | Spustit testy |
| `lint-command` | string | `'npm run lint'` | Příkaz pro lint |
| `typecheck-command` | string | `'npm run type-check'` | Příkaz pro type check |
| `test-command` | string | `'npm test -- --run'` | Příkaz pro testy |

### Outputs

| Output | Popis |
|--------|-------|
| `lint-result` | `success` / `failure` / `skipped` |
| `typecheck-result` | `success` / `failure` / `skipped` |
| `tests-result` | `success` / `failure` / `skipped` |

---

## claude-code.yml

Claude Code integration - reaguje na @claude zmínky v issues a PR.

### Použití

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

### Inputs

| Input | Type | Default | Popis |
|-------|------|---------|-------|
| `allowed-tools` | string | `''` | Omezení povolených nástrojů |
| `model` | string | `''` | Claude model |

### Secrets

- `CLAUDE_CODE_OAUTH_TOKEN` (required)

---

## claude-code-review.yml

Automatické code review pro pull requesty.

### Použití

```yaml
name: Claude Code Review

on:
  pull_request:
    branches: [develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'

jobs:
  review:
    uses: i4b-cz/github-workflows/.github/workflows/claude-code-review.yml@v1
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

### Inputs

| Input | Type | Default | Popis |
|-------|------|---------|-------|
| `skip-title-keyword` | string | `'[skip-review]'` | Keyword pro skip |
| `review-prompt` | string | `''` | Custom prompt |
| `model` | string | `''` | Claude model |

### Secrets

- `CLAUDE_CODE_OAUTH_TOKEN` (required)

---

## deploy-ssh.yml

Generický SSH deploy via rsync s podporou backup a health checků.

### Použití

```yaml
jobs:
  prepare:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Create .env.prod.local
        run: |
          cat > backend/.env.prod.local << 'EOF'
          APP_SECRET=${{ secrets.APP_SECRET }}
          DATABASE_URL=${{ secrets.DATABASE_URL }}
          EOF
      - uses: actions/upload-artifact@v4
        with:
          name: deploy-config
          path: backend/.env.prod.local

  deploy:
    needs: prepare
    uses: i4b-cz/github-workflows/.github/workflows/deploy-ssh.yml@v1
    with:
      environment: staging
      deploy-config: |
        {
          "backend": { "enabled": true, "path": "./backend" },
          "frontend": { "enabled": true, "path": "./frontend" },
          "backup": { "enabled": false },
          "health-check": { "enabled": true, "url": "/api/config" },
          "post-deploy": {
            "composer-install": true,
            "run-migrations": true,
            "clear-cache": true
          }
        }
    secrets:
      SSH_KEY: ${{ secrets.SSH_KEY }}
```

### Inputs

| Input | Type | Default | Popis |
|-------|------|---------|-------|
| `environment` | string | required | GitHub environment |
| `node-version` | string | `'20'` | Node.js verze |
| `deploy-config` | string | required | JSON konfigurace |

### Deploy Config JSON

```json
{
  "backend": {
    "enabled": true,
    "path": "./backend",
    "rsync-exclude": [".git", "node_modules", "vendor", "var/cache", "var/log", "public/"],
    "public-includes": ["index.php", ".htaccess", "cron/"]
  },
  "frontend": {
    "enabled": true,
    "path": "./frontend",
    "build-command": "npm run build",
    "post-build-commands": ["cp .htaccess.production dist/.htaccess"],
    "deploy-to": "backend/public/",
    "rsync-exclude": ["index.php", "cron/", ".htaccess"],
    "env-vars": true
  },
  "backup": {
    "enabled": true,
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
    "custom-commands": ["php bin/console app:custom-command"]
  }
}
```

### Config Options

| Sekce | Klíč | Default | Popis |
|-------|------|---------|-------|
| `backend` | `enabled` | `true` | Povolit backend deploy |
| `backend` | `path` | `"./backend"` | Cesta k backend kódu |
| `backend` | `rsync-exclude` | `[".git", "node_modules", ...]` | Soubory/složky vyloučené z rsync |
| `backend` | `public-includes` | `["index.php"]` | Soubory/složky z public/ k deploynutí |
| `frontend` | `enabled` | `true` | Povolit frontend build a deploy |
| `frontend` | `path` | `"./frontend"` | Cesta k frontend kódu |
| `frontend` | `build-command` | `"npm run build"` | Build příkaz |
| `frontend` | `post-build-commands` | `[]` | Příkazy po buildu |
| `frontend` | `deploy-to` | `"backend/public/"` | Cílová složka pro frontend |
| `frontend` | `rsync-exclude` | `[]` | Soubory vyloučené z frontend deploy |
| `frontend` | `env-vars` | `true` | Vytvořit .env.production s VITE_* proměnnými |
| `backup` | `enabled` | `false` | Povolit backup před deploy |
| `backup` | `database` | `true` | Zálohovat databázi |
| `health-check` | `enabled` | `true` | Povolit health check po deploy |
| `health-check` | `url` | `"/api/config"` | URL pro health check |
| `health-check` | `verify-production-mode` | `false` | Ověřit testing_mode: false |
| `post-deploy` | `composer-install` | `true` | Spustit composer install |
| `post-deploy` | `run-migrations` | `true` | Spustit migrace |
| `post-deploy` | `clear-cache` | `true` | Vyčistit cache |
| `post-deploy` | `restart-workers` | `false` | Restartovat Supervisor workers |
| `post-deploy` | `custom-commands` | `[]` | Vlastní příkazy po deploy |

### Secrets

- `SSH_KEY` (required) - SSH private key

### Environment Variables

Nastavte v GitHub environment (Settings → Environments):

- `SSH_HOST` - SSH server hostname
- `SSH_USER` - SSH username
- `SSH_PORT` - SSH port (default 22)
- `REMOTE_PATH` - Cesta na serveru
- `APP_URL` - URL aplikace (pro health check)
- `API_URL` - API URL (pro frontend build jako VITE_API_BASE_URL)

---

## Verzování

Doporučujeme používat konkrétní verze:

```yaml
uses: i4b-cz/github-workflows/.github/workflows/php-ci.yml@v1      # Major verze
uses: i4b-cz/github-workflows/.github/workflows/php-ci.yml@v1.2.0  # Konkrétní verze
uses: i4b-cz/github-workflows/.github/workflows/php-ci.yml@main    # Latest (pro vývoj)
```

---

## Příklad kompletního test workflow

```yaml
name: Tests

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]

jobs:
  # Určení test scope
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

  summary:
    runs-on: ubuntu-latest
    needs: [php-ci, node-ci]
    if: always()
    steps:
      - run: |
          echo "## Test Results" >> $GITHUB_STEP_SUMMARY
          echo "- PHP CI: ${{ needs.php-ci.result }}" >> $GITHUB_STEP_SUMMARY
          echo "- Node CI: ${{ needs.node-ci.result }}" >> $GITHUB_STEP_SUMMARY
```

---

## License

MIT
