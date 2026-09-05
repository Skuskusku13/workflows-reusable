# workflows-reusable

Workflows GitHub Actions réutilisables (`workflow_call`) partagés entre plusieurs projets.

Toutes les actions tierces sont épinglées sur le SHA de commit de leur dernière release (commentaire `# vX.Y.Z` en fin de ligne). Les projets appelants référencent un tag de ce repo (`@v1.2.1`), jamais `@main`.

## Workflows disponibles

| Workflow | Type | Langage | Quand l'utiliser |
|---|---|---|---|
| `php-quality.yml` | qualité | PHP | PR / push |
| `npm-security.yml` | dépendances + secrets | tous | PR / push / mensuel |
| `owasp-zap-baseline.yml` | DAST passif | tous | PR / push |
| `owasp-zap-full-scan.yml` | DAST actif | tous | planifié (hebdo) |

### `php-quality.yml`

Analyse qualité pour un projet PHP/Composer (`composer audit` + PHPStan).

### `npm-security.yml`

Deux jobs parallèles et indépendants :

- **Trivy** (`trivy fs`) : vulnérabilités des dépendances à partir du lockfile, sans installer quoi que ce soit. Base téléchargée depuis le miroir ECR (GitHub Advisory Database + NVD). Malgré le nom du fichier, fonctionne pour tout écosystème que Trivy détecte : `package-lock.json`, `composer.lock`, `go.sum`, `requirements.txt`, etc. L'input `trivy-scanners` permet d'ajouter `misconfig` (Dockerfile, compose, K8s).
- **Gitleaks** : détection de secrets sur tout l'historique git.

Depuis `v1.2.1`, il n'y a plus de job `npm audit` : l'endpoint d'audit du registre npm ne répondait pas depuis les runners GitHub (step tué au timeout sur tous les runs observés), et Trivy couvre les mêmes avis sans dépendre du réseau npm. Les inputs `node-version`, `audit-level`, `audit-omit-dev` et `enable-npm-audit` sont conservés pour compatibilité mais ignorés, et seront retirés en `v2.0.0`.

```yaml
jobs:
  security:
    uses: Skuskusku13/workflows-reusable/.github/workflows/npm-security.yml@v1.2.1
    with:
      project-name: portfolio
      # trivy-severity: CRITICAL,HIGH   (défaut)
      # trivy-scanners: vuln,misconfig  (défaut : vuln)
```

### `owasp-zap-baseline.yml`

Scan DAST **passif** via OWASP ZAP : spider + analyse des réponses, aucune attaque envoyée. Rapide, prévu pour les pull requests. Rapport HTML / JSON / Markdown publié en artifact.

Agnostique du langage. Deux modes :

1. **App locale** : le workflow installe, builde et démarre l'app dans le runner avec les commandes fournies, attend qu'elle réponde sur `health-check-path`, puis scanne `http://localhost:<port>`.
2. **Cible distante** : `target-url` renseigné, les étapes install / build / start sont ignorées. À réserver aux cibles dont vous êtes propriétaire.

SPA Node :

```yaml
jobs:
  zap-baseline:
    uses: Skuskusku13/workflows-reusable/.github/workflows/owasp-zap-baseline.yml@v1.2.1
    with:
      project-name: portfolio
      node-version: '22'
      install-command: npm ci --no-audit --no-fund
      build-command: npm run build
      start-command: npm run preview -- --port 4173 --host
      port: '4173'
```

Backend conteneurisé (PHP, Go, Java, ...), API authentifiée :

```yaml
jobs:
  zap-baseline:
    uses: Skuskusku13/workflows-reusable/.github/workflows/owasp-zap-baseline.yml@v1.2.1
    with:
      project-name: mon-api
      start-command: docker compose -f .docker/docker-compose.yml up -d --build
      stop-command: docker compose -f .docker/docker-compose.yml down -v
      port: '8080'
      health-check-path: /health
      startup-timeout-seconds: '180'
    secrets:
      zap-auth-header-value: ${{ secrets.ZAP_AUTH_TOKEN }}   # ex : "Bearer eyJ..."
```

Backend PHP sans Docker (PHP est préinstallé sur `ubuntu-latest`) :

```yaml
    with:
      project-name: mon-api
      install-command: composer install --no-dev --no-interaction
      start-command: php -S 0.0.0.0:8000 -t public
      port: '8000'
```

Options ZAP utiles : `zap-ajax-spider: true` pour une SPA (spider navigateur, plus lent), `zap-rules-file` pour ignorer ou durcir certaines règles, `zap-fail-on-warn: true` pour échouer aussi sur les WARN.

### `owasp-zap-full-scan.yml`

Scan DAST **actif** : mêmes inputs que le baseline, mais ZAP envoie de vraies attaques (injections, XSS, traversée de chemin, ...) sur chaque point d'entrée découvert. Long et bruyant : à planifier, pas à lancer sur les PR. Le mode « app locale » est sans risque pour l'extérieur.

Inputs supplémentaires : `zap-active-scan-minutes` (borne le scan actif, défaut 30) et `job-timeout-minutes` (défaut 90). Le spider est par défaut à 5 minutes.

```yaml
on:
  schedule:
    - cron: '0 8 * * 1'   # lundi 8h UTC
  workflow_dispatch:

jobs:
  zap-full-scan:
    uses: Skuskusku13/workflows-reusable/.github/workflows/owasp-zap-full-scan.yml@v1.2.1
    with:
      project-name: portfolio
      node-version: '22'
      install-command: npm ci --no-audit --no-fund
      build-command: npm run build
      start-command: npm run preview -- --port 4173 --host
      port: '4173'
      zap-ajax-spider: true
      zap-active-scan-minutes: '30'
```

## Versions

Voir [CHANGELOG.md](CHANGELOG.md). Les tags suivent semver : un patch ne casse jamais un caller existant.

Publier une version : merger dans `main`, ajouter une section `## vX.Y.Z` au CHANGELOG, puis `git tag -a vX.Y.Z -m "vX.Y.Z" && git push origin vX.Y.Z`. Le workflow interne `release.yml` crée la Release GitHub automatiquement avec les notes du CHANGELOG.
