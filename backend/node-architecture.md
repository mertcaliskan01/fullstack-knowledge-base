# Scalable Node.js Backend Architecture

## Why Architecture Matters

Backend architecture isn't just about organizing code—it's about building systems that can grow with your business while remaining maintainable. A well-designed Node.js architecture enables teams to ship features faster, debug issues efficiently, and scale horizontally without technical debt accumulation.

The difference between a successful production system and a maintenance nightmare often lies in architectural decisions made early in development.

## Project Structure

### Recommended Folder Organization

```
src/
├── controllers/          # HTTP request handlers
├── services/            # Business logic layer
├── repositories/        # Data access layer
├── middleware/          # Custom middleware functions
├── models/              # Data models and schemas
├── utils/               # Shared utilities and helpers
├── config/              # Configuration management
├── types/               # TypeScript type definitions
└── tests/               # Test files mirroring src structure
```

### Express.js vs NestJS Structure

**Express.js (Minimalist Approach):**
```javascript
// controllers/userController.js
const UserService = require('../services/userService');

class UserController {
  async createUser(req, res, next) {
    try {
      const user = await UserService.createUser(req.body);
      res.status(201).json(user);
    } catch (error) {
      next(error);
    }
  }
}
```

**NestJS (Decorator-Based):**
```typescript
// controllers/user.controller.ts
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Post()
  @HttpCode(201)
  async createUser(@Body() createUserDto: CreateUserDto) {
    return this.userService.createUser(createUserDto);
  }
}
```

## Service Layer Architecture

### Business Logic Separation

Services encapsulate business rules and coordinate between multiple repositories. They should be framework-agnostic and easily testable.

```javascript
// services/userService.js
class UserService {
  constructor(userRepository, emailService, logger) {
    this.userRepository = userRepository;
    this.emailService = emailService;
    this.logger = logger;
  }

  async createUser(userData) {
    // Validate business rules
    await this.validateUserData(userData);
    
    // Check for existing user
    const existingUser = await this.userRepository.findByEmail(userData.email);
    if (existingUser) {
      throw new ConflictError('User already exists');
    }

    // Create user with hashed password
    const hashedPassword = await bcrypt.hash(userData.password, 12);
    const user = await this.userRepository.create({
      ...userData,
      password: hashedPassword
    });

    // Send welcome email
    await this.emailService.sendWelcomeEmail(user.email);

    this.logger.info('User created successfully', { userId: user.id });
    return user;
  }
}
```

### Service Dependencies

Use dependency injection to make services testable and loosely coupled:

```javascript
// config/container.js
const container = {
  userService: new UserService(
    container.userRepository,
    container.emailService,
    container.logger
  ),
  userRepository: new UserRepository(container.database),
  emailService: new EmailService(container.emailConfig),
  logger: new Logger(container.logConfig)
};
```

## Controller Patterns

### Request Validation and Response Handling

Controllers should be thin—they handle HTTP concerns and delegate business logic to services.

```javascript
// controllers/userController.js
class UserController {
  constructor(userService, validator) {
    this.userService = userService;
    this.validator = validator;
  }

  async createUser(req, res, next) {
    try {
      // Validate request
      const validation = await this.validator.validate(req.body, userSchema);
      if (!validation.isValid) {
        return res.status(400).json({
          error: 'Validation failed',
          details: validation.errors
        });
      }

      // Delegate to service
      const user = await this.userService.createUser(req.body);
      
      // Transform response
      const userResponse = this.transformUserResponse(user);
      res.status(201).json(userResponse);
    } catch (error) {
      next(error);
    }
  }

  transformUserResponse(user) {
    const { password, ...userResponse } = user;
    return userResponse;
  }
}
```

## Middleware Architecture

### Custom Middleware Functions

Middleware should be composable and focused on single responsibilities:

```javascript
// middleware/errorHandler.js
const errorHandler = (err, req, res, next) => {
  const logger = req.app.get('logger');
  
  // Log error with context
  logger.error('Request failed', {
    error: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    userId: req.user?.id
  });

  // Handle different error types
  if (err instanceof ValidationError) {
    return res.status(400).json({
      error: 'Validation failed',
      details: err.details
    });
  }

  if (err instanceof AuthenticationError) {
    return res.status(401).json({
      error: 'Authentication required'
    });
  }

  // Default error response
  res.status(500).json({
    error: 'Internal server error'
  });
};

// middleware/requestLogger.js
const requestLogger = (req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    req.app.get('logger').info('Request completed', {
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration,
      userAgent: req.get('User-Agent')
    });
  });

  next();
};
```

## Error Handling Strategy

### Custom Error Classes

Create domain-specific error classes for better error handling:

```javascript
// errors/AppError.js
class AppError extends Error {
  constructor(message, statusCode, isOperational = true) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    Error.captureStackTrace(this, this.constructor);
  }
}

class ValidationError extends AppError {
  constructor(message, details) {
    super(message, 400);
    this.details = details;
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, 404);
  }
}

class ConflictError extends AppError {
  constructor(message) {
    super(message, 409);
  }
}
```

### Async Error Handling

Use async/await with proper error boundaries:

```javascript
// utils/asyncHandler.js
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Usage in controllers
const createUser = asyncHandler(async (req, res) => {
  const user = await userService.createUser(req.body);
  res.status(201).json(user);
});
```

## Database Abstraction Patterns

### Repository Pattern

Repositories abstract data access logic and make your application database-agnostic:

```javascript
// repositories/userRepository.js
class UserRepository {
  constructor(database) {
    this.db = database;
  }

  async findById(id) {
    const user = await this.db.query(
      'SELECT * FROM users WHERE id = ? AND deleted_at IS NULL',
      [id]
    );
    return user[0] || null;
  }

  async findByEmail(email) {
    const users = await this.db.query(
      'SELECT * FROM users WHERE email = ? AND deleted_at IS NULL',
      [email]
    );
    return users[0] || null;
  }

  async create(userData) {
    const result = await this.db.query(
      'INSERT INTO users (email, password, name) VALUES (?, ?, ?)',
      [userData.email, userData.password, userData.name]
    );
    
    return this.findById(result.insertId);
  }

  async update(id, updates) {
    const fields = Object.keys(updates).map(key => `${key} = ?`).join(', ');
    const values = Object.values(updates);
    
    await this.db.query(
      `UPDATE users SET ${fields}, updated_at = NOW() WHERE id = ?`,
      [...values, id]
    );
    
    return this.findById(id);
  }

  async softDelete(id) {
    await this.db.query(
      'UPDATE users SET deleted_at = NOW() WHERE id = ?',
      [id]
    );
  }
}
```

### Query Builder Pattern

For complex queries, use a query builder to maintain readability:

```javascript
// repositories/queryBuilder.js
class QueryBuilder {
  constructor(table) {
    this.table = table;
    this.conditions = [];
    this.params = [];
    this.limit = null;
    this.offset = null;
  }

  where(field, operator, value) {
    this.conditions.push(`${field} ${operator} ?`);
    this.params.push(value);
    return this;
  }

  limit(count) {
    this.limit = count;
    return this;
  }

  offset(count) {
    this.offset = count;
    return this;
  }

  build() {
    let query = `SELECT * FROM ${this.table}`;
    
    if (this.conditions.length > 0) {
      query += ` WHERE ${this.conditions.join(' AND ')}`;
    }
    
    if (this.limit) {
      query += ` LIMIT ${this.limit}`;
    }
    
    if (this.offset) {
      query += ` OFFSET ${this.offset}`;
    }
    
    return { query, params: this.params };
  }
}
```

## Testing Strategy

### Unit Testing Services

Test business logic in isolation with mocked dependencies:

```javascript
// tests/services/userService.test.js
describe('UserService', () => {
  let userService;
  let mockUserRepository;
  let mockEmailService;
  let mockLogger;

  beforeEach(() => {
    mockUserRepository = {
      findByEmail: jest.fn(),
      create: jest.fn()
    };
    
    mockEmailService = {
      sendWelcomeEmail: jest.fn()
    };
    
    mockLogger = {
      info: jest.fn(),
      error: jest.fn()
    };

    userService = new UserService(
      mockUserRepository,
      mockEmailService,
      mockLogger
    );
  });

  describe('createUser', () => {
    it('should create user successfully', async () => {
      const userData = {
        email: 'test@example.com',
        password: 'password123',
        name: 'Test User'
      };

      mockUserRepository.findByEmail.mockResolvedValue(null);
      mockUserRepository.create.mockResolvedValue({
        id: 1,
        ...userData,
        password: 'hashedPassword'
      });

      const result = await userService.createUser(userData);

      expect(mockUserRepository.findByEmail).toHaveBeenCalledWith(userData.email);
      expect(mockUserRepository.create).toHaveBeenCalled();
      expect(mockEmailService.sendWelcomeEmail).toHaveBeenCalledWith(userData.email);
      expect(result).toHaveProperty('id', 1);
    });

    it('should throw error if user already exists', async () => {
      const userData = { email: 'existing@example.com' };
      mockUserRepository.findByEmail.mockResolvedValue({ id: 1 });

      await expect(userService.createUser(userData))
        .rejects
        .toThrow('User already exists');
    });
  });
});
```

### Integration Testing

Test the full request-response cycle:

```javascript
// tests/integration/user.test.js
describe('User API', () => {
  let app;
  let testDb;

  beforeAll(async () => {
    app = createTestApp();
    testDb = await setupTestDatabase();
  });

  afterAll(async () => {
    await testDb.destroy();
  });

  beforeEach(async () => {
    await testDb.query('DELETE FROM users');
  });

  describe('POST /users', () => {
    it('should create user successfully', async () => {
      const userData = {
        email: 'test@example.com',
        password: 'password123',
        name: 'Test User'
      };

      const response = await request(app)
        .post('/users')
        .send(userData)
        .expect(201);

      expect(response.body).toHaveProperty('id');
      expect(response.body.email).toBe(userData.email);
      expect(response.body).not.toHaveProperty('password');

      // Verify user was saved to database
      const savedUser = await testDb.query(
        'SELECT * FROM users WHERE email = ?',
        [userData.email]
      );
      expect(savedUser).toHaveLength(1);
    });
  });
});
```

## Configuration Management

### Environment-Based Configuration

Separate configuration by environment and use validation:

```javascript
// config/index.js
const Joi = require('joi');

const configSchema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
  PORT: Joi.number().default(3000),
  DATABASE_URL: Joi.string().required(),
  JWT_SECRET: Joi.string().min(32).required(),
  REDIS_URL: Joi.string().uri().optional(),
  LOG_LEVEL: Joi.string().valid('error', 'warn', 'info', 'debug').default('info')
});

const config = {
  NODE_ENV: process.env.NODE_ENV,
  PORT: parseInt(process.env.PORT, 10),
  DATABASE_URL: process.env.DATABASE_URL,
  JWT_SECRET: process.env.JWT_SECRET,
  REDIS_URL: process.env.REDIS_URL,
  LOG_LEVEL: process.env.LOG_LEVEL
};

const { error, value } = configSchema.validate(config);
if (error) {
  throw new Error(`Configuration error: ${error.message}`);
}

module.exports = value;
```

## Performance Considerations

### Connection Pooling

Configure database connection pools for optimal performance:

```javascript
// config/database.js
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
  acquireTimeout: 60000,
  timeout: 60000,
  reconnect: true
});

module.exports = pool;
```

### Caching Strategy

Implement caching for frequently accessed data:

```javascript
// services/cacheService.js
class CacheService {
  constructor(redisClient) {
    this.redis = redisClient;
  }

  async get(key) {
    try {
      const value = await this.redis.get(key);
      return value ? JSON.parse(value) : null;
    } catch (error) {
      console.error('Cache get error:', error);
      return null;
    }
  }

  async set(key, value, ttl = 3600) {
    try {
      await this.redis.setex(key, ttl, JSON.stringify(value));
    } catch (error) {
      console.error('Cache set error:', error);
    }
  }

  async invalidate(pattern) {
    try {
      const keys = await this.redis.keys(pattern);
      if (keys.length > 0) {
        await this.redis.del(keys);
      }
    } catch (error) {
      console.error('Cache invalidate error:', error);
    }
  }
}
```

## Security Best Practices

### Input Validation and Sanitization

Always validate and sanitize user inputs:

```javascript
// middleware/validation.js
const Joi = require('joi');
const xss = require('xss');

const sanitizeInput = (obj) => {
  if (typeof obj === 'string') {
    return xss(obj.trim());
  }
  if (typeof obj === 'object' && obj !== null) {
    const sanitized = {};
    for (const [key, value] of Object.entries(obj)) {
      sanitized[key] = sanitizeInput(value);
    }
    return sanitized;
  }
  return obj;
};

const validateRequest = (schema) => {
  return (req, res, next) => {
    const sanitizedBody = sanitizeInput(req.body);
    const { error, value } = schema.validate(sanitizedBody);
    
    if (error) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.details.map(detail => detail.message)
      });
    }
    
    req.body = value;
    next();
  };
};
```

### Rate Limiting

Implement rate limiting to prevent abuse:

```javascript
// middleware/rateLimiter.js
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

const createRateLimiter = (redisClient) => {
  return rateLimit({
    store: new RedisStore({
      client: redisClient,
      prefix: 'rate_limit:'
    }),
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per windowMs
    message: {
      error: 'Too many requests, please try again later.'
    },
    standardHeaders: true,
    legacyHeaders: false
  });
};
```

## Deployment Considerations

### Health Checks

Implement comprehensive health checks:

```javascript
// routes/health.js
const healthCheck = async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    externalServices: await checkExternalServices()
  };

  const isHealthy = Object.values(checks).every(check => check.status === 'ok');
  const statusCode = isHealthy ? 200 : 503;

  res.status(statusCode).json({
    status: isHealthy ? 'healthy' : 'unhealthy',
    timestamp: new Date().toISOString(),
    checks
  });
};

const checkDatabase = async () => {
  try {
    await db.query('SELECT 1');
    return { status: 'ok', responseTime: Date.now() };
  } catch (error) {
    return { status: 'error', error: error.message };
  }
};
```

### Graceful Shutdown

Handle application shutdown gracefully:

```javascript
// server.js
const gracefulShutdown = (signal) => {
  console.log(`Received ${signal}. Starting graceful shutdown...`);
  
  server.close(() => {
    console.log('HTTP server closed');
    
    // Close database connections
    pool.end((err) => {
      if (err) {
        console.error('Error closing database pool:', err);
        process.exit(1);
      }
      console.log('Database connections closed');
      process.exit(0);
    });
  });
};

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

## Conclusion

Building a scalable Node.js backend requires thoughtful architecture from the start. The patterns and practices outlined here provide a solid foundation for applications that can grow with your business needs.

Key takeaways:
- Separate concerns with clear layer boundaries
- Make dependencies explicit and injectable
- Handle errors consistently across the application
- Test business logic in isolation
- Plan for horizontal scaling from day one
- Implement comprehensive monitoring and logging

Remember: good architecture is not about following every pattern—it's about choosing the right patterns for your specific use case and team constraints.
