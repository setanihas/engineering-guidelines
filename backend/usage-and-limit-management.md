# 📊 Usage & Limit Management Overview

## 1. Purpose
This document describes the design and implementation of **usage and limit tracking** for both **user-level** and **project-level** resources.  
The system supports:
- Numeric limits  
- Boolean flags  
- Periodic limits  
- Lifetime limits  

---

## 2. Core Concepts

### 🔹 Limit Entries
Stored in the `limit_entries` collection.  
Defines the rules and configuration for each limit.

**Key Fields**
- `scope`: `"user"` or `"project"`
- `key`: unique identifier for the limit (e.g. `page_count`, `visitor_count`, `storage`, `ab_test_enabled`)
- `limit`: numeric limit, if applicable
- `active`: boolean limit
- `period`: `"day" | "month" | "lifetime"`

**Examples**
```json
{
  "scope": "project",
  "projectId": "proj_42",
  "key": "page_count",
  "limit": 100,
  "period": "month"
}
```

```json
{
  "scope": "user",
  "userId": "user_123",
  "key": "storage",
  "limit": 1048576,
  "period": "lifetime"
}
```

---

### 🔹 Limit Usages
Stored in the `limit_usages` collection.  
Tracks real-time usage for each limit.

**Key Fields**
- `used`: numeric usage value
- `periodStart` / `periodEnd`: used for periodic limits only  
  Lifetime limits do **not** use these fields.

**Examples**
```json
{
  "scope": "project",
  "projectId": "proj_42",
  "key": "page_count",
  "used": 37,
  "periodStart": "2025-10-01T00:00:00Z",
  "periodEnd": "2025-10-31T23:59:59Z"
}
```

```json
{
  "scope": "user",
  "userId": "user_123",
  "key": "storage",
  "used": 153600
}
```

---

## 3. Design Principles

### 🧩 Separate Collections
- `limit_entries` for configuration  
- `limit_usages` for dynamic state  
→ This separation ensures atomic updates, better scalability, and clear audit trails.

### 🔑 Key-Based Tracking
- Each limit has its **own document**.
- Prevents write contention and supports per-limit indexing.

### ⏳ Period vs. Lifetime
- **Periodic limits** (daily/monthly) reset after their defined period.  
- **Lifetime limits** (e.g. storage usage) never reset.

### ⚡ Atomic Increment & Limit Check
- `$inc` operations ensure atomic updates.  
- Limit checks occur **immediately after increment** to prevent over-usage.

### 🔄 Reset Strategies
- Periodic limits can be reset by:
  - A scheduled cron job  
  - Manual trigger via Admin API  
- Lifetime limits are **never reset**.

---

## 4. Usage Example

### Increment usage for a monthly limit
```ts
await incrementLimitUsage({
  scope: "project",
  key: "page_count",
  projectId: "proj_42",
  increment: 1
});
```

### Increment usage for a lifetime limit
```ts
await incrementLimitUsage({
  scope: "user",
  key: "storage",
  userId: "user_123",
  increment: 1024
});
```

---

## 5. Advantages

| Benefit | Description |
|----------|--------------|
| ⚙️ **Scalable** | Suitable for high-traffic environments with minimal write conflicts. |
| 🧾 **Audit-Friendly** | Lifetime usage data is never lost; periodic data can reset safely. |
| 🧠 **Flexible** | Supports numeric, boolean, periodic, and lifetime configurations. |
| 🛡️ **Safe** | Per-limit documents prevent race conditions during concurrent updates. |

---

## 6. Summary
The **Usage & Limit Management System** provides a consistent and scalable foundation for managing per-user and per-project resource constraints.  
Its separation of configuration and dynamic state allows fine-grained tracking, predictable resets, and high data integrity across all usage types.

---
