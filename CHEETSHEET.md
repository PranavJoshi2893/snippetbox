# 📝 Snippetbox Go ServeMux Cheat Sheet

## 1. Handler Basics

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    // write response
    w.Write([]byte("Hello"))
}
```

* `w http.ResponseWriter` → used to send response
* `r *http.Request` → request info (path, method, headers)

**Important:**

* `w.WriteHeader(statusCode)` → sends status **once**
* `w.Write([]byte("msg"))` → sends response body
* If no `WriteHeader` → first `Write()` sends 200 OK automatically

---

## 2. Creating a Router (ServeMux)

```go
// New ServeMux (recommended)
mux := http.NewServeMux()
mux.HandleFunc("/path", handler)
http.ListenAndServe(":4000", mux)

// Default ServeMux (global, not recommended)
http.HandleFunc("/path", handler)
http.ListenAndServe(":4000", nil)
```

**Note:** Default ServeMux is a **global variable**, unsafe in production.

---

## 3. URL Patterns

| Type       | Ends with `/` | Match Behavior                     | Example         |
| ---------- | ------------- | ---------------------------------- | --------------- |
| Fixed Path | ❌             | Exact match only                   | `/snippet/view` |
| Subtree    | ✅             | Prefix match (acts like catch-all) | `/static/`, `/` |

**Catch-all `/`** → restrict to root with:

```go
if r.URL.Path != "/" {
    http.NotFound(w, r)
    return
}
```

---

## 4. Automatic Path Sanitization

* Removes `.` and `..`
* Collapses `//` to `/`
* Redirects to clean path with **301 Permanent Redirect**

**Example:**

```
Request: /foo/bar/..//baz
Redirect: 301 → /foo/baz
```

* Subtree missing trailing slash → automatic redirect

```
Registered: /foo/
Request: /foo
Redirect: 301 → /foo/
```

---

## 5. HTTP Method Handling

### Before Go 1.22

* Must manually check `r.Method`:

```go
if r.Method != http.MethodPost {
    w.WriteHeader(http.StatusMethodNotAllowed)
    w.Write([]byte("Method Not Allowed"))
    return
}
```

### Go 1.22+

* Method-aware routing supported:

```go
mux.HandleFunc("GET /snippet/{id}", viewSnippet)
mux.HandleFunc("POST /snippet/create", createSnippet)
```

---

## 6. Handler Tips

* Always check `r.URL.Path` if handler is supposed to serve **specific paths**
* Use `http.NotFound(w, r)` for 404
* Only call `w.WriteHeader()` **once**
* Use `w.Write([]byte(...))` after `WriteHeader` (or default 200 OK)

---

## 7. ServeMux Limitations

* No regex patterns (still)
* No built-in middleware chaining
* Limited compared to frameworks (Chi, Echo, Fiber)
* Before Go 1.22 → must manually check methods and parse path variables

---

## 8. Quick Mental Recap

**Think of routes as:**

* `/snippet/view` → fixed path → exact match
* `/static/` → subtree → prefix match → can serve multiple files
* `/` → catch-all → check `r.URL.Path` to restrict
* Clean path automatically → Go redirects for `.`, `..`, `//`, and missing `/`
