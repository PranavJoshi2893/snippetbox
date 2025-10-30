# Snippetbox

Snippetbox is a simple Go web server built using the `net/http` package.
It demonstrates how handlers, routers, and the built-in web server work together — and how `ServeMux` evolved before and after Go 1.22.

---

## Overview

Go’s standard library provides everything needed to create a web server.
When the server receives an HTTP request, it passes it to a **ServeMux** (router), which checks the request path and calls the appropriate handler.

No external web server (like Nginx or Apache) is required.

---

## Components

**Handler**
Executes the application’s business logic — processes incoming requests and writes responses.

**Router (ServeMux)**
Maps URL patterns to their corresponding handlers. When a request comes in, ServeMux finds the correct handler based on the request path.

**Web Server**
The built-in Go web server listens for incoming HTTP requests and passes them to the ServeMux.

---

## Using ServeMux

You have two options when registering routes:

1. **Create a new ServeMux instance** *(Recommended for production; keeps routes isolated)*

   ```go
   mux := http.NewServeMux()
   mux.HandleFunc("/", home)
   http.ListenAndServe(":4000", mux)
   ```

2. **Use the default ServeMux (`http.DefaultServeMux`)**

   * Shorter code, but **not recommended for production**.
   * `http.HandleFunc` and `http.Handle` register routes on a global variable.
   * Any package can access or modify it, which can compromise application safety.

   ```go
   http.HandleFunc("/", home) // uses http.DefaultServeMux
   http.ListenAndServe(":4000", nil)
   ```

---

## URL Patterns: Fixed vs Subtree

Go’s ServeMux supports two types of URL patterns:

* **Fixed paths**

  * Do not end with a trailing slash.
  * Match only when the request path exactly matches the pattern.
  * Example: `/snippet/view` matches only `/snippet/view`.

* **Subtree paths**

  * End with a trailing slash.
  * Match when the start of the request path matches the pattern.
  * Example: `/static/` matches `/static/`, `/static/css/`, `/static/js/`, etc.

* `/` is a **subtree path** → acts as a catch-all.

---

## Handling `/` Without Catch-All Behavior

If you want a handler for `/` to respond **only** to the root path:

```go
func home(w http.ResponseWriter, r *http.Request) {
    if r.URL.Path != "/" {
        http.NotFound(w, r)
        return
    }
    w.Write([]byte("Hello from Snippetbox"))
}
```

* Ensures `/` handler responds **only** to `/`.
* All other unmatched paths return 404.

---

## Automatic URL Path Sanitization

Go’s ServeMux automatically **cleans request paths** before matching them:

* Removes `.` (current directory) and `..` (parent directory) elements.
* Replaces multiple consecutive slashes `//` with a single slash `/`.
* If the request path is not clean, Go sends a **301 Permanent Redirect** to the sanitized path.

**Example:**

```
Request:  /foo/bar/..//baz
Sanitized: /foo/baz
Redirect:  301 Permanent Redirect to /foo/baz
```

This ensures:

* Handlers always receive **consistent, canonical paths**.
* Reduces the chance of duplicate routes and directory traversal issues.

---

## Subtree Path Trailing Slash Redirect

If a **subtree path** is registered with a trailing slash, but the request **omits the slash**, Go automatically sends a **301 Permanent Redirect** to the trailing-slash version.

**Example:**

```
Registered subtree path: /foo/
Request: /foo
Redirect: 301 Permanent Redirect → /foo/
```

This ensures canonical URLs and correct resolution of relative paths within the subtree.

---

## ServeMux Features Before and After Go 1.22

### 🕰️ **Before Go 1.22**

Go’s built-in `http.ServeMux` was intentionally lightweight:

* Didn’t support routing based on HTTP **methods** (e.g., GET vs POST).
* Didn’t support **URL parameters** or path variables (like `/snippet/{id}`).
* Didn’t support **regexp-based patterns**.
* Matched routes strictly by string prefix and required manual logic inside handlers for method checks or path parsing.

#### 🔧 Manual Method Checking (Before 1.22)

```go
if r.Method != http.MethodPost {
    w.WriteHeader(http.StatusMethodNotAllowed)
    w.Write([]byte("Method Not Allowed"))
    return
}
```

* The ServeMux didn’t differentiate between methods — all requests for a path went to the same handler.
* Developers had to manually block unwanted methods in every handler.

---

### 🚀 **After Go 1.22 (Go 1.22 – 1.25)**

New features introduced include:

* **Method-based routing**: `"GET /snippet"` or `"POST /snippet/create"`.
* **URL variables (wildcard segments)**: `"/snippet/{id}"` and access via `r.PathValue("id")`.
* **Automatic sanitization** continues as before (clean `.` / `..`, collapsing `//`, trailing slashes).

**Example:**

```go
mux.HandleFunc("GET /snippet/{id}", viewSnippet)
mux.HandleFunc("POST /snippet/create", createSnippet)
```

---

### ⚖️ **Still Limited Compared to Frameworks**

Even with these improvements, `ServeMux` remains minimal by design. It **still doesn’t support**:

* Full **regexp-based route patterns**
* Advanced **middleware chaining** or **route grouping**
* Built-in request context helpers or advanced parameter binding

For complex routing, developers often use:

* [Chi](https://github.com/go-chi/chi)
* [Gorilla Mux](https://github.com/gorilla/mux)
* [Echo](https://echo.labstack.com/)
* [Fiber](https://gofiber.io/)

---

### 💡 **Summary**

| Version            | Supported Features                                                   | Missing Features                                        |
| ------------------ | -------------------------------------------------------------------- | ------------------------------------------------------- |
| **Before Go 1.22** | Basic string-based routing; manual method checking (`405`)           | Built-in method routing, path variables, regex patterns |
| **Go 1.22 +**      | Method routing (`"GET /path"`), URL variables (`{id}`), sanitization | Regex routing, advanced middleware, route groups        |

---

## Quick Tips

* **ServeMux vs DefaultServeMux** — custom is safer; default is global and risky.
* **Fixed vs Subtree paths** — exact vs prefix matching.
* **Catch-all `/`** — restrict with `r.URL.Path` check.
* **404 handling** — `http.NotFound` for unmatched paths.
* **Automatic sanitization** — cleans `.` / `..` and repeated slashes.
* **Subtree redirect** — `/foo` → `/foo/`.
* **Method & variable routing (Go 1.22 +)** — `"GET /foo/{id}"` + `r.PathValue("id")`.
* **Handler signature** — `func(w http.ResponseWriter, r *http.Request)`.
