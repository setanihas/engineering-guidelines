# 🔄 Circular Dependencies — Prevention & Resolution Guide

### Overview

A **circular dependency** occurs when two or more modules depend on each other, directly or indirectly.  
Example:

```
A → B → C → A
```

This creates a closed loop where modules cannot be initialized in a deterministic order.  
In TypeScript or Node.js, this often causes:
- `undefined` imports
- “Cannot access X before initialization”
- Unexpected runtime behavior
- Re-render loops in React

---

## 🧩 Why Circular Dependencies Happen

Common causes include:
- **Two-way imports** (A imports B, and B imports A)
- **Improper use of barrel files** (`index.ts` with `export *`)
- **Mixed responsibilities** (modules doing too many unrelated things)
- **Tight coupling** between services or contexts
- **Shared types/constants** defined in the wrong place

---

## 🧠 How to Think to Prevent Them

Circular dependencies are not just a code issue — they’re an *architecture and thinking issue*.  
Here’s how to reason about them before they happen.

### 1. Think in Layers

Design your project as a **dependency hierarchy**:

```
UI (components)
↓
Hooks / State
↓
Services
↓
Utils
↓
Core Types / Shared Models
```

Each layer can depend only on layers *below* it — never above.

> 🧭 Ask: “Am I depending downward or upward?”

---

### 2. Keep Responsibilities Narrow

Each file or module should have a **single purpose**.

If a module handles both **data fetching** and **UI logic**, it likely creates unnecessary imports that can lead to cycles.

> Ask: “Does this file know too much?”

---

### 3. Separate Data from Behavior

Circular dependencies often occur when:
- A service calls another service’s method,  
- …and that second service also depends on the first one’s data.

The fix: extract the **shared data model or logic** into a neutral shared module.

> Ask: “Can this shared piece of logic live in a `shared/` or `core/` folder?”

---

### 4. Be Cautious with Barrel Files (`index.ts`)

Barrel files make imports cleaner but **hide dependency directions**.  
They are the #1 silent cause of circular imports.

Rules:
- Use barrels only for **one-directional** exports (e.g., only inside `components/`).
- Avoid re-exporting across different layers.

---

### 5. Use a Shared Layer for Common Entities

If two modules both need the same type, enum, or utility:
→ move it to a `shared/`, `core/`, or `common/` folder.

```ts
// ❌ Before
// A.ts
import { somethingFromB } from "./B";

// ✅ After
// shared/types.ts
export interface SomethingCommon { ... }

// A.ts / B.ts
import { SomethingCommon } from "../shared/types";
```

---

### 6. Type-Only Imports Help

If you only need types, use `import type`:

```ts
import type { User } from "./User";
```

This ensures TypeScript doesn’t generate a runtime import — breaking the circular chain.

---

## 🧭 When It Already Exists — How to Fix It

If you detect a circular dependency:

### 1️⃣ Visualize It

Use a dependency graph tool:

```bash
npx madge --circular src/
```

It will show you exactly which files are part of the loop.

---

### 2️⃣ Identify the Direction Problem

Draw it mentally or on paper:

```
A → B → C → A
```

Then ask:
> “Which direction *should* the dependency flow?”

Usually, one of the modules shouldn’t depend on another directly.

---

### 3️⃣ Use Abstraction or Events

If two modules truly need to communicate both ways:
- Introduce an **interface** or **event system** between them.

**Example (interface abstraction):**
```ts
export interface IUserEvents {
  onLogin(user: User): void;
}

class AuthService {
  constructor(private userEvents: IUserEvents) {}
}
```

Now `AuthService` no longer needs to import `UserService` directly.

---

### 4️⃣ Extract Shared Responsibility

If both sides depend on each other for a shared concept,
extract that concept into a new module.

> “If two modules both need each other,  
> there’s usually a *third one missing*.”

---

## ⚙️ Engineering Checklist

| Question | Goal |
|-----------|------|
| Does this import go downward (not upward)? | Maintain directionality |
| Is this module doing too much? | Enforce single responsibility |
| Are we using a barrel file correctly? | Prevent hidden imports |
| Can this logic be moved to `shared/`? | Remove tight coupling |
| Is this just a type dependency? | Use `import type` |
| Do two modules depend on each other? | Introduce abstraction or event |

---

## 🚀 Quick Summary

| Stage | What to Do |
|--------|-------------|
| **Before writing code** | Plan layered architecture |
| **While coding** | Question every cross-module dependency |
| **When it happens** | Analyze the dependency direction |
| **To fix** | Extract shared logic, or use abstraction/events |

---

## 🛠 Tools

- **Madge:**  
  `npx madge --circular src/` — visualizes circular dependencies.
- **Dependency Cruiser:**  
  `npx depcruise --include-only "^src" --circular` — deeper analysis.

---

## 💬 Final Thought

> Circular dependencies are not bugs —  
> they are *architecture feedback*.  
> They tell you that two modules know too much about each other.  
> The solution is always to **simplify ownership and direction**.
