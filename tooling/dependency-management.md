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

```json
// package.json (root)
{
  "name": "my-workspace",
  "workspaces": ["packages/*", "apps/*"],
  "devDependencies": {
    // Shared development dependencies
    "eslint": "^8.45.0",
    "prettier": "^3.0.0",
    "typescript": "^5.1.6",
    "@types/node": "^18.0.0"
  }
}

// packages/shared/package.json
{
  "name": "@mycompany/shared",
  "version": "1.0.0",
  "dependencies": {
    "lodash-es": "^4.17.21",
    "date-fns": "^2.30.0"
  },
  "peerDependencies": {
    "@angular/core": "^16.0.0"
  }
}

// apps/frontend/package.json
{
  "name": "@mycompany/frontend",
  "version": "1.0.0",
  "dependencies": {
    "@mycompany/shared": "workspace:*",
    "@angular/core": "^16.2.0",
    "@angular/common": "^16.2.0"
  }
}
```

**Workspace Commands**:
```bash
# Install dependencies for all packages
npm install

# Install dependency in specific workspace
npm install express --workspace=@mycompany/backend

# Run scripts across workspaces
npm run test --workspaces
npm run build --workspace=@mycompany/frontend

# Check for duplicate dependencies across workspaces
npx npm-check-updates --workspace=@mycompany/shared
```

## Package Registry Management

### Private Registry Configuration **[RECOMMENDED]**

```bash
# .npmrc - Project-level configuration
registry=https://registry.npmjs.org/
@mycompany:registry=https://npm.mycompany.com/
//npm.mycompany.com/:_authToken=${NPM_TOKEN}

# Scoped package registry
@angular:registry=https://registry.npmjs.org/
@types:registry=https://registry.npmjs.org/

# Security settings
audit-level=moderate
fund=false
```

**Registry Security**:
```json
// package.json
{
  "publishConfig": {
    "registry": "https://npm.mycompany.com/",
    "access": "restricted"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/mycompany/project.git"
  }
}
```

## Dependency Validation and Policy

### Dependency Policy Enforcement **[ADVANCED]**

```javascript
// scripts/validate-dependencies.js
const fs = require('fs');
const path = require('path');

const packageJson = JSON.parse(fs.readFileSync('package.json', 'utf8'));
const allowedLicenses = ['MIT', 'Apache-2.0', 'BSD-3-Clause', 'ISC'];
const prohibitedPackages = ['colors', 'faker']; // Known security issues
const maxPackageSize = 10 * 1024 * 1024; // 10MB

async function validateDependencies() {
  const errors = [];
  
  // Check for prohibited packages
  const allDeps = {
    ...packageJson.dependencies,
    ...packageJson.devDependencies
  };
  
  for (const [packageName, version] of Object.entries(allDeps)) {
    if (prohibitedPackages.includes(packageName)) {
      errors.push(`Prohibited package detected: ${packageName}`);
    }
    
    // Check license compatibility
    try {
      const packageInfo = require(`${packageName}/package.json`);
      if (!allowedLicenses.includes(packageInfo.license)) {
        errors.push(`Incompatible license for ${packageName}: ${packageInfo.license}`);
      }
    } catch (e) {
      // Package info not available
    }
  }
  
  if (errors.length > 0) {
    console.error('Dependency validation failed:');
    errors.forEach(error => console.error(`  - ${error}`));
    process.exit(1);
  }
  
  console.log('✅ All dependencies validated successfully');
}

validateDependencies();
```

**GitHub Actions Integration**:
```yaml
# .github/workflows/dependency-validation.yml
name: Dependency Validation
on:
  pull_request:
    paths:
      - 'package.json'
      - 'package-lock.json'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Validate dependencies
        run: node scripts/validate-dependencies.js
      
      - name: Check for security vulnerabilities
        run: npm audit --audit-level high
      
      - name: Check for outdated dependencies
        run: npm outdated --depth=0 || true
      
      - name: Analyze bundle size impact
        run: |
          npm run build
          npx bundlesize
```

## Performance Optimization

### Bundle Size Management **[RECOMMENDED]**

```json
// package.json - Bundle size monitoring
{
  "bundlesize": [
    {
      "path": "dist/main.*.js",
      "maxSize": "2MB"
    },
    {
      "path": "dist/vendor.*.js",
      "maxSize": "5MB"
    },
    {
      "path": "dist/runtime.*.js",
      "maxSize": "50KB"
    }
  ],
  "scripts": {
    "size:analyze": "webpack-bundle-analyzer dist/",
    "size:check": "bundlesize",
    "size:report": "npm run build && npm run size:check"
  }
}
```

### Tree Shaking Optimization **[RECOMMENDED]**

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    usedExports: true,
    sideEffects: false,
  },
  resolve: {
    alias: {
      // Prefer ES modules for better tree shaking
      'lodash': 'lodash-es'
    }
  }
};

// Package-specific optimizations
// ✅ GOOD - Tree-shakable imports
import { debounce } from 'lodash-es';
import { format } from 'date-fns';
import { Observable, map } from 'rxjs';

// ❌ BAD - Imports entire library
import _ from 'lodash';
import * as dateFns from 'date-fns';
import * as rxjs from 'rxjs';

// ✅ GOOD - Individual packages when tree-shaking isn't available
import debounce from 'lodash.debounce';
import formatDate from 'date-fns/format';
```

## Dependency Documentation

### Dependency Rationale Documentation **[RECOMMENDED]**

```markdown
# Dependency Documentation

## Production Dependencies

### Core Framework
- **@angular/core** (^16.2.0) - Main Angular framework
- **@angular/common** (^16.2.0) - Common Angular utilities and directives
- **rxjs** (^7.8.0) - Reactive programming library, required by Angular

### UI Components
- **@angular/material** (^16.2.0) - Material Design components
  - Alternative considered: PrimeNG (rejected due to licensing costs)
  - Bundle size: ~500KB gzipped
  - Last security audit: 2023-08-15

### Utilities
- **lodash-es** (^4.17.21) - Utility functions (tree-shakable ES modules)
  - Alternative: Ramda (rejected due to learning curve)
  - Usage: Array manipulation, object utilities
  - Tree-shaking effectiveness: ~90% unused code eliminated

- **date-fns** (^2.30.0) - Date manipulation library
  - Alternative: moment.js (rejected due to bundle size and immutability)
  - Bundle size: ~20KB for used functions
  - Tree-shakable: Yes

### HTTP Client
- **@angular/common/http** (^16.2.0) - Angular's built-in HTTP client
  - Alternative: axios (unnecessary given Angular's built-in solution)

## Development Dependencies

### Build Tools
- **@angular/cli** (^16.2.0) - Angular command line tools
- **typescript** (^5.1.6) - TypeScript compiler
- **webpack** (^5.88.0) - Module bundler (via Angular CLI)

### Testing
- **jest** (^29.6.0) - JavaScript testing framework
  - Alternative: Jasmine/Karma (rejected due to performance)
  - Configuration: Custom Jest configuration for Angular
  
- **@testing-library/angular** (^14.2.0) - Testing utilities
  - Philosophy: Test user behavior, not implementation details

### Code Quality
- **eslint** (^8.45.0) - JavaScript/TypeScript linting
- **prettier** (^3.0.0) - Code formatting
- **@typescript-eslint/parser** (^6.2.0) - TypeScript parser for ESLint

## Removed Dependencies

### Recently Removed
- **moment.js** (removed 2023-07-15) - Replaced with date-fns for bundle size
- **bootstrap** (removed 2023-08-01) - Replaced with Angular Material for consistency
- **lodash** (removed 2023-08-10) - Replaced with lodash-es for tree-shaking

### Security Removals
- **colors** (removed 2023-06-20) - Security vulnerability (infinite loop DoS)
- **node-ipc** (removed 2023-05-15) - Malicious code injection

## Upgrade Schedule

### Quarterly Major Updates
- Angular framework packages (test thoroughly)
- TypeScript (coordinate with Angular compatibility)

### Monthly Minor Updates  
- Utility libraries (lodash-es, date-fns)
- Development tools (ESLint, Prettier)

### Weekly Security Updates
- All packages with high/critical security vulnerabilities
- Automated via Dependabot and npm audit

## Bundle Analysis Results

Last analysis: 2023-08-15
- Main bundle: 1.8MB (target: <2MB) ✅
- Vendor bundle: 4.2MB (target: <5MB) ✅
- Largest contributors:
  1. @angular/material (890KB)
  2. @angular/core (650KB)
  3. rxjs (420KB)
  4. lodash-es (actual usage: 45KB)

## Decision Log

### 2023-08-15: Material Design vs PrimeNG
**Decision**: Angular Material
**Reasoning**: 
- Better Angular integration
- Smaller bundle size
- No licensing fees
- Better accessibility support

### 2023-07-20: Jest vs Karma/Jasmine
**Decision**: Jest
**Reasoning**:
- 3x faster test execution
- Better TypeScript support
- Snapshot testing capabilities
- Industry standard outside Angular
```

This comprehensive dependency management guide ensures consistent, secure, and optimized dependency handling across your development lifecycle. Regular audits and clear documentation prevent technical debt accumulation and security vulnerabilities.
