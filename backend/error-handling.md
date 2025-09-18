# Backend Error Handling Guidelines

Consistent, localized, and maintainable error handling patterns for robust backend services.

## Named Error Constants

### Define Error Constants **[MANDATORY]**

Replace magic strings with named constants for consistency and maintainability.

```typescript
// errors/constants.ts
export const ERROR_CODES = {
  // Authentication & Authorization
  AUTHENTICATION_FAILED: 'AUTHENTICATION_FAILED',
  INVALID_CREDENTIALS: 'INVALID_CREDENTIALS',
  TOKEN_EXPIRED: 'TOKEN_EXPIRED',
  TOKEN_INVALID: 'TOKEN_INVALID',
  INSUFFICIENT_PERMISSIONS: 'INSUFFICIENT_PERMISSIONS',
  ACCOUNT_LOCKED: 'ACCOUNT_LOCKED',
  ACCOUNT_SUSPENDED: 'ACCOUNT_SUSPENDED',
  
  // Validation Errors
  VALIDATION_FAILED: 'VALIDATION_FAILED',
  INVALID_EMAIL_FORMAT: 'INVALID_EMAIL_FORMAT',
  PASSWORD_TOO_WEAK: 'PASSWORD_TOO_WEAK',
  REQUIRED_FIELD_MISSING: 'REQUIRED_FIELD_MISSING',
  INVALID_DATE_RANGE: 'INVALID_DATE_RANGE',
  FILE_TOO_LARGE: 'FILE_TOO_LARGE',
  UNSUPPORTED_FILE_TYPE: 'UNSUPPORTED_FILE_TYPE',
  
  // Business Logic Errors
  USER_NOT_FOUND: 'USER_NOT_FOUND',
  USER_ALREADY_EXISTS: 'USER_ALREADY_EXISTS',
  ORDER_ALREADY_PROCESSED: 'ORDER_ALREADY_PROCESSED',
  INSUFFICIENT_INVENTORY: 'INSUFFICIENT_INVENTORY',
  PAYMENT_FAILED: 'PAYMENT_FAILED',
  OPERATION_NOT_ALLOWED: 'OPERATION_NOT_ALLOWED',
  
  // System Errors
  DATABASE_CONNECTION_FAILED: 'DATABASE_CONNECTION_FAILED',
  EXTERNAL_SERVICE_UNAVAILABLE: 'EXTERNAL_SERVICE_UNAVAILABLE',
  RATE_LIMIT_EXCEEDED: 'RATE_LIMIT_EXCEEDED',
  MAINTENANCE_MODE: 'MAINTENANCE_MODE',
  INTERNAL_SERVER_ERROR: 'INTERNAL_SERVER_ERROR'
} as const;

// Type for compile-time safety
export type ErrorCode = typeof ERROR_CODES[keyof typeof ERROR_CODES];
```

### Error Messages with Localization **[RECOMMENDED]**

```typescript
// errors/messages.ts
import { ERROR_CODES } from './constants';

export const ERROR_MESSAGES = {
  [ERROR_CODES.AUTHENTICATION_FAILED]: {
    en: 'Authentication failed. Please check your credentials.',
    es: 'Falló la autenticación. Por favor, verifique sus credenciales.',
    fr: 'Échec de l\'authentification. Veuillez vérifier vos identifiants.'
  },
  [ERROR_CODES.INVALID_EMAIL_FORMAT]: {
    en: 'Please enter a valid email address.',
    es: 'Por favor, ingrese una dirección de correo válida.',
    fr: 'Veuillez entrer une adresse e-mail valide.'
  },
  [ERROR_CODES.USER_NOT_FOUND]: {
    en: 'User not found.',
    es: 'Usuario no encontrado.',
    fr: 'Utilisateur non trouvé.'
  },
  [ERROR_CODES.INSUFFICIENT_PERMISSIONS]: {
    en: 'You do not have permission to perform this action.',
    es: 'No tiene permisos para realizar esta acción.',
    fr: 'Vous n\'avez pas l\'autorisation d\'effectuer cette action.'
  }
} as const;

// Message retrieval function
export const getErrorMessage = (
  errorCode: ErrorCode, 
  language: string = 'en'
): string => {
  const messages = ERROR_MESSAGES[errorCode];
  return messages?.[language as keyof typeof messages] || messages?.en || errorCode;
};
```

## Error Response Structure

### Standardized Error Response **[MANDATORY]**

```typescript
// types/error-response.ts
export interface ErrorResponse {
  success: false;
  error: {
    code: ErrorCode;
    message: string;
    details?: Record<string, any>;
    timestamp: string;
    requestId: string;
    field?: string; // For validation errors
  };
}

export interface SuccessResponse<T = any> {
  success: true;
  data: T;
  timestamp: string;
  requestId: string;
}

export type ApiResponse<T = any> = SuccessResponse<T> | ErrorResponse;
```

### Error Response Factory **[RECOMMENDED]**

```typescript
// utils/error-response.ts
import { Request } from 'express';
import { ERROR_CODES, ErrorCode } from '../errors/constants';
import { getErrorMessage } from '../errors/messages';
import { ErrorResponse } from '../types/error-response';

export class ErrorResponseFactory {
  static create(
    errorCode: ErrorCode,
    req: Request,
    details?: Record<string, any>,
    field?: string
  ): ErrorResponse {
    const language = req.headers['accept-language']?.split(',')[0] || 'en';
    
    return {
      success: false,
      error: {
        code: errorCode,
        message: getErrorMessage(errorCode, language),
        details,
        timestamp: new Date().toISOString(),
        requestId: req.id || generateRequestId(),
        field
      }
    };
  }
  
  static validation(
    field: string,
    errorCode: ErrorCode,
    req: Request,
    details?: Record<string, any>
  ): ErrorResponse {
    return this.create(errorCode, req, details, field);
  }
  
  static businessLogic(
    errorCode: ErrorCode,
    req: Request,
    details?: Record<string, any>
  ): ErrorResponse {
    return this.create(errorCode, req, details);
  }
}

const generateRequestId = (): string => {
  return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
};
```

## Custom Error Classes

### Structured Error Hierarchy **[RECOMMENDED]**

```typescript
// errors/custom-errors.ts
import { ErrorCode } from './constants';

export abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly errorCode: ErrorCode;
  readonly timestamp: Date;
  readonly requestId?: string;
  
  constructor(message: string, requestId?: string) {
    super(message);
    this.name = this.constructor.name;
    this.timestamp = new Date();
    this.requestId = requestId;
    
    // Ensure proper prototype chain for instanceof checks
    Object.setPrototypeOf(this, new.target.prototype);
  }
}

export class ValidationError extends AppError {
  readonly statusCode = 400;
  readonly field: string;
  
  constructor(
    public readonly errorCode: ErrorCode,
    field: string,
    message?: string,
    requestId?: string
  ) {
    super(message || `Validation failed for field: ${field}`, requestId);
    this.field = field;
  }
}

export class AuthenticationError extends AppError {
  readonly statusCode = 401;
  
  constructor(
    public readonly errorCode: ErrorCode,
    message?: string,
    requestId?: string
  ) {
    super(message || 'Authentication failed', requestId);
  }
}

export class AuthorizationError extends AppError {
  readonly statusCode = 403;
  
  constructor(
    public readonly errorCode: ErrorCode,
    message?: string,
    requestId?: string
  ) {
    super(message || 'Access denied', requestId);
  }
}

export class NotFoundError extends AppError {
  readonly statusCode = 404;
  readonly resource: string;
  
  constructor(
    public readonly errorCode: ErrorCode,
    resource: string,
    identifier?: string | number,
    requestId?: string
  ) {
    const message = identifier 
      ? `${resource} with id '${identifier}' not found`
      : `${resource} not found`;
    super(message, requestId);
    this.resource = resource;
  }
}

export class ConflictError extends AppError {
  readonly statusCode = 409;
  
  constructor(
    public readonly errorCode: ErrorCode,
    message?: string,
    requestId?: string
  ) {
    super(message || 'Resource conflict', requestId);
  }
}

export class BusinessLogicError extends AppError {
  readonly statusCode = 422;
  
  constructor(
    public readonly errorCode: ErrorCode,
    message?: string,
    requestId?: string
  ) {
    super(message || 'Business rule violation', requestId);
  }
}

export class ExternalServiceError extends AppError {
  readonly statusCode = 502;
  readonly serviceName: string;
  
  constructor(
    public readonly errorCode: ErrorCode,
    serviceName: string,
    message?: string,
    requestId?: string
  ) {
    super(message || `External service '${serviceName}' failed`, requestId);
    this.serviceName = serviceName;
  }
}
```

## Service Layer Error Handling

### Result Pattern Implementation **[RECOMMENDED]**

```typescript
// utils/result.ts
export type Result<T, E = AppError> = 
  | { success: true; data: T }
  | { success: false; error: E };

export const Success = <T>(data: T): Result<T, never> => ({
  success: true,
  data
});

export const Failure = <E extends AppError>(error: E): Result<never, E> => ({
  success: false,
  error
});

// Helper functions for working with Results
export const isSuccess = <T, E>(result: Result<T, E>): result is { success: true; data: T } => {
  return result.success;
};

export const isFailure = <T, E>(result: Result<T, E>): result is { success: false; error: E } => {
  return !result.success;
};
```

### Service Implementation with Result Pattern **[RECOMMENDED]**

```typescript
// services/user.service.ts
import { Result, Success, Failure } from '../utils/result';
import { ERROR_CODES } from '../errors/constants';
import { ValidationError, NotFoundError, ConflictError } from '../errors/custom-errors';

export class UserService {
  constructor(
    private userRepository: UserRepository,
    private passwordService: PasswordService
  ) {}
  
  async createUser(
    userData: CreateUserRequest,
    requestId: string
  ): Promise<Result<User, ValidationError | ConflictError>> {
    // Validate input
    const validationResult = this.validateCreateUserRequest(userData, requestId);
    if (isFailure(validationResult)) {
      return validationResult;
    }
    
    // Check for existing user
    const existingUser = await this.userRepository.findByEmail(userData.email);
    if (existingUser) {
      return Failure(new ConflictError(
        ERROR_CODES.USER_ALREADY_EXISTS,
        `User with email '${userData.email}' already exists`,
        requestId
      ));
    }
    
    try {
      // Hash password
      const hashedPassword = await this.passwordService.hash(userData.password);
      
      // Create user
      const user = await this.userRepository.create({
        ...userData,
        password: hashedPassword
      });
      
      return Success(user);
      
    } catch (error) {
      // Log error for debugging but don't expose internal details
      console.error('Failed to create user:', error);
      
      return Failure(new ConflictError(
        ERROR_CODES.USER_ALREADY_EXISTS,
        'Failed to create user',
        requestId
      ));
    }
  }
  
  async getUserById(
    userId: number,
    requestId: string
  ): Promise<Result<User, NotFoundError>> {
    try {
      const user = await this.userRepository.findById(userId);
      
      if (!user) {
        return Failure(new NotFoundError(
          ERROR_CODES.USER_NOT_FOUND,
          'User',
          userId,
          requestId
        ));
      }
      
      return Success(user);
      
    } catch (error) {
      console.error('Failed to fetch user:', error);
      
      return Failure(new NotFoundError(
        ERROR_CODES.USER_NOT_FOUND,
        'User',
        userId,
        requestId
      ));
    }
  }
  
  private validateCreateUserRequest(
    userData: CreateUserRequest,
    requestId: string
  ): Result<void, ValidationError> {
    // Email validation
    if (!userData.email || !this.isValidEmail(userData.email)) {
      return Failure(new ValidationError(
        ERROR_CODES.INVALID_EMAIL_FORMAT,
        'email',
        'Invalid email format',
        requestId
      ));
    }
    
    // Password validation
    if (!userData.password || !this.isStrongPassword(userData.password)) {
      return Failure(new ValidationError(
        ERROR_CODES.PASSWORD_TOO_WEAK,
        'password',
        'Password does not meet strength requirements',
        requestId
      ));
    }
    
    // Name validation
    if (!userData.name || userData.name.trim().length < 2) {
      return Failure(new ValidationError(
        ERROR_CODES.REQUIRED_FIELD_MISSING,
        'name',
        'Name must be at least 2 characters long',
        requestId
      ));
    }
    
    return Success(undefined);
  }
  
  private isValidEmail(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }
  
  private isStrongPassword(password: string): boolean {
    // At least 8 characters, 1 uppercase, 1 lowercase, 1 number
    const strongPasswordRegex = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)[a-zA-Z\d@$!%*?&]{8,}$/;
    return strongPasswordRegex.test(password);
  }
}
```

## Controller Layer Error Handling

### Express Controller with Error Handling **[MANDATORY]**

```typescript
// controllers/user.controller.ts
import { Request, Response, NextFunction } from 'express';
import { UserService } from '../services/user.service';
import { ErrorResponseFactory } from '../utils/error-response';
import { isFailure, isSuccess } from '../utils/result';
import { AppError } from '../errors/custom-errors';

export class UserController {
  constructor(private userService: UserService) {}
  
  createUser = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const result = await this.userService.createUser(req.body, req.id);
      
      if (isSuccess(result)) {
        res.status(201).json({
          success: true,
          data: result.data,
          timestamp: new Date().toISOString(),
          requestId: req.id
        });
        return;
      }
      
      // Handle specific error types
      const errorResponse = ErrorResponseFactory.create(
        result.error.errorCode,
        req,
        { originalError: result.error.message },
        result.error instanceof ValidationError ? result.error.field : undefined
      );
      
      res.status(result.error.statusCode).json(errorResponse);
      
    } catch (error) {
      next(error); // Pass to global error handler
    }
  };
  
  getUserById = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const userId = parseInt(req.params.id);
      
      if (isNaN(userId)) {
        const errorResponse = ErrorResponseFactory.validation(
          'id',
          ERROR_CODES.VALIDATION_FAILED,
          req,
          { providedValue: req.params.id }
        );
        res.status(400).json(errorResponse);
        return;
      }
      
      const result = await this.userService.getUserById(userId, req.id);
      
      if (isSuccess(result)) {
        res.json({
          success: true,
          data: result.data,
          timestamp: new Date().toISOString(),
          requestId: req.id
        });
        return;
      }
      
      const errorResponse = ErrorResponseFactory.create(
        result.error.errorCode,
        req
      );
      
      res.status(result.error.statusCode).json(errorResponse);
      
    } catch (error) {
      next(error);
    }
  };
}
```

### Global Error Handler Middleware **[MANDATORY]**

```typescript
// middleware/error-handler.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../errors/custom-errors';
import { ERROR_CODES } from '../errors/constants';
import { ErrorResponseFactory } from '../utils/error-response';

export const globalErrorHandler = (
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
): void => {
  // Log error for monitoring
  console.error('Unhandled error:', {
    error: error.message,
    stack: error.stack,
    requestId: req.id,
    url: req.url,
    method: req.method,
    userId: req.user?.id
  });
  
  // Handle known application errors
  if (error instanceof AppError) {
    const errorResponse = ErrorResponseFactory.create(
      error.errorCode,
      req,
      { originalError: error.message }
    );
    
    res.status(error.statusCode).json(errorResponse);
    return;
  }
  
  // Handle validation errors from express-validator
  if (error.name === 'ValidationError') {
    const errorResponse = ErrorResponseFactory.create(
      ERROR_CODES.VALIDATION_FAILED,
      req,
      { validationErrors: error.message }
    );
    
    res.status(400).json(errorResponse);
    return;
  }
  
  // Handle database errors
  if (error.name === 'SequelizeValidationError' || error.name === 'MongoError') {
    const errorResponse = ErrorResponseFactory.create(
      ERROR_CODES.VALIDATION_FAILED,
      req,
      { databaseError: error.message }
    );
    
    res.status(400).json(errorResponse);
    return;
  }
  
  // Handle unexpected errors
  const errorResponse = ErrorResponseFactory.create(
    ERROR_CODES.INTERNAL_SERVER_ERROR,
    req
  );
  
  res.status(500).json(errorResponse);
};

// 404 handler for unmatched routes
export const notFoundHandler = (req: Request, res: Response): void => {
  const errorResponse = ErrorResponseFactory.create(
    ERROR_CODES.USER_NOT_FOUND, // Or create ROUTE_NOT_FOUND
    req,
    { requestedPath: req.path }
  );
  
  res.status(404).json(errorResponse);
};
```

## Error Monitoring and Logging

### Structured Logging **[RECOMMENDED]**

```typescript
// utils/logger.ts
import winston from 'winston';

export interface ErrorLogData {
  errorCode: string;
  message: string;
  stack?: string;
  requestId: string;
  userId?: number;
  userAgent?: string;
  ip?: string;
  url: string;
  method: string;
  timestamp: Date;
  severity: 'low' | 'medium' | 'high' | 'critical';
}

class Logger {
  private logger: winston.Logger;
  
  constructor() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json()
      ),
      defaultMeta: { service: 'user-api' },
      transports: [
        new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
        new winston.transports.File({ filename: 'logs/combined.log' }),
        new winston.transports.Console({
          format: winston.format.simple()
        })
      ]
    });
  }
  
  logError(error: AppError | Error, req: Request): void {
    const errorData: ErrorLogData = {
      errorCode: error instanceof AppError ? error.errorCode : 'UNKNOWN_ERROR',
      message: error.message,
      stack: error.stack,
      requestId: req.id || 'unknown',
      userId: req.user?.id,
      userAgent: req.headers['user-agent'],
      ip: req.ip,
      url: req.url,
      method: req.method,
      timestamp: new Date(),
      severity: this.determineSeverity(error)
    };
    
    this.logger.error('Application error occurred', errorData);
    
    // Send critical errors to monitoring service
    if (errorData.severity === 'critical') {
      this.sendToMonitoring(errorData);
    }
  }
  
  private determineSeverity(error: Error): 'low' | 'medium' | 'high' | 'critical' {
    if (error instanceof AppError) {
      switch (error.statusCode) {
        case 400:
        case 401:
        case 403:
        case 404:
          return 'low';
        case 409:
        case 422:
          return 'medium';
        case 500:
        case 502:
        case 503:
          return 'high';
        default:
          return 'critical';
      }
    }
    
    // Unknown errors are critical
    return 'critical';
  }
  
  private async sendToMonitoring(errorData: ErrorLogData): Promise<void> {
    // Integration with monitoring services like Sentry, DataDog, etc.
    try {
      // Example: Sentry integration
      // Sentry.captureException(error, { extra: errorData });
      
      // Example: Custom monitoring webhook
      // await fetch('/api/monitoring/alert', {
      //   method: 'POST',
      //   headers: { 'Content-Type': 'application/json' },
      //   body: JSON.stringify(errorData)
      // });
    } catch (monitoringError) {
      console.error('Failed to send error to monitoring:', monitoringError);
    }
  }
}

export const logger = new Logger();
```

### Error Metrics and Analytics **[ADVANCED]**

```typescript
// utils/error-metrics.ts
import { ErrorCode } from '../errors/constants';

interface ErrorMetrics {
  errorCode: ErrorCode;
  count: number;
  lastOccurrence: Date;
  affectedUsers: Set<number>;
  averageResponseTime: number;
}

class ErrorMetricsCollector {
  private metrics = new Map<ErrorCode, ErrorMetrics>();
  private metricsBuffer: Array<{
    errorCode: ErrorCode;
    userId?: number;
    responseTime: number;
    timestamp: Date;
  }> = [];
  
  recordError(errorCode: ErrorCode, userId?: number, responseTime?: number): void {
    const timestamp = new Date();
    
    // Add to buffer for batch processing
    this.metricsBuffer.push({
      errorCode,
      userId,
      responseTime: responseTime || 0,
      timestamp
    });
    
    // Update real-time metrics
    const existing = this.metrics.get(errorCode) || {
      errorCode,
      count: 0,
      lastOccurrence: timestamp,
      affectedUsers: new Set(),
      averageResponseTime: 0
    };
    
    existing.count++;
    existing.lastOccurrence = timestamp;
    if (userId) existing.affectedUsers.add(userId);
    if (responseTime) {
      existing.averageResponseTime = 
        (existing.averageResponseTime * (existing.count - 1) + responseTime) / existing.count;
    }
    
    this.metrics.set(errorCode, existing);
    
    // Process buffer periodically
    if (this.metricsBuffer.length >= 100) {
      this.flushMetrics();
    }
  }
  
  private async flushMetrics(): Promise<void> {
    if (this.metricsBuffer.length === 0) return;
    
    const batch = [...this.metricsBuffer];
    this.metricsBuffer = [];
    
    try {
      // Send to analytics service
      await this.sendToAnalytics(batch);
    } catch (error) {
      console.error('Failed to flush error metrics:', error);
      // Re-add to buffer for retry
      this.metricsBuffer.unshift(...batch);
    }
  }
  
  private async sendToAnalytics(
    batch: Array<{ errorCode: ErrorCode; userId?: number; responseTime: number; timestamp: Date }>
  ): Promise<void> {
    // Integration with analytics services
    // Example: Google Analytics, Mixpanel, custom analytics API
  }
  
  getMetrics(): Map<ErrorCode, ErrorMetrics> {
    return new Map(this.metrics);
  }
  
  getTopErrors(limit: number = 10): Array<ErrorMetrics> {
    return Array.from(this.metrics.values())
      .sort((a, b) => b.count - a.count)
      .slice(0, limit);
  }
}

export const errorMetrics = new ErrorMetricsCollector();
```

## Testing Error Handling

### Unit Testing Error Scenarios **[RECOMMENDED]**

```typescript
// tests/user.service.test.ts
import { UserService } from '../services/user.service';
import { ERROR_CODES } from '../errors/constants';
import { ValidationError, ConflictError, NotFoundError } from '../errors/custom-errors';
import { isSuccess, isFailure } from '../utils/result';

describe('UserService Error Handling', () => {
  let userService: UserService;
  let mockUserRepository: jest.Mocked<UserRepository>;
  let mockPasswordService: jest.Mocked<PasswordService>;
  
  beforeEach(() => {
    mockUserRepository = {
      findByEmail: jest.fn(),
      findById: jest.fn(),
      create: jest.fn()
    };
    
    mockPasswordService = {
      hash: jest.fn()
    };
    
    userService = new UserService(mockUserRepository, mockPasswordService);
  });
  
  describe('createUser', () => {
    const validUserData = {
      name: 'John Doe',
      email: 'john@example.com',
      password: 'StrongPass123'
    };
    
    it('should return validation error for invalid email', async () => {
      const invalidUserData = { ...validUserData, email: 'invalid-email' };
      
      const result = await userService.createUser(invalidUserData, 'req-123');
      
      expect(isFailure(result)).toBe(true);
      if (isFailure(result)) {
        expect(result.error).toBeInstanceOf(ValidationError);
        expect(result.error.errorCode).toBe(ERROR_CODES.INVALID_EMAIL_FORMAT);
        expect(result.error.field).toBe('email');
      }
    });
    
    it('should return validation error for weak password', async () => {
      const invalidUserData = { ...validUserData, password: '123' };
      
      const result = await userService.createUser(invalidUserData, 'req-123');
      
      expect(isFailure(result)).toBe(true);
      if (isFailure(result)) {
        expect(result.error).toBeInstanceOf(ValidationError);
        expect(result.error.errorCode).toBe(ERROR_CODES.PASSWORD_TOO_WEAK);
        expect(result.error.field).toBe('password');
      }
    });
    
    it('should return conflict error when user already exists', async () => {
      mockUserRepository.findByEmail.mockResolvedValue({ id: 1 } as User);
      
      const result = await userService.createUser(validUserData, 'req-123');
      
      expect(isFailure(result)).toBe(true);
      if (isFailure(result)) {
        expect(result.error).toBeInstanceOf(ConflictError);
        expect(result.error.errorCode).toBe(ERROR_CODES.USER_ALREADY_EXISTS);
      }
    });
    
    it('should create user successfully with valid data', async () => {
      const createdUser = { id: 1, ...validUserData };
      mockUserRepository.findByEmail.mockResolvedValue(null);
      mockPasswordService.hash.mockResolvedValue('hashed-password');
      mockUserRepository.create.mockResolvedValue(createdUser);
      
      const result = await userService.createUser(validUserData, 'req-123');
      
      expect(isSuccess(result)).toBe(true);
      if (isSuccess(result)) {
        expect(result.data).toEqual(createdUser);
      }
    });
  });
  
  describe('getUserById', () => {
    it('should return not found error when user does not exist', async () => {
      mockUserRepository.findById.mockResolvedValue(null);
      
      const result = await userService.getUserById(999, 'req-123');
      
      expect(isFailure(result)).toBe(true);
      if (isFailure(result)) {
        expect(result.error).toBeInstanceOf(NotFoundError);
        expect(result.error.errorCode).toBe(ERROR_CODES.USER_NOT_FOUND);
      }
    });
    
    it('should return user when found', async () => {
      const user = { id: 1, name: 'John Doe', email: 'john@example.com' };
      mockUserRepository.findById.mockResolvedValue(user);
      
      const result = await userService.getUserById(1, 'req-123');
      
      expect(isSuccess(result)).toBe(true);
      if (isSuccess(result)) {
        expect(result.data).toEqual(user);
      }
    });
  });
});
```

### Integration Testing with Error Scenarios **[RECOMMENDED]**

```typescript
// tests/user.controller.integration.test.ts
import request from 'supertest';
import { app } from '../app';
import { ERROR_CODES } from '../errors/constants';

describe('User Controller Integration Tests', () => {
  describe('POST /api/users', () => {
    it('should return 400 for invalid email format', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({
          name: 'John Doe',
          email: 'invalid-email',
          password: 'StrongPass123'
        });
      
      expect(response.status).toBe(400);
      expect(response.body).toMatchObject({
        success: false,
        error: {
          code: ERROR_CODES.INVALID_EMAIL_FORMAT,
          field: 'email',
          message: expect.stringContaining('valid email')
        }
      });
    });
    
    it('should return 409 when user already exists', async () => {
      // Create user first
      await request(app)
        .post('/api/users')
        .send({
          name: 'John Doe',
          email: 'john@example.com',
          password: 'StrongPass123'
        });
      
      // Try to create same user again
      const response = await request(app)
        .post('/api/users')
        .send({
          name: 'Jane Doe',
          email: 'john@example.com',
          password: 'AnotherPass123'
        });
      
      expect(response.status).toBe(409);
      expect(response.body).toMatchObject({
        success: false,
        error: {
          code: ERROR_CODES.USER_ALREADY_EXISTS,
          message: expect.stringContaining('already exists')
        }
      });
    });
    
    it('should create user successfully with valid data', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({
          name: 'John Doe',
          email: 'john@example.com',
          password: 'StrongPass123'
        });
      
      expect(response.status).toBe(201);
      expect(response.body).toMatchObject({
        success: true,
        data: {
          id: expect.any(Number),
          name: 'John Doe',
          email: 'john@example.com'
        }
      });
      expect(response.body.data.password).toBeUndefined(); // Password should not be returned
    });
  });
  
  describe('GET /api/users/:id', () => {
    it('should return 400 for invalid user ID', async () => {
      const response = await request(app)
        .get('/api/users/invalid-id');
      
      expect(response.status).toBe(400);
      expect(response.body).toMatchObject({
        success: false,
        error: {
          code: ERROR_CODES.VALIDATION_FAILED,
          field: 'id'
        }
      });
    });
    
    it('should return 404 for non-existent user', async () => {
      const response = await request(app)
        .get('/api/users/99999');
      
      expect(response.status).toBe(404);
      expect(response.body).toMatchObject({
        success: false,
        error: {
          code: ERROR_CODES.USER_NOT_FOUND
        }
      });
    });
  });
});
```

## Error Documentation

### API Error Documentation **[RECOMMENDED]**

```markdown
# API Error Reference

## Standard Error Response Format

All API endpoints return errors in the following format:

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE_CONSTANT",
    "message": "Human-readable error message",
    "details": {},
    "timestamp": "2023-08-15T10:30:00.000Z",
    "requestId": "req_1692095400000_abc123def",
    "field": "fieldName"
  }
}
```

## Error Codes Reference

### Authentication Errors (401)

| Code | Message | Description |
|------|---------|-------------|
| `AUTHENTICATION_FAILED` | Authentication failed. Please check your credentials. | Invalid username/password combination |
| `TOKEN_EXPIRED` | Your session has expired. Please log in again. | JWT token has expired |
| `TOKEN_INVALID` | Invalid authentication token provided. | Malformed or tampered JWT token |

### Authorization Errors (403)

| Code | Message | Description |
|------|---------|-------------|
| `INSUFFICIENT_PERMISSIONS` | You do not have permission to perform this action. | User lacks required role/permission |
| `ACCOUNT_SUSPENDED` | Your account has been suspended. Contact support. | Account temporarily disabled |

### Validation Errors (400)

| Code | Field | Message | Description |
|------|-------|---------|-------------|
| `INVALID_EMAIL_FORMAT` | `email` | Please enter a valid email address. | Email format validation failed |
| `PASSWORD_TOO_WEAK` | `password` | Password does not meet strength requirements. | Password complexity requirements not met |
| `REQUIRED_FIELD_MISSING` | varies | This field is required. | Required field not provided |

### Business Logic Errors (422)

| Code | Message | Description |
|------|---------|-------------|
| `USER_ALREADY_EXISTS` | User with this email already exists. | Attempting to create duplicate user |
| `INSUFFICIENT_INVENTORY` | Not enough items in stock. | Order quantity exceeds available inventory |
| `ORDER_ALREADY_PROCESSED` | This order has already been processed. | Attempting to modify completed order |

### System Errors (500+)

| Code | Status | Message | Description |
|------|--------|---------|-------------|
| `INTERNAL_SERVER_ERROR` | 500 | An internal error occurred. Please try again. | Unexpected server error |
| `DATABASE_CONNECTION_FAILED` | 503 | Service temporarily unavailable. | Database connectivity issues |
| `EXTERNAL_SERVICE_UNAVAILABLE` | 502 | External service is currently unavailable. | Third-party service failure |

## Error Handling Best Practices

### Client-Side Error Handling

```typescript
// TypeScript client example
interface ApiError {
  code: string;
  message: string;
  field?: string;
  details?: any;
}

const handleApiError = (error: ApiError): void => {
  switch (error.code) {
    case 'AUTHENTICATION_FAILED':
      redirectToLogin();
      break;
      
    case 'VALIDATION_FAILED':
      showFieldError(error.field!, error.message);
      break;
      
    case 'USER_NOT_FOUND':
      showNotification('User not found', 'error');
      break;
      
    default:
      showNotification('An unexpected error occurred', 'error');
      logErrorToMonitoring(error);
  }
};
```

### Retry Logic

Implement exponential backoff for transient errors:

```typescript
const retryableErrors = [
  'DATABASE_CONNECTION_FAILED',
  'EXTERNAL_SERVICE_UNAVAILABLE',
  'RATE_LIMIT_EXCEEDED'
];

const shouldRetry = (errorCode: string): boolean => {
  return retryableErrors.includes(errorCode);
};
```
```

This comprehensive error handling system ensures consistent, maintainable, and user-friendly error management across your backend services. The structured approach makes debugging easier, improves user experience, and facilitates internationalization.
