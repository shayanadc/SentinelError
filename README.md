# SentinelError
How to handle exceptions professionally from Go to NodeJs

### The Sentinel Pattern: Elevating Node.js Error Handling with Go-Inspired Logic

In computer science, an error is more than just a failure; it is a critical piece of state that communicates the boundaries of a system. Fundamentally, error handling is the process of responding to unexpected conditions to maintain a program's integrity. Traditionally, errors were often treated as simple return codes or unstructured strings, but as systems grew in complexity, the need for **Sentinel Errors** emerged. A sentinel error is a specific, predefined value used to signify a unique error condition. By using these identifiable markers, developers move away from "blind" error handling—where a program simply knows *that* something failed—to "informed" error handling, where the program knows exactly *why* it failed and can decide whether to retry, recover, or halt.

### The Evolution: From Beginner to Professional in Go

In the Go programming language, the evolution of error handling is defined by the transition from string-based checks to the `errors.Is` pattern. A beginner approach is brittle because it relies on text; the professional approach utilizes sentinel variables and "wrapping" to preserve identity while adding context.

**Theory:** Decouple the **identity** of the error from the **description** of the error.

```go
// PROFESSIONAL GO EXAMPLE
var ErrUserNotFound = errors.New("user not found") // The Sentinel

func DatabaseCall() error {
    return ErrUserNotFound
}

func UserService() error {
    err := DatabaseCall()
    // Wrapping the sentinel with %w to add context
    return fmt.Errorf("service layer failed: %w", err) 
}

func main() {
    err := UserService()
    // Check for the "Identity" even though it's wrapped
    if errors.Is(err, ErrUserNotFound) {
        fmt.Println("Handle specifically: Show 404 Page")
    }
}
```

### The Gap: What Node.js Lacks

Despite its ubiquity, Node.js has historically lacked the robust primitives that make Go’s error handling so effective. In JavaScript, adding context usually meant "losing" the original error object. While Go treats errors as values that can be inspected via a standard library chain, Node.js developers often resorted to string concatenation, which destroys the ability to perform a strict identity check. There is no native `errors.is()` to recursively "un-nest" an error chain, leaving developers to choose between clean logs and functional error logic.

### The Proposed Solution: The "Cause" Chain

To bridge this gap, we use the modern `cause` property (Node.js v16.9.0+). This allows us to propagate an error from Function C to B to A, with each layer adding diagnostic information without destroying the original sentinel at the bottom of the stack.

**Theory:** Use a recursive utility to "peel" the error layers.

```go
// PROPOSED NODE.JS UTILITY
const isError = (err, target) => {
    let current = err;
    while (current) {
        if (current === target) return true;
        current = current.cause; // Traverse the "Wrap"
    }
    return false;
};

// General Example: C -> B -> A
const ERR_NOT_FOUND = new Error("Resource Missing");

function C() { throw ERR_NOT_FOUND; }
function B() { 
    try { C(); } 
    catch (e) { throw new Error("B failed", { cause: e }); } 
}
function A() { 
    try { B(); } 
    catch (e) { throw new Error("A failed", { cause: e }); } 
}

// Check at the top level
try { A(); } catch (e) {
    if (isError(e, ERR_NOT_FOUND)) { /* Handle the sentinel */ }
}
```

### Two Implementation Strategies for Node.js

There are two primary ways to implement this pattern depending on your project's architecture:

**1. The Constant Sentinel (Simple & Direct)** Best for small to medium projects where you want a shared "Dictionary" of errors.

```go
export const ERR_UNAUTHORIZED = new Error("Auth Failed");
```

**2. The Class-Based Sentinel (Robust & Scalable)** Best for TypeScript/Large-scale apps. It allows you to check for "categories" of errors using `instanceof` during the recursive search.

```go
class DatabaseError extends Error {}
const ERR_QUERY_TIMEOUT = new DatabaseError("Timeout");
```

### 1. Handling Wrapped Errors as Types (The "Result" Pattern)

Instead of using `try/catch` which is computationally expensive and disrupts the flow, we return a "Result" object. This mimics Go's `return value, err` pattern in a single JavaScript object.

**Theory:** Avoid exceptions for "expected" failures (like a record not found) and treat them as data.

```go
// A standard "Result" type
const success = (data) => ({ ok: true, data, err: null });
const fail = (err) => ({ ok: false, data: null, err });

function FetchUser(id) {
    if (!id) return fail(ERR_INVALID_INPUT);
    const user = db.find(id);
    if (!user) return fail(ERR_NOT_FOUND); // Returning a Sentinel
    return success(user);
}

// Usage: No try/catch needed
const { data, err } = FetchUser(null);
if (err) {
    // Handle error as a standard variable
}
```

### 2. Efficiently Comparing Wrapped Errors

Comparing errors by string or by identity `==` can fail if the error is wrapped. To be efficient, we use a **Recursive Unwrapper**. This ensures that no matter how deep the "Service" layer hides the "Database" error, we can find it in O(N)time, where N is the depth of the wrap.

**The Professional Unwrapper:**

```go
/**
 * Efficiently checks if 'target' exists anywhere in the 'err' chain.
 */
function is(err, target) {
    let current = err;
    while (current) {
        // 1. Check Identity (Sentinel)
        if (current === target) return true;
        
        // 2. Check Constructor (Class/Type)
        if (target.prototype && current instanceof target) return true;
        
        // 3. Move to the next link in the chain
        current = current.cause;
    }
    return false;
}
```

### 3. Wrapping Multiple Different Errors

A service often coordinates multiple dependencies (e.g., a Database and an Email API). When multiple things go wrong, we wrap the specific error inside a **Contextual Error**. This tells the caller *where* it happened while preserving *what*happened.

**Example: The "User Registration" Service**

```go
// Define high-level service error types
class ServiceError extends Error {
    constructor(message, cause) {
        super(message, { cause });
    }
}

function RegisterUser(userData) {
    // 1. Database Error
    const dbResult = saveToDb(userData);
    if (!dbResult.ok) {
        return fail(new ServiceError("Failed to persist user", dbResult.err));
    }

    // 2. Email Error
    const emailResult = sendWelcomeEmail(userData.email);
    if (!emailResult.ok) {
        // We wrap a different error type (EmailError) into the same ServiceError
        return fail(new ServiceError("User saved, but email failed", emailResult.err));
    }

    return success(userData);
}
```
