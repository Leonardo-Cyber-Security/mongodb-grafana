# MongoDB-Grafana Plugin — Repository Health Analysis

## 1. Project Health Overview

| Dimension | Status |
|-----------|--------|
| Last upstream update | ~2018 (8 years ago) |
| Node.js engine | Pinned to **6.10.0** (EOL since April 2019) |
| Grafana target | **3.x.x** (current is 11.x) |
| MongoDB driver | **3.0.8** (current is 6.x) |
| Build system | Grunt + Babel 6 (obsolete toolchain) |
| Tests | Minimal (frontend only, no proxy tests) |
| CI/CD | None |
| Linting / Formatting | None |
| Security posture | Several critical issues |

---

## 2. Critical Issues

### 2.1 **[HIGH] NoSQL / Aggregation Injection via `JSON.parse` of user input**
mongodb-proxy.js — The query string coming from Grafana is directly parsed with `JSON.parse` and sent to MongoDB's `aggregate()`. A malicious or accidental query can execute arbitrary aggregation pipelines (e.g., `$out` to overwrite collections, `$merge`, `$lookup` to exfiltrate data).

```js
// CURRENT — no validation
args = '[' + args + ']'
docs = JSON.parse(args)
doc.pipeline = docs[0]
```

**Fix:** Whitelist allowed aggregation stages and reject dangerous ones:

```js
const BLOCKED_STAGES = new Set([
  '$out', '$merge', '$collStats', '$indexStats',
  '$planCacheStats', '$currentOp', '$listSessions'
]);

function validatePipeline(pipeline) {
  for (const stage of pipeline) {
    const key = Object.keys(stage)[0];
    if (BLOCKED_STAGES.has(key)) {
      throw new Error(`Aggregation stage "${key}" is not allowed`);
    }
  }
}
```

---

### 2.2 **[HIGH] MongoDB Connection String Sent in Plaintext over HTTP**
The frontend stores the MongoDB URL (including credentials) in `jsonData` and POSTs it in every request body to the proxy. This means credentials are visible in browser devtools, network logs, and any proxy in between.

**Fix:**
- Store the MongoDB URL in `secureJsonData` (Grafana's encrypted storage).
- Have the proxy read the connection string from its own config or environment variables, **not** from the request body.
- At minimum, use the proxy's default.json to hold the connection info server-side.

---

### 2.3 **[HIGH] Wildcard CORS: `Access-Control-Allow-Origin: *`**
mongodb-proxy.js allows any origin to call the proxy, which could be exploited via CSRF if the proxy is network-accessible.

```js
// CURRENT
res.setHeader("Access-Control-Allow-Origin", "*");

// FIX — restrict to the Grafana origin
const ALLOWED_ORIGIN = process.env.GRAFANA_ORIGIN || 'http://localhost:3000';
res.setHeader("Access-Control-Allow-Origin", ALLOWED_ORIGIN);
```

---

### 2.4 **[HIGH] MongoDB Client Connection Leak**
Every request in `/query` and `/search` calls `MongoClient.connect()` and creates a brand-new connection. On errors in several code paths, `client.close()` is never called. Additionally, the `doTemplateQuery` function uses `assert.equal(err, null)` which will **crash the process** on a DB error instead of returning an error response.

**Fix:** Use a connection pool (singleton client) and handle errors properly:

```js
let clientPromise = null;

function getClient(url) {
  if (!clientPromise) {
    clientPromise = MongoClient.connect(url, { 
      maxPoolSize: 10,
      minPoolSize: 1 
    });
  }
  return clientPromise;
}
```

---

### 2.5 **[HIGH] `assert.equal(err, null)` in Production Code**
mongodb-proxy.js — In `doTemplateQuery`, aggregation errors cause `assert.equal` to throw an `AssertionError`, crashing the Node.js process without returning an HTTP error.

```js
// CURRENT
assert.equal(err, null)  // crashes the process

// FIX
if (err) {
  client.close();
  return queryError(requestId, err, next);
}
```

---

## 3. Recommended Improvements

### 3.1 **[HIGH] Upgrade Node.js Engine** — `6.10.0` → `18.x` or `20.x`
Node 6 is 7 years past EOL. No security patches. All dependencies are stuck at old versions because of it.

In package.json:
```json
"engines": {
  "node": ">=18.0.0"
}
```

### 3.2 **[HIGH] Upgrade MongoDB Driver** — `3.0.8` → `6.x`
The `mongodb@3` driver uses the legacy callback API. Version 6.x uses Promises, supports modern MongoDB (7.x/8.x), and has numerous security fixes.

```bash
npm install mongodb@latest
```

Then refactor from callbacks to async/await:
```js
// BEFORE
MongoClient.connect(url, function(err, client) { ... });

// AFTER
const client = await MongoClient.connect(url);
const db = client.db(dbName);
const results = await collection.aggregate(pipeline).toArray();
```

### 3.3 **[HIGH] Replace Build Toolchain** — Grunt/Babel 6 → modern tooling
Grunt and Babel 6 are unmaintained. The Grafana plugin ecosystem now uses:
- **@grafana/create-plugin** (official scaffolding)
- **Webpack 5** or **Vite** for bundling
- **TypeScript** for type safety
- **@grafana/toolkit** or the newer **@grafana/plugin-e2e** for testing

Minimal migration: Replace Grunt with **npm scripts + esbuild/webpack** and upgrade Babel to 7.x if you stay with JS.

### 3.4 **[MEDIUM] Migrate Grafana Plugin to Modern SDK**
The plugin targets Grafana 3.x using AngularJS (`QueryCtrl`, `ConfigCtrl`). Grafana deprecated Angular plugins in **v7** and plans to remove support entirely. The modern approach uses **React** with the `@grafana/data` and `@grafana/ui` packages.

Key changes needed:
- plugin.json — `grafanaVersion` should become `">=9.0.0"` (or whatever minimum you target)
- module.js — Replace Angular class exports with React component exports
- datasource.js — Extend `DataSourceApi` from `@grafana/data`
- Replace `backendSrv.datasourceRequest()` → `getBackendSrv().fetch()` (uses RxJS)
- Consider making it a **backend plugin** (Go) to avoid sending credentials through the frontend

### 3.5 **[MEDIUM] Add Environment Variable Support for Config**
default.json hardcodes port 3333. Support env vars:

```js
const port = process.env.PORT || serverConfig.port || 3333;
app.listen(port);
```

### 3.6 **[MEDIUM] Implicit Globals in Proxy**
Several variables in mongodb-proxy.js are used without `var`/`let`/`const`, making them implicit globals — a source of subtle bugs:

| Line(s) | Variable |
|---------|----------|
| ~207 | `doc = {}` |
| ~208 | `queryErrors = []` |
| ~259 | `docs = JSON.parse(args)` |
| ~136 | `queryArgs = parseQuery(...)` |
| ~137 | `tg = req.body.targets[queryId]` |
| ~345 | `rows = []`, `row = []` |
| ~396 | `output = []` |

**Fix:** Add `const` or `let` to every declaration. Enable strict mode:
```js
'use strict';
```

### 3.7 **[MEDIUM] Add Linting and Formatting**
No `.eslintrc`, no Prettier config. Add:

```bash
npm install --save-dev eslint prettier eslint-config-prettier
```

```json
// .eslintrc.json
{
  "env": { "node": true, "es2020": true },
  "extends": ["eslint:recommended"],
  "rules": {
    "no-unused-vars": "warn",
    "no-undef": "error",
    "strict": ["error", "global"]
  }
}
```

### 3.8 **[MEDIUM] Add Tests for the Proxy Server**
The spec files only test the frontend datasource logic. There are **zero tests** for the proxy server, which is where the most critical logic lives (query parsing, aggregation execution, result formatting).

Add tests for:
- `parseQuery()` — valid and malformed queries
- `getTimeseriesResults()` / `getTableResults()` — output formatting
- `getBucketCount()` — bucket calculation
- HTTP endpoint integration tests with supertest

### 3.9 **[LOW] Remove Unnecessary `fs` Dependency**
package.json lists `"fs": "0.0.1-security"` — this is a stub package. The built-in `fs` module requires no install. Remove it.

### 3.10 **[LOW] Remove `q` Promise Library**
package.json — `q` is a legacy promise library. Node.js has native Promises since v4. Remove it and use native promises in tests.

### 3.11 **[LOW] Update `lodash`**
`lodash@4.17.10` has known prototype pollution vulnerabilities. Update to `>=4.17.21`. Better yet, replace it with native JS — the usage is minimal (just `_.map`, `_.filter` in the datasource).

### 3.12 **[LOW] Add .gitignore Entries**
.gitignore only ignores `node_modules/` and `.gradle/`. Add:
```
dist/
*.log
.env
coverage/
.vscode/
```

---

## 4. Optional Modernization Ideas

| Idea | Effort | Impact |
|------|--------|--------|
| **Rewrite proxy in TypeScript** | Medium | Catches type bugs, better IDE support |
| **Dockerize** with `docker-compose.yml` (proxy + Grafana + MongoDB) | Low | Easy local dev/testing |
| **Add CI/CD** with GitHub Actions (lint, test, build, release) | Low | Prevents regressions |
| **Convert to Grafana backend plugin (Go)** | High | Eliminates the JS proxy entirely; credentials stay server-side |
| **Add Helmet.js** to Express proxy for security headers | Low | Mitigates XSS/clickjacking |
| **Add rate limiting** to proxy endpoints | Low | Prevents abuse |
| **Support `find()` queries** in addition to `aggregate()` | Medium | User convenience |
| **Health check endpoint** (`GET /health`) | Trivial | Monitoring/load-balancer integration |

---

## 5. Example Patches

### Patch A — Fix implicit globals and add strict mode to proxy

```diff
--- a/server/mongodb-proxy.js
+++ b/server/mongodb-proxy.js
@@ -1,3 +1,4 @@
+'use strict';
 var express = require('express');
 var bodyParser = require('body-parser');
 ...
@@ -207,8 +208,8 @@
 function parseQuery(query, substitutions)
 {
-  doc = {}
-  queryErrors = []
+  let doc = {}
+  const queryErrors = []
 ...
@@ -259,7 +260,7 @@
-        docs = JSON.parse(args)
+        const docs = JSON.parse(args)
```

### Patch B — Replace `assert.equal` crash in template query

```diff
--- a/server/mongodb-proxy.js
+++ b/server/mongodb-proxy.js
@@ -451,8 +451,12 @@
         collection.aggregate(queryArgs.pipeline).toArray(function(err, result)
           {
-            assert.equal(err, null)
-
-            output = []
+            if (err) {
+              client.close();
+              return next(err);
+            }
+            const output = []
```

### Patch C — Connection pooling

```diff
--- a/server/mongodb-proxy.js
+++ b/server/mongodb-proxy.js
@@ -1,5 +1,15 @@
+'use strict';
 const MongoClient = require('mongodb').MongoClient;
+
+// Connection pool cache: url -> client promise
+const clientCache = {};
+
+function getClient(url) {
+  if (!clientCache[url]) {
+    clientCache[url] = MongoClient.connect(url, { useNewUrlParser: true, useUnifiedTopology: true });
+  }
+  return clientCache[url];
+}
```

---

## Priority Summary

| Priority | Issue | Section |
|----------|-------|---------|
| **Critical** | NoSQL injection via unvalidated aggregation pipeline | 2.1 |
| **Critical** | MongoDB credentials in plaintext request bodies | 2.2 |
| **Critical** | Wildcard CORS | 2.3 |
| **Critical** | Connection leaks + `assert` crash | 2.4, 2.5 |
| **High** | Node 6, MongoDB driver 3.x, Grunt/Babel 6 | 3.1–3.3 |
| **Medium** | Angular plugin deprecation, implicit globals, no linting, no proxy tests | 3.4–3.8 |
| **Low** | Unnecessary deps, lodash CVE, gitignore | 3.9–3.12 |

I recommend tackling the **Critical** items first (security hardening), then upgrading the runtime and driver, and finally migrating to the modern Grafana plugin SDK. Let me know if you'd like me to implement any of these fixes. 

Completed: *Compile findings & recommendations* (4/4)