# TypeScript Guidelines

Type-safe coding standards for maintainable, scalable TypeScript applications with clear examples and reasoning.

## Variable Declarations

### Declaration Keywords **[MANDATORY]**

```typescript
// ✅ GOOD - Clear intent and scope
const API_BASE_URL = 'https://api.example.com';
const users = await fetchUsers();
let currentPage = 1;
let isLoading = false;

// ❌ BAD - Outdated and problematic
var globalCounter = 0; // Function-scoped, can be redeclared
var users = getData();
```

**Rules**:
- Use `const` for values that won't be reassigned
- Use `let` for variables that will be reassigned  
- Never use `var` - it has function scope and can be redeclared

**Why**: `const` and `let` have block scope, preventing accidental redeclaration and making code more predictable.

### Type Annotations **[RECOMMENDED]**

```typescript
// ✅ GOOD - Explicit when TypeScript can't infer
const userId: number = parseInt(routeParams.id);
const userPreferences: UserPreferences = {
  theme: 'dark',
  notifications: true
};

// ✅ GOOD - Let TypeScript infer obvious types
const userName = 'John Doe'; // string inferred
const isActive = true; // boolean inferred
const items = [1, 2, 3]; // number[] inferred

// ❌ UNNECESSARY - TypeScript already knows
const message: string = 'Hello'; // string is obvious
const count: number = 42; // number is obvious
```

**Guideline**: Add type annotations when they clarify intent or when TypeScript's inference isn't sufficient.

## Function Declarations

### Function Syntax **[MANDATORY]**

```typescript
// ✅ GOOD - Arrow functions with explicit return types
const calculateTotal = (items: CartItem[]): number => {
  return items.reduce((sum, item) => sum + item.price, 0);
};

const formatCurrency = (amount: number, currency = 'USD'): string => {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency
  }).format(amount);
};

// ❌ BAD - Function expressions without types
const calculateTotal = function(items) {
  return items.reduce((sum, item) => sum + item.price, 0);
};
```

**Benefits of Arrow Functions**:
- Lexical `this` binding (no context confusion)
- Consistent syntax across codebase
- More concise for simple functions

### Function Overloading **[RECOMMENDED]**

```typescript
// ✅ GOOD - Clear overloads for different use cases
function findUser(id: number): Promise<User | undefined>;
function findUser(email: string): Promise<User | undefined>;
function findUser(criteria: Partial<User>): Promise<User[]>;
function findUser(
  idOrEmailOrCriteria: number | string | Partial<User>
): Promise<User | User[] | undefined> {
  if (typeof idOrEmailOrCriteria === 'number') {
    return userRepository.findById(idOrEmailOrCriteria);
  }
  
  if (typeof idOrEmailOrCriteria === 'string') {
    return userRepository.findByEmail(idOrEmailOrCriteria);
  }
  
  return userRepository.findByCriteria(idOrEmailOrCriteria);
}

// Usage - TypeScript knows return types
const user1 = await findUser(123); // User | undefined
const user2 = await findUser('john@example.com'); // User | undefined  
const users = await findUser({ isActive: true }); // User[]
```

### Generic Functions **[RECOMMENDED]**

```typescript
// ✅ GOOD - Reusable with type safety
function mapToViewModel<T, U>(
  items: T[], 
  mapper: (item: T) => U
): U[] {
  return items.map(mapper);
}

function createRepository<T extends { id: number }>(
  entityName: string
): Repository<T> {
  return new GenericRepository<T>(entityName);
}

// Usage
const userViewModels = mapToViewModel(users, UserMapper.toViewModel);
const userRepo = createRepository<User>('users');

// ❌ BAD - Loses type information
function mapItems(items: any[], mapper: Function): any[] {
  return items.map(mapper);
}
```

## Type Definitions

### Interfaces vs Types **[RECOMMENDED]**

Use interfaces for object shapes, type aliases for complex types.

```typescript
// ✅ GOOD - Interface for object structure
interface User {
  readonly id: number;
  name: string;
  email: string;
  isActive: boolean;
  roles: Role[];
  
  // Optional properties
  avatar?: string;
  lastLoginAt?: Date;
}

// ✅ GOOD - Interface extension
interface AdminUser extends User {
  permissions: Permission[];
  canModifyUsers: boolean;
}

// ✅ GOOD - Type aliases for unions, functions, complex types
type Status = 'pending' | 'approved' | 'rejected';
type EventHandler<T> = (event: T) => void;
type ApiResponse<T> = {
  data: T;
  success: boolean;
  message?: string;
};

// ❌ AVOID - Type for simple objects (use interface)
type User = {
  id: number;
  name: string;
};
```

**When to Use Each**:
- **Interfaces**: Object shapes, can be extended, can be merged
- **Types**: Union types, computed types, function signatures

### Advanced Type Patterns **[RECOMMENDED]**

```typescript
// Utility types for safer code
type CreateUserRequest = Omit<User, 'id' | 'createdAt'>;
type UpdateUserRequest = Partial<Pick<User, 'name' | 'email' | 'isActive'>>;
type UserKeys = keyof User; // 'id' | 'name' | 'email' | 'isActive'

// Mapped types for consistency
type UserFormErrors = {
  [K in keyof CreateUserRequest]?: string;
};

// Conditional types for complex logic
type ApiResult<T> = T extends string 
  ? { message: T }
  : { data: T };

// Template literal types for type-safe strings
type EventType = 'user' | 'order' | 'product';
type EventAction = 'created' | 'updated' | 'deleted';
type EventName = `${EventType}:${EventAction}`; // 'user:created', 'order:updated', etc.

// Usage
const handleEvent = (eventName: EventName, data: unknown): void => {
  // TypeScript ensures only valid event names are passed
};

handleEvent('user:created', userData); // ✅ Valid
handleEvent('invalid:event', data); // ❌ TypeScript error
```

## Class Design

### Class Structure **[RECOMMENDED]**

```typescript
abstract class BaseEntity {
  constructor(
    public readonly id: number,
    public readonly createdAt: Date
  ) {}
  
  abstract validate(): boolean;
  
  protected formatDate(date: Date): string {
    return date.toISOString();
  }
}

class User extends BaseEntity {
  private _isActive: boolean = true;
  
  constructor(
    id: number,
    createdAt: Date,
    public name: string,
    public email: string
  ) {
    super(id, createdAt);
  }
  
  // Getter/setter for controlled access
  get isActive(): boolean {
    return this._isActive;
  }
  
  set isActive(value: boolean) {
    if (this._isActive !== value) {
      this._isActive = value;
      this.logStatusChange(value);
    }
  }
  
  // Override abstract method
  override validate(): boolean {
    return this.name.length > 0 && this.email.includes('@');
  }
  
  // Private methods prefixed with underscore
  private logStatusChange(newStatus: boolean): void {
    console.log(`User ${this.name} status changed to: ${newStatus}`);
  }
}
```

### Method Override **[RECOMMENDED]**

```typescript
class BaseComponent {
  protected initialize(): void {
    console.log('Base initialization');
  }
  
  public render(): void {
    this.initialize();
    this.doRender();
  }
  
  protected abstract doRender(): void;
}

class UserComponent extends BaseComponent {
  // ✅ GOOD - Explicit override annotation
  protected override initialize(): void {
    super.initialize();
    console.log('User component initialization');
  }
  
  protected doRender(): void {
    // Render user-specific content
  }
  
  // ❌ BAD - Missing override (still works but less clear)
  protected initialize(): void {
    super.initialize();
    console.log('User component initialization');
  }
}
```

**Benefits of `override`**:
- Makes inheritance relationships explicit
- Catches errors when parent method signatures change
- Improves code readability and maintainability

## Error Handling

### Type-Safe Error Handling **[RECOMMENDED]**

```typescript
// Define specific error types
class ValidationError extends Error {
  constructor(
    message: string,
    public field: string,
    public code: string
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

class NotFoundError extends Error {
  constructor(resource: string, id: string | number) {
    super(`${resource} with id ${id} not found`);
    this.name = 'NotFoundError';
  }
}

// Result type for operations that can fail
type Result<T, E = Error> = 
  | { success: true; data: T }
  | { success: false; error: E };

// Usage in service methods
class UserService {
  async createUser(userData: CreateUserRequest): Promise<Result<User, ValidationError>> {
    try {
      // Validate input
      if (!userData.email.includes('@')) {
        return {
          success: false,
          error: new ValidationError('Invalid email format', 'email', 'INVALID_EMAIL')
        };
      }
      
      const user = await this.repository.create(userData);
      return { success: true, data: user };
      
    } catch (error) {
      return {
        success: false,
        error: error instanceof ValidationError ? error : new Error('Unknown error')
      };
    }
  }
  
  // Alternative: Maybe pattern
  async findUserById(id: number): Promise<User | undefined> {
    try {
      return await this.repository.findById(id);
    } catch {
      return undefined; // Clear contract: undefined means not found or error
    }
  }
}

// Type-safe error handling in components
const handleCreateUser = async (userData: CreateUserRequest): Promise<void> => {
  const result = await userService.createUser(userData);
  
  if (result.success) {
    // TypeScript knows result.data is User
    console.log('Created user:', result.data.name);
  } else {
    // TypeScript knows result.error is ValidationError
    if (result.error instanceof ValidationError) {
      showFieldError(result.error.field, result.error.message);
    }
  }
};
```

## Utility Types and Helpers

### Common Utility Patterns **[RECOMMENDED]**

```typescript
// Type-safe environment configuration
interface EnvironmentConfig {
  apiUrl: string;
  apiKey: string;
  enableLogging: boolean;
  features: {
    userManagement: boolean;
    analytics: boolean;
  };
}

// Ensure all required env vars are typed
const createConfig = (): EnvironmentConfig => ({
  apiUrl: process.env['API_URL'] || 'http://localhost:3000',
  apiKey: process.env['API_KEY'] || '',
  enableLogging: process.env['NODE_ENV'] !== 'production',
  features: {
    userManagement: process.env['FEATURE_USER_MGMT'] === 'true',
    analytics: process.env['FEATURE_ANALYTICS'] === 'true'
  }
});

// Type guards for runtime type checking
const isString = (value: unknown): value is string => {
  return typeof value === 'string';
};

const isUser = (obj: unknown): obj is User => {
  return obj !== null && 
         typeof obj === 'object' && 
         'id' in obj && 
         'name' in obj && 
         'email' in obj;
};

// Safe type assertions
const parseUserFromApi = (response: unknown): User => {
  if (isUser(response)) {
    return response;
  }
  throw new Error('Invalid user data received from API');
};

// Branded types for ID safety
type UserId = number & { readonly __brand: 'UserId' };
type OrderId = number & { readonly __brand: 'OrderId' };

const createUserId = (id: number): UserId => id as UserId;
const createOrderId = (id: number): OrderId => id as OrderId;

// Now these can't be accidentally mixed
const getUserOrders = (userId: UserId): Promise<Order[]> => {
  // Implementation
  return Promise.resolve([]);
};

const userId = createUserId(123);
const orderId = createOrderId(456);

getUserOrders(userId); // ✅ Correct
getUserOrders(orderId); // ❌ TypeScript error - can't pass OrderId where UserId expected
```
## Best Practices Summary

### Type Safety Checklist **[RECOMMENDED]**

- [ ] **Strict TypeScript config** - Enable `strict: true`, `noImplicitAny`, `strictNullChecks`
- [ ] **Explicit return types** for public functions and methods
- [ ] **Type guards** for runtime type checking of external data
- [ ] **Utility types** (`Partial`, `Pick`, `Omit`) instead of manual type definitions
- [ ] **Generic constraints** (`<T extends SomeType>`) for type safety
- [ ] **Branded types** for primitive values that shouldn't be interchangeable
- [ ] **Result types** or explicit error handling instead of throwing exceptions
- [ ] **Interface segregation** - small, focused interfaces over large ones

### Performance Considerations **[RECOMMENDED]**

```typescript
// ✅ GOOD - Lazy evaluation with computed properties
class UserViewModel {
  constructor(private user: User) {}
  
  // Computed only when accessed
  get fullName(): string {
    return `${this.user.firstName} ${this.user.lastName}`;
  }
  
  get displayStatus(): string {
    return this.user.isActive ? 'Active' : 'Inactive';
  }
}

// ✅ GOOD - Memoization for expensive calculations
const memoizedCalculateRisk = (() => {
  const cache = new Map<string, number>();
  
  return (user: User, transactions: Transaction[]): number => {
    const key = `${user.id}-${transactions.length}`;
    
    if (cache.has(key)) {
      return cache.get(key)!;
    }
    
    const risk = calculateUserRisk(user, transactions); // Expensive operation
    cache.set(key, risk);
    return risk;
  };
})();

// ❌ AVOID - Computing values in constructor
class BadUserViewModel {
  public fullName: string;
  public displayStatus: string;
  
  constructor(user: User) {
    // These are computed even if never used
    this.fullName = `${user.firstName} ${user.lastName}`;
    this.displayStatus = user.isActive ? 'Active' : 'Inactive';
  }
}
```

### Module Organization **[RECOMMENDED]**

```typescript
// types/index.ts - Centralized type definitions
export interface User {
  id: number;
  name: string;
  email: string;
}

export interface CreateUserRequest extends Omit<User, 'id'> {
  password: string;
}

export type UserRole = 'admin' | 'user' | 'moderator';

// constants/index.ts - Application constants
export const USER_ROLES = {
  ADMIN: 'admin' as const,
  USER: 'user' as const,
  MODERATOR: 'moderator' as const
} as const;

export const API_ENDPOINTS = {
  USERS: '/api/users',
  AUTH: '/api/auth',
  ORDERS: '/api/orders'
} as const;

// utils/type-guards.ts - Reusable type guards
export const isUser = (obj: unknown): obj is User => {
  return obj !== null &&
         typeof obj === 'object' &&
         typeof (obj as User).id === 'number' &&
         typeof (obj as User).name === 'string' &&
         typeof (obj as User).email === 'string';
};

export const isUserRole = (role: string): role is UserRole => {
  return Object.values(USER_ROLES).includes(role as UserRole);
};

// services/user.service.ts - Clean imports
import { User, CreateUserRequest, UserRole } from '../types';
import { USER_ROLES, API_ENDPOINTS } from '../constants';
import { isUser, isUserRole } from '../utils/type-guards';
```

## Advanced Patterns

### Conditional Types **[ADVANCED]**

```typescript
// API response wrapper that adapts based on data type
type ApiResponse<T> = T extends string
  ? { message: T; timestamp: Date }
  : { data: T; count: number; timestamp: Date };

// Function that returns different types based on input
function processApiData<T>(input: T): ApiResponse<T> {
  if (typeof input === 'string') {
    return {
      message: input,
      timestamp: new Date()
    } as ApiResponse<T>;
  }
  
  return {
    data: input,
    count: Array.isArray(input) ? input.length : 1,
    timestamp: new Date()
  } as ApiResponse<T>;
}

// Usage - TypeScript infers correct return types
const stringResponse = processApiData('Hello'); // { message: string; timestamp: Date }
const dataResponse = processApiData({ id: 1 }); // { data: object; count: number; timestamp: Date }
```

### Template Literal Types **[ADVANCED]**

```typescript
// Type-safe event system
type EntityType = 'user' | 'order' | 'product';
type ActionType = 'created' | 'updated' | 'deleted';
type EventName = `${EntityType}:${ActionType}`;

// Event payload types
interface EventPayload {
  'user:created': { user: User };
  'user:updated': { user: User; changes: Partial<User> };
  'user:deleted': { userId: number };
  'order:created': { order: Order };
  'order:updated': { order: Order; changes: Partial<Order> };
  'order:deleted': { orderId: number };
  // ... more events
}

// Type-safe event emitter
class TypedEventEmitter {
  private listeners: Partial<{
    [K in EventName]: Array<(payload: EventPayload[K]) => void>
  }> = {};
  
  on<T extends EventName>(
    event: T, 
    listener: (payload: EventPayload[T]) => void
  ): void {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(listener);
  }
  
  emit<T extends EventName>(event: T, payload: EventPayload[T]): void {
    const eventListeners = this.listeners[event];
    if (eventListeners) {
      eventListeners.forEach(listener => listener(payload));
    }
  }
}

// Usage - fully type-safe
const emitter = new TypedEventEmitter();

emitter.on('user:created', (payload) => {
  // TypeScript knows payload is { user: User }
  console.log('New user created:', payload.user.name);
});

emitter.emit('user:created', { user: newUser }); // ✅ Correct payload type
emitter.emit('user:created', { userId: 123 }); // ❌ TypeScript error - wrong payload
```

### Discriminated Unions **[ADVANCED]**

```typescript
// State management with discriminated unions
interface LoadingState {
  type: 'loading';
  startTime: Date;
}

interface SuccessState<T> {
  type: 'success';
  data: T;
  loadTime: number;
}

interface ErrorState {
  type: 'error';
  error: Error;
  retryCount: number;
}

type AsyncState<T> = LoadingState | SuccessState<T> | ErrorState;

// Type-safe state handling
const handleUserState = (state: AsyncState<User[]>): string => {
  switch (state.type) {
    case 'loading':
      // TypeScript knows this is LoadingState
      return `Loading started at ${state.startTime.toLocaleTimeString()}`;
      
    case 'success':
      // TypeScript knows this is SuccessState<User[]>
      return `Loaded ${state.data.length} users in ${state.loadTime}ms`;
      
    case 'error':
      // TypeScript knows this is ErrorState
      return `Error: ${state.error.message} (attempt ${state.retryCount})`;
      
    default:
      // TypeScript ensures exhaustive checking
      const _exhaustive: never = state;
      return _exhaustive;
  }
};

// Usage in components
const UserListComponent = () => {
  const [userState, setUserState] = useState<AsyncState<User[]>>({
    type: 'loading',
    startTime: new Date()
  });
  
  useEffect(() => {
    userService.getUsers().then(
      users => setUserState({
        type: 'success',
        data: users,
        loadTime: Date.now() - userState.startTime.getTime()
      }),
      error => setUserState({
        type: 'error',
        error,
        retryCount: 1
      })
    );
  }, []);
  
  return <div>{handleUserState(userState)}</div>;
};
```

## Common Anti-Patterns

### Avoid These TypeScript Mistakes **[MANDATORY]**

```typescript
// ❌ BAD - Any defeats the purpose of TypeScript
const processData = (data: any): any => {
  return data.someProperty.doSomething();
};

// ✅ GOOD - Use generics or specific types
const processData = <T extends { someProperty: { doSomething(): unknown } }>(
  data: T
): unknown => {
  return data.someProperty.doSomething();
};

// ❌ BAD - Type assertion without validation
const userFromApi = response as User;

// ✅ GOOD - Type guard with validation
const userFromApi = (() => {
  if (isUser(response)) {
    return response;
  }
  throw new Error('Invalid user data from API');
})();

// ❌ BAD - Ignoring null/undefined possibilities
const getUserName = (user: User | null): string => {
  return user.name; // Runtime error if user is null
};

// ✅ GOOD - Explicit null handling
const getUserName = (user: User | null): string => {
  return user?.name ?? 'Unknown User';
};

// ❌ BAD - Using function declarations in modules
function helper() {
  // Function declarations are hoisted and harder to tree-shake
}

// ✅ GOOD - Use const with arrow functions
const helper = (): void => {
  // More predictable, better for bundling
};

// ❌ BAD - Mutating readonly properties
interface ReadonlyUser {
  readonly id: number;
  readonly name: string;
}

const updateUser = (user: ReadonlyUser): void => {
  (user as any).name = 'New Name'; // Circumvents type safety
};

// ✅ GOOD - Create new objects for updates
const updateUser = (user: ReadonlyUser, newName: string): ReadonlyUser => {
  return { ...user, name: newName };
};
```

## TypeScript Configuration

### Recommended tsconfig.json **[MANDATORY]**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "lib": ["ES2022", "DOM"],
    "moduleResolution": "node",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**Key Settings Explained**:
- `strict: true` - Enables all strict type checking options
- `noImplicitAny: true` - Prevents accidental `any` types
- `strictNullChecks: true` - Requires explicit handling of `null`/`undefined`
- `noUnusedLocals: true` - Catches unused variables (prevents dead code)
- `exactOptionalPropertyTypes: true` - Distinguishes between `undefined` and missing properties
