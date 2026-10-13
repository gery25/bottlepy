# Arquitectura del Framework Bottle - Explicación Completa

Bottle es un **micro-framework WSGI en un solo archivo** (~4,500 líneas) con cero dependencias más allá de la librería estándar de Python.

---

## 1. Ciclo Request/Response (Flujo Principal)

```
Petición HTTP → Servidor WSGI → Bottle.wsgi() → _handle() → Router.match() → Route.call() → Respuesta
```

### Punto de Entrada: `Bottle.wsgi(environ, start_response)` (línea 1059)

Llamado por cualquier servidor WSGI (gunicorn, uwsgi, wsgiref, etc.). Delega en `_handle(environ)` que hace el trabajo real.

### Enlace de Request (líneas 947-949)

```python
environ['bottle.app'] = self
request.bind(environ)    # Thread-local request.environ = environ
response.bind()          # Inicialización thread-local response
```

### Routing: `Router.match(environ)` (línea 430)

1. Extrae `REQUEST_METHOD` + `PATH_INFO`
2. Verifica **rutas estáticas** primero (búsqueda en dict - O(1))
3. Luego **rutas dinámicas** (regex combinado con grupos nombrados)
4. Retorna `(target, url_args)` o lanza `HTTPError(404/405)`

### Ejecución de Ruta: `Route.call(**args)` (línea 498)

- Aplica todos los plugins vía `_make_callback()` (cadena de decoradores)
- Ejecuta el callback final envuelto con los argumentos de la URL

### Conversión de Respuesta: `Bottle._cast(out)` (línea 984)

Convierte valores de retorno a iterables compatibles WSGI:

| Tipo de Retorno | Manejo |
|----------------|--------|
| `dict` | → JSON (vía `JSONPlugin`) |
| `str` | → bytes codificados |
| `bytes` | → directo |
| `HTTPResponse` | → aplicar headers/status |
| file-like | → `wsgi.file_wrapper` o `WSGIFileWrapper` |
| iterable | → inspeccionar primer item, codificar si necesario |

---

## 2. Arquitectura Thread-Local

```python
# Singletons globales (líneas 1884-1905)
request = LocalRequest()   # threading.local() por hilo
response = LocalResponse() # threading.local() por hilo
```

Cada hilo obtiene su propio `request.environ` / `response._headers` / etc. Por esto funciona:

```python
@route('/')
def index():
    return request.query.get('name')  # Funciona mágicamente por request
```

---

## 3. Sistema de Routing

### Definición de Rutas (línea 847)

```python
@app.route('/wiki/<page>')
def show(page):
    return page
```

### Parseo de Patrones: `Router._itertokens()` (línea 308)

Tokens: `:name` (legacy) o `<name>`, `<name:filtro>`, `<name:filtro:config>`

- `filtro` → patrón regex + conversores to_python + to_url
- Integrados: `int`, `float`, `path`, `re` (default)

### Compilación Regex (línea 363)

```python
pattern = '(?P<page>[^/]+)'  # para <page>
re_pattern = re.compile('^(?P<page>[^/]+)$')
```

### Optimización de Matching (línea 405)

- Agrupa rutas por método
- Combina hasta 99 rutas en un solo regex: `(^pat1$)|(^pat2$)|...`
- Usa `match.lastindex` para identificar qué ruta coincidió

---

## 4. Sistema de Plugins

### Instalación (línea 778)

```python
app.install(JSONPlugin())  # Añade a app.plugins list
```

### Aplicación (líneas 523-532)

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

### Ejemplo: JSONPlugin (líneas 1955-1992)

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

## 5. Sistema de Configuración: `ConfigDict` (línea 2256)

Overlay de tres capas:

```
Route.config (overlay) → App.config (overlay) → Global config (base)
```

```python
app.config['debug'] = True           # Nivel app
route.config['template'] = 'my.tpl'  # Nivel ruta (hereda config app)
```

Características: namespaces (`db.host`), validadores, meta-datos, listeners de cambios.

---

## 6. Motor de Templates

### Integrado: `SimpleTemplate` (línea 4110)

Sintaxis: `{{ var }}`, `% if condition:`, `% for x in y:`, `% include('sub')`

```python
# Compila template a código Python, cachea code object compilado
code = parser.translate()  # → código fuente Python
co = compile(code, filename, 'exec')
exec(co, env)              # env tiene _printlist, include, rebase, etc.
```

### Adaptadores (líneas 4038-4098): Mako, Jinja2, Cheetah vía `TemplatePlugin`

---

## 7. Adaptadores de Servidor (líneas 3372-3752)

18+ adaptadores implementando `ServerAdapter.run(handler)`:

- `WSGIRefServer` (stdlib)
- `WaitressServer`, `GunicornServer`, `CherootServer`
- `GeventServer`, `EventletServer` (async)
- `AiohttpServer` (asyncio)
- `AutoServer` (prueba todos hasta que uno importe)

```python
run(app, server='waitress', host='0.0.0.0', port=8080)
```

---

## 8. Utilidades Clave

| Utilidad | Propósito |
|----------|-----------|
| `static_file()` | Servir archivos seguros con ETag, Range, 304 |
| `redirect()` | 302/303 con header `Location` |
| `abort()` | Lanzar `HTTPError` |
| `auth_basic()` | Decorador HTTP Basic Auth |
| `hook()` | `before_request`, `after_request`, `app_reset` |
| `mount()` | Montaje de sub-apps (WSGI o Bottle) |

---

## 9. Flujo de Inicio

```python
run(app)  # línea 3797
    → load_app() si string
    → instalar plugins
    → crear ServerAdapter
    → server.run(app)  # bloquea indefinidamente
    → (opcional) FileCheckerThread para auto-reload
```

---

## 10. Diagrama Resumen

```
┌─────────────────────────────────────────────────────────────┐
│                        Servidor WSGI                        │
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
│   ├─ _cast(result) → iterable WSGI                          │
│   ├─ trigger_hook('after_request')                          │
│   └─ start_response(status, headers)                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Principios de Diseño Clave

1. **Un solo archivo** - Fácil de vendor, deployar, entender
2. **Cero dependencias** - Solo librería estándar de Python requerida
3. **Request/Response thread-local** - API simple tipo global, thread-safe
4. **Arquitectura de plugins** - Extensible sin modificar el core
5. **WSGI nativo** - Funciona con cualquier servidor WSGI
6. **Configuration overlays** - Herencia Route → App → Global
7. **Auto-reloader** - File watching amigable para desarrollo

---

*Generado desde análisis de `bottle.py` (v0.14-dev)*