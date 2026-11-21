# agent.md — Reglas técnicas del proyecto

Reglas de arquitectura, estilo y trabajo que se respetan en **todos** los repositorios del TPI.
El *qué* (modelos, endpoints, reglas de negocio) está en `spec.md`; este archivo define el *cómo*.

---

## 1. Reglas generales (todos los servicios)

1. **Secretos fuera del código.** Contraseñas, cadenas de conexión, `SECRET_KEY`, claves JWT y
   `SERVICE_TOKEN` se leen de un archivo `.env` que está en el `.gitignore`. Cada repo sube un
   `.env.example` con las variables y valores de ejemplo, nunca los reales. Si un secreto se sube por
   error, se genera uno nuevo: borrarlo no alcanza, porque queda en el historial de Git (Clase 8).
2. **Formato de error uniforme** `{"error": true, "mensaje": "..."}` en todas las respuestas de error
   (`spec.md`, sección 3), resuelto en **un único lugar** de cada servicio (manejador centralizado), no
   endpoint por endpoint.
3. **Arquitectura en capas:** la capa HTTP (rutas/vistas) no contiene lógica de negocio ni consultas;
   la lógica va en servicios y las consultas en repositorios (Clase 5).
4. **Llamadas entre servicios con timeout de 5 segundos** y manejo del error: si el otro servicio falla,
   se responde `503` o una respuesta degradada, nunca un `500` sin control.
5. **Borrado lógico:** ninguna operación de baja hace un `DELETE` físico en la base de datos.
6. **Nombres claros antes que comentarios.** El dominio se nombra en español (producto, proveedor,
   pedido), igual que en `spec.md`. Los comentarios explican el *porqué*, no el *qué*.
7. **Endpoint de salud** (`/api/health`) en cada servicio, sin autenticación.
8. **Puertos fijos:** Inventario 3000, Pedidos 8000, Pagos 8001, Frontend 5500.

---

## 2. Inventario — Node.js + Express + MongoDB

- **ES Modules** (`import`/`export`) con `"type": "module"` en `package.json`.
- **Estructura:**
  ```
  index.js                 → arranque: carga .env, conecta a MongoDB, levanta el servidor
  src/
    app.js                 → configuración de Express (middlewares y rutas); se exporta para los tests
    config/db.js           → conexión a MongoDB
    models/                → esquemas de Mongoose (validaciones del spec)
    repositories/          → consultas a la base (find, populate, paginación, $inc)
    controllers/           → reciben la petición, llaman al repositorio, arman la respuesta
    routes/                → definición de rutas
    middlewares/           → manejo centralizado de errores, autenticación JWT (TP 2), validaciones
  tests/
  ```
- Toda operación asíncrona va en `try/catch` y los errores se derivan con `next(error)` al middleware
  de errores.
- Las validaciones de formato viven en el esquema de Mongoose; las reglas que involucran otra colección
  (proveedor activo) van en el repositorio o el controlador.
- El descuento de stock es **atómico** (`findOneAndUpdate` con condición y `$inc`). No se usa el patrón
  "leer, restar y guardar".
- Scripts: `npm start` (producción, `node`), `npm run dev` (desarrollo, `nodemon`), `npm test`.
- Los tests usan **una base de datos propia** (variable `MONGO_URI_TEST`), nunca la de desarrollo:
  los tests borran colecciones.

## 3. Pedidos — Python + Django REST Framework + PostgreSQL

- Entorno virtual `venv/` (fuera de Git) y `requirements.txt` actualizado con `pip freeze` después de
  cada instalación (Clase 1).
- Configuración leída con `python-dotenv` (`load_dotenv()` + `os.getenv`), como en la Clase 1. En
  `settings.py` no hay ningún valor sensible escrito.
- **Estructura de la app `pedidos`:**
  ```
  models.py          → modelo Pedido
  serializers.py     → validación de entrada y formato de salida
  repositories.py    → consultas al ORM (select_related, filtros, orden estable)
  services.py        → lógica de negocio y llamadas HTTP a Inventario y Pagos (requests, timeout=5)
  views.py           → ModelViewSet / acciones: solo HTTP, permisos y códigos de estado
  exceptions.py      → manejador centralizado con el formato de error del spec
  urls.py
  tests/
  ```
- El manejador de excepciones sigue el de la Clase 4 (`custom_exception_handler`), adaptado a la clave
  `mensaje` que pide el spec.
- Paginación con orden estable (`order_by("-creado")`) y filtros con `django-filter` (Clase 5).
- Producción (TP 4/5): Daphne (ASGI) en lugar de `runserver`, `DEBUG=False`, `ALLOWED_HOSTS` desde el
  `.env`, WhiteNoise para los archivos estáticos, PostgreSQL en lugar de SQLite (Clase 12).
- Documentación OpenAPI con `drf-yasg` (cronograma, 18/11).

## 4. Pagos — Python + FastAPI

- Esquemas **Pydantic** para validar la entrada (`order_id: int > 0`, `monto: float > 0`) y la salida.
- Documentación OpenAPI automática en `/docs`.
- Manejadores de excepciones para `RequestValidationError` y `HTTPException` que devuelven el formato
  uniforme `{"error": true, "mensaje": ...}` con código `400`, en lugar del `422` que FastAPI usa por
  defecto.
- No tiene base de datos: es un simulador sin estado.
- Se ejecuta con `uvicorn` en el puerto 8001.

## 5. Frontend — HTML + CSS + JavaScript

- Sin frameworks ni librerías (enfoque de la Clase 13): `fetch` con `async/await`.
- Siempre se verifica `response.ok` después de un `fetch`, porque `fetch` no falla ante un `4xx`/`5xx`.
- Los datos que vienen de la API se muestran con `textContent`, **nunca** con `innerHTML` (defensa
  contra XSS, Clases 8 y 13).
- Las URLs de las APIs se definen en un único archivo de configuración.
- Se sirve por HTTP (`python -m http.server 5500` o Live Server), no abriendo el archivo con doble clic.
  El origen `http://localhost:5500` debe figurar en la política CORS de cada backend.

---

## 6. Flujo de trabajo con Git

- **`main` siempre funciona.** Nadie hace push directo a `main`: cada tarea se hace en una rama y entra
  por Pull Request revisado por otro integrante.
- Nombres de ramas: `feature/<tema>`, `fix/<tema>`, `docs/<tema>`.
- Mensajes de commit con el formato `tipo(alcance): descripción`, en español y en presente.
  Tipos: `feat`, `fix`, `docs`, `test`, `refactor`, `chore`.
  Ejemplo: `feat(pedidos): crea el modelo Pedido y su serializer`.
- Antes de cada `git add`, revisar `git status` y confirmar que no aparezcan `.env`, `venv/`,
  `node_modules/` ni `db.sqlite3`.
- Cada integrante configura su identidad real antes de commitear:
  `git config --global user.name "Nombre Apellido"`.
- Después de mergear cambios en un servicio, se actualiza el submódulo en `TPI_Ecommerce`
  (ver README del orquestador).
- Toda tarea se registra en el tablero del equipo antes de empezarla.

## 7. Calidad

- Cada servicio tiene tests automatizados de sus reglas de negocio (stock insuficiente, borrado lógico,
  validaciones, flujo de compra).
- Un PR no se mergea si rompe los tests existentes.
- Los tests se ejecutan automáticamente con GitHub Actions (TP 4/5).
