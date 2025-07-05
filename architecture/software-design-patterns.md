# Software Architecture and Design Patterns

Design patterns are the building blocks of maintainable, scalable software systems. They're not just academic concepts—they're proven solutions to recurring problems that every experienced developer encounters. Understanding when and how to apply these patterns can transform your code from a tangled mess into a clean, extensible architecture.

This guide focuses on the patterns you'll actually use in real-world projects, with practical examples and the wisdom that comes from applying them in production environments.

## Core Design Patterns

### Singleton Pattern

**Purpose:** Ensure a class has only one instance and provide global access to it.

**When to use:** Configuration management, database connections, logging systems, or any resource that should be shared across your application.

**Real-world analogy:** A company's CEO—there's only one, and everyone needs access to the same person.

```typescript
class DatabaseConnection {
  private static instance: DatabaseConnection;
  private constructor() {}
  
  public static getInstance(): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }
  
  public query(sql: string): any {
    // Database query logic
  }
}

// Usage
const db = DatabaseConnection.getInstance();
```

**Pros:** Guarantees single instance, lazy initialization, global access point.

**Cons:** Global state can make testing difficult, potential memory leaks, violates single responsibility principle.

**Pitfalls:** Don't use for everything—singletons can become a code smell when overused. Consider dependency injection instead.

### Factory Pattern

**Purpose:** Create objects without specifying their exact classes.

**When to use:** When object creation logic is complex, when you need to create objects based on runtime conditions, or when you want to decouple object creation from object usage.

**Real-world analogy:** A car factory—you specify what type of car you want, and the factory handles the complex assembly process.

```typescript
interface PaymentProcessor {
  process(amount: number): void;
}

class CreditCardProcessor implements PaymentProcessor {
  process(amount: number): void {
    console.log(`Processing ${amount} via credit card`);
  }
}

class PayPalProcessor implements PaymentProcessor {
  process(amount: number): void {
    console.log(`Processing ${amount} via PayPal`);
  }
}

class PaymentProcessorFactory {
  static createProcessor(type: 'credit' | 'paypal'): PaymentProcessor {
    switch (type) {
      case 'credit':
        return new CreditCardProcessor();
      case 'paypal':
        return new PayPalProcessor();
      default:
        throw new Error('Unknown payment type');
    }
  }
}

// Usage
const processor = PaymentProcessorFactory.createProcessor('credit');
processor.process(100);
```

**Pros:** Encapsulates object creation, easy to extend with new types, follows open/closed principle.

**Cons:** Can become complex with many types, may violate single responsibility principle if too large.

**Pitfalls:** Don't create factories for simple object creation. Use when you have complex creation logic or multiple related types.

### Strategy Pattern

**Purpose:** Define a family of algorithms, encapsulate each one, and make them interchangeable.

**When to use:** When you have multiple ways to perform the same operation, when you want to avoid large conditional statements, or when you need to switch algorithms at runtime.

**Real-world analogy:** Different sorting algorithms—bubble sort, quicksort, merge sort—all sort, but with different trade-offs.

```typescript
interface SortingStrategy {
  sort(data: number[]): number[];
}

class BubbleSort implements SortingStrategy {
  sort(data: number[]): number[] {
    // Bubble sort implementation
    return [...data].sort((a, b) => a - b);
  }
}

class QuickSort implements SortingStrategy {
  sort(data: number[]): number[] {
    // Quick sort implementation
    return [...data].sort((a, b) => a - b);
  }
}

class Sorter {
  private strategy: SortingStrategy;
  
  constructor(strategy: SortingStrategy) {
    this.strategy = strategy;
  }
  
  setStrategy(strategy: SortingStrategy): void {
    this.strategy = strategy;
  }
  
  sort(data: number[]): number[] {
    return this.strategy.sort(data);
  }
}

// Usage
const sorter = new Sorter(new BubbleSort());
const result = sorter.sort([3, 1, 4, 1, 5]);
sorter.setStrategy(new QuickSort());
```

**Pros:** Eliminates conditional statements, easy to add new strategies, follows open/closed principle.

**Cons:** Can lead to many small classes, may be overkill for simple cases.

**Pitfalls:** Don't use for simple if/else scenarios. Strategy pattern shines when you have complex, interchangeable algorithms.

### Observer Pattern

**Purpose:** Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

**When to use:** Event handling, model-view architectures, reactive programming, or any scenario where objects need to react to state changes.

**Real-world analogy:** A newsletter subscription—when the publisher releases new content, all subscribers automatically receive it.

```typescript
interface Observer {
  update(data: any): void;
}

class Subject {
  private observers: Observer[] = [];
  private state: any;
  
  attach(observer: Observer): void {
    this.observers.push(observer);
  }
  
  detach(observer: Observer): void {
    const index = this.observers.indexOf(observer);
    if (index > -1) {
      this.observers.splice(index, 1);
    }
  }
  
  notify(): void {
    this.observers.forEach(observer => observer.update(this.state));
  }
  
  setState(state: any): void {
    this.state = state;
    this.notify();
  }
}

class ConcreteObserver implements Observer {
  private name: string;
  
  constructor(name: string) {
    this.name = name;
  }
  
  update(data: any): void {
    console.log(`${this.name} received update:`, data);
  }
}

// Usage
const subject = new Subject();
const observer1 = new ConcreteObserver('Observer 1');
const observer2 = new ConcreteObserver('Observer 2');

subject.attach(observer1);
subject.attach(observer2);
subject.setState('New data');
```

**Pros:** Loose coupling, supports broadcast communication, easy to add/remove observers.

**Cons:** Can cause memory leaks if observers aren't properly detached, may lead to unexpected updates.

**Pitfalls:** Always detach observers when they're no longer needed. Be careful with circular references.

### Adapter Pattern

**Purpose:** Allow incompatible interfaces to work together by wrapping an existing class with a new interface.

**When to use:** When integrating third-party libraries, legacy code, or when you need to make incompatible interfaces work together.

**Real-world analogy:** A power adapter—converts one type of plug to work with a different type of socket.

```typescript
// Legacy interface
interface LegacyPayment {
  makePayment(amount: number, currency: string): boolean;
}

// New interface
interface ModernPayment {
  pay(amount: number): Promise<boolean>;
}

// Legacy implementation
class LegacyPaymentSystem implements LegacyPayment {
  makePayment(amount: number, currency: string): boolean {
    console.log(`Legacy payment: ${amount} ${currency}`);
    return true;
  }
}

// Adapter
class PaymentAdapter implements ModernPayment {
  private legacyPayment: LegacyPayment;
  
  constructor(legacyPayment: LegacyPayment) {
    this.legacyPayment = legacyPayment;
  }
  
  async pay(amount: number): Promise<boolean> {
    return this.legacyPayment.makePayment(amount, 'USD');
  }
}

// Usage
const legacySystem = new LegacyPaymentSystem();
const adapter = new PaymentAdapter(legacySystem);
await adapter.pay(100);
```

**Pros:** Allows integration of incompatible interfaces, follows single responsibility principle, easy to test.

**Cons:** Can add complexity, may not expose all features of the adapted class.

**Pitfalls:** Don't create adapters for simple interface changes. Use when you truly need to bridge incompatible systems.

### Dependency Injection

**Purpose:** Provide objects with their dependencies instead of having them create dependencies themselves.

**When to use:** When you want to decouple classes from their dependencies, make code more testable, or manage complex object graphs.

**Real-world analogy:** A restaurant kitchen—the chef doesn't grow vegetables or raise animals; ingredients are provided (injected) by suppliers.

```typescript
interface Logger {
  log(message: string): void;
}

class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(message);
  }
}

class UserService {
  private logger: Logger;
  
  constructor(logger: Logger) {
    this.logger = logger;
  }
  
  createUser(name: string): void {
    this.logger.log(`Creating user: ${name}`);
    // User creation logic
  }
}

// Usage
const logger = new ConsoleLogger();
const userService = new UserService(logger);
userService.createUser('John Doe');
```

**Pros:** Improves testability, reduces coupling, makes dependencies explicit, easier to swap implementations.

**Cons:** Can add complexity, requires understanding of DI containers for advanced usage.

**Pitfalls:** Don't inject everything—some dependencies are truly internal. Use for external dependencies and services.

## Object-Oriented vs Functional Design

Both paradigms offer valuable insights for design patterns, but they approach problems differently:

**Object-Oriented Design** focuses on:
- Encapsulation and data hiding
- Inheritance and polymorphism
- State management within objects
- Design patterns as class structures

**Functional Design** emphasizes:
- Immutability and pure functions
- Composition over inheritance
- Higher-order functions
- Avoiding side effects

In modern development, you'll often use both. React hooks demonstrate this beautifully—functional components with object-oriented patterns like observer (useEffect) and strategy (custom hooks).

## When to Apply Patterns

Design patterns are tools, not rules. Here's when to use them:

**Use patterns when:**
- You have a recurring problem that patterns solve
- The complexity justifies the abstraction
- You're building a framework or library
- You need to communicate design intent to other developers

**Prioritize simplicity when:**
- The problem is straightforward
- You're prototyping or building MVPs
- The pattern adds more complexity than it solves
- You're the only developer and won't be for long

Remember: The best code is the code that's easiest to understand and maintain. Patterns should serve this goal, not hinder it.

## Conclusion

Design patterns are powerful tools that can elevate your code from functional to elegant. But they're not silver bullets—they're solutions to specific problems. The key is recognizing when a pattern fits your situation and implementing it cleanly.

Start with the patterns that solve your immediate problems. As you gain experience, you'll develop an intuition for when patterns add value and when they add unnecessary complexity. The goal isn't to use every pattern—it's to write code that's clear, maintainable, and solves real problems.

In the end, the best architecture is the one that makes your code easier to understand, test, and extend. Patterns are just one way to achieve that goal.
