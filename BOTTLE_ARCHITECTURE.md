# Bottle Framework Architecture - Complete Explanation

Bottle is a **single-file WSGI micro-framework** (~4,500 lines) with zero dependencies beyond Python's standard library.

---

## 1. Request/Response Cycle (Core Flow)

```
HTTP Request → WSGI Server → Bottle.wsgi() → _handle() → Router.match() → Route.call() → Response
```

### Entry Point: `Bottle.wsgi(environ, start_response)` (line 1059)

Called by any WSGI server (gunicorn, uwsgi, wsgiref, etc.). Delegates to `_handle(environ)` which does the real work.

### Request Binding (lines 947-949)

```python
environ['bottle.app'] = self
request.bind(environ)    # Thread-local request.environ = environ
response.bind()          # Thread-local response init
```

### Routing: `Router.match(environ)` (line 430)

1. Extracts `REQUEST_METHOD` + `PATH_INFO`
2. Checks **static routes** first (dict lookup - O(1))
3. Then **dynamic routes** (combined regex with named groups)
4. Returns `(target, url_args)` or raises `HTTPError(404/405)`

### Route Execution: `Route.call(**args)` (line 498)

- Applies all plugins via `_make_callback()` (chain of decorators)
- Executes the final wrapped callback with URL args

### Response Casting: `Bottle._cast(out)` (line 984)

Converts return values to WSGI-compatible iterables:

| Return Type | Handling |
|-------------|----------|
| `dict` | → JSON (via `JSONPlugin`) |
| `str` | → encoded bytes |
| `bytes` | → direct |
| `HTTPResponse` | → apply headers/status |
| file-like | → `wsgi.file_wrapper` or `WSGIFileWrapper` |
| iterable | → peek first item, encode if needed |

---

## 2. Thread-Local Architecture

```python
# Global singletons (lines 1884-1905)
request = LocalRequest()   # threading.local() per thread
response = LocalResponse() # threading.local() per thread
```

Each thread gets its own `request.environ` / `response._headers` / etc. This is why you can do:

```python
@route('/')
def index():
    return request.query.get('name')  # Works magically per-request
```

---

## 3. Routing System

### Route Definition (line 847)

```python
@app.route('/wiki/<page>')
def show(page):
    return page
```

### Pattern Parsing: `Router._itertokens()` (line 308)

Tokens: `:name` (legacy) or `<name>`, `<name:filter>`, `<name:filter:config>`

- `filter` → regex pattern + to_python + to_url converters
- Built-in: `int`, `float`, `path`, `re` (default)

### Regex Compilation (line 363)

```python
pattern = '(?P<page>[^/]+)'  # for <page>
re_pattern = re.compile('^(?P<page>[^/]+)$')
```

### Matching Optimization (line 405)

- Groups routes by method
- Combines up to 99 routes into single regex: `(^pat1$)|(^pat2$)|...`
- Uses `match.lastindex` to identify which route matched

---

## 4. Plugin System

### Installation (line 778)

```python
app.install(JSONPlugin())  # Adds to app.plugins list
```

### Application (lines 523-532)

```python
def _make_callback(self):
    callback = self.callback
    for plugin in self.all_plugins():  # app.plugins + route.plugins - skiplist
        if hasattr(plugin, 'apply'):
            callback = plugin.apply(callback, self)
        else:
            callback = plugin(callback)
    return callback
```

### Example: JSONPlugin (lines 1955-1992)

```python
def apply(self, callback, route):
    def wrapper(*a, **ka):
        rv = callback(*a, **ka)
        if isinstance(rv, dict):
            response.content_type = 'application/json'
            return json_dumps(rv)
        return rv
    return wrapper
```

---

## 5. Configuration System: `ConfigDict` (line 2256)

Three-layer overlay:

```
Route.config (overlay) → App.config (overlay) → Global config (base)
```

```python
app.config['debug'] = True           # App level
route.config['template'] = 'my.tpl'  # Route level (inherits app config)
```

Features: namespaces (`db.host`), validators, meta-data, change listeners.

---

## 6. Template Engine

### Built-in: `SimpleTemplate` (line 4110)

Syntax: `{{ var }}`, `% if condition:`, `% for x in y:`, `% include('sub')`

```python
# Compiles template to Python code, caches compiled code object
code = parser.translate()  # → Python source
co = compile(code, filename, 'exec')
exec(co, env)              # env has _printlist, include, rebase, etc.
```

### Adapters (lines 4038-4098): Mako, Jinja2, Cheetah via `TemplatePlugin`

---

## 6. Server Adapters (lines 3372-3752)

18+ adapters implementing `ServerAdapter.run(handler)`:

- `WSGIRefServer` (stdlib)
- `WaitressServer`, `GunicornServer`, `CherootServer`
- `GeventServer`, `EventletServer` (async)
- `AiohttpServer` (asyncio)
- `AutoServer` (tries all until one imports)

```python
run(app, server='waitress', host='0.0.0.0', port=8080)
```

---

## 7. Key Utilities

| Utility | Purpose |
|---------|---------|
| `static_file()` | Secure file serving with ETag, Range, 304 |
| `redirect()` | 302/303 with `Location` header |
| `abort()` | Raise `HTTPError` |
| `auth_basic()` | HTTP Basic Auth decorator |
| `hook()` | `before_request`, `after_request`, `app_reset` |
| `mount()` | Sub-app mounting (WSGI or Bottle) |

---

## 8. Startup Flow

```python
run(app)  # line 3797
    → load_app() if string
    → install plugins
    → create ServerAdapter
    → server.run(app)  # blocks forever
    → (optional) FileCheckerThread for auto-reload
```

---

## 9. Summary Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        WSGI Server                          │
└──────────────────────────┬──────────────────────────────────┘
                           │ environ, start_response
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Bottle.wsgi() → _handle()                                  │
│   ├─ request.bind(environ)    ← Thread-local                │
│   ├─ response.bind()          ← Thread-local                │
│   ├─ trigger_hook('before_request')                         │
│   ├─ router.match(environ) → (Route, url_args)              │
│   ├─ route.call(**url_args) → plugins → callback            │
│   ├─ _cast(result) → WSGI iterable                          │
│   ├─ trigger_hook('after_request')                          │
│   └─ start_response(status, headers)                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Design Principles

1. **Single file** - Easy to vendor, deploy, understand
2. **Zero dependencies** - Only Python stdlib required
3. **Thread-local request/response** - Simple global-like API, thread-safe
4. **Plugin architecture** - Extensible without modifying core
5. **WSGI native** - Works with any WSGI server
6. **Configuration overlays** - Route → App → Global inheritance
7. **Auto-reloader** - Development-friendly file watching

---

*Generated from analysis of `bottle.py` (v0.14-dev)*