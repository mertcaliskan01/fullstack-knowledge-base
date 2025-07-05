# Scalable Backend Architecture with NestJS

NestJS has emerged as the go-to framework for building enterprise-grade Node.js applications. Its decorator-based architecture, dependency injection system, and TypeScript-first approach make it ideal for creating scalable, maintainable backend systems. This guide covers the architectural patterns and best practices I've learned from building production applications with NestJS.

## Why NestJS for Scalable Systems?

NestJS provides a structured approach to building Node.js applications that scales with your team and business requirements. The framework's opinionated architecture reduces decision fatigue while maintaining flexibility where it matters most.

**Key Benefits:**
- **Type Safety**: Full TypeScript support prevents runtime errors and improves developer experience
- **Dependency Injection**: Built-in IoC container enables loose coupling and testability
- **Modular Architecture**: Clear separation of concerns through modules, controllers, and services
- **Enterprise Patterns**: Familiar patterns from Angular and Spring Boot reduce learning curve
- **Performance**: Built on Express/Fastify with minimal overhead

## Project Structure and Module Organization

A well-organized NestJS project follows a domain-driven structure that scales with complexity:

```
src/
├── common/                 # Shared utilities, decorators, guards
│   ├── decorators/
│   ├── guards/
│   ├── interceptors/
│   └── pipes/
├── config/                 # Configuration management
│   ├── database.config.ts
│   ├── app.config.ts
│   └── validation.schema.ts
├── modules/                # Feature modules
│   ├── users/
│   │   ├── dto/
│   │   ├── entities/
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   └── users.module.ts
│   └── auth/
├── database/               # Database configuration and migrations
│   ├── migrations/
│   └── seeds/
└── main.ts
```

**Module Organization Principles:**
- Group related functionality into feature modules
- Keep modules focused and cohesive
- Use shared modules for common functionality
- Implement lazy loading for large applications

## Core Concepts Deep Dive

### Modules: The Foundation

Modules are the building blocks of NestJS applications. They encapsulate related functionality and define clear boundaries:

```typescript
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService, UserRepository],
  exports: [UsersService], // Make service available to other modules
})
export class UsersModule {}
```

**Module Design Patterns:**
- **Feature Modules**: Encapsulate business logic (UsersModule, OrdersModule)
- **Shared Modules**: Provide common services (DatabaseModule, LoggerModule)
- **Core Modules**: Handle application-wide concerns (ConfigModule, AuthModule)

### Controllers: API Layer

Controllers handle HTTP requests and define your API contract. They should be thin and delegate business logic to services:

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(':id')
  @UseGuards(AuthGuard)
  async findOne(@Param('id') id: string): Promise<User> {
    return this.usersService.findById(id);
  }

  @Post()
  @UsePipes(new ValidationPipe({ transform: true }))
  async create(@Body() createUserDto: CreateUserDto): Promise<User> {
    return this.usersService.create(createUserDto);
  }
}
```

### Services: Business Logic Layer

Services contain your business logic and coordinate between different parts of your application:

```typescript
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async create(createUserDto: CreateUserDto): Promise<User> {
    const user = this.userRepository.create(createUserDto);
    const savedUser = await this.userRepository.save(user);
    
    // Emit domain event
    this.eventEmitter.emit('user.created', savedUser);
    
    return savedUser;
  }

  async findById(id: string): Promise<User> {
    const user = await this.userRepository.findOne({ where: { id } });
    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }
}
```

### Providers: Dependency Injection

Providers are the backbone of NestJS's dependency injection system. They can be services, repositories, or any class that provides functionality:

```typescript
// Custom provider with factory
export const DATABASE_CONNECTION = 'DATABASE_CONNECTION';

@Module({
  providers: [
    {
      provide: DATABASE_CONNECTION,
      useFactory: async (configService: ConfigService) => {
        return createConnection({
          type: 'postgres',
          host: configService.get('DB_HOST'),
          port: configService.get('DB_PORT'),
          username: configService.get('DB_USERNAME'),
          password: configService.get('DB_PASSWORD'),
          database: configService.get('DB_NAME'),
        });
      },
      inject: [ConfigService],
    },
  ],
})
export class DatabaseModule {}
```

## Dependency Injection Best Practices

NestJS's dependency injection system enables loose coupling and testability. Here are the patterns I've found most effective:

### Constructor Injection

Always use constructor injection for required dependencies:

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly paymentService: PaymentService,
    private readonly notificationService: NotificationService,
  ) {}
}
```

### Optional Dependencies

Use `@Optional()` decorator for optional dependencies:

```typescript
@Injectable()
export class AnalyticsService {
  constructor(
    @Optional() private readonly analyticsProvider?: AnalyticsProvider,
  ) {}

  trackEvent(event: string, data: any) {
    if (this.analyticsProvider) {
      this.analyticsProvider.track(event, data);
    }
  }
}
```

### Circular Dependencies

Handle circular dependencies using forward references:

```typescript
@Injectable()
export class UserService {
  constructor(
    @Inject(forwardRef(() => OrderService))
    private readonly orderService: OrderService,
  ) {}
}

@Injectable()
export class OrderService {
  constructor(
    @Inject(forwardRef(() => UserService))
    private readonly userService: UserService,
  ) {}
}
```

## Error Handling and Logging

### Global Exception Filter

Implement a global exception filter for consistent error responses:

```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(private readonly logger: Logger) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      message = exception.message;
    } else if (exception instanceof QueryFailedError) {
      status = HttpStatus.BAD_REQUEST;
      message = 'Database operation failed';
    }

    this.logger.error(
      `${request.method} ${request.url} - ${status}: ${message}`,
      exception instanceof Error ? exception.stack : undefined,
    );

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message,
    });
  }
}
```

### Structured Logging

Use structured logging for better observability:

```typescript
@Injectable()
export class LoggerService {
  private readonly logger = new Logger(LoggerService.name);

  logUserAction(userId: string, action: string, metadata?: any) {
    this.logger.log({
      message: 'User action performed',
      userId,
      action,
      metadata,
      timestamp: new Date().toISOString(),
    });
  }

  logError(error: Error, context?: string) {
    this.logger.error({
      message: error.message,
      stack: error.stack,
      context,
      timestamp: new Date().toISOString(),
    });
  }
}
```

## Configuration Management

### Environment-based Configuration

Use the ConfigModule for environment-based configuration:

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test')
          .default('development'),
        PORT: Joi.number().default(3000),
        DATABASE_URL: Joi.string().required(),
        JWT_SECRET: Joi.string().required(),
      }),
    }),
  ],
})
export class AppModule {}
```

### Feature-specific Configuration

Create feature-specific configuration classes:

```typescript
@Injectable()
export class DatabaseConfig {
  constructor(private configService: ConfigService) {}

  get host(): string {
    return this.configService.get<string>('DB_HOST');
  }

  get port(): number {
    return this.configService.get<number>('DB_PORT');
  }

  get username(): string {
    return this.configService.get<string>('DB_USERNAME');
  }

  get password(): string {
    return this.configService.get<string>('DB_PASSWORD');
  }

  get database(): string {
    return this.configService.get<string>('DB_NAME');
  }
}
```

## Database Integration Strategies

### Repository Pattern with TypeORM

Implement the repository pattern for clean data access:

```typescript
@Entity()
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  email: string;

  @Column()
  name: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}

@Injectable()
export class UserRepository {
  constructor(
    @InjectRepository(User)
    private readonly repository: Repository<User>,
  ) {}

  async findByEmail(email: string): Promise<User | null> {
    return this.repository.findOne({ where: { email } });
  }

  async create(userData: Partial<User>): Promise<User> {
    const user = this.repository.create(userData);
    return this.repository.save(user);
  }

  async update(id: string, updates: Partial<User>): Promise<User> {
    await this.repository.update(id, updates);
    return this.findById(id);
  }

  async findById(id: string): Promise<User> {
    const user = await this.repository.findOne({ where: { id } });
    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }
}
```

### Prisma Integration

For Prisma users, create a service layer that abstracts the Prisma client:

```typescript
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }

  async enableShutdownHooks(app: INestApplication) {
    this.$on('beforeExit', async () => {
      await app.close();
    });
  }
}

@Injectable()
export class UserService {
  constructor(private readonly prisma: PrismaService) {}

  async create(data: CreateUserDto): Promise<User> {
    return this.prisma.user.create({
      data,
      include: {
        profile: true,
      },
    });
  }

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { id },
      include: {
        profile: true,
        orders: true,
      },
    });
  }
}
```

## Testing Strategies

### Unit Testing

Test individual services and controllers in isolation:

```typescript
describe('UserService', () => {
  let service: UserService;
  let repository: jest.Mocked<UserRepository>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: UserRepository,
          useValue: {
            create: jest.fn(),
            findById: jest.fn(),
            findByEmail: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
    repository = module.get(UserRepository);
  });

  describe('create', () => {
    it('should create a new user', async () => {
      const createUserDto = { email: 'test@example.com', name: 'Test User' };
      const expectedUser = { id: '1', ...createUserDto };

      repository.create.mockResolvedValue(expectedUser);

      const result = await service.create(createUserDto);

      expect(repository.create).toHaveBeenCalledWith(createUserDto);
      expect(result).toEqual(expectedUser);
    });
  });
});
```

### Integration Testing

Test the complete request-response cycle:

```typescript
describe('UsersController (e2e)', () => {
  let app: INestApplication;
  let userService: UserService;

  beforeEach(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    userService = moduleFixture.get<UserService>(UserService);
    await app.init();
  });

  afterEach(async () => {
    await app.close();
  });

  describe('/users (POST)', () => {
    it('should create a new user', async () => {
      const createUserDto = {
        email: 'test@example.com',
        name: 'Test User',
      };

      const response = await request(app.getHttpServer())
        .post('/users')
        .send(createUserDto)
        .expect(201);

      expect(response.body).toHaveProperty('id');
      expect(response.body.email).toBe(createUserDto.email);
      expect(response.body.name).toBe(createUserDto.name);
    });
  });
});
```

### Test Database Setup

Use a separate test database with proper cleanup:

```typescript
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      port: 5432,
      username: 'test',
      password: 'test',
      database: 'test_db',
      entities: [User],
      synchronize: true, // Only for testing
    }),
  ],
})
export class TestDatabaseModule {}
```

## Performance Optimization

### Caching Strategies

Implement caching at multiple levels:

```typescript
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly cacheManager: Cache,
  ) {}

  async findById(id: string): Promise<User> {
    const cacheKey = `user:${id}`;
    let user = await this.cacheManager.get<User>(cacheKey);

    if (!user) {
      user = await this.userRepository.findById(id);
      await this.cacheManager.set(cacheKey, user, 300); // 5 minutes
    }

    return user;
  }
}
```

### Database Query Optimization

Use query optimization techniques:

```typescript
@Injectable()
export class OrderService {
  async findUserOrders(userId: string): Promise<Order[]> {
    return this.orderRepository
      .createQueryBuilder('order')
      .leftJoinAndSelect('order.items', 'items')
      .leftJoinAndSelect('order.payment', 'payment')
      .where('order.userId = :userId', { userId })
      .orderBy('order.createdAt', 'DESC')
      .getMany();
  }
}
```

## Security Best Practices

### Input Validation

Use DTOs with validation decorators:

```typescript
export class CreateUserDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password must contain at least one uppercase letter, one lowercase letter, and one number',
  })
  password: string;
}
```

### Authentication and Authorization

Implement JWT-based authentication:

```typescript
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get<string>('JWT_SECRET'),
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, email: payload.email };
  }
}

@Controller('users')
@UseGuards(JwtAuthGuard)
export class UsersController {
  @Get('profile')
  @UseGuards(RolesGuard)
  @Roles('user', 'admin')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

## Deployment and Production Considerations

### Health Checks

Implement health checks for monitoring:

```typescript
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }
}
```

### Graceful Shutdown

Handle graceful shutdown properly:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Graceful shutdown
  const signals = ['SIGTERM', 'SIGINT'];
  for (const signal of signals) {
    process.on(signal, async () => {
      console.log(`Received ${signal}, starting graceful shutdown`);
      await app.close();
      process.exit(0);
    });
  }

  await app.listen(3000);
}
```

## Conclusion

Building scalable applications with NestJS requires understanding its architectural patterns and following established best practices. The framework's dependency injection system, modular architecture, and TypeScript support provide a solid foundation for creating maintainable, testable, and scalable backend systems.

Key takeaways:
- Organize code into focused, cohesive modules
- Use dependency injection for loose coupling
- Implement proper error handling and logging
- Follow testing strategies for all layers
- Optimize for performance and security
- Plan for production deployment from the start

The patterns and practices outlined in this guide have been proven in production environments and will help you build robust, scalable NestJS applications that can grow with your business requirements.
