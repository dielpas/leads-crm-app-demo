# Code Quality Review

**Branch:** Current Working Branch vs `main`
**Scope:** Reviewing all changes on this branch, including the newly introduced `/api/leads/search` endpoint and the recently applied security patches.
**Reference:** `code-review-and-quality` guidelines.

## Overview
The overall implementation of the search feature is functional and follows the existing architectural patterns of the application. The recent security patches have effectively mitigated the most critical risks (SQL Injection, Error Information Disclosure, DoS via unbounded queries). 

However, applying the rigorous five-axis review reveals a few remaining architectural, correctness, and performance issues that should be addressed before or shortly after merging.

---

## 1. Correctness & Input Validation

### Issue: Unsafe Runtime Type Casting
*   **Severity:** **Required**
*   **Location:** `server.ts` - `const query = req.query.q as string;`
*   **Description:** In Express, `req.query.q` is not guaranteed to be a string. If a client sends `?q[]=foo&q[]=bar` or `?q[foo]=bar`, `req.query.q` will be parsed as an `Array` or an `Object`. 
    *   Casting `as string` only silences TypeScript; it does not convert the runtime type.
    *   If an array or object is passed, `query.length > 100` might evaluate unexpectedly (e.g., `undefined > 100` is `false`, bypassing the check).
    *   The template literal `%${query}%` would evaluate to `%[object Object]%` or `%foo,bar%`.
*   **Proposed Improvement:** Replace the truthiness check and cast with an explicit runtime type check.
    ```typescript
    const rawQuery = req.query.q;
    if (!rawQuery || typeof rawQuery !== 'string') {
      return res.status(400).json({ error: "Search query must be a non-empty string" });
    }
    const query = rawQuery; // Now correctly inferred as string
    ```

---

## 2. Performance

### Issue: Full Table Scans via Leading Wildcards (`LIKE '%...%'`)
*   **Severity:** **FYI / Consider for Future**
*   **Location:** `server.ts` - `LIKE ?` with `%${query}%`
*   **Description:** The query uses leading wildcards (`%`) for `name`, `company`, and `email`. Standard B-Tree indexes cannot optimize queries that start with a wildcard. This forces the SQLite database engine to perform a **Full Table Scan**, checking every single row in the `leads` table.
*   **Proposed Improvement:** 
    *   **Short-term:** Acceptable for a small CRM with limited records. The `LIMIT 50` added previously protects the application memory, but the database will still do heavy lifting.
    *   **Long-term:** Implement SQLite's **FTS5 (Full-Text Search)** extension for the `leads` table. This creates a virtual table optimized for searching text, which will be magnitudes faster as the database grows.

---

## 3. Architecture & Security

### Issue: Missing Authentication Strategy
*   **Severity:** **Consider** (Contextual)
*   **Description:** As noted in the prior security review, the `/api/leads/search` endpoint (and the entire API) is publicly accessible without any authentication middleware. 
*   **Proposed Improvement:** If this application is intended to hold real customer data, an authentication strategy (e.g., Session cookies, JWTs) must be implemented and a global middleware (e.g., `app.use('/api', authenticate)`) should be applied to protect all lead data.

### Issue: Route Handler Fatness
*   **Severity:** **Nit**
*   **Description:** The application currently defines all database operations and routing logic inline within `server.ts`. As the application grows, this file will become unmaintainable (a "God object").
*   **Proposed Improvement:** Consider extracting the database queries into a dedicated service layer (e.g., `services/leadService.ts`) and the route definitions into separate controllers or routers (e.g., `routes/leads.ts`).

---

## Verdict

- [ ] **Request Changes:** The runtime type casting issue (`typeof query !== 'string'`) should be fixed to ensure correct application behavior and robust input validation.
- [ ] **Approve (with nits):** Once the type check is fixed, the code is structurally sound and secure enough for merge, given the current scope of the application.
