# Dependency Management Guidelines

Strategies for maintaining clean, secure, and reproducible dependency trees across development environments.

## Package Installation

### Use npm ci for Production Builds **[MANDATORY]**

```bash
# ✅ GOOD - Production and CI environments
npm ci
# - Installs from package-lock.json exactly
# - Fails if package.json and package-lock.json are out of sync
# - 2-10x faster than npm install
# - Removes node_modules before installing (clean slate)

# ✅ GOOD - Development when adding/updating packages
npm install
npm install express@^4.18.0
npm install --save-dev @types/jest

# ❌ BAD - Using npm install in CI/production
npm install
# - Can install different versions than development
# - Modifies package-lock.json
# - Slower and less predictable
```

**Why This Matters**:
- **Reproducible builds** - Everyone gets identical dependency versions
- **Security** - Prevents supply chain attacks through dependency confusion
- **Performance** - CI builds run significantly faster
- **Reliability** - Eliminates "works on my machine" dependency issues

### Package Lock Management **[MANDATORY]**

```bash
# ✅ GOOD - Always commit package-lock.json
git add package-lock.json
git commit -m "Update dependencies"

# ✅ GOOD - Keep package-lock.json up to date
npm install        # When adding new dependencies
npm update         # When updating existing dependencies

# ❌ BAD - Never do these
echo "package-lock.json" >> .gitignore  # Don't ignore the lock file
rm package-lock.json                    # Don't delete the lock file
npm install --no-package-lock          # Don't skip lock file generation
```

**Lock File Benefits**:
- Ensures exact dependency versions across all environments
- Prevents automatic updates that could break the application
- Provides security audit trail for dependency changes
- Enables faster installs through dependency caching

## Dependency Audit and Security

### Regular Security Audits **[RECOMMENDED]**

```bash
# Check for known vulnerabilities
npm audit
npm audit --audit-level high  # Only show high severity issues

# Fix vulnerabilities automatically (use with caution)
npm audit fix
npm audit fix --force  # May introduce breaking changes

# Alternative: Use yarn for better security reporting
yarn audit
yarn audit --level high
```

**Automated Security Monitoring**:
```json
// .github/workflows/security-audit.yml
name: Security Audit
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # Weekly on Mondays

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm audit --audit-level high
      - run: npm outdated
```

### Dependency Updates Strategy **[RECOMMENDED]**

```bash
# Check for outdated packages
npm outdated
npm outdated --depth=0  # Only direct dependencies

# Update patch versions safely
npm update

# Update major versions carefully (test thoroughly)
npm install express@latest
npm install @angular/core@16

# Use tools for systematic updates
npx npm-check-updates -u    # Update package.json to latest versions
npm install                 # Install updated versions
```

**Update Priority Matrix**:

| Type | Update Frequency | Risk Level | Testing Required |
|------|-----------------|------------|------------------|
| **Security patches** | Immediately | Low | Basic smoke tests |
| **Patch versions (x.x.X)** | Weekly | Low | Automated tests |
| **Minor versions (x.X.x)** | Monthly | Medium | Full test suite |
| **Major versions (X.x.x)** | Quarterly | High | Comprehensive testing + manual QA |

## Dependency Cleanup

### Remove Unused Dependencies **[RECOMMENDED]**

```bash
# Find unused dependencies
npx depcheck
npx depcheck --ignores="@types/*,eslint-*"  # Ignore common dev dependencies

# Remove unused packages
npm uninstall unused-package-name
npm uninstall --save-dev unused-dev-package

# Clean up after removal
npm dedupe  # Remove duplicate dependencies
npm prune   # Remove packages not in package.json
```

**Automated Cleanup Script**:
```json
// package.json
{
  "scripts": {
    "deps:check": "npx depcheck",
    "deps:clean": "npm dedupe && npm prune",
    "deps:audit": "npm audit && npm outdated",
    "deps:update-patch": "npm update",
    "deps:update-check": "npx npm-check-updates"
  }
}
```

### Dependency Size Analysis **[RECOMMENDED]**

```bash
# Analyze bundle size impact
npx webpack-bundle-analyzer dist/
npx source-map-explorer dist/main.*.js

# Check package sizes before installing
npx package-phobia express
npx bundlephobia lodash

# Find heavy dependencies
npx cost-of-modules
```

**Size Optimization Strategies**:
```javascript
// ✅ GOOD - Import only what you need
import { debounce } from 'lodash-es';
import { Observable } from 'rxjs';
import { map, filter } from 'rxjs/operators';

// ❌ BAD - Imports entire library
import * as _ from 'lodash';
import * as rxjs from 'rxjs';

// ✅ GOOD - Use tree-shakable alternatives
import debounce from 'lodash.debounce';  // Individual package
import { format } from 'date-fns';        // Tree-shakable date library
```

## Dependency Version Management

### Semantic Versioning Strategy **[RECOMMENDED]**

```json
// package.json - Version range strategies
{
  "dependencies": {
    // ✅ GOOD - Patch updates only (safest)
    "critical-package": "1.2.3",
    
    // ✅ GOOD - Minor updates (recommended for most packages)
    "stable-package": "^1.2.3",
    
    // ⚠️ CAUTION - Major updates (use sparingly)
    "frequently-updated-package": "~1.2.3",
    
    // ❌ AVOID - Latest updates (unpredictable)
    "unpredictable-package": "*"
  },
  "devDependencies": {
    // More flexible versioning for dev tools
    "@types/node": "^18.0.0",
    "eslint": "^8.0.0",
    "jest": "^29.0.0"
  }
}
```

**Version Range Guide**:
- `1.2.3` - Exact version (most secure, least flexible)
- `^1.2.3` - Compatible within major version (1.x.x)
- `~1.2.3` - Compatible within minor version (1.2.x)
- `*` or `latest` - Any version (never use in production)

### Pinning Critical Dependencies **[RECOMMENDED]**

```json
// package.json
{
  "dependencies": {
    // Pin exact versions for:
    // 1. Security-critical packages
    "bcrypt": "5.1.0",
    "jsonwebtoken": "9.0.0",
    
    // 2. Packages with frequent breaking changes
    "@angular/core": "16.2.5",
    "@angular/common": "16.2.5",
    
    // 3. Build tools and bundlers
    "webpack": "5.88.2",
    "typescript": "5.1.6"
  },
  "devDependencies": {
    // More flexibility for development tools
    "eslint": "^8.45.0",
    "prettier": "^3.0.0",
    "jest": "^29.6.0"
  }
}
```

## Environment-Specific Dependencies

### Separate Production and Development **[MANDATORY]**

```json
// package.json
{
  "dependencies": {
    // Runtime dependencies - included in production bundle
    "express": "^4.18.2",
    "lodash": "^4.17.21",
    "@angular/core": "^16.2.0"
  },
  "devDependencies": {
    // Development-only dependencies - not in production bundle
    "@types/express": "^4.17.17",
    "@types/lodash": "^4.14.195",
    "eslint": "^8.45.0",
    "jest": "^29.6.0",
    "typescript": "^5.1.6"
  },
  "optionalDependencies": {
    // Optional enhancements that can fail to install
    "fsevents": "^2.3.2"  // macOS-specific file watching
  }
}
```

**Installation Commands**:
```bash
# Install only production dependencies
npm ci --production
npm ci --omit=dev

# Install all dependencies (development)
npm ci

# Install specific dependency types
npm install express --save          # Add to dependencies
npm install @types/express --save-dev  # Add to devDependencies
npm install fsevents --save-optional   # Add to optionalDependencies
```

### Environment Configuration **[RECOMMENDED]**

```javascript
// config/dependencies.js
const productionDependencies = {
  // Core runtime dependencies
  core: ['@angular/core', '@angular/common', 'rxjs'],
  ui: ['@angular/material', '@angular/cdk'],
  http: ['@angular/common/http'],
  
  // Third-party runtime dependencies
  utilities: ['lodash-es', 'date-fns'],
  charts: ['chart.js', 'd3']
};

const developmentDependencies = {
  // Build and development tools
  build: ['@angular/cli', 'webpack', 'typescript'],
  testing: ['jest', '@testing-library/angular', 'cypress'],
  linting: ['eslint', 'prettier', '@typescript-eslint/parser'],
  types: ['@types/node', '@types/lodash', '@types/jest']
};

module.exports = { productionDependencies, developmentDependencies };
```

## Monorepo Dependency Management

### Workspace Dependencies **[ADVANCED]**
[ PENDING ]
