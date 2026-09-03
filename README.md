# workflows-reusable

Workflows GitHub Actions réutilisables (`workflow_call`) partagés entre plusieurs projets.

## Workflows disponibles

### `php-quality.yml`

Analyse qualité pour un projet PHP/Composer (`composer audit` + PHPStan).

### `npm-security.yml`

Scan sécurité pour un projet npm/Node : `npm audit`, scan Trivy (filesystem/dépendances) et détection de secrets (Gitleaks). Chaque outil est activable/désactivable indépendamment.

Exemple d'appel depuis un autre repo :

```yaml
jobs:
  security:
    uses: Skuskusku13/workflows-reusable/.github/workflows/npm-security.yml@v1.0.0
    with:
      project-name: portfolio
      node-version: '22'
      audit-level: high
```

Voir les `inputs` du fichier `.github/workflows/npm-security.yml` pour la liste complète des paramètres (chemin du projet, seuils de sévérité, activation/désactivation des scanners).

### `owasp-zap-baseline.yml`

Scan DAST (dynamique, sur l'app en cours d'exécution) via OWASP ZAP en mode **baseline uniquement** (scan passif, non intrusif — pas de scan actif). Build et démarre l'app appelante localement dans le runner, attend qu'elle soit prête (health check), puis scanne `http://localhost:<port>` avec ZAP. Le rapport ZAP (HTML/JSON/MD) est publié comme artifact GitHub Actions. Complète `npm-security.yml` (SAST/dépendances) avec une couche DAST.

Exemple d'appel depuis un autre repo :

```yaml
jobs:
  zap-baseline:
    uses: Skuskusku13/workflows-reusable/.github/workflows/owasp-zap-baseline.yml@v1.0.0
    with:
      project-name: portfolio
      node-version: '22'
      build-command: npm run build
      start-command: npm run preview -- --port 4173 --host
      port: '4173'
      health-check-path: /
      zap-fail-on-warn: false
```

Voir les `inputs` du fichier `.github/workflows/owasp-zap-baseline.yml` pour la liste complète des paramètres.
