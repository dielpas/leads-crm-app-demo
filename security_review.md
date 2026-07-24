# Security & Code Quality Review: `/api/leads/search` Endpoint

Based on the multi-axis code review principles (`security-and-hardening` and `code-review-and-quality` skills), here is the review of the recent changes to `server.ts` introducing the `/api/leads/search` endpoint.

## 🚨 Critical Security Findings

### 1. SQL Injection (SQLi) Vulnerability
*   **Severity:** **Critical**
*   **Description:** The user input (`req.query.q`) is directly concatenated into the SQL statement using template literals. This allows an attacker to input malicious SQL commands, potentially reading unauthorized data, modifying records, or dropping tables.
*   **Reference:** *Security and Hardening > Injection (SQL, NoSQL, OS Command)*
*   **Plan to Fix:** Refactor the query to use parameterized inputs. 
    ```typescript
    const searchTerm = `%${query}%`;
    const sql = `SELECT * FROM leads WHERE name LIKE ? OR company LIKE ? OR email LIKE ?`;
    const results = db.prepare(sql).all(searchTerm, searchTerm, searchTerm);
    ```

### 2. Sensitive Data Exposure via Error Handling
*   **Severity:** **Critical**
*   **Description:** In the `catch` block, `error.message` is exposed directly to the client in the `details` field of the 500 response. This can leak internal database schemas, table names, and query structures to attackers.
*   **Reference:** *Security and Hardening > Sensitive Data Exposure*
*   **Plan to Fix:** Keep detailed errors in server-side logs only (`console.error`) and return a generic, safe error message to the client (e.g., `"An unexpected error occurred during search"`).

## ⚠️ High/Medium Quality & Security Findings

### 3. Unbounded Query Results (Denial of Service / Performance Issue)
*   **Severity:** **High**
*   **Description:** The query uses `.all()` without any `LIMIT` or pagination. A broad search term (e.g., `"a"`) could return thousands of records, leading to memory exhaustion, slow response times, and potential Denial of Service (DoS).
*   **Reference:** *Code Review and Quality > Performance* and *Security and Hardening > Denial of Service*
*   **Plan to Fix:** 
    *   Add a `LIMIT` clause to the SQL query (e.g., `LIMIT 50`).
    *   Implement pagination if the UI requires viewing more results.

### 4. Insufficient Input Validation
*   **Severity:** **Medium**
*   **Description:** The `query` is only checked for truthiness. There are no constraints on the maximum length or format of the input, which could allow overly long inputs to consume server resources or be used for DoS attacks.
*   **Reference:** *Security and Hardening > Input Validation Patterns*
*   **Plan to Fix:** Enforce a strict length constraint (e.g., max 100 characters) before executing the query. If possible, use a schema validation library like `zod`.

### 5. Potential Missing Access Control
*   **Severity:** **Medium** (Pending context)
*   **Description:** The route `app.get("/api/leads/search", ...)` is registered without inline authentication middleware. Unless a global middleware protects the `/api/leads` path, this exposes sensitive customer data (leads) to unauthenticated users.
*   **Reference:** *Security and Hardening > Broken Access Control*
*   **Plan to Fix:** Verify that authentication middleware (e.g., `authenticate`) is applied to this route or globally. If not, add it inline: `app.get("/api/leads/search", authenticate, ...)`

---

## 🛠️ Execution Plan

I will not apply these fixes yet per the instructions. Once approved, the changes to `server.ts` will involve:
1.  **Modifying the SQL statement** to remove string interpolation and use `?` placeholders.
2.  **Updating `db.prepare(sql).all()`** to pass the bounded `searchTerm` multiple times.
3.  **Appending `LIMIT 50`** to the SQL query to cap results.
4.  **Adding an input length check** (e.g., `if (query.length > 100) return res.status(400)...`) right after the empty check.
5.  **Sanitizing the `catch` block** to only return `{ error: "Search failed" }`.
