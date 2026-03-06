---
layout: default
title: npm install vs. npm ci
parent: Infrastructure as Code
nav_order: 6
---

## `npm install` vs. `npm ci`

**Flow (CI):** Checkout → `npm ci` → test → build → deploy. Never `npm install` in CI.

`npm install` resolves dependencies against `package.json` and *updates* `package-lock.json` if versions drift. In CI, this means your build might use different dependency versions than what the developer tested locally. `npm ci` enforces the lockfile as the single source of truth.

### Comparison

| Behavior                        | `npm install`                         | `npm ci`                              |
|---------------------------------|---------------------------------------|---------------------------------------|
| Reads                           | `package.json` (primary)              | `package-lock.json` (primary)         |
| Modifies `package-lock.json`    | Yes, if tree differs                  | Never — fails if lockfile is outdated |
| Deletes `node_modules/`         | No — merges incrementally             | Yes — clean install every time        |
| Speed (cold)                    | Slower (resolves tree)                | Faster (skips resolution)             |
| Deterministic                   | No                                    | Yes                                   |
| Use in                          | Local development                     | CI/CD pipelines                       |

### Jenkinsfile Stage

```groovy
stage('Install Dependencies') {
    steps {
        sh '''
            node --version
            npm --version
            npm ci --ignore-scripts  # skip postinstall in CI if not needed
        '''
    }
}
```

### When `npm ci` Fails

```bash
$ npm ci
npm ERR! `npm ci` can only install packages when your package.json
npm ERR! and package-lock.json are in sync.

# Fix: developer forgot to commit the updated lockfile
npm install          # regenerates lockfile
git add package-lock.json
git commit -m "Sync lockfile after dependency update"
```

### `.npmrc` for CI Environments

```ini
# .npmrc (committed to repo)
engine-strict=true
save-exact=true
audit-level=high
```

`save-exact=true` pins exact versions in `package.json` (no `^` or `~`), reducing lockfile churn and version surprises.

> **Reliability Note:** `npm ci` deletes and recreates `node_modules/` from scratch on every run. On large projects this takes 30–60 seconds. To speed up CI, cache `~/.npm` (npm's download cache, not `node_modules/`) between builds. Jenkins: use the `stash`/`unstash` steps or a shared volume. This cuts install time to ~5 seconds for warm caches while keeping the deterministic lockfile guarantee.
