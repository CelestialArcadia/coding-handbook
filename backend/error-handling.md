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
  [ PENDING ]
  }
