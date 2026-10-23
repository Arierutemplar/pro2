# Distributed Task Queue Engine

Motor de alto rendimiento orientado a producción para aceptar y procesar tareas distribuidas de forma asíncrona. Está construido con Node.js en modo ESM, TypeScript estricto, Express, BullMQ, Redis (ioredis) y Zod. Bull-Board ofrece una interfaz para inspeccionar el estado de la cola; Pino proporciona logs estructurados.

El endpoint de entrada valida la solicitud y la encola sin esperar a que termine el procesamiento. Los workers consumen los trabajos de forma independiente, lo que permite escalar la API y el procesamiento por separado.

## Casos de uso en arquitecturas de microservicios

Una cola desacopla la recepción de solicitudes del trabajo que puede tardar, requerir reintentos o depender de servicios externos. La API puede confirmar la recepción con rapidez y los consumidores procesan los trabajos bajo su propia capacidad.

- **Procesamiento de archivos y documentos (`PROCESS_DOCUMENT`)**: mueve análisis, extracción de metadatos, generación de hashes, conversiones o indexación fuera del ciclo HTTP. El handler actual calcula cantidad de caracteres y palabras, genera un hash SHA-256 y devuelve un resultado de procesamiento. En un despliegue real, el contenido de gran tamaño normalmente se almacena en object storage y el trabajo en cola lleva una referencia al objeto.
- **Notificaciones y correos (`SEND_NOTIFICATION`)**: permite distribuir envíos masivos entre workers y reintentar fallos temporales de proveedores. El handler actual procesa el trabajo y registra el resultado; **no está conectado a un proveedor de correo o notificaciones**. Para enviar mensajes reales se debe integrar un proveedor y definir su política de idempotencia y manejo de errores.
- **Control de tasa para APIs de terceros**: una cola puede amortiguar picos de solicitudes y los límites de concurrencia o tasa del consumidor pueden adaptarse a las cuotas del proveedor. La configuración actual establece concurrencia de workers; no implementa un rate limiter de proveedor.
- **Tareas programadas o diferidas**: BullMQ permite programar trabajos futuros y repetitivos. La cola puede servir como base para cron jobs, pero este proyecto todavía no registra tareas programadas.
- **Webhooks de pasarelas de pago**: un endpoint de webhook puede validar la firma, persistir el evento y encolarlo para responder rápidamente al proveedor; los workers procesan luego la conciliación y actualizaciones internas. Esta aplicación no incluye actualmente un endpoint de webhook ni validación de firmas.

Los últimos tres patrones describen extensiones adecuadas para esta arquitectura, no funcionalidades que ya estén habilitadas en el código actual.

## Arquitectura y funcionalidades

El flujo principal sigue capas desacopladas:

```text
HTTP Router → Controller → Service → BullMQ Queue → Worker
```

- **Validación runtime**: Zod valida los payloads usando una unión discriminada por `type` antes de encolar el trabajo.
- **Cola y workers distribuidos**: BullMQ utiliza Redis como almacenamiento y coordinación. `REDIS_URL` acepta una URL `rediss://` para una conexión TLS, como la proporcionada por Upstash Redis Cloud.
- **Resiliencia**: cada trabajo tiene hasta 5 intentos, con retroceso exponencial a partir de 2 segundos. Los trabajos completados se conservan hasta 24 horas y los fallidos hasta 7 días, además de los límites de cantidad definidos en la cola.
- **Observabilidad**: Pino registra eventos estructurados de Redis y del worker, incluidos el ID, tipo, resultado o error e intento del trabajo.
- **Dashboard**: Bull-Board muestra la cola `tasks` en `/admin/queues`, incluidos trabajos activos, completados, diferidos y fallidos.
- **Pruebas**: Vitest y Supertest verifican la validación de solicitudes y la aceptación de trabajos sin requerir una instancia real de Redis para estas pruebas.
- **Cierre ordenado**: el proceso detiene la recepción HTTP y cierra worker, cola y conexión Redis al recibir `SIGTERM` o `SIGINT`.

> **Seguridad del dashboard:** el endpoint `/admin/queues` no tiene autenticación ni autorización incorporadas. En producción, no lo expongas públicamente: protégelo mediante autenticación/autorización de la aplicación, una red privada o controles equivalentes en el proxy de entrada.

## Stack tecnológico

| Tecnología | Uso |
|---|---|
| Node.js (ESM) | Entorno de ejecución y módulos ECMAScript |
| TypeScript (Strict) | Tipado estático y compilación estricta |
| Express | API HTTP y composición de rutas |
| BullMQ | Encolamiento, reintentos y consumo de trabajos |
| Redis (ioredis / Upstash) | Backend distribuido de BullMQ; TLS con `rediss://` |
| Zod | Validación de payloads y contratos |
| Pino | Logging estructurado |
| Bull-Board | Dashboard de monitoreo de colas |
| Vitest | Ejecutor de pruebas |
| Supertest | Pruebas HTTP de integración |

## Estructura del proyecto

```text
.
├── src/
│   ├── config/
│   │   └── redis.ts
│   ├── controllers/
│   │   └── task.controller.ts
│   ├── queues/
│   │   └── task.queue.ts
│   ├── routes/
│   │   ├── board.routes.ts
│   │   └── task.routes.ts
│   ├── services/
│   │   └── task.service.ts
│   ├── types/
│   │   └── task.types.ts
│   ├── workers/
│   │   └── task.worker.ts
│   ├── app.ts
│   └── index.ts
├── tests/
│   └── task.test.ts
├── vitest.config.ts
├── package.json
├── package-lock.json
└── tsconfig.json
```

## Variables de entorno

Crea un archivo `.env` en la raíz del proyecto. No incluyas credenciales reales en el repositorio.

```dotenv
PORT=3000
REDIS_URL=rediss://default:<TOKEN>@<HOST>:<PORT>

# Opcionales
WORKER_CONCURRENCY=10
LOG_LEVEL=info
SERVICE_NAME=distributed-task-queue
```

Para Redis local también se puede omitir `REDIS_URL` y configurar `REDIS_HOST`, `REDIS_PORT` y, si aplica, `REDIS_PASSWORD`. El cliente usa `127.0.0.1:6379` como valores locales predeterminados. Para Upstash, utiliza la URL TLS `rediss://` y conserva el token como secreto.

## Instalación y ejecución

Se requiere una versión de Node.js compatible con las dependencias y una instancia de Redis accesible para ejecutar la aplicación.

```bash
npm install
```

Iniciar el servidor de desarrollo (carga las variables de `.env`):

```bash
npm run dev
```

Ejecutar las pruebas automatizadas:

```bash
npm test
```

Comprobar los tipos de TypeScript:

```bash
npx tsc --noEmit
```

## API y endpoints

### `POST /api/v1/tasks/process`

Encola un trabajo y responde `202 Accepted` si BullMQ lo acepta. El cuerpo debe incluir el discriminador `type` y los campos requeridos para el tipo de tarea.

Ejemplo de solicitud `PROCESS_DOCUMENT`:

```http
POST /api/v1/tasks/process
Content-Type: application/json
```

```json
{
  "type": "PROCESS_DOCUMENT",
  "documentId": "document-42",
  "content": "Contenido del documento para procesar."
}
```

Respuesta esperada (`202 Accepted`):

```json
{
  "taskId": "1",
  "taskType": "PROCESS_DOCUMENT",
  "status": "accepted",
  "acceptedAt": "2026-10-23T20:00:00.000Z"
}
```

`taskId` es el identificador de trabajo generado por BullMQ. El contrato HTTP actual utiliza ese nombre; el ID no se devuelve como `jobId`.

Payload admitido para `SEND_NOTIFICATION`:

```json
{
  "type": "SEND_NOTIFICATION",
  "recipient": "persona@example.com",
  "subject": "Actualización",
  "message": "El procesamiento terminó."
}
```

Un payload que no cumpla el esquema devuelve `400 Bad Request`, con `error` y una lista de `issues` que indica las rutas y mensajes de validación. Si falla el encolamiento, el error pasa al middleware HTTP de errores y se registra con Pino.

### Dashboard Bull-Board

Con el servidor ejecutándose, abre:

```text
http://localhost:3000/admin/queues
```

El dashboard muestra el estado de la cola `tasks` y sus trabajos. Protege este endpoint en cualquier entorno accesible fuera de una red de confianza.
