# Análisis interno de `bottle.py` (versión `0.14-dev`)

Guía para entender cómo funciona Bottle por dentro, con foco en los decoradores `@route`, `@get`, `@post`, etc. Los números de línea corresponden al fichero analizado (4598 líneas) y pueden variar en otras versiones.

---

## 1. Idea general

Bottle es un microframework **WSGI** de un solo fichero. Una aplicación WSGI es un objeto al que se llama así:

```python
app(environ, start_response)
```

- `environ`: diccionario con toda la información de la petición (método, ruta, cabeceras...).
- `start_response`: función para indicar código de estado y cabeceras.

Todo Bottle consiste en: **recibir `environ` → encontrar la función que corresponde a la URL → ejecutarla → convertir lo que devuelve en una respuesta WSGI.**

---

## 2. Mapa del fichero

| Zona | Líneas aprox. | Contenido |
|---|---|---|
| Utilidades y descriptores | 174-231 | `DictProperty`, `cached_property`, `lazy_attribute` |
| Excepciones de rutas | 233-265 | `BottleException`, `RouteError`... |
| **Router** | 267-475 | `Router.add`, `Router.match`, `Router.build` |
| **Route** | 477-591 | Objeto que une URL + método + función + plugins |
| **Bottle** | 593-1127 | La aplicación: `route`, `get`, `post`, `_handle`, `wsgi`... |
| Request | 1129-1591 | `BaseRequest` |
| Response | 1611-1898 | `BaseResponse` |
| `LocalRequest`/`LocalResponse` | 1900-1926 | Versiones thread-local |
| `HTTPResponse`/`HTTPError` | 1928-1965 | Respuestas usadas como excepciones |
| Plugins | 1967-2031 | `JSONPlugin`, `TemplatePlugin` |
| Estructuras de datos | 2079-2531 | `MultiDict`, `FormsDict`, `HeaderDict`, `WSGIHeaderDict`, `ConfigDict`, `AppStack` |
| Ficheros y recursos | 2532-2700 | `WSGIFileWrapper`, `_closeiter`, `ResourceManager`, `FileUpload` |
| **Atajos globales** | 3108-3125 | `route = make_default_app_wrapper('route')`... |
| Multipart | 3139-3386 | Parser de formularios multipart |
| Servidores | 3388-3810 | Adaptadores (`WSGIRefServer`, `GunicornServer`...) |
| `run()` | 3813 | Arranque del servidor |
| Plantillas | 3968 en adelante | `SimpleTemplate`, `StplParser`, etc. |

---

## 3. Orden recomendado de lectura

1. `Bottle.route` (859) y `Bottle.get`/`post`/... (911-925)
2. `Bottle.add_route` (850)
3. `make_default_app_wrapper` (3108)
4. `Router.add` (335) y `Router.match` (437)
5. `Route` (477), especialmente `_make_callback` (533)
6. `Bottle._handle` (956), `_cast` (999), `wsgi` (1075), `__call__` (1107)
7. Plugins: `Bottle.install` (788), `JSONPlugin` (1971), `TemplatePlugin` (2011)
8. `BaseRequest` (1129) y `BaseResponse` (1611)

**Se puede ignorar al principio:** las clases `*Server`, las plantillas, el parser multipart, `MultiDict`/`ConfigDict` en detalle.

---

## 4. Cómo funcionan los decoradores `@route`, `@get`, `@post`...

### 4.1 `Bottle.route` (línea ~859)

Firma:

```python
def route(self, path=None, method='GET', callback=None,
          name=None, apply=None, skip=None, **config):
```

Es una **fábrica de decoradores**: devuelve una función `decorator(callback)`.

Esquema simplificado:

```python
def route(self, path=None, method='GET', callback=None, ...):
    if callable(path): path, callback = None, path   # permite @route sin paréntesis
    plugins = makelist(apply)
    skiplist = makelist(skip)

    def decorator(callback):
        if isinstance(callback, str):
            callback = load(callback)                # permite pasar "modulo:funcion"
        for rule in makelist(path) or yieldroutes(callback):
            for verb in makelist(method):
                verb = verb.upper()
                route = Route(self, rule, verb, callback,
                              name=name, plugins=plugins,
                              skiplist=skiplist, **config)
                self.add_route(route)
        return callback                               # devuelve la función ORIGINAL

    return decorator(callback) if callback else decorator
```

Puntos clave:

- **El decorador no modifica la función**: la registra y la devuelve tal cual. Por eso `hola('mundo')` sigue funcionando como una función normal.
- `path` puede ser una lista (varias URLs para la misma función) y `method` también (varios verbos).
- Si no se da `path`, se genera desde la firma de la función con `yieldroutes(callback)`.
- Se crea **un objeto `Route` por cada combinación (ruta × método)**.
- `callable(path)` permite usar `@route` sin paréntesis: en ese caso `path` es en realidad la función.
- `**config` se guarda como configuración específica de la ruta y se pasa a los plugins.

### 4.2 `get`, `post`, `put`, `delete`, `patch` (líneas ~911-925)

Son simples atajos que cambian el valor por defecto de `method`:

```python
def get(self, path=None, method='GET', **options):
    return self.route(path, method, **options)

def post(self, path=None, method='POST', **options):
    return self.route(path, method, **options)
```

### 4.3 `add_route` (línea 850)

```python
def add_route(self, route):
    self.routes.append(route)                                   # lista de Route
    self.router.add(route.rule, route.method, route, name=route.name)  # registro en el Router
    if DEBUG: route.prepare()
```

Hace dos cosas: guarda la ruta en `self.routes` (lista) y la registra en el `Router` (estructura de búsqueda).

### 4.4 Atajos globales: `from bottle import get, route` (línea 3108)

```python
def make_default_app_wrapper(name):
    @functools.wraps(getattr(Bottle, name))
    def wrapper(*a, **ka):
        return getattr(app(), name)(*a, **ka)
    return wrapper

route = make_default_app_wrapper('route')
get   = make_default_app_wrapper('get')
post  = make_default_app_wrapper('post')
...
```

Cada función global (`route`, `get`, `post`, `put`, `delete`, `patch`, `error`, `mount`, `hook`, `install`, `uninstall`, `url`) **reenvía la llamada a la aplicación por defecto** (`app()`, que es el último elemento de un `AppStack`). Por eso `@get('/')` funciona sin haber creado ninguna instancia de `Bottle`.

---

## 5. El Router

### 5.1 Qué hace

Dada una petición (método + ruta), encuentra el objeto `Route` correcto y extrae los parámetros de la URL.

### 5.2 Estructuras internas (`Router.__init__`, ~línea 286)

| Atributo | Tipo | Uso |
|---|---|---|
| `rules` | `list` | Todas las reglas en orden de registro |
| `static` | `dict` anidado `{método: {path: (target, getargs)}}` | Rutas sin parámetros: búsqueda O(1) |
| `dyna_routes` | `dict` de `list` `{método: [reglas]}` | Rutas dinámicas pendientes de compilar |
| `dyna_regexes` | `dict` de `list` de tuplas `{método: [(regex_combinada, reglas)]}` | Rutas dinámicas compiladas |
| `builder` | `dict` `{regla_o_nombre: [(key, filtro), ...]}` | Construcción inversa de URLs (`get_url`) |
| `_groups` | `dict` `{(patrón, método): índice}` | Evita duplicar reglas dinámicas |
| `filters` | `dict` de funciones | Filtros `int`, `float`, `path`, `re`... |

### 5.3 Sintaxis de rutas

```
/hello/<name>            # comodín por defecto (un segmento)
/item/<id:int>           # filtro int
/price/<p:float>
/static/<ruta:path>      # coincide con barras
/user/<n:re:[a-z]+>      # expresión regular propia
```

### 5.4 `Router.add` (línea 335)

1. Recorre los tokens de la regla (`_itertokens`).
2. Si hay comodines, construye un patrón regex y una lista de filtros de conversión.
3. Guarda el *builder* (para construir URLs).
4. **Si la ruta es estática**: la guarda en `self.static[método][ruta] = (target, None)`.
5. **Si es dinámica**: compila su regex, añade la regla a `dyna_routes[método]` y llama a `_compile(método)`.

### 5.5 `Router._compile`

Combina varias reglas dinámicas del mismo método en **una sola regex grande** (con grupos alternativos, hasta un máximo de grupos por patrón). Así, al hacer `match`, se prueba una única expresión en lugar de una por ruta, y `match.lastindex` indica qué regla coincidió.

### 5.6 `Router.match` (línea 437)

Algoritmo:

1. Obtiene `verb` (método HTTP) y `path`.
2. Define los métodos a probar: `('PROXY', verb, 'ANY')`; para `HEAD` también prueba `GET`.
3. Para cada método:
   - Busca `path` en `self.static[método]` (acierto inmediato).
   - Si no, prueba las regex combinadas de `dyna_regexes[método]`.
4. Si no hay resultado, recoge en un `set` los métodos que **sí** aceptan esa ruta.
   - Si hay alguno → `HTTPError 405` con cabecera `Allow`.
   - Si no → `HTTPError 404`.

Devuelve `(Route, args)` donde `args` es un diccionario con los parámetros convertidos (por ejemplo `{'id': 42}`).

---

## 6. La clase `Route` (línea 477)

Une todos los datos de una ruta: aplicación, regla, método, callback, nombre, plugins propios y configuración.

Métodos importantes:

- `call` (línea 508): propiedad que devuelve el callback ya envuelto con los plugins (se prepara la primera vez).
- `prepare` (518): construye el callback final.
- `all_plugins` (522): plugins de la app + los de la ruta, respetando `skip`.
- **`_make_callback` (533)**:

```python
def _make_callback(self):
    callback = self.callback
    for plugin in self.all_plugins():
        if hasattr(plugin, 'apply'):
            callback = plugin.apply(callback, self)
        else:
            callback = plugin(callback)
        if callback is not self.callback:
            update_wrapper(callback, self.callback)
    return callback
```

Esto es **el segundo uso de decoradores en Bottle**: cada plugin recibe el callback y devuelve otro callback que lo envuelve (patrón decorador clásico). `update_wrapper` conserva nombre y docstring.

- `get_undecorated_callback` (544): recorre las clausuras para encontrar la función original bajo los decoradores.
- `get_callback_args` (565): inspecciona los argumentos de la función original.
- `get_config` (574): lee configuración específica de la ruta.

---

## 7. Ciclo de vida de una petición

```
Servidor (wsgiref, gunicorn...)
        │  app(environ, start_response)
        ▼
Bottle.__call__ (1107)  →  Bottle.wsgi (1075)
        │
        ▼
Bottle._handle (956)
   1. guarda la ruta original en environ['bottle.raw_path']
   2. request.bind(environ);  response.bind()
   3. trigger_hook('before_request')
   4. route, args = self.router.match(environ)
   5. out = route.call(**args)          ← ejecuta tu función (con plugins)
   6. si es HTTPResponse → out.apply(response)
   7. trigger_hook('after_request')
   8. si hay excepción no controlada → HTTPError 500
        │
        ▼
Bottle._cast (999)
   convierte `out` (str, bytes, dict, list, generador, HTTPResponse...)
   en un iterable de bytes y rellena cabeceras
        │
        ▼
start_response(status, headers) + cuerpo
```

Notas:

- Las excepciones `HTTPResponse` (incluido `HTTPError`) se usan como **mecanismo de control de flujo**: `abort(404)`, `redirect(...)` las lanzan y `_handle` las captura.
- Si `catchall` está activo, cualquier excepción genérica se convierte en un 500 y el traceback se escribe en `wsgi.errors`.
- `_cast` es la función que decide qué hacer según el tipo devuelto:
  - `dict` → lo trata `JSONPlugin` (a nivel de plugin).
  - `str` → se codifica a bytes.
  - `bytes`, listas de bytes, generadores → se iteran.
  - Objetos tipo fichero → `WSGIFileWrapper`.

---

## 8. Plugins, hooks y errores

### 8.1 Plugins

- `Bottle.install(plugin)` (788) añade el plugin a `self.plugins`.
- Un plugin es un objeto con método `apply(callback, route)` (o simplemente un callable decorador).
- En el constructor de `Bottle` se instalan por defecto:
  - **`JSONPlugin`** (1971): convierte `dict` devuelto en JSON y ajusta `Content-Type`.
  - **`TemplatePlugin`** (2011): soporta `@view` / `@template` sobre rutas.

### 8.2 Hooks

- `self._hooks` es un diccionario `{nombre: [funciones]}`.
- `@app.hook('before_request')` / `'after_request'` / `'config'`, etc.
- `trigger_hook(nombre, ...)` ejecuta todas las funciones registradas (se itera sobre una copia de la lista).

### 8.3 Manejo de errores

- `self.error_handler` es un `dict` `{código: función}`.
- `@app.error(404)` registra el manejador.
- `default_error_handler` genera la página HTML por defecto.

### 8.4 Aplicaciones montadas

- `Bottle.mount(prefix, app)` (748) permite componer aplicaciones bajo un prefijo.
- Se registra en `self._mounts`.

---

## 9. Estructuras de datos usadas

### 9.1 Estructuras estándar de Python

| Estructura | Dónde | Para qué |
|---|---|---|
| `list` | `Bottle.routes`, `Bottle.plugins`, `Router.rules`, `_mounts`, listas de hooks | Mantener **orden** |
| `dict` | `Router.static`, `builder`, `error_handler`, `_hooks`, filtros | **Búsqueda rápida** |
| `tuple` | `(target, getargs)`, `(key, filtro)`, `(regex, reglas)` | Valores inmutables agrupados |
| `set` | `Router.match` (métodos permitidos para el 405) | Acumular sin duplicados |
| Generadores | `_itertokens`, `yieldroutes`, respuestas de `_cast` | Iteración perezosa |
| Closures | `getargs`, `decorator`, `make_default_app_wrapper` | Guardar estado sin clases |

### 9.2 Clases propias

| Clase | Línea | Base | Descripción |
|---|---|---|---|
| `AppStack` | 2509 | `list` | **Pila** de aplicaciones; `app()` devuelve `self[-1]`; `push()` añade |
| `MultiDict` | 2079 | `DictMixin` | Clave → **varios valores**. `.get()` devuelve el último; `.getall()` todos |
| `FormsDict` | 2157 | `MultiDict` | Añade acceso por atributo y decodificación unicode (`request.forms.nombre`) |
| `HeaderDict` | 2187 | `MultiDict` | Cabeceras HTTP con claves normalizadas (sin distinguir mayúsculas) |
| `WSGIHeaderDict` | 2225 | `DictMixin` | **Vista** sobre `environ`: traduce `HTTP_USER_AGENT` ⇄ `User-Agent` sin copiar |
| `ConfigDict` | 2272 | `dict` | Config jerárquica (`'json.enable'`), listeners de cambios y *overlays* |

### 9.3 Descriptores y propiedades perezosas

- **`DictProperty`** (174): guarda atributos dentro de un diccionario (típicamente `environ`). El objeto `request` apenas tiene estado propio.
- **`cached_property`** (200): calcula el valor una vez y lo cachea.
- **`lazy_attribute`** (215): igual, pero lo sustituye en la clase.

### 9.4 Estado thread-local

`LocalRequest` (1900) y `LocalResponse` (1910) guardan su estado por hilo, de modo que `request` y `response` son globales seguros en entornos con varios hilos.

---

## 10. Experimentos para ver el funcionamiento

### 10.1 Inspeccionar qué registra un decorador

```python
import bottle
bottle.DEBUG = True

@bottle.get('/hola/<nombre>')
def hola(nombre):
    return f'Hola {nombre}'

app = bottle.default_app()
print(app.routes)
r = app.routes[0]
print(r.rule, r.method, r.callback)
print(hola('mundo'))            # la función original sigue intacta
```

### 10.2 Ver las estructuras del Router

```python
import bottle

app = bottle.Bottle()

@app.get('/about')
def about(): return 'about'

@app.get('/user/<id:int>')
def user(id): return str(id)

print(app.router.static)
print(app.router.dyna_routes)
print(app.router.dyna_regexes)
print(app.router.builder)
```

Deberías ver `/about` en `static` y `/user/<id:int>` en `dyna_regexes`.

### 10.3 Probar `match` sin servidor

```python
env = {'REQUEST_METHOD': 'GET', 'PATH_INFO': '/user/42'}
route, args = app.router.match(env)
print(route, args)      # args == {'id': 42}
```

### 10.4 Simular una petición WSGI completa

```python
from wsgiref.util import setup_testing_defaults

environ = {}
setup_testing_defaults(environ)
environ['PATH_INFO'] = '/user/42'
environ['REQUEST_METHOD'] = 'GET'

def start_response(status, headers):
    print(status, headers)

print(b''.join(app(environ, start_response)))
```

### 10.5 Depurar el flujo

- Pon `breakpoint()` dentro de `Bottle._handle` y `Router.match`.
- Esta versión incluye llamadas a `_trace(...)` en `_handle`, `Router.match` y `Bottle.route/add_route`, útiles para seguir el recorrido.

### 10.6 Ver cómo actúan los plugins

```python
def mi_plugin(callback):
    def wrapper(*a, **kw):
        print('antes')
        res = callback(*a, **kw)
        print('después')
        return res
    return wrapper

app.install(mi_plugin)
```

Luego mira `Route._make_callback` para ver cómo se encadena con `JSONPlugin` y `TemplatePlugin`.

---

## 11. Resumen mental

1. `@get('/x')` → `make_default_app_wrapper` → `app().get(...)` → `route(...)`.
2. `route` devuelve un **decorador** que crea objetos `Route` y los registra en `Bottle.routes` y en `Router`.
3. El `Router` separa rutas **estáticas** (dict) y **dinámicas** (regex combinadas).
4. En cada petición, `_handle` hace `match` → `route.call(**args)` → `_cast`.
5. `route.call` es tu función **envuelta por plugins** (`_make_callback`).
6. Las estructuras son sencillas: **dicts para buscar, listas para ordenar**, y subclases mínimas solo donde hace falta (`MultiDict`, `ConfigDict`, `AppStack`...).

---

## 12. Siguientes pasos sugeridos

- Leer a fondo `Router.add` y `Router._compile` (cómo `<id:int>` se convierte en regex y filtro).
- Escribir un plugin propio y seguir cómo `Route._make_callback` lo aplica.
- Estudiar `BaseRequest` (`query`, `forms`, `json`, `files`) y `BaseResponse` (cabeceras y cookies).
- Implementar un mini-framework de ~50 líneas con la misma idea (dict de rutas + decorador + WSGI) para consolidar lo aprendido.