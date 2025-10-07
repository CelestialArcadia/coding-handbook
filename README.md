# Complete Development Guidelines

An amalgamation of coding standards, best practices, and architectural guidelines for maintainable web development with a pinch of personal preference.

<b> How to navigate this repo? </b> It is structured following two approaches:

* This README file is a general overall approach that has a certain amount of depth.
* Other files are divided by categories with more in-depth approaches- they may or may not overlap with some of the general README file points.

## Table of Contents

- [Purpose & Benefits](#-purpose--benefits)
- [Legend](#-legend)
- [Quick Reference](#-quick-reference)
- [Dependency Management](#-dependency-management)
- [TypeScript Guidelines](#-typescript-guidelines)
- [Angular Standards](#-angular-standards)
- [SCSS/CSS Conventions](#-scsscss-conventions)
- [Backend Guidelines](#-backend-guidelines)
- [Universal Principles](#-universal-principles)
- [Git & Workflow](#-git--workflow)
- [Implementation Guide](#-implementation-guide)

## Purpose & Benefits

These guidelines ensure:
- **Consistency** - Standardized practices across teams and projects
- **Readability** - Code that clearly communicates intent and functionality
- **Maintainability** - Easier debugging, refactoring, and feature addition
- **Collaboration** - Common conventions reduce onboarding issues
- **Scalability** - Architectural patterns that support growth

## Legend

- **[MANDATORY]** - Non-negotiable standards
- **[RECOMMENDED]** - Best practices with clear benefits
- **[TEAM_PREFERENCE]** - Consistency improvements that may vary

## Quick Reference

### Essential Rules
1. **[MANDATORY]** Use`const`/`let` instead of `var`
2. **[MANDATORY]** Arrow functions over function expressions
3. **[MANDATORY]** Named error constants instead of magic strings
4. **[RECOMMENDED]** Observable variables end with `$`
5. **[RECOMMENDED]** Private methods prefixed with `_`
6. **[RECOMMENDED]** ID format: `{object}Id` (userId, clientId)

### Common Anti-Patterns
```typescript
// AVOID
var data = getData();
const process = function(input) { /* ... */ };
throw new Error('Something went wrong');

// PREFER
const data = getData();
const process = (input: InputType): OutputType => { /* ... */ };
throw new Error(PROCESSING_ERROR);
```

## Dependency Management

### Installing Dependencies **[MANDATORY]**
```bash
# Use npm ci for deterministic builds
npm ci  # Preferred
npm i   # Avoid in CI/production
```

### Package Management **[RECOMMENDED]**
- **Remove unused dependencies** regularly to prevent bloat
- **Commit `package-lock.json`** to ensure consistent environments
- **Review dependencies** before adding - evaluate necessity and maintenance status

### Why This Matters
Consistent dependency versions eliminate "works on my machine" issues and security vulnerabilities from outdated packages.

## TypeScript Guidelines

### Variable Declarations **[MANDATORY]**

```typescript
//  GOOD
const immutableValue = 'constant';
let mutableValue = 0;

//  BAD
var outdatedDeclaration = 'avoid';
```

**Rule**: Use `const` for values that won't be reassigned, `let` for mutable variables.

### Function Declarations **[MANDATORY]**

```typescript
//  GOOD
const add = (a: number, b: number): number => {
  return a + b;
};

//  BAD
const add = function(a: number, b: number) {
  return a + b;
};
```

**Benefits**: Arrow functions provide lexical `this` binding and consistent syntax.

### Type Definitions **[RECOMMENDED]**

```typescript
//  GOOD - Use interfaces for object shapes
interface User {
  id: number;
  name: string;
  email: string;
}

//  GOOD - Type aliases for complex types
type EventHandler = (event: Event) => void;

//  GOOD - Generics for reusability
interface Repository<T> {
  findById(id: number): T | undefined;
}
```

### Override Keyword **[RECOMMENDED]**

```typescript
class ChildComponent extends BaseComponent {
  override ngOnInit(): void {
    super.ngOnInit();
    // Child-specific initialization
  }
}
```

**Purpose**: Makes inheritance relationships explicit and catches signature mismatches.

## Angular Standards

### Component Architecture **[MANDATORY]**

```typescript
//  GOOD
export class UserProfileComponent implements OnInit {
  @Input() userId: number;
  @Output() userUpdated = new EventEmitter<User>();
  
  private _userService = inject(UserService); // Modern injection
  
  private _loadUserData(): void { /* ... */ }
}

//  BAD
export class userProfile {
  @Input('id') userId: number;
  @Output() updated = new EventEmitter();
}
```

### Dependency Injection Decision Matrix

| Situation | Use Constructor | Use inject() |
|-----------|----------------|--------------|
| Few dependencies (1-3) |  Clear signature | Optional |
| Many dependencies (4+) |  Bloated constructor |  Clean fields |
| Functional APIs (guards, resolvers) |  Not available |  Required |
| Need DI decorators (@Optional, @Self) |  Required |  Not supported |

### Template Safety **[MANDATORY]**

```html
<!--  GOOD - Safe navigation for optional properties !! -->
<div>{{ user?.profile?.displayName || 'Guest' }}</div>

<!--  BAD - Potential runtime errors !! -->
<div>{{ user.profile.displayName }}</div>
```

### Models & View Models **[RECOMMENDED]**

```typescript
// user.model.ts - Backend alignment
export interface User {
  id: number;
  firstName: string;
  lastName: string;
  email: string;
  isActive: boolean;
}

// user.view.model.ts - Frontend-specific needs
export interface UserViewModel {
  id: number;
  fullName: string;
  email: string;
  displayStatus: 'Active' | 'Inactive';
}
```

### Forms **[RECOMMENDED]**

```typescript
//  GOOD - Type-safe access
const nameValue = this.userForm.controls['firstName'].value;

//  AVOID - Less refactor-friendly  
const nameValue = this.userForm.controls.firstName.value;
```

### Observable Naming **[TEAM_PREFERENCE]**

```typescript
//  CONSISTENT
private usersSubject = new BehaviorSubject<User[]>([]);
public users$ = this.usersSubject.asObservable();

private _loadUsers(): void { /* ... */ }
```

### Accessibility **[MANDATORY]**

Accessibility isn't just compliance; it's about creating inclusive experiences that work for everyone.  
Poor accessibility excludes users and creates legal liability.

#### Semantic HTML
```html
<!--  SEMANTIC HTML !! -->
<!--  GOOD - Browser provides built-in keyboard support, focus management !! -->
<button (click)="submit()" [disabled]="!isValid">
  Submit Form
</button>

<!--  Always associate labels with form controls for screen reader context. !! -->
<label for="email">Email Address</label>
<input id="email" type="email" aria-required="true">

<!--  ARIA ATTRIBUTES !! -->
<span *ngIf="hasError" aria-live="assertive">
  {{ errorMessage }}
</span>

<!--  PROPER ALT TEXT !! -->
<img src="user-avatar.jpg" alt="User profile picture">
<img src="decorative-border.png" alt="" aria-hidden="true">

<!--  BAD - Requires manual keyboard handling, no semantic meaning !! -->
<div (click)="submit()" class="fake-button">
  Submit Form
</div>
```
* Why: Semantic elements provide automatic keyboard navigation, screen reader context, and focus management.  Users with disabilities rely on these built-in behaviors.

#### Heading Hierarchy
Maintain logical heading order for screen reader navigation. ```<h1> -> <h2>``` etc...

* Why: Screen readers use headings as navigation landmarks.  
Users jump between headings to quickly find content.  
An illogical hierarchy breaks this navigation pattern.

#### Form Labels
* Why: Screen readers announce the label when the input receives focus. Without proper association, users don't know what information to enter.

#### ARIA Attributes
Use ARIA to provide additional context when HTML semantics aren't sufficient.

```
<!--  ARIA-LIVE for dynamic content !! -->
<span *ngIf="hasError" aria-live="assertive">
  {{ errorMessage }}
</span>

<span *ngIf="saveStatus" aria-live="polite">
  Document saved successfully
</span>

<!--  ARIA-LABEL for context without visible text !! -->
<button aria-label="Close modal" (click)="closeModal()">
  <i class="icon-close"></i>
</button>

<!--  ARIA-EXPANDED for collapsible content !! -->
<button 
  [attr.aria-expanded]="isExpanded"
  (click)="toggle()"
  aria-controls="dropdown-menu">
  Options
</button>
<div id="dropdown-menu" [hidden]="!isExpanded">
  <!-- menu items !! -->
</div>
```

#### Aria Live Regions

* aria-live="assertive" - Interrupts screen reader immediately (errors, urgent alerts)
* aria-live="polite" - Waits for pause in speech (status updates, confirmations)
* aria-atomic="true" - Reads entire region when updated, not just changes

#### Keyboard Navigation 
Ensure all interactive elements are keyboard accessible.

```
<!--  GOOD - Custom interactive element with keyboard support !! -->
<span
  role="button"
  tabindex="0"
  [inlineSVG]="'./assets/icons/clear.svg'"
  (click)="clear()"
  (keydown.enter)="clear()"
  (keydown.space)="clear(); $event.preventDefault()"
  aria-label="Clear search">
</span>

<!--  BAD - Click-only interaction !! -->
<span (click)="clear()">
  <i class="icon-clear"></i>
</span>
```

* Why: Many users navigate entirely by keyboard (motor disabilities, power users, mobile screen readers).
  Every interactive element must be reachable and operable via keyboard.

#### Image Alt Text
Provide context-appropriate descriptions for images.
```
<!--  GOOD - Descriptive alt for meaningful images !! -->
<img src="sales-chart-q4.png" alt="Q4 sales increased 23% from previous quarter">

<!--  GOOD - Empty alt for decorative images !! -->
<img src="decorative-border.png" alt="" aria-hidden="true">

<!--  GOOD - Context-aware descriptions !! -->
<img src="user-avatar.jpg" alt="John Smith's profile picture">

<!--  BAD - Redundant or useless alt !! -->
<img src="chart.png" alt="chart"> <!-- Doesn't describe the data !! -->
<img src="decoration.png" alt="image"> <!-- Screen reader already says "image" !! -->
```

Alt Text Rules:

* Informative images: Describe the meaning/data, not appearance
* Decorative images: Use alt="" and aria-hidden="true"
* Functional images (icons in buttons): Describe the action, not the icon
* Complex images: Consider longer descriptions or data tables

**Testing Accessibility**:
1. Use Lighthouse audits (Chrome DevTools)
2. Test keyboard navigation (Tab, Enter, Space)
3. Verify screen reader compatibility (NVDA, JAWS)

Testing Checklist:
- [ ] All interactive elements focusable via keyboard
- [ ] Focus indicators visible and clear
- [ ] Heading structure logical (h1 → h2 → h3)
- [ ] Form labels properly associated
- [ ] Images have appropriate alt text
- [ ] Error messages announced by screen readers
- [ ] Color information supplemented with icons/text
- [ ] Minimum contrast ratios met

## SCSS/CSS Conventions

### BEM with Prefixes **[TEAM_PREFERENCE]**

```scss
//  GOOD - Clear component structure
.c-user-card {
  padding: 1rem;
  
  &__header {
    font-weight: bold;
  }
  
  &__content {
    margin-top: 0.5rem;
  }
  
  &--featured {
    border: 2px solid gold;
  }
}

//  GOOD - Reusable objects
.o-dropdown {
  position: relative;
  display: inline-block;
}

//  GOOD - Utility classes
.u-text-center { text-align: center; }
.u-margin-small { margin: 0.5rem; }
```

### Naming Convention
- **Components**: `.c-component-name`
- **Objects**: `.o-object-name` 
- **Utilities**: `.u-utility-name`

## Backend Guidelines

### Error Handling **[MANDATORY]**

```typescript
//  GOOD - Named constants
const INPUT_VALIDATION_ERROR = 'INPUT_VALIDATION_FAILED';
const AUTHENTICATION_ERROR = 'AUTHENTICATION_FAILED';
const PERMISSION_DENIED_ERROR = 'INSUFFICIENT_PERMISSIONS';

// Usage
return BadRequest(INPUT_VALIDATION_ERROR);

//  BAD - Magic strings
return BadRequest('Invalid input provided');
```

**Benefits**:
- Consistent error messages across the application
- Easier localization and translation
- Better error tracking and monitoring
- Reduced typos in error handling

### Service Return Types **[RECOMMENDED]**

```typescript
//  GOOD - Explicit undefined handling
getUserById(id: number): User | undefined {
  return this.users.find(user => user.id === id);
}

//  AVOID - Unclear return type
getUserById(id: number): any {
  return this.users.find(user => user.id === id);
}
```

## Universal Principles

### SOLID Principles **[RECOMMENDED]**

1. **Single Responsibility** - Each class/function has one reason to change
2. **Open/Closed** - Open for extension, closed for modification
3. **Liskov Substitution** - Subtypes must be substitutable for base types
4. **Interface Segregation** - Clients shouldn't depend on unused interfaces
5. **Dependency Inversion** - Depend on abstractions, not concretions

### Naming Conventions **[MANDATORY]**

- **Variables**: `camelCase` for most languages
- **Constants**: `UPPER_SNAKE_CASE`
- **Private members**: Prefix with `_` 
- **Booleans**: `is`, `has`, `can` prefixes (`isValid`, `hasPermission`)
- **IDs**: `{entity}Id` format (`userId`, `orderId`)

### Code Organization **[RECOMMENDED]**

```typescript
//  GOOD - Clear separation of concerns
class UserService {
  // Public API first
  public getUsers(): Observable<User[]> { /* ... */ }
  public createUser(user: CreateUserRequest): Observable<User> { /* ... */ }
  
  // Private implementation details last
  private _validateUser(user: User): boolean { /* ... */ }
  private _mapToViewModel(user: User): UserViewModel { /* ... */ }
}
```

## Git & Workflow

### Branch Naming **[MANDATORY]**
```bash
# Format: {ticket-id}-brief-description
feature/PROJ-123-user-authentication
bugfix/PROJ-456-form-validation-error
hotfix/PROJ-789-security-patch
```

### Commit Messages **[RECOMMENDED]**
```bash
# Format: [TICKET-ID] Brief description
[PROJ-123] Add user authentication endpoints
[PROJ-456] Fix form validation error handling
[PROJ-789] Update security headers configuration
```

**Benefits**: Clear traceability between code changes and business requirements.

## Implementation Guide

### For New Projects
1. **Set up linting** rules that enforce these standards
2. **Configure IDE** with appropriate extensions and settings
3. **Create templates** for common patterns (components, services)
4. **Document exceptions** when deviating from guidelines

### For Existing Projects
1. **Audit current code** against these guidelines
2. **Prioritize high-impact changes** (error handling, accessibility)
3. **Refactor incrementally** during feature development
4. **Update gradually** to avoid disrupting ongoing work

### Team Adoption
1. **Code review checklist** based on these guidelines
2. **Regular team discussions** about guideline effectiveness
3. **Update guidelines** based on real-world experience
4. **Share knowledge** through pair programming and documentation

## Contributing & Maintenance

### When to Update Guidelines
- New framework versions with breaking changes
- Team retrospectives identify pain points
- Industry best practices evolve
- Security vulnerabilities require new approaches

### Guideline Evolution Process
1. **Propose changes** with clear reasoning and examples
2. **Team review** to assess impact and benefits
3. **Pilot implementation** on small features
4. **Document lessons learned** and update accordingly

---

*For questions or suggestions, create an issue or reach out to me. And let that hill connect us*
