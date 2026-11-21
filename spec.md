# spec.md — Especificación del sistema

Desarrollo Guiado por Especificaciones (SDD): este archivo es la **fuente de verdad** sobre qué datos
maneja cada servicio y cómo se comunican. Si el código y este documento no coinciden, se corrige uno
de los dos con un Pull Request; nunca se deja la diferencia.

> Las secciones marcadas con **(TP 2)**, **(TP 3)** o **(TP 4/5)** describen lo que se implementa en esa
> entrega. Lo que no tiene marca corresponde al TP 1.

---

## 1. Mapa de servicios

| Servicio | Tecnología | Base de datos | Puerto | URL base |
|---|---|---|---|---|
| Inventario | Node.js + Express | MongoDB | 3000 | `http://localhost:3000/api` |
| Pedidos | Django REST Framework | PostgreSQL | 8000 | `http://localhost:8000/api` |
| Pagos | FastAPI | — | 8001 | `http://localhost:8001/api` |
| Frontend | HTML + CSS + JS | — | 5500 | `http://localhost:5500` |

Dentro de Docker Compose (TP 4/5) los servicios se llaman entre sí por el nombre del contenedor:
`http://inventario:3000/api`, `http://pedidos:8000/api`, `http://pagos:8001/api`.

```
                 ┌──────────────┐
                 │   Frontend   │
                 └──────┬───────┘
          HTTP + JWT    │       WebSocket (TP 3)
        ┌───────────────┼─────────────────┐
        ▼               ▼                 │
 ┌─────────────┐  ┌─────────────┐◄────────┘
 │ Inventario  │◄─│   Pedidos   │──────────►┌──────────┐
 │  (Express)  │  │  (Django)   │           │  Pagos   │
 │   MongoDB   │  │ PostgreSQL  │           │(FastAPI) │
 └─────────────┘  └─────────────┘           └──────────┘
   descontar / reponer stock          procesar pago
```

---

## 2. Modelos de datos

### 2.1 Proveedor — Inventario (MongoDB)

| Campo | Tipo | Restricciones | Descripción |
|---|---|---|---|
| `razonSocial` | String | Requerido, `trim`, mayúsculas | Nombre de la empresa proveedora |
| `cuit` | String | Requerido, único, 11 dígitos (`/^\d{11}$/`) | Identificador fiscal |
| `contacto.email` | String | Requerido, minúsculas | Correo de contacto |
| `contacto.telefono` | String | Opcional | Teléfono de contacto |
| `categorias` | [String] | Al menos una; enum `HARDWARE`, `SOFTWARE`, `INSUMOS`, `SERVICIOS` | Rubros que distribuye |
| `activo` | Boolean | Por defecto `true` | Borrado lógico |
| `createdAt`, `updatedAt` | Date | Automáticos (`timestamps`) | Auditoría |

### 2.2 Producto — Inventario (MongoDB)

| Campo | Tipo | Restricciones | Descripción |
|---|---|---|---|
| `codigoSKU` | String | Requerido, único, mayúsculas, formato `AAA-000` (`/^[A-Z]{3}-\d{3}$/`) | Código interno del artículo |
| `nombre` | String | Requerido, `trim` | Nombre comercial |
| `descripcion` | String | Opcional | Descripción libre |
| `precio` | Number | Requerido, mínimo 0 | Precio unitario de venta |
| `stock` | Number | Requerido, entero, mínimo 0 | Unidades disponibles |
| `especificaciones` | Map\<String\> | Opcional | Datos técnicos variables (ej. `memoria: "4GB"`) |
| `proveedor` | ObjectId → Proveedor | Requerido | Proveedor del producto |
| `activo` | Boolean | Por defecto `true` | Borrado lógico |
| `createdAt`, `updatedAt` | Date | Automáticos | Auditoría |

### 2.3 Pedido — Pedidos (PostgreSQL)

Un pedido corresponde a **un producto** con su cantidad.

| Campo | Tipo Django | Restricciones | Descripción |
|---|---|---|---|
| `id` | `BigAutoField` | Automático | Identificador del pedido |
| `cliente` | `ForeignKey(User)` | Requerido, `on_delete=PROTECT` | Usuario autenticado que compra (TP 2; en TP 1 puede ser nulo) |
| `producto_id` | `CharField(24)` | Requerido | `_id` del producto en MongoDB |
| `cantidad` | `PositiveIntegerField` | Requerido, mínimo 1 | Unidades solicitadas |
| `precio_unitario` | `DecimalField(12, 2)` | Lo informa Inventario | Precio al momento de la compra |
| `monto_total` | `DecimalField(12, 2)` | Calculado: `precio_unitario × cantidad` | Importe a cobrar |
| `estado` | `CharField` con `choices` | `PENDIENTE`, `PAGADO`, `FALLIDO`; por defecto `PENDIENTE` | Estado en el flujo de compra |
| `transaccion_id` | `CharField` | Nulo hasta que Pagos aprueba | Comprobante devuelto por Pagos |
| `activo` | `BooleanField` | Por defecto `True` | Borrado lógico |
| `creado`, `actualizado` | `DateTimeField` | `auto_now_add` / `auto_now` | Auditoría |

Se guarda `precio_unitario` (y no solo el total) porque el precio del producto puede cambiar después
de la compra: el pedido tiene que conservar el precio que se cobró.

### 2.4 Usuario — Pedidos (TP 2)

Se usa el modelo `User` que trae Django. El rol se determina así:

| Rol | Cómo se identifica | Puede |
|---|---|---|
| `ADMIN` | `is_staff = True` | Administrar proveedores y productos; ver todos los pedidos |
| `CLIENTE` | `is_staff = False` | Ver el catálogo; crear y ver **sus propios** pedidos |

---

## 3. Formato uniforme de respuestas de error

**Todos** los servicios responden los errores con esta estructura:

```json
{
  "error": true,
  "mensaje": "Descripción clara del motivo del fallo",
  "detalles": ["opcional: lista de errores de validación por campo"]
}
```

| Código | Cuándo |
|---|---|
| `400 Bad Request` | Datos inválidos, regla de negocio incumplida (ej. stock insuficiente) |
| `401 Unauthorized` | Falta el token o es inválido (TP 2) |
| `403 Forbidden` | El usuario no tiene el rol necesario (TP 2) |
| `404 Not Found` | El recurso no existe o está inactivo |
| `429 Too Many Requests` | Se superó el límite de peticiones (TP 2) |
| `500 Internal Server Error` | Error no previsto |
| `503 Service Unavailable` | Un servicio del que dependemos no responde o tardó demasiado |

---

## 4. Contratos de la API

### 4.1 Inventario (puerto 3000)

| Método | Ruta | Descripción | Rol (TP 2) |
|---|---|---|---|
| GET | `/api/health` | Estado del servicio | Público |
| GET | `/api/proveedores?page=&limit=&search=` | Lista paginada de proveedores activos; `search` filtra por razón social | ADMIN |
| GET | `/api/proveedores/:id` | Un proveedor | ADMIN |
| POST | `/api/proveedores` | Crea un proveedor | ADMIN |
| PUT | `/api/proveedores/:id` | Modifica un proveedor | ADMIN |
| DELETE | `/api/proveedores/:id` | Baja lógica (`activo: false`) | ADMIN |
| GET | `/api/productos?page=&limit=&search=` | Catálogo paginado de productos activos, con `populate('proveedor', 'razonSocial cuit')` | Público |
| GET | `/api/productos/:id` | Un producto con su proveedor y sus pedidos recientes (ver abajo) | Público |
| POST | `/api/productos` | Crea un producto | ADMIN |
| PUT | `/api/productos/:id` | Modifica un producto | ADMIN |
| DELETE | `/api/productos/:id` | Baja lógica | ADMIN |
| PATCH | `/api/productos/:id/descontar` | Descuenta stock (uso interno de Pedidos) | Servicio |
| PATCH | `/api/productos/:id/reponer` | Devuelve stock tras un pago fallido (uso interno de Pedidos) | Servicio |

**Respuesta de los listados:**

```json
{ "total": 42, "pagina": 1, "paginasTotales": 5, "resultados": [ ... ] }
```

**`PATCH /api/productos/:id/descontar`**

Petición:
```json
{ "cantidad": 2 }
```

- `200 OK` — hay stock suficiente. El descuento se hace **en una sola operación atómica**
  (`findOneAndUpdate` con la condición `stock >= cantidad` y `$inc`), para que dos pedidos simultáneos
  no puedan dejar el stock negativo. Devuelve el producto actualizado; Pedidos usa el campo `precio`:
  ```json
  { "_id": "665f1c...", "nombre": "Placa de Video RX 560", "precio": 185000, "stock": 13 }
  ```
- `400 Bad Request` — `cantidad` inválida, o stock insuficiente:
  ```json
  { "error": true, "mensaje": "Stock insuficiente. Disponible: 1" }
  ```
- `404 Not Found` — el producto no existe o está inactivo.

**`PATCH /api/productos/:id/reponer`**

Petición: `{ "cantidad": 2 }` → `200 OK` con el producto actualizado (suma la cantidad al stock).

**`GET /api/productos/:id` — integración con Pedidos y respuesta degradada**

Inventario consulta `GET /api/orders/?producto_id=:id` en Pedidos (timeout de 5 s) y agrega el
resultado. Si Pedidos no responde, igual devuelve `200` con los datos del producto y el estado
degradado; una falla de otro servicio no debe romper esta API:

```json
{
  "producto": { "_id": "665f1c...", "codigoSKU": "GPU-560", "stock": 15, "...": "..." },
  "integracionPedidos": {
    "estado": "DEGRADADO: el servicio de Pedidos no está disponible temporalmente",
    "pedidosRecientes": []
  }
}
```

### 4.2 Pedidos (puerto 8000)

| Método | Ruta | Descripción | Rol (TP 2) |
|---|---|---|---|
| GET | `/api/health/` | Estado del servicio | Público |
| POST | `/api/auth/register/` | Registro de un cliente (TP 2) | Público |
| POST | `/api/token/` | Login: devuelve `access` y `refresh` (TP 2) | Público |
| POST | `/api/token/refresh/` | Renueva el `access` (TP 2) | Público |
| GET | `/api/orders/?estado=&producto_id=` | Lista paginada. El CLIENTE ve solo los suyos; el ADMIN, todos | Autenticado |
| GET | `/api/orders/{id}/` | Detalle de un pedido | Dueño o ADMIN |
| POST | `/api/orders/` | Crea un pedido y ejecuta el flujo de compra (sección 5) | CLIENTE |
| — | `/graphql/` | Consultas GraphQL sobre los pedidos (TP 4/5) | Autenticado |
| WS | `/ws/pedidos/?token=<access>` | Notificaciones en tiempo real (TP 3) | Autenticado |

No hay `PUT` ni `DELETE` de pedidos: son registros financieros. El estado solo lo cambia el flujo de
compra.

**`POST /api/orders/`**

Petición:
```json
{ "producto_id": "665f1c2a9b1e8a0012345678", "cantidad": 2 }
```

Respuesta `201 Created` (el pedido se creó; su `estado` indica cómo terminó el cobro):
```json
{
  "id": 15,
  "producto_id": "665f1c2a9b1e8a0012345678",
  "cantidad": 2,
  "precio_unitario": "185000.00",
  "monto_total": "370000.00",
  "estado": "PAGADO",
  "transaccion_id": "b3f6c1e2-...",
  "creado": "2026-11-22T14:30:00Z"
}
```

### 4.3 Pagos (puerto 8001)

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/health` | Estado del servicio |
| POST | `/api/payments/process` | Simula el cobro |
| GET | `/docs` | Documentación OpenAPI automática de FastAPI |

**`POST /api/payments/process`**

Petición (validada con Pydantic):
```json
{ "order_id": 15, "monto": 370000.00 }
```

| Condición | Respuesta |
|---|---|
| `order_id` no es entero positivo, o `monto` no es mayor a 0 | `400` con el formato de error uniforme |
| `0 < monto <= 10.000.000` | `200` `{ "estado": "APROBADO", "transaccion_id": "<uuid>", "mensaje": "Pago aprobado" }` |
| `monto > 10.000.000` | `200` `{ "estado": "RECHAZADO", "transaccion_id": null, "mensaje": "El monto supera el límite autorizado" }` |

El caso `RECHAZADO` no figura en la planificación original. Se agrega para poder **demostrar** el
camino del pedido `FALLIDO` sin apagar el servicio de Pagos.

---

## 5. Flujo de compra (`POST /api/orders/`)

```
Cliente          Pedidos                         Inventario            Pagos
   │ POST /orders/  │                                │                    │
   │───────────────►│ 1. valida producto_id/cantidad │                    │
   │                │ 2. PATCH /productos/:id/descontar                   │
   │                │───────────────────────────────►│                    │
   │                │◄─────── 200 {precio} ──────────│                    │
   │                │ 3. crea Pedido PENDIENTE        │                    │
   │                │    monto_total = precio × cantidad                  │
   │                │ 4. POST /payments/process ─────────────────────────►│
   │                │◄──────────── 200 {APROBADO, transaccion_id} ────────│
   │                │ 5. Pedido → PAGADO                                  │
   │                │ 6. notificación WebSocket (TP 3)                    │
   │◄── 201 ────────│                                                     │
```

| Situación | Qué hace Pedidos | Respuesta al cliente |
|---|---|---|
| Datos inválidos | No llama a nadie | `400` |
| Inventario responde `400` (stock insuficiente) | **No crea el pedido** ni llama a Pagos | `400` con el mensaje de Inventario |
| Inventario responde `404` | No crea el pedido | `404` |
| Inventario no responde en 5 s | No crea el pedido | `503` |
| Pagos responde `APROBADO` | Pedido → `PAGADO`, guarda `transaccion_id` | `201` |
| Pagos responde `RECHAZADO` | Pedido → `FALLIDO`; llama a `/reponer` en Inventario | `201` (estado `FALLIDO`) |
| Pagos no responde en 5 s | Pedido → `FALLIDO`; llama a `/reponer` en Inventario | `201` (estado `FALLIDO`) |

El paso de **reponer** es una *compensación*: el stock ya se descontó en el paso 2, así que si el cobro
falla hay que devolverlo. Si la reposición también falla, se registra el error en el log para
corregirlo a mano.

---

## 6. Reglas de negocio (criterios de aceptación)

1. **Control estricto de stock:** ningún pedido se envía a cobro ni pasa a `PAGADO` si Inventario
   informa stock insuficiente.
2. **Descuento atómico:** el stock se verifica y descuenta en una sola operación de base de datos.
3. **Borrado lógico obligatorio:** ningún producto, proveedor ni pedido se elimina físicamente; una baja
   solo pone `activo` en `false`. Los listados muestran únicamente registros activos.
4. **Integridad de referencias:** no se puede crear ni modificar un producto con un proveedor
   inexistente o inactivo.
5. **Precio histórico:** el pedido conserva el `precio_unitario` informado por Inventario al momento
   de la compra.
6. **Formato uniforme de errores** en los tres servicios (sección 3).
7. **Timeouts:** toda llamada entre servicios tiene un tiempo máximo de 5 segundos.

---

## 7. Autenticación entre servicios (TP 2)

- **Emisor único de tokens:** Pedidos (Django + `djangorestframework-simplejwt`) es el único servicio
  que registra usuarios y emite tokens JWT. El token incluye el `user_id` y un claim `rol`
  (`ADMIN` o `CLIENTE`).
- **Verificación en Inventario:** Express valida el mismo token con la misma clave (`JWT_SIGNING_KEY`,
  algoritmo HS256) y lee el `rol` para autorizar las rutas de administración.
- **Llamadas de servicio a servicio:** los endpoints `descontar` y `reponer` no los usa el cliente, sino
  Pedidos. Se protegen con el header `X-Service-Token`, que lleva una clave compartida
  (`SERVICE_TOKEN`) definida en el `.env` de ambos servicios. La misma clave autoriza a Inventario a
  consultar `GET /api/orders/?producto_id=` en Pedidos (sección 4.1).
- **Por qué una clave compartida (HS256) y no un par de claves (RS256):** es el esquema que se vio en
  la Clase 6 y alcanza para un sistema donde todos los servicios son nuestros. Su limitación, también
  vista en clase, es que quien puede verificar tokens también podría fabricarlos; con servicios de
  terceros convendría RS256.
- **WebSocket:** el token se envía como parámetro de la URL (`?token=`), porque el navegador no permite
  agregar headers al conectarse (Clase 10).

---

## 8. Notificaciones en tiempo real (TP 3)

Pedidos usa Django Channels. Al conectarse a `/ws/pedidos/?token=<access>`:

- un **CLIENTE** se une al grupo `pedidos_usuario_<id>` (recibe solo sus pedidos);
- un **ADMIN** se une además al grupo `pedidos_admin` (recibe todos).

Cuando un pedido termina el flujo de compra se envía:

```json
{
  "tipo": "pedido_actualizado",
  "pedido": { "id": 15, "estado": "PAGADO", "monto_total": "370000.00", "producto_id": "665f1c..." }
}
```
