---
tags: ["NestJS", "Interceptors"]
categories: ["Backend", "NestJS"]
---

# What are Interceptors?

Interceptors in NestJS are powerful tools that allow you to intercept and manipulate the request-response cycle of your application. They provide a way to:

1. Execute custom logic before a request reaches your route handler
2. Transform the result returned from your route handler before it's sent back to the client
3. Extend basic function behavior with extra functionality
4. Completely override a function depending on specific conditions (e.g., for caching purposes)

Now, let's dive deeper into the world of interceptors and explore their key components.

## Exploring the `intercept` method, `ExecutionContext`, and `CallHandler`

The `intercept` method is the heart of a custom interceptor. It takes two crucial arguments: `ExecutionContext` and `CallHandler`. Let's break down their roles:

### Execution Context

- Represents the context of the current execution point within your application.
- Provides access to various details about the ongoing request or event.
- Obtained using the `context` argument in the `intercept` method of an interceptor.

Common properties and methods of `ExecutionContext`:

- `getClass()`: Returns the class associated with the current execution context (e.g., the controller class).
- `getHandler()`: Returns the handler function being executed (e.g., the controller method).
- `switchToHttp()`: Switches the context to the HTTP context if applicable, allowing access to HTTP request and response objects.

### What is `next.handle()` Really Doing?

When you call `next.handle()` inside an interceptor, it represents the **continuation** of the request lifecycle. Here's how it works in both cases: when there **are** or **are not** any other interceptors.

#### 1. When There Are No Other Interceptors

If there are no other interceptors in the request lifecycle, `next.handle()` will execute the **controller method** that handles the request. In this case:

- **`next.handle()` retrieves the response that the controller method will send to the client.**
- After you call `next.handle()`, you can subscribe to the `Observable` returned by the controller and manipulate or inspect the response before it is sent to the client.

For example:

```ts
return next.handle().pipe(
  tap((data) => {
    console.log("Controller response:", data); // Manipulate or log the response
  })
);
```

In this case, `next.handle()` means:

- "Pass the control to the controller method and handle its response."

#### 2. When There Are Other Interceptors

When there are multiple interceptors in the chain:

- Each interceptor calls `next.handle()`, which passes control to the **next interceptor**.
- Eventually, the last interceptor in the chain will call `next.handle()`, which executes the controller method.

In this case, the lifecycle works like this:

1. First interceptor calls `next.handle()` and waits for the result.
2. The second interceptor (if it exists) calls `next.handle()` and passes control down the chain.
3. Eventually, the controller method is called and returns a response as an `Observable`.
4. The response then bubbles back up the interceptor chain, allowing each interceptor to manipulate or inspect the response.

### Why Call `next.handle()`?

You must call `next.handle()` to:

- **Ensure the request completes**: If you don't call it, the request stops at the interceptor, and the controller method is never reached.
- **Allow response modification**: The response from the controller passes back through the interceptors, giving you a chance to modify or observe it.

### `next.handle()` and the Client Response

Now, let's focus on this part of your question:

> "The docs are saying that inside the `next.handle()` will be the response going to the client."

This is correct, but it happens after the controller method is executed. Here's the sequence of events:

1. **`next.handle()` calls the controller method** (or the next interceptor).
2. The controller method returns a response as an **`Observable`**.
3. That **`Observable`** is passed back through the interceptor(s), where it can be modified.
4. Finally, the **modified response** is sent to the client.

The docs mean that the **result of `next.handle()`** is the **response stream** that the client will eventually receive, **after** it has gone through the full lifecycle.

### Example of Interceptor Flow:

Here's a simplified flow with two interceptors and a controller:

```ts
// First Interceptor
intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
  console.log('First Interceptor: Before Controller');
  return next.handle().pipe(
    tap((response) => {
      console.log('First Interceptor: After Controller', response);
    })
  );
}

// Second Interceptor
intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
  console.log('Second Interceptor: Before Controller');
  return next.handle().pipe(
    tap((response) => {
      console.log('Second Interceptor: After Controller', response);
    })
  );
}

// Controller Method
getData() {
  return of({ message: 'Hello from the Controller' });
}
```

- **Before the Controller**: Both interceptors log before calling `next.handle()`.
- **After the Controller**: Both interceptors log after receiving the response from `next.handle()`.

The **controller** returns an `Observable` of `{ message: 'Hello from the Controller' }`, which is passed back through both interceptors before reaching the client.

### Key Points:

- `next.handle()` **continues the request lifecycle**. If there are no more interceptors, it invokes the controller method.
- The result of `next.handle()` is the **response stream**, which can be modified by the interceptor before it is sent to the client.
- In NestJS, the response from a controller is always treated as an `Observable`, making it easy to apply transformations.

## Understanding Observables and RxJS Operators

Let's break down the concepts of **Observable**, **pipe**, **tap**, and **map** in a simple way.

### 1. Observable (from RxJS)

An **Observable** is like a **data stream** or a "box" that can hold and emit values over time. It's a way to handle **asynchronous** data or events in JavaScript.

- You can think of it like a **Netflix stream**: It sends data (like episodes of a series) at different times, and you can "subscribe" to it to watch them. In an Observable, the "data" could be anything: HTTP responses, user inputs, or even emitted values from a backend controller.

- **Why use it?** It allows us to handle data that doesn't arrive immediately, like the response from an HTTP call or database query.

### 2. `pipe()` Method

The `pipe()` method is used to **combine multiple operations** (called "operators") that we want to apply to the data inside the Observable.

Think of `pipe()` as a **processing pipeline** where you can add steps to **modify or handle** the data in sequence.

- Example: If you want to log some data, modify it, and then return the result, you can use `pipe()` to chain these steps together.

### 3. `tap()` Method

`tap()` is an operator that lets you **do something with the data** flowing through the Observable **without changing it**. You use it when you want to **peek** into the data but not modify it.

- Example: You want to log the response from the server **without changing** it, you use `tap()`.

```ts
return next.handle().pipe(
  tap((data) => {
    console.log("Response from the server:", data);
  })
);
```

- **Purpose of `tap()`**: Inspect or log data in the middle of a stream **without affecting the result**.

### 4. `map()` Method

`map()` is used when you want to **modify the data** in some way. It **transforms** the data inside the Observable and passes the transformed result down the stream.

- Example: Let's say the response from the server includes a user's name, and you want to make the name all uppercase before sending it to the client. You can use `map()` to transform the response.

```ts
return next.handle().pipe(
  map((data) => {
    return { ...data, name: data.name.toUpperCase() }; // Modify the data
  })
);
```

- **Purpose of `map()`**: **Transform** the data as it moves through the Observable stream.

### Putting it All Together:

Here's how it all works together in an interceptor example:

```ts
intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
  return next.handle().pipe(
    tap((response) => {
      console.log('Response before modification:', response);  // Use tap to log the response
    }),
    map((response) => {
      return { ...response, additionalData: 'This was added!' };  // Use map to modify the response
    })
  );
}
```

### What's Happening Here?

1. **`next.handle()`** calls the controller, which returns an `Observable` of the response.
2. The **`pipe()`** method allows you to apply multiple transformations to the response data:
   - **`tap()`** is used to log the response without modifying it.
   - **`map()`** modifies the response (in this case, adding `additionalData`).
3. After these operations, the **modified response** is sent to the client.

## Comparing Interceptors, Middleware, and Exception Filters

When working with NestJS, you'll encounter three powerful tools for handling requests and responses: Interceptors, Middleware, and Exception Filters. Each serves a unique purpose in the request lifecycle. Let's compare them:

### Interceptors

Interceptors are versatile tools that can interact with both the request and response:

- They have access to the execution context, which includes both the request and response objects.
- Interceptors can execute logic **before and after** the route handler is called.
- They're ideal for tasks like logging, transforming the response, or adding metadata to the response.

**Key Point**: Interceptors offer the most flexibility, allowing you to modify or enhance both incoming requests and outgoing responses.

### Middleware

Middleware functions are more focused on the request phase:

- They are executed **only before** the route handler is called.
- Middleware is great for tasks that need to be done before the request reaches your route handlers.
- Common use cases include parsing request bodies, setting headers, or performing authentication checks.

**Key Point**: Middleware is perfect for preprocessing requests but doesn't have access to the response after the route handler has executed.

### Exception Filters

Exception Filters play a crucial role in error handling:

- They are called **after** both the route handler and any interceptors have completed.
- Exception Filters catch and process unhandled exceptions that occur during request processing.
- They're used to customize error responses or perform logging when exceptions occur.

**Key Point**: Exception Filters are your last line of defense for handling errors and shaping how your application responds to exceptions.

### Putting It All Together

Here's a simplified view of how these components fit into the request lifecycle:

1. **Middleware** processes the incoming request.
2. **Interceptors** (pre-handler) can perform actions before the route handler.
3. The **Route Handler** executes.
4. **Interceptors** (post-handler) can modify the response.
5. If an exception occurs at any point, **Exception Filters** handle it.

By understanding the unique roles of Interceptors, Middleware, and Exception Filters, you can choose the right tool for each task in your NestJS application, ensuring clean, maintainable, and efficient code.

## Summary:

- **Observable**: A stream of data or events that you can subscribe to (like async data from the controller).
- **`pipe()`**: A method to combine multiple steps (like tap, map) into a single stream.
- **`tap()`**: A way to do something (like log) with the data **without changing it**.
- **`map()`**: A way to **transform** the data in the stream.
- **Interceptors**: Flexible tools for modifying both requests and responses.
- **Middleware**: Focused on preprocessing requests.
- **Exception Filters**: Specialized for handling and customizing error responses.

By mastering these concepts, you'll be well-equipped to leverage the full power of NestJS in your applications!
