# TPI Programación IV — Plataforma de E-commerce con Microservicios

**Equipo 12** · UTN FRRe · Tecnicatura Universitaria en Programación · 2026

Repositorio **orquestador** del Trabajo Práctico Integrador. Reúne los microservicios del sistema
(cada uno en su propio repositorio) y define cómo se comunican y cómo se levantan juntos.

## Integrantes

| Integrante | GitHub |
|---|---|
| Quijano, Joaquín Gabriel | [@Joabriel](https://github.com/Joabriel) |
| Alarcón, Gustavo | [@Gustavoctes](https://github.com/Gustavoctes) |
| Micheloud, Joaquín | [@Joaquin-Micheloud](https://github.com/Joaquin-Micheloud) |

## Alcance

Plataforma backend de comercio electrónico dividida en tres programas independientes (microservicios)
conectados por HTTP. Permite administrar un catálogo de productos vinculados a sus proveedores,
registrar órdenes de compra verificando el stock real y simular el cobro de cada operación mediante
una pasarela de pagos propia. Un frontend mínimo permite demostrar el circuito completo.

Fuera de alcance: cobros con tarjetas reales.

## Componentes

| Carpeta | Repositorio | Tecnología | Base de datos | Puerto | Rol |
|---|---|---|---|---|---|
| `inventario/` | [TPI_BackendA_Inventario](https://github.com/ProogIV/TPI_BackendA_Inventario) | Node.js + Express | MongoDB (Mongoose) | 3000 | Catálogo de productos, proveedores y stock |
| `pedidos/` | [TPI_BackendB_Pedidos](https://github.com/ProogIV/TPI_BackendB_Pedidos) | Python + Django REST Framework | PostgreSQL | 8000 | Órdenes de compra; coordina Inventario y Pagos |
| `pagos/` | [TPI_BackendC_Pagos](https://github.com/ProogIV/TPI_BackendC_Pagos) | Python + FastAPI | — | 8001 | Simula la pasarela bancaria |
| `frontend/` | [TPI_Frontend](https://github.com/ProogIV/TPI_Frontend) | HTML + CSS + JavaScript | — | a definir | Interfaz mínima de demostración |

## Documentación del proyecto

- `spec.md` — contratos entre servicios: modelos de datos, endpoints y reglas de negocio.
- `agent.md` — reglas técnicas y de estilo de cada microservicio.

## Cómo clonar el proyecto

Este repositorio usa **submódulos de Git**, así que hay que clonarlo con la opción
`--recurse-submodules` para que se descarguen también los microservicios:

```bash
git clone --recurse-submodules https://github.com/ProogIV/TPI_Ecommerce.git
cd TPI_Ecommerce
```

Si ya se clonó sin esa opción, las carpetas de los servicios quedan vacías. Se completan con:

```bash
git submodule update --init --recursive
```

## Cómo levantar el proyecto con Docker Compose

Requisito: Docker Desktop (o Docker Engine con el plugin Compose) en ejecución.

```bash
git clone --recurse-submodules https://github.com/ProogIV/TPI_Ecommerce.git
cd TPI_Ecommerce
cp .env.example .env          # Windows (PowerShell): Copy-Item .env.example .env
```

Completar en `.env` las tres claves (`SECRET_KEY`, `JWT_SIGNING_KEY` y `SERVICE_TOKEN`) con valores
generados con `python -c "import secrets; print(secrets.token_urlsafe(50))"`, y levantar todo:

```bash
docker compose up --build
```

Al arrancar, sin intervención manual:

- Pedidos espera a PostgreSQL, aplica las migraciones, **crea los usuarios de prueba** y levanta Daphne (ASGI).
- Inventario se conecta a MongoDB y, si la base está vacía, **carga un proveedor y 4 productos**.

| Servicio | URL |
|---|---|
| Frontend | http://localhost:8080 |
| Inventario | http://localhost:3000/api/productos |
| Pedidos | http://localhost:8000/api/orders/ (admin: http://localhost:8000/admin/) |
| Pagos | http://localhost:8001/docs |

### Credenciales de prueba

| Rol | Usuario | Contraseña | Puede |
|---|---|---|---|
| Administrador | `admin` | `Admin12345!` | Administrar productos y proveedores; ver todos los pedidos |
| Comprador | `cliente` | `Cliente12345!` | Comprar y ver sus propios pedidos |

### Demostración del circuito de compra

1. Entrar a http://localhost:8080 e ingresar con `cliente` / `Cliente12345!`.
2. Comprar 2 unidades de la **Placa de Video**: el pedido queda **PAGADO** y el stock baja de 15 a 13.
3. Comprar 2 unidades del **Servidor Rack** ($12.000.000, supera el límite de Pagos): el pedido queda
   **FALLIDO** y el stock vuelve a su valor original (compensación).

Para detener todo: `docker compose down`. Para borrar también los datos: `docker compose down -v`.

## Cómo trabajar con los submódulos

Cada carpeta (`inventario/`, `pedidos/`, `pagos/`, `frontend/`) es un repositorio Git completo e
independiente. El repositorio orquestador **no guarda el código de los servicios**: guarda una
referencia a un commit puntual de cada uno.

**Para programar en un servicio**, se trabaja dentro de su carpeta (o en su propio clon) con el flujo
de siempre: rama, commit, push y Pull Request en el repositorio del servicio.

**Para actualizar el orquestador** a la última versión de los servicios:

```bash
git submodule update --remote          # trae lo último de la rama principal de cada servicio
git add inventario pedidos pagos frontend
git commit -m "Actualiza submódulos a la última versión de cada servicio"
git push
```

> Si no se hace este paso, el orquestador sigue apuntando a la versión vieja de los servicios
> aunque ya se hayan mergeado cambios en ellos.

## Decisiones técnicas fuera del contenido de la cátedra

| Decisión | Motivo |
|---|---|
| Un repositorio por microservicio, más este orquestador con **submódulos de Git** (la planificación propone un monorepo; los submódulos no se vieron en la guía de Git de la cátedra) | Cada microservicio es independiente y tiene su propio ciclo de versiones, como en una arquitectura de microservicios real: un equipo puede trabajar y versionar su servicio sin afectar a los demás. El orquestador fija qué versión de cada servicio compone el sistema y permite levantar todo con un único comando, que es el objetivo del monorepo propuesto. |

## Hoja de ruta

| Entrega | Contenido | Estado |
|---|---|---|
| TP 0 | Repositorios, entornos, `spec.md` y `agent.md` | ✅ |
| TP 1 | CRUD de proveedores y productos; órdenes; integración de stock | ✅ |
| TP 2 | JWT, roles (Administrador / Cliente), validación y rate limiting | ✅ |
| TP 3 | Microservicio de Pagos y notificaciones en tiempo real (WebSockets con Django Channels y Redis) | ✅ |
| TP 4 y 5 | Docker Compose, CI con GitHub Actions, OpenAPI, GraphQL y defensa | En curso |
