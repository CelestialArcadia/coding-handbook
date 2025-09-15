# Angular Guidelines

A more in-depth Angular-focused Component architecture, services, dependency injection, and framework-specific best practices for maintainable applications.

## Component Architecture

### Component Declaration **[MANDATORY]**

```typescript
// ✅ GOOD
@Component({
  selector: 'app-user-profile',
  templateUrl: './user-profile.component.html',
  styleUrls: ['./user-profile.component.scss']
})
export class UserProfileComponent implements OnInit, OnDestroy {
  @Input() userId: number;
  @Output() userUpdated = new EventEmitter<User>();
  
  private destroy$ = new Subject<void>();
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// ❌ BAD
export class userProfile {
  @Input('id') userId: number;
  @Output() updated = new EventEmitter();
}
```

**Rules**:
- Use PascalCase for component class names
- Implement lifecycle interfaces explicitly
- Use descriptive `@Input` and `@Output` names
- Handle subscriptions cleanup in `ngOnDestroy`

### Template Safety **[MANDATORY]**

```html
<!-- ✅ GOOD - Safe navigation prevents runtime errors -->
<div class="user-info">
  <h2>{{ user?.profile?.displayName || 'Guest User' }}</h2>
  <p>{{ user?.email || 'No email provided' }}</p>
</div>

<!-- ✅ GOOD - Structural directive with safe checks -->
<div *ngIf="users$ | async as users">
  <div *ngFor="let user of users; trackBy: trackByUserId">
    {{ user.name }}
  </div>
</div>

<!-- ❌ BAD - Potential null reference errors -->
<div>{{ user.profile.displayName }}</div>
<div *ngFor="let user of users">{{ user.name }}</div>
```

**Why**: Template errors break the entire component. Safe navigation and proper null checks prevent runtime failures.

## Dependency Injection

### Modern Injection Pattern **[RECOMMENDED]**

Angular v14+ provides the `inject()` function as an alternative to constructor injection.

```typescript
// ✅ NEW APPROACH - Clean and functional
export class UserService {
  private http = inject(HttpClient);
  private router = inject(Router);
  private notificationService = inject(NotificationService);
  
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
}

// ✅ TRADITIONAL APPROACH - Still valid
export class UserService {
  constructor(
    private http: HttpClient,
    private router: Router,
    private notificationService: NotificationService
  ) {}
}
```

### When to Use Each Approach

| Situation | Constructor Injection | inject() Function |
|-----------|----------------------|-------------------|
| **Few dependencies (1-3)** | ✅ Clear signature | Optional |
| **Many dependencies (4+)** | ❌ Bloated constructor | ✅ Clean field injection |
| **Functional APIs** (guards, resolvers) | ❌ Not available | ✅ Required |
| **DI decorators needed** (@Optional, @Self) | ✅ Required | ❌ Not supported |
| **Testing** | ✅ Easy to mock in constructor | ⚠️ Requires TestBed setup |

## Models and Data Flow

### Model Organization **[RECOMMENDED]**

Keep backend alignment while supporting frontend needs through view models.

```typescript
// user.model.ts - Direct backend mapping
export interface User {
  id: number;
  firstName: string;
  lastName: string;
  email: string;
  isActive: boolean;
  createdAt: string; // ISO string from backend
  roles: Role[];
}

// user.view.model.ts - Frontend-optimized
export interface UserViewModel {
  id: number;
  fullName: string;
  email: string;
  displayStatus: 'Active' | 'Inactive';
  memberSince: Date;
  primaryRole: string;
}

// user.mapper.ts - Transformation logic
export class UserMapper {
  static toViewModel(user: User): UserViewModel {
    return {
      id: user.id,
      fullName: `${user.firstName} ${user.lastName}`,
      email: user.email,
      displayStatus: user.isActive ? 'Active' : 'Inactive',
      memberSince: new Date(user.createdAt),
      primaryRole: user.roles[0]?.name || 'No role assigned'
    };
  }
}
```

**Benefits**:
- Backend changes don't immediately break frontend
- Frontend can optimize data structure for UI needs
- Clear separation between API contracts and presentation logic

### Associative Models **[RECOMMENDED]**

Create explicit models for many-to-many relationships to improve code clarity.

```typescript
// user-role.association.model.ts
export interface UserRoleAssociation {
  userId: number;
  roleId: number;
  assignedAt: Date;
  assignedBy: number;
  isActive: boolean;
}

// Usage in services
export class UserRoleService {
  assignRole(userId: number, roleId: number): Observable<UserRoleAssociation> {
    const assignment: Partial<UserRoleAssociation> = {
      userId,
      roleId,
      assignedAt: new Date(),
      assignedBy: this.currentUserId,
      isActive: true
    };
    
    return this.http.post<UserRoleAssociation>('/api/user-roles', assignment);
  }
}
```

## Service Architecture

### Service Design **[MANDATORY]**

```typescript
@Injectable({
  providedIn: 'root'
})
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = '/api/users';
  
  // Public API - what components use
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl)
      .pipe(
        catchError(this._handleError),
        map(users => users.map(user => this._sanitizeUser(user)))
      );
  }
  
  getUserById(id: number): Observable<User | undefined> {
    return this.http.get<User>(`${this.apiUrl}/${id}`)
      .pipe(
        catchError(this._handleError),
        map(user => user ? this._sanitizeUser(user) : undefined)
      );
  }
  
  // Private implementation - internal logic only
  private _handleError(error: HttpErrorResponse): Observable<never> {
    console.error('User service error:', error);
    return throwError(() => new Error(USER_SERVICE_ERROR));
  }
  
  private _sanitizeUser(user: User): User {
    // Remove sensitive data, normalize formats, etc.
    return { ...user, email: user.email.toLowerCase() };
  }
}
```

**Principles**:
- Public methods define the service contract
- Private methods (prefixed with `_`) handle implementation details
- Explicit return types improve IDE support and catch errors
- Use `Observable<Type | undefined>` instead of `Observable<any>`

## Forms Management

### Reactive Forms **[RECOMMENDED]**

```typescript
export class UserFormComponent implements OnInit {
  userForm = this.fb.group({
    firstName: ['', [Validators.required, Validators.minLength(2)]],
    lastName: ['', [Validators.required, Validators.minLength(2)]],
    email: ['', [Validators.required, Validators.email]],
    preferences: this.fb.group({
      notifications: [true],
      newsletter: [false]
    })
  });
  
  constructor(private fb: FormBuilder) {}
  
  onSubmit(): void {
    if (this.userForm.valid) {
      const formData = this.userForm.value;
      // TypeScript knows the structure due to FormBuilder typing
    }
  }
  
  // ✅ GOOD - Type-safe access with refactor support
  get firstNameControl() {
    return this.userForm.controls['firstName'];
  }
  
  // Alternative access pattern
  private getControlValue(controlName: string): any {
    return this.userForm.controls[controlName].value;
  }
}
```

### Form Validation **[RECOMMENDED]**

```html
<form [formGroup]="userForm" (ngSubmit)="onSubmit()">
  <div class="form-field">
    <label for="firstName">First Name</label>
    <input 
      id="firstName" 
      type="text" 
      formControlName="firstName"
      [attr.aria-invalid]="firstNameControl.invalid && firstNameControl.touched"
      aria-required="true">
    
    <span 
      *ngIf="firstNameControl.invalid && firstNameControl.touched" 
      class="error-message"
      aria-live="assertive">
      <span *ngIf="firstNameControl.errors?.['required']">
        First name is required
      </span>
      <span *ngIf="firstNameControl.errors?.['minlength']">
        First name must be at least 2 characters
      </span>
    </span>
  </div>
  
  <button type="submit" [disabled]="userForm.invalid">
    Save User
  </button>
</form>
```

## State Management

### Observable Patterns **[TEAM_PREFERENCE]**

Follow consistent naming for observables and subjects.

```typescript
export class UserStateService {
  // Private state - internal to service
  private _usersSubject = new BehaviorSubject<User[]>([]);
  private _loadingSubject = new BehaviorSubject<boolean>(false);
  private _errorSubject = new BehaviorSubject<string | null>(null);
  
  // Public observables - what components subscribe to
  public readonly users$ = this._usersSubject.asObservable();
  public readonly loading$ = this._loadingSubject.asObservable();
  public readonly error$ = this._errorSubject.asObservable();
  
  // Derived observables
  public readonly activeUsers$ = this.users$.pipe(
    map(users => users.filter(user => user.isActive))
  );
  
  // Actions - what components call
  loadUsers(): void {
    this._loadingSubject.next(true);
    this._errorSubject.next(null);
    
    this.userService.getUsers().subscribe({
      next: (users) => {
        this._usersSubject.next(users);
        this._loadingSubject.next(false);
      },
      error: (error) => {
        this._errorSubject.next('Failed to load users');
        this._loadingSubject.next(false);
      }
    });
  }
  
  private _updateUser(updatedUser: User): void {
    const currentUsers = this._usersSubject.value;
    const userIndex = currentUsers.findIndex(u => u.id === updatedUser.id);
    
    if (userIndex >= 0) {
      const updatedUsers = [...currentUsers];
      updatedUsers[userIndex] = updatedUser;
      this._usersSubject.next(updatedUsers);
    }
  }
}
```

**Naming Convention**:
- `privateDataSubject` - BehaviorSubject for internal state
- `publicData$` - Observable exposed to components  
- `_privateMethod()` - Internal service methods

## Performance Optimization

### Change Detection **[RECOMMENDED]**

```typescript
@Component({
  selector: 'app-user-list',
  templateUrl: './user-list.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush // Opt into performance
})
export class UserListComponent {
  @Input() users: User[] = [];
  
  // TrackBy function prevents unnecessary DOM updates
  trackByUserId(index: number, user: User): number {
    return user.id;
  }
}
```

```html
<div *ngFor="let user of users; trackBy: trackByUserId" class="user-card">
  <h3>{{ user.name }}</h3>
  <p>{{ user.email }}</p>
</div>
```

### Subscription Management **[MANDATORY]**

```typescript
export class UserComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    // ✅ GOOD - Automatic cleanup
    this.userService.getUsers().pipe(
      takeUntil(this.destroy$)
    ).subscribe(users => {
      this.users = users;
    });
    
    // ✅ GOOD - Multiple subscriptions with same cleanup
    merge(
      this.route.params,
      this.userService.currentUser$,
      this.permissionService.permissions$
    ).pipe(
      takeUntil(this.destroy$)
    ).subscribe(([params, user, permissions]) => {
      // Handle combined data
    });
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Why**: Unsubscribed observables cause memory leaks and unexpected behavior. The `takeUntil` pattern provides clean, centralized cleanup.

## Testing Patterns

### Component Testing **[RECOMMENDED]**

```typescript
describe('UserProfileComponent', () => {
  let component: UserProfileComponent;
  let fixture: ComponentFixture<UserProfileComponent>;
  let userService: jasmine.SpyObj<UserService>;
  
  beforeEach(async () => {
    const userServiceSpy = jasmine.createSpyObj('UserService', ['getUserById']);
    
    await TestBed.configureTestingModule({
      declarations: [UserProfileComponent],
      providers: [
        { provide: UserService, useValue: userServiceSpy }
      ]
    }).compileComponents();
    
    userService = TestBed.inject(UserService) as jasmine.SpyObj<UserService>;
  });
  
  beforeEach(() => {
    fixture = TestBed.createComponent(UserProfileComponent);
    component = fixture.componentInstance;
  });
  
  it('should display user name when user data is loaded', () => {
    // Arrange
    const mockUser = { id: 1, name: 'John Doe', email: 'john@example.com' };
    userService.getUserById.and.returnValue(of(mockUser));
    component.userId = 1;
    
    // Act
    fixture.detectChanges();
    
    // Assert
    const nameElement = fixture.debugElement.query(By.css('.user-name'));
    expect(nameElement.nativeElement.textContent).toBe('John Doe');
  });
  
  it('should emit userUpdated when save button is clicked', () => {
    // Arrange
    spyOn(component.userUpdated, 'emit');
    const mockUser = { id: 1, name: 'John Doe', email: 'john@example.com' };
    component.user = mockUser;
    
    // Act
    const saveButton = fixture.debugElement.query(By.css('.save-button'));
    saveButton.nativeElement.click();
    
    // Assert
    expect(component.userUpdated.emit).toHaveBeenCalledWith(mockUser);
  });
});
```

## Common Anti-Patterns

### Avoid These Patterns **[MANDATORY]**

```typescript
// ❌ BAD - Direct DOM manipulation
export class BadComponent {
  ngAfterViewInit(): void {
    document.getElementById('myButton').addEventListener('click', () => {
      // This bypasses Angular's change detection
    });
  }
}

// ❌ BAD - Nested subscriptions
export class BadComponent {
  loadUserData(): void {
    this.userService.getUser(this.userId).subscribe(user => {
      this.userService.getUserPreferences(user.id).subscribe(preferences => {
        this.userService.getUserRoles(user.id).subscribe(roles => {
          // Nested callback hell
        });
      });
    });
  }
}

// ❌ BAD - Mutating @Input properties
export class BadComponent {
  @Input() user: User;
  
  updateUser(): void {
    this.user.name = 'New Name'; // Mutates parent's data
    this.user.isActive = false;
  }
}
```

### Preferred Alternatives

```typescript
// ✅ GOOD - Use Angular's Renderer2 for DOM manipulation
export class GoodComponent {
  constructor(private renderer: Renderer2, private el: ElementRef) {}
  
  ngAfterViewInit(): void {
    const button = this.el.nativeElement.querySelector('.my-button');
    this.renderer.listen(button, 'click', () => {
      // Properly integrated with Angular
    });
  }
}

// ✅ GOOD - Use RxJS operators for combining observables
export class GoodComponent {
  loadUserData(): void {
    const userId$ = of(this.userId);
    
    userId$.pipe(
      switchMap(userId => 
        combineLatest([
          this.userService.getUser(userId),
          this.userService.getUserPreferences(userId),
          this.userService.getUserRoles(userId)
        ])
      ),
      takeUntil(this.destroy$)
    ).subscribe(([user, preferences, roles]) => {
      // Clean, flat data loading
    });
  }
}

// ✅ GOOD - Emit changes instead of mutating inputs
export class GoodComponent {
  @Input() user: User;
  @Output() userChanged = new EventEmitter<Partial<User>>();
  
  updateUser(): void {
    this.userChanged.emit({
      name: 'New Name',
      isActive: false
    });
  }
}
```
