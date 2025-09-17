# Accessibility Guidelines

Comprehensive accessibility standards for inclusive web applications. Accessibility isn't compliance—it's about creating experiences that work for everyone.

## Why Accessibility Matters

**Legal Reality**: Web accessibility is legally required in many jurisdictions (ADA, WCAG compliance).

**User Impact**: 15% of the global population has some form of disability. Poor accessibility excludes millions of potential users.

**Business Case**: Accessible apps have better SEO, broader market reach, and often improved UX for all users.

**Technical Benefits**: Semantic HTML and proper ARIA usage improve code maintainability and automated testing.

## Semantic HTML

### Use Proper Elements **[MANDATORY]**

HTML elements have built-in accessibility features. Don't recreate them with generic elements.

```html
<!-- ✅ GOOD - Browser provides built-in keyboard support -->
<button (click)="submit()" [disabled]="!isValid">
  Submit Form
</button>

<nav aria-label="Main navigation">
  <ul>
    <li><a href="/dashboard">Dashboard</a></li>
    <!-- Generally frameworks have a substitute for 'aria-current', like Angular's 'routerLinkActive'  -->
    <li><a href="/profile" aria-current="page">Profile</a></li>
    <li><a href="/settings">Settings</a></li>
  </ul>
</nav>

<main>
  <h1>User Dashboard</h1>
  <!-- Referencing Existing Labels: When an element's accessible name should be derived from visible text
       that already exists in the DOM but cannot be directly associated using a standard <label for="...">
       (e.g., for elements like <div> or <span> with a specific ARIA role), aria-labelledby provides a way to link them.
  -->
  <section aria-labelledby="recent-activity">
    <h2 id="recent-activity">Recent Activity</h2>
    <!-- content -->
  </section>
</main>

<!-- ❌ BAD - Requires manual accessibility implementation -->
<div class="fake-button" (click)="submit()">
  Submit Form
</div>

<div class="navigation">
  <div class="nav-item" (click)="navigate('/dashboard')">Dashboard</div>
</div>
```

**Why Semantic HTML Matters**:
- **Screen readers** understand element roles and can navigate efficiently
- **Keyboard users** get automatic focus management and navigation
- **Voice control** software can identify interactive elements
- **Search engines** better understand page structure

### Heading Hierarchy **[MANDATORY]**

Maintain logical heading order for navigation landmarks.

```html
<!-- ✅ GOOD - Logical progression -->
<h1>E-commerce Dashboard</h1>
  <h2>Sales Overview</h2>
    <h3>This Month's Revenue</h3>
    <h3>Top-Selling Products</h3>
  <h2>Recent Orders</h2>
    <h3>Pending Orders</h3>
    <h3>Completed Orders</h3>
  <h2>User Management</h2>

<!-- ❌ BAD - Breaks navigation -->
<h1>Dashboard</h1>
<h3>Sales</h3> <!-- Skipped h2 -->
<h1>Orders</h1> <!-- Multiple h1s confuse hierarchy -->
<h4>Recent</h4> <!-- Skipped h2 and h3 -->
```

**Screen Reader Impact**: Users with screen readers rely on headings to navigate content quickly. They jump between heading levels to find relevant sections. Illogical hierarchy makes navigation impossible.

**Testing**: Use the Web Developer browser extension to view document outline and verify heading structure makes sense.

## Form Accessibility

### Labels and Form Controls **[MANDATORY]**

Every form control must have an accessible name.

```html
<!-- ✅ GOOD - Explicit association -->
<div class="form-field">
  <label for="email-input">Email Address</label>
  <input 
    id="email-input" 
    type="email" 
    name="email"
    aria-required="true"
    aria-describedby="email-help email-error"
    [attr.aria-invalid]="emailControl.invalid && emailControl.touched">
  
  <div id="email-help" class="help-text">
    We'll use this for account notifications
  </div>
  
  <div 
    id="email-error" 
    class="error-message"
    *ngIf="emailControl.invalid && emailControl.touched"
    aria-live="assertive">
    Please enter a valid email address
  </div>
</div>

<!-- ✅ GOOD - Implicit association for simple cases -->
<label class="checkbox-label">
  <input type="checkbox" [(ngModel)]="acceptTerms" aria-required="true">
  I accept the terms and conditions
</label>

<!-- ✅ GOOD - Fieldset for grouped inputs -->
<fieldset>
  <legend>Preferred Contact Method</legend>
  <label>
    <input type="radio" name="contact" value="email" [(ngModel)]="contactMethod">
    Email
  </label>
  <label>
    <input type="radio" name="contact" value="phone" [(ngModel)]="contactMethod">
    Phone
  </label>
  <label>
    <input type="radio" name="contact" value="mail" [(ngModel)]="contactMethod">
    Mail
  </label>
</fieldset>

<!-- ❌ BAD - No accessible name -->
<div class="form-row">
  <div class="label-text">Email</div>
  <input type="email"> <!-- Screen reader can't connect label -->
</div>

<!-- ❌ BAD - Placeholder as label -->
<input type="email" placeholder="Enter your email"> <!-- Disappears when typing -->
```

### Form Validation **[MANDATORY]**

Make validation messages accessible to screen readers.

```html
<form [formGroup]="userForm" (ngSubmit)="onSubmit()" novalidate>
  <div class="form-field">
    <label for="password">Password</label>
    <input 
      id="password" 
      type="password" 
      formControlName="password"
      aria-required="true"
      aria-describedby="password-requirements password-error"
      [attr.aria-invalid]="passwordControl.invalid && passwordControl.touched">
    
    <!-- Requirements always visible -->
    <ul id="password-requirements" class="requirements-list">
      <li [class.met]="hasMinLength">At least 8 characters</li>
      <li [class.met]="hasUppercase">One uppercase letter</li>
      <li [class.met]="hasNumber">One number</li>
    </ul>
    
    <!-- Error message announced when validation fails -->
    <div 
      id="password-error"
      class="error-message"
      *ngIf="passwordControl.invalid && passwordControl.touched"
      aria-live="assertive"
      role="alert">
      <span *ngIf="passwordControl.errors?.['required']">
        Password is required
      </span>
      <span *ngIf="passwordControl.errors?.['minlength']">
        Password must be at least 8 characters
      </span>
    </div>
  </div>
  
  <button type="submit" [disabled]="userForm.invalid">
    Create Account
  </button>
</form>
```

**Angular Implementation**:
```typescript
export class RegisterComponent {
  userForm = this.fb.group({
    password: ['', [
      Validators.required,
      Validators.minLength(8),
      this.customPasswordValidator
    ]]
  });
  
  get passwordControl() {
    return this.userForm.controls['password'];
  }
  
  get hasMinLength(): boolean {
    const value = this.passwordControl.value || '';
    return value.length >= 8;
  }
  
  get hasUppercase(): boolean {
    const value = this.passwordControl.value || '';
    return /[A-Z]/.test(value);
  }
  
  get hasNumber(): boolean {
    const value = this.passwordControl.value || '';
    return /\d/.test(value);
  }
  
  private customPasswordValidator(control: AbstractControl): ValidationErrors | null {
    const value = control.value || '';
    const hasUppercase = /[A-Z]/.test(value);
    const hasNumber = /\d/.test(value);
    
    if (!hasUppercase || !hasNumber) {
      return { customPassword: true };
    }
    
    return null;
  }
}
```

## ARIA Attributes

### ARIA Live Regions **[RECOMMENDED]**

Use ARIA live regions to announce dynamic content changes.

```html
<!-- Status messages that should interrupt immediately -->
<div 
  class="alert alert-error" 
  *ngIf="submitError"
  aria-live="assertive" 
  role="alert">
  Form submission failed. Please try again.
</div>

<!-- Status updates that can wait for a pause -->
<div 
  class="status-message" 
  *ngIf="saveStatus"
  aria-live="polite">
  {{ saveStatus }}
</div>

<!-- Loading states -->
<div 
  *ngIf="isLoading" 
  aria-live="polite" 
  aria-atomic="true">
  Loading user data, please wait...
</div>

<!-- Dynamic counters -->
<div class="character-counter">
  <span aria-live="polite">
    {{ remainingChars }} characters remaining
  </span>
</div>
```

**ARIA Live Region Types**:
- `aria-live="assertive"` - Interrupts screen reader immediately (errors, critical alerts)
- `aria-live="polite"` - Waits for pause in speech (status updates, confirmations)  
- `aria-atomic="true"` - Reads entire region when updated, not just changes
- `role="alert"` - Equivalent to `aria-live="assertive"`

### Interactive Elements **[MANDATORY]**

Provide proper roles and states for custom interactive components.

```html
<!-- Custom dropdown component -->
<div class="dropdown-container">
  <button 
    [attr.aria-expanded]="isOpen"
    aria-haspopup="listbox"
    aria-controls="user-options"
    (click)="toggle()"
    (keydown.arrowDown)="openAndFocusFirst()"
    class="dropdown-trigger">
    {{ selectedUser?.name || 'Select a user' }}
    <span aria-hidden="true">▼</span>
  </button>
  
  <ul 
    id="user-options"
    role="listbox"
    [attr.aria-activedescendant]="focusedOptionId"
    [hidden]="!isOpen"
    class="dropdown-menu"
    (keydown)="handleKeyboardNavigation($event)">
    
    <li 
      *ngFor="let user of users; trackBy: trackByUserId"
      [id]="'option-' + user.id"
      role="option"
      [attr.aria-selected]="selectedUser?.id === user.id"
      [class.focused]="focusedUserId === user.id"
      (click)="selectUser(user)">
      {{ user.name }}
    </li>
  </ul>
</div>

<!-- Toggle button with state -->
<button 
  [attr.aria-pressed]="isNotificationsEnabled"
  (click)="toggleNotifications()"
  class="toggle-button">
  <span class="sr-only">
    {{ isNotificationsEnabled ? 'Disable' : 'Enable' }} notifications
  </span>
  <span aria-hidden="true">🔔</span>
</button>

<!-- Custom close button -->
<button 
  aria-label="Close modal"
  (click)="closeModal()"
  class="close-button">
  <span aria-hidden="true">×</span>
</button>
```

**Implementation**:
```typescript
export class DropdownComponent {
  isOpen = false;
  selectedUser: User | null = null;
  focusedUserId: number | null = null;
  
  get focusedOptionId(): string | null {
    return this.focusedUserId ? `option-${this.focusedUserId}` : null;
  }
  
  openAndFocusFirst(): void {
    this.isOpen = true;
    if (this.users.length > 0) {
      this.focusedUserId = this.users[0].id;
    }
  }
  
  handleKeyboardNavigation(event: KeyboardEvent): void {
    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        this.focusNext();
        break;
      case 'ArrowUp':
        event.preventDefault();
        this.focusPrevious();
        break;
      case 'Enter':
      case ' ':
        event.preventDefault();
        this.selectFocusedUser();
        break;
      case 'Escape':
        this.close();
        break;
    }
  }
  
  private focusNext(): void {
    if (!this.focusedUserId) return;
    
    const currentIndex = this.users.findIndex(u => u.id === this.focusedUserId);
    const nextIndex = (currentIndex + 1) % this.users.length;
    this.focusedUserId = this.users[nextIndex].id;
  }
  
  private focusPrevious(): void {
    if (!this.focusedUserId) return;
    
    const currentIndex = this.users.findIndex(u => u.id === this.focusedUserId);
    const prevIndex = currentIndex === 0 ? this.users.length - 1 : currentIndex - 1;
    this.focusedUserId = this.users[prevIndex].id;
  }
}
```

## Keyboard Navigation

### Focus Management **[MANDATORY]**

Ensure logical focus order and visible focus indicators.

```typescript
@Component({
  template: `
    <div class="modal-overlay" *ngIf="isOpen" (click)="closeOnOverlayClick()">
      <div 
        class="modal-content"
        role="dialog"
        aria-labelledby="modal-title"
        aria-describedby="modal-description"
        #modalContent
        (keydown.escape)="close()">
        
        <h2 id="modal-title">{{ title }}</h2>
        <p id="modal-description">{{ description }}</p>
        
        <form (ngSubmit)="save()">
          <input type="text" [(ngModel)]="userData.name" #firstInput>
          <input type="email" [(ngModel)]="userData.email">
          
          <div class="modal-actions">
            <button type="button" (click)="close()">Cancel</button>
            <button type="submit">Save</button>
          </div>
        </form>
      </div>
    </div>
  `
})
export class ModalComponent implements AfterViewInit, OnDestroy {
  @ViewChild('modalContent') modalContent!: ElementRef<HTMLElement>;
  @ViewChild('firstInput') firstInput!: ElementRef<HTMLInputElement>;
  
  private previousFocusElement: HTMLElement | null = null;
  
  ngAfterViewInit(): void {
    if (this.isOpen) {
      this.trapFocus();
    }
  }
  
  ngOnDestroy(): void {
    this.restoreFocus();
  }
  
  open(): void {
    this.previousFocusElement = document.activeElement as HTMLElement;
    this.isOpen = true;
    
    // Focus first input after modal is rendered
    setTimeout(() => {
      this.firstInput.nativeElement.focus();
    }, 0);
  }
  
  close(): void {
    this.isOpen = false;
    this.restoreFocus();
  }
  
  private trapFocus(): void {
    const modalElement = this.modalContent.nativeElement;
    const focusableElements = modalElement.querySelectorAll(
      'button, input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    
    const firstElement = focusableElements[0] as HTMLElement;
    const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement;
    
    modalElement.addEventListener('keydown', (event: KeyboardEvent) => {
      if (event.key === 'Tab') {
        if (event.shiftKey) {
          if (document.activeElement === firstElement) {
            event.preventDefault();
            lastElement.focus();
          }
        } else {
          if (document.activeElement === lastElement) {
            event.preventDefault();
            firstElement.focus();
          }
        }
      }
    });
  }
  
  private restoreFocus(): void {
    if (this.previousFocusElement) {
      this.previousFocusElement.focus();
      this.previousFocusElement = null;
    }
  }
}
```

### Keyboard Shortcuts **[RECOMMENDED]**

```html
<!-- Skip navigation link for keyboard users -->
<a href="#main-content" class="skip-link">
  Skip to main content
</a>

<nav aria-label="Main navigation">
  <!-- navigation items -->
</nav>

<main id="main-content" tabindex="-1">
  <!-- main content -->
</main>

<!-- Custom interactive elements need keyboard support -->
<div 
  role="button"
  tabindex="0"
  [inlineSVG]="'./assets/icons/clear.svg'"
  (click)="clear()"
  (keydown.enter)="clear()"
  (keydown.space)="clear(); $event.preventDefault()"
  aria-label="Clear search results">
</div>
```

**CSS for Skip Links**:
```scss
.skip-link {
  position: absolute;
  top: -40px;
  left: 6px;
  background: #000;
  color: #fff;
  padding: 8px;
  text-decoration: none;
  z-index: 1000;
  
  &:focus {
    top: 6px;
  }
}

// Focus indicators for all interactive elements
button:focus,
input:focus,
select:focus,
textarea:focus,
a:focus,
[tabindex]:focus {
  outline: 2px solid #007acc;
  outline-offset: 2px;
}

// Never remove focus indicators without replacement
*:focus {
  outline: none; /* ❌ NEVER DO THIS */
}
```

## Images and Media

### Alternative Text **[MANDATORY]**

Provide context-appropriate descriptions for images.

```html
<!-- ✅ INFORMATIVE IMAGES - Describe the content/meaning -->
<img 
  src="sales-chart-q4.png" 
  alt="Q4 sales increased 23% compared to Q3, reaching $2.4 million">

<img 
  src="user-avatar.jpg" 
  alt="Profile picture of Sarah Johnson, Software Engineer">

<!-- ✅ FUNCTIONAL IMAGES - Describe the action -->
<button>
  <img src="edit-icon.svg" alt="Edit user profile">
</button>

<a href="/download/report.pdf">
  <img src="pdf-icon.svg" alt="Download quarterly report (PDF)">
</a>

<!-- ✅ DECORATIVE IMAGES - Empty alt, hidden from screen readers -->
<img 
  src="decorative-border.png" 
  alt="" 
  aria-hidden="true">

<!-- ✅ COMPLEX IMAGES - Use longdesc or accessible data table -->
<img 
  src="complex-chart.png" 
  alt="Revenue by department for 2024"
  aria-describedby="chart-details">

<div id="chart-details" class="sr-only">
  Detailed breakdown: Sales department generated $1.2M (40%), 
  Marketing $800K (27%), Engineering $600K (20%), 
  Operations $400K (13%).
</div>

<!-- ❌ BAD - Useless or redundant alt text -->
<img src="chart.png" alt="chart"> <!-- Doesn't describe the data -->
<img src="photo.jpg" alt="image"> <!-- Screen reader already says "image" -->
<img src="btn-submit.png" alt="submit button image"> <!-- Redundant -->
```

**Alt Text Decision Tree**:
1. **Is the image decorative only?** → Use `alt=""` and `aria-hidden="true"`
2. **Is the image functional (button, link)?** → Describe the action, not the image
3. **Does the image convey information?** → Describe the meaning/data, not appearance
4. **Is the image complex (chart, diagram)?** → Provide summary in alt, details elsewhere

### Video and Audio Content **[MANDATORY]**

```html
<!-- Video with captions and transcript -->
<video controls preload="metadata" aria-describedby="video-description">
  <source src="tutorial.mp4" type="video/mp4">
  <track kind="captions" src="tutorial-captions.vtt" srclang="en" label="English" default>
  <track kind="descriptions" src="tutorial-descriptions.vtt" srclang="en" label="Audio descriptions">
  Your browser does not support the video element.
</video>

<div id="video-description">
  <p>This 5-minute tutorial demonstrates how to create a new user account.</p>
  <details>
    <summary>Full transcript</summary>
    <div class="transcript">
      <p><strong>00:00 - 00:15:</strong> Welcome to our user management system...</p>
      <p><strong>00:15 - 00:30:</strong> To create a new account, click the "Add User" button...</p>
    </div>
  </details>
</div>

<!-- Audio with controls and transcript -->
<audio controls aria-describedby="audio-description">
  <source src="announcement.mp3" type="audio/mpeg">
  <source src="announcement.ogg" type="audio/ogg">
  Your browser does not support the audio element.
</audio>

<div id="audio-description">
  <p>Company announcement regarding new remote work policy changes.</p>
  <a href="announcement-transcript.html">View full transcript</a>
</div>
```

## Color and Contrast

### Contrast Requirements **[MANDATORY]**

Ensure sufficient color contrast for readability.

```scss
// ✅ GOOD - Meets WCAG AA standards
.primary-text {
  color: #212529; // Dark gray
  background: #ffffff; // White
  // Contrast ratio: 16.75:1 (exceeds 4.5:1 requirement)
}

.secondary-text {
  color: #6c757d; // Medium gray
  background: #ffffff; // White
  // Contrast ratio: 4.54:1 (meets 4.5:1 requirement)
}

.large-text {
  color: #495057; // Lighter gray - acceptable for large text only
  background: #ffffff; // White
  font-size: 18px; // Large text only needs 3:1 ratio
  // Contrast ratio: 9.35:1 (exceeds 3:1 requirement for large text)
}

.error-text {
  color: #dc3545; // Red
  background: #ffffff; // White
  // Contrast ratio: 5.52:1
}

// ❌ BAD - Insufficient contrast
.poor-contrast {
  color: #cccccc; // Light gray
  background: #ffffff; // White
  // Contrast ratio: 1.61:1 (fails WCAG standards)
}
```

### Don't Rely on Color Alone **[MANDATORY]**

Supplement color with icons, patterns, or text to convey information.

```html
<!-- ✅ GOOD - Multiple visual cues -->
<div class="form-field" [class.error]="hasError" [class.success]="isValid">
  <label for="email">Email Address</label>
  <input 
    id="email" 
    type="email" 
    [attr.aria-invalid]="hasError"
    class="form-control">
  
  <!-- Icon + color + text for error state -->
  <div *ngIf="hasError" class="validation-message error">
    <span class="icon" aria-hidden="true">⚠️</span>
    <span>Please enter a valid email address</span>
  </div>
  
  <!-- Icon + color + text for success state -->
  <div *ngIf="isValid" class="validation-message success">
    <span class="icon" aria-hidden="true">✅</span>
    <span>Email address is valid</span>
  </div>
</div>

<!-- Status indicators with patterns -->
<div class="status-list">
  <div class="status-item online">
    <span class="status-indicator online" aria-hidden="true"></span>
    <span class="status-text">John Doe</span>
    <span class="sr-only">is online</span>
  </div>
  <div class="status-item busy">
    <span class="status-indicator busy" aria-hidden="true"></span>
    <span class="status-text">Jane Smith</span>
    <span class="sr-only">is busy</span>
  </div>
  <div class="status-item offline">
    <span class="status-indicator offline" aria-hidden="true"></span>
    <span class="status-text">Bob Wilson</span>
    <span class="sr-only">is offline</span>
  </div>
</div>

<!-- ❌ BAD - Color-only indicators -->
<div class="status-item" style="color: green;">John Doe</div>
<div class="status-item" style="color: red;">Jane Smith</div>
```

**CSS for Visual Patterns**:
```scss
.status-indicator {
  display: inline-block;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  margin-right: 8px;
  
  &.online {
    background: #28a745;
  }
  
  &.busy {
    background: #ffc107;
    // Add diagonal stripes pattern for color-blind users
    background-image: repeating-linear-gradient(
      -45deg,
      transparent,
      transparent 2px,
      rgba(0,0,0,0.2) 2px,
      rgba(0,0,0,0.2) 4px
    );
  }
  
  &.offline {
    background: #dc3545;
    // Add dots pattern
    background-image: radial-gradient(
      circle at 2px 2px,
      rgba(255,255,255,0.5) 1px,
      transparent 1px
    );
    background-size: 4px 4px;
  }
}
```

## Testing Accessibility

### Automated Testing **[RECOMMENDED]**

Use multiple tools as automated tests catch only ~30% of accessibility issues.

```typescript
// Angular component testing with accessibility checks
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { axe, toHaveNoViolations } from 'jest-axe';

describe('UserFormComponent Accessibility', () => {
  let component: UserFormComponent;
  let fixture: ComponentFixture<UserFormComponent>;
  
  beforeEach(() => {
    expect.extend(toHaveNoViolations);
  });
  
  it('should be accessible', async () => {
    fixture.detectChanges();
    const results = await axe(fixture.nativeElement);
    expect(results).toHaveNoViolations();
  });
  
  it('should have proper form labels', () => {
    fixture.detectChanges();
    const emailInput = fixture.debugElement.query(By.css('#email-input'));
    const emailLabel = fixture.debugElement.query(By.css('label[for="email-input"]'));
    
    expect(emailInput).toBeTruthy();
    expect(emailLabel).toBeTruthy();
    expect(emailLabel.nativeElement.textContent.trim()).toBe('Email Address');
  });
  
  it('should announce validation errors', async () => {
    component.userForm.controls['email'].setErrors({ required: true });
    component.userForm.controls['email'].markAsTouched();
    fixture.detectChanges();
    
    const errorMessage = fixture.debugElement.query(By.css('[aria-live="assertive"]'));
    expect(errorMessage).toBeTruthy();
    expect(errorMessage.nativeElement.textContent).toContain('Email is required');
  });
});
```

### Manual Testing Checklist **[MANDATORY]**

**Keyboard Navigation Testing**:
- [ ] Tab through entire interface - all interactive elements focusable
- [ ] Shift+Tab works in reverse order
- [ ] Enter and Space activate buttons appropriately
- [ ] Arrow keys work for custom components (dropdowns, tabs)
- [ ] Escape closes modals and dropdowns
- [ ] Focus indicators visible and clear
- [ ] Focus doesn't get trapped inappropriately
- [ ] Skip links work for main content

**Screen Reader Testing** (use NVDA on Windows or VoiceOver on Mac):
- [ ] All content is announced logically
- [ ] Form labels are associated and announced
- [ ] Validation errors are announced immediately
- [ ] Dynamic content changes are announced
- [ ] Images have appropriate alt text
- [ ] Heading structure makes sense for navigation
- [ ] Tables have proper headers

**Visual Testing**:
- [ ] Interface works at 200% zoom level
- [ ] Content reflows appropriately on mobile
- [ ] Color contrast meets WCAG standards
- [ ] Information isn't conveyed by color alone
- [ ] Text remains readable when user stylesheets are applied

### Browser Extension Tools **[RECOMMENDED]**

1. **axe DevTools** - Most comprehensive automated testing
2. **WAVE** - Visual accessibility evaluation
3. **Lighthouse** - Quick accessibility audit in Chrome DevTools
4. **Colour Contrast Analyser** - Precise contrast ratio checking
5. **Web Developer** - Document outline and structure analysis

### Testing with Real Users **[ADVANCED]**

```typescript
// User testing scenario template
interface AccessibilityTestScenario {
  userProfile: 'screenReader' | 'keyboardOnly' | 'lowVision' | 'cognitiveImpairment';
  task: string;
  successCriteria: string[];
  assistiveTechnology?: string;
  expectedChallenges: string[];
}

const testScenarios: AccessibilityTestScenario[] = [
  {
    userProfile: 'screenReader',
    task: 'Create a new user account',
    successCriteria: [
      'Can navigate form using heading structure',
      'All form fields have clear labels',
      'Validation errors are announced immediately',
      'Success confirmation is announced'
    ],
    assistiveTechnology: 'NVDA or JAWS',
    expectedChallenges: [
      'Complex form validation',
      'Dynamic content updates',
      'Modal focus management'
    ]
  },
  {
    userProfile: 'keyboardOnly',
    task: 'Search and filter user list',
    successCriteria: [
      'Can access all filters using Tab key',
      'Search results update without mouse',
      'Can navigate through results with arrow keys',
      'Can perform all actions keyboard-only'
    ],
    expectedChallenges: [
      'Custom dropdown components',
      'Drag and drop interactions',
      'Complex data table navigation'
    ]
  }
];
```

## Common Accessibility Anti-Patterns

### Avoid These Mistakes **[MANDATORY]**

```html
<!-- ❌ BAD - Removes focus indicators -->
<style>
*:focus { outline: none; }
</style>

<!-- ❌ BAD - Inaccessible custom components -->
<div class="fake-button" (click)="submit()">Submit</div>
<div class="fake-select" (click)="showOptions()">Choose option</div>

<!-- ❌ BAD - Auto-playing media -->
<video autoplay loop>
  <source src="background-video.mp4">
</video>

<!-- ❌ BAD - Time-limited content without controls -->
<div class="notification" *ngIf="showNotification">
  <!-- Disappears after 3 seconds with no way to extend -->
  Important message that users might miss
</div>

<!-- ❌ BAD - Placeholder as label -->
<input type="email" placeholder="Enter email address">
<!-- Placeholder disappears when typing, no persistent label -->

<!-- ❌ BAD - Non-descriptive link text -->
<a href="/user/123">Click here</a>
<a href="/report.pdf">Read more</a>

<!-- ❌ BAD - Empty headings or skipped levels -->
<h1>Dashboard</h1>
<h3>User Stats</h3> <!-- Skipped h2 -->
<h2></h2> <!-- Empty heading -->

<!-- ❌ BAD - Redundant alt text -->
<img src="user-photo.jpg" alt="Photo of user John Doe's photo">
<button>
  <img src="save-icon.png" alt="Save icon"> Save
  <!-- Alt text is redundant with button text -->
</button>

<!-- ❌ BAD - Form without labels -->
<form>
  <input type="text" name="firstName">
  <input type="email" name="email">
  <button type="submit">Submit</button>
</form>
```

### Correct Implementations **[RECOMMENDED]**

```html
<!-- ✅ GOOD - Accessible alternatives -->
<style>
button:focus,
input:focus,
a:focus {
  outline: 2px solid #007acc;
  outline-offset: 2px;
}
</style>

<!-- ✅ GOOD - Proper semantic components -->
<button type="submit" (click)="submit()">Submit</button>
<select aria-label="Choose option" (change)="onOptionChange($event)">
  <option value="">Select an option</option>
  <option value="1">Option 1</option>
</select>

<!-- ✅ GOOD - User-controlled media -->
<video controls preload="metadata" aria-label="Product demonstration video">
  <source src="demo-video.mp4">
  <track kind="captions" src="captions.vtt" default>
</video>

<!-- ✅ GOOD - Persistent notifications with controls -->
<div 
  class="notification" 
  *ngIf="showNotification"
  role="alert"
  aria-live="assertive">
  <span>Important message about your account</span>
  <button (click)="dismissNotification()" aria-label="Dismiss notification">
    <span aria-hidden="true">×</span>
  </button>
</div>

<!-- ✅ GOOD - Proper labels with helpful placeholders -->
<label for="email">Email Address</label>
<input 
  id="email" 
  type="email" 
  placeholder="example@company.com"
  aria-describedby="email-help">
<div id="email-help">We'll use this for account recovery</div>

<!-- ✅ GOOD - Descriptive link text -->
<a href="/user/123">View John Doe's profile</a>
<a href="/report.pdf">Download Q4 financial report (PDF, 2.3MB)</a>

<!-- ✅ GOOD - Logical heading structure -->
<h1>User Management Dashboard</h1>
<h2>User Statistics</h2>
<h3>Active Users This Month</h3>
<h3>New Registrations</h3>
<h2>Recent Activity</h2>

<!-- ✅ GOOD - Meaningful alt text -->
<img src="user-photo.jpg" alt="John Doe, Senior Developer">
<button>
  <img src="save-icon.png" alt="" aria-hidden="true"> Save Changes
</button>

<!-- ✅ GOOD - Accessible form with proper labels -->
<form>
  <div>
    <label for="firstName">First Name</label>
    <input id="firstName" type="text" name="firstName" required>
  </div>
  <div>
    <label for="email">Email Address</label>
    <input id="email" type="email" name="email" required>
  </div>
  <button type="submit">Create Account</button>
</form>
```

## Advanced Accessibility Patterns

### Progressive Enhancement **[RECOMMENDED]**

Build accessibility in from the start, don't retrofit it.

```html
<!-- Base HTML that works without JavaScript -->
<form action="/search" method="GET">
  <label for="search-input">Search users</label>
  <input id="search-input" type="search" name="q" required>
  <button type="submit">Search</button>
</form>

<!-- Enhanced with Angular for better UX -->
<form [formGroup]="searchForm" (ngSubmit)="onSearch()">
  <label for="search-input">Search users</label>
  <input 
    id="search-input" 
    type="search" 
    formControlName="query"
    [attr.aria-expanded]="showSuggestions"
    [attr.aria-activedescendant]="activeSuggestionId"
    aria-autocomplete="list"
    aria-describedby="search-help"
    (input)="onSearchInput($event)"
    (keydown)="handleSearchKeydown($event)">
  
  <div id="search-help">
    Use keywords to find users by name, email, or department
  </div>
  
  <!-- Suggestions enhanced with ARIA -->
  <ul 
    *ngIf="showSuggestions"
    role="listbox"
    aria-label="Search suggestions"
    class="suggestions-list">
    <li 
      *ngFor="let suggestion of suggestions; let i = index"
      [id]="'suggestion-' + i"
      role="option"
      [attr.aria-selected]="i === activeSuggestionIndex"
      (click)="selectSuggestion(suggestion)"
      class="suggestion-item">
      {{ suggestion.name }} - {{ suggestion.department }}
    </li>
  </ul>
  
  <button type="submit" [disabled]="searchForm.invalid">
    Search
  </button>
</form>
```

### Accessible Data Tables **[ADVANCED]**

```html
<table role="table" aria-labelledby="users-table-caption">
  <caption id="users-table-caption">
    User accounts ({{ users.length }} total)
  </caption>
  
  <thead>
    <tr>
      <th scope="col" 
          [attr.aria-sort]="getSortDirection('name')"
          (click)="sortBy('name')"
          (keydown.enter)="sortBy('name')"
          tabindex="0">
        Name
      </th>
      <th scope="col" 
          [attr.aria-sort]="getSortDirection('email')"
          (click)="sortBy('email')"
          (keydown.enter)="sortBy('email')"
          tabindex="0">
        Email
      </th>
      <th scope="col">Department</th>
      <th scope="col">Actions</th>
    </tr>
  </thead>
  
  <tbody>
    <tr *ngFor="let user of users; trackBy: trackByUserId">
      <th scope="row">{{ user.name }}</th>
      <td>{{ user.email }}</td>
      <td>{{ user.department }}</td>
      <td>
        <button 
          [attr.aria-label]="'Edit ' + user.name + ' profile'"
          (click)="editUser(user)">
          Edit
        </button>
        <button 
          [attr.aria-label]="'Delete ' + user.name + ' account'"
          (click)="deleteUser(user)"
          class="btn-danger">
          Delete
        </button>
      </td>
    </tr>
  </tbody>
</table>
```

This comprehensive accessibility guide ensures your Angular applications work for everyone, not just users without disabilities. Remember: accessibility is not a feature to add later—it's a foundational aspect of good web development.
