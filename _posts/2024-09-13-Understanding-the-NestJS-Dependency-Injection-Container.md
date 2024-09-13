---
tags:
  [
    "NestJS",
    "Dependency Injection",
    "Dependency Injection Container",
    "Inversion of Control",
  ]
categories: ["Backend", "Design Patterns"]
---

NestJS, a powerful framework for building efficient and scalable server-side applications, relies heavily on its Dependency Injection (DI) container. This IoC (Inversion of Control) container is at the heart of NestJS, managing dependencies and orchestrating the application lifecycle. In this post, we'll take a deep dive into the steps the DI container goes through when you start a NestJS application.

## What is Dependency Injection?

Before we dive into the specifics, let's briefly discuss what Dependency Injection is and why it's important.

Dependency Injection is a design pattern that allows us to develop loosely coupled code. The basic idea is that instead of creating objects directly inside a class, we "inject" these dependencies from the outside. This makes our code more modular, easier to test, and more maintainable.

In NestJS, the DI container automates this process, making it seamless for developers to work with complex dependency graphs.

## What is an IoC Container?

An IoC (Inversion of Control) container, also known as a DI container, is a framework for implementing automatic dependency injection. It manages object creation and lifetime, and injects dependencies into classes. The term "Inversion of Control" refers to the way it inverts the usual control flow of the application.

Key features of an IoC container include:

1. **Object creation**: The container creates instances of classes as needed.
2. **Dependency resolution**: It figures out what dependencies a class needs and provides them.
3. **Lifetime management**: It can manage the lifetime of objects (e.g., creating singletons or new instances for each request).
4. **Configuration**: It allows for flexible configuration of how objects are created and injected.

In NestJS, the IoC container is a core part of the framework, handling all these aspects automatically based on the decorators and module definitions you provide.

Now, let's explore the steps the NestJS DI container takes when bootstrapping an application.

## Step 1: Module Discovery

When you start a NestJS application, the first thing the DI container does is analyze the root module (typically `AppModule`) to identify all the modules that need to be loaded.

- Modules are the building blocks of NestJS apps.
- Every NestJS application has at least one root module.
- The IoC container discovers all modules, including imported modules.

Here's an example of a root module:

```typescript
@Module({
  imports: [AuthModule, UserModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

## Step 2: Provider Registration

Once all modules are discovered, the IoC container goes through each module and registers the providers defined in the `providers` array.

Providers can be:

- Services (e.g., `AuthService`, `UserService`)
- Repositories (for database operations)
- Other classes that provide functionalities

For each provider:

1. It checks for the `@Injectable()` decorator.
2. It adds the provider to the container, mapping it by its class or a custom token (if specified).

Example of a provider:

```typescript
@Injectable()
export class AuthService {
  constructor(private readonly userService: UserService) {}
}
```

## Step 3: Dependency Resolution

For each provider, the DI container resolves its dependencies by analyzing the constructor's parameters:

- When a provider declares a dependency (e.g., `AuthService` needs `UserService`), the IoC container automatically searches for the `UserService` in the container.
- If `UserService` is registered in the same module or a module imported by the current module, it is injected automatically.

The DI container uses type-based injection, resolving dependencies based on the type of the class.

## Step 4: Instantiation of Providers

Once dependencies are resolved, the IoC container instantiates the providers:

- If a provider has dependencies, they are instantiated recursively before the main provider is created.
- By default, providers are instantiated as singletons — only one instance is created for the whole application.

```typescript
const authService = new AuthService(userService);
```

## Step 5: Controller Instantiation

After providers are set up, the IoC container moves to controller instantiation:

- Controllers are the entry points for handling HTTP requests.
- They often depend on providers (services) to handle business logic.
- The container analyzes the controller's constructor and injects any required dependencies.

Example:

```typescript
@Controller("auth")
export class AuthController {
  constructor(private readonly authService: AuthService) {}
}
```

## Step 6: Lifecycle Hooks (if any)

If any providers or controllers implement lifecycle hooks, the IoC container calls these methods:

- `onModuleInit()`: Called once the host module's dependencies have been resolved.
- `onApplicationBootstrap()`: Called once all modules have been initialized, but before listening for connections.
- `onModuleDestroy()` and `beforeApplicationShutdown()`: Called during the shutdown process.

Example:

```typescript
@Injectable()
export class AuthService implements OnModuleInit {
  onModuleInit() {
    console.log("AuthService initialized");
  }
}
```

## Step 7: Application Bootstrapping

Finally, once everything is set up, NestJS bootstraps the application:

- The application starts listening for incoming HTTP requests (for HTTP applications).
- Controllers are now ready to handle requests using the injected services.

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

## Conclusion

Understanding the steps the NestJS DI container goes through helps developers appreciate the power and flexibility of the framework. It allows for:

- Modular architecture
- Easy testing through dependency mocking
- Scalable applications
- Clear separation of concerns

By leveraging the DI container, NestJS encourages best practices in software development, leading to more maintainable and robust applications.

Remember, while these steps happen automatically, you can customize the behavior of the DI container using custom providers, scope options, and more advanced features that NestJS offers.

### **Summary of Steps**

1. **Module Discovery**: NestJS identifies all modules in the app and their relationships.
2. **Provider Registration**: The IoC container registers all providers defined in the modules.
3. **Dependency Resolution**: The container resolves dependencies by analyzing constructor parameters.
4. **Provider Instantiation**: The container creates instances of the providers and injects their dependencies.
5. **Controller Instantiation**: Controllers are instantiated with their dependencies (e.g., services) injected.
6. **Lifecycle Hooks**: Calls lifecycle methods (`onModuleInit()`, etc.) if implemented by providers.
7. **Application Bootstrapping**: The application starts, and the HTTP server begins listening for requests.
