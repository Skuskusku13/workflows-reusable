# Changelog

## v1.2.1 — 2026-09-05

### `npm-security.yml`
- **Retrait du job `npm audit`.** L'endpoint `registry.npmjs.org/-/npm/v1/security/advisories/bulk` ne répondait pas depuis les runners GitHub : le step était tué au `timeout-minutes` de 6 min à chaque run, sans jamais atteindre la boucle de retry (npm consomme lui-même le budget avec `fetch-timeout` × `fetch-retries`). Trivy couvre les mêmes avis (GitHub Advisory Database) depuis une base locale.
- Inputs `node-version`, `audit-level`, `audit-omit-dev`, `enable-npm-audit` conservés mais ignorés (compatibilité). Retrait prévu en v2.0.0.
- Nouvel input `trivy-scanners` (défaut `vuln`) pour activer `misconfig` ou `license`.

### `owasp-zap-baseline.yml`
- **Agnostique du langage.** `node-version` devient optionnel ; nouveaux inputs `install-command`, `stop-command`, `startup-timeout-seconds`. Le comportement v1.2.0 est préservé : `node-version` renseigné sans `install-command` exécute toujours `npm ci --no-audit --no-fund`.
- Nouveau mode **cible distante** via `target-url` (ignore install / build / start).
- **Scan authentifié** : secret `zap-auth-header-value` + input `zap-auth-header` (défaut `Authorization`), transmis à ZAP via `ZAP_AUTH_HEADER` / `ZAP_AUTH_HEADER_VALUE`.
- Nouveaux inputs `zap-ajax-spider` (spider navigateur pour les SPA), `zap-rules-file` (règles ZAP `-c`), `job-timeout-minutes`.
- **Correctif** : `zap-fail-on-warn: true` était sans effet. L'expression `cond && '' || '-I'` renvoie toujours `-I` car une chaîne vide est falsy dans les expressions GitHub Actions.
- Pin de `zaproxy/action-baseline` corrigé sur le SHA du commit de v0.15.0 (`de8ad96`) ; l'ancien pin (`6c5a007`) était le SHA de l'objet tag annoté.
- Le health check suit les redirections (`curl -L`).

### `owasp-zap-full-scan.yml` (nouveau)
- Scan DAST actif via `zaproxy/action-full-scan` v0.13.0, mêmes inputs que le baseline, plus `zap-active-scan-minutes` (défaut 30, via `-z "-config scanner.maxScanDurationInMins=N"`). Spider par défaut à 5 min, `job-timeout-minutes` par défaut 90. Prévu pour un `schedule` hebdomadaire.

### Divers
- Nouveau workflow interne `release.yml` : au push d'un tag `vX.Y.Z`, crée la Release GitHub avec les notes de la section correspondante de ce CHANGELOG (token Actions du repo, aucun secret à configurer).
- Versions vérifiées le 2026-09-05, toutes les actions sont à leur dernière release : `actions/checkout` v7.0.1, `actions/setup-node` v7.0.0, `aquasecurity/trivy-action` v0.36.0, `gitleaks/gitleaks-action` v3.0.0, `zaproxy/action-baseline` v0.15.0, `zaproxy/action-full-scan` v0.13.0.

## v1.2.0 — 2026-09-04
- `npm-security.yml` : 3 jobs parallèles indépendants (npm audit, Trivy, Gitleaks) au lieu d'un seul job avec `continue-on-error`. Retry avec backoff sur npm audit, `TRIVY_DB_REPOSITORY` avec miroir ECR, input `audit-omit-dev`.
- `owasp-zap-baseline.yml` : input `zap-spider-minutes`, étape « Stop app » en `always()`.

## v1.1.1 — 2026-09-04
- `npm ci --no-audit` partout où l'audit n'est pas le but du step (le rapport d'audit automatique de `npm ci` tapait sur le même endpoint lent).

## v1.1.0 — 2026-09-03
- Ajout de `owasp-zap-baseline.yml`. Références des actions mises à jour.

## v1.0.0
- `php-quality.yml`, `npm-security.yml`.
