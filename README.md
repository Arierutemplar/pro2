# Distributed Task Queue Engine

Motor de tareas distribuidas de alto rendimiento, diseñado con criterios de
producción para aceptar, validar y procesar trabajos asíncronos. La aplicación
combina Node.js, TypeScript estricto, Express, BullMQ, Redis y Zod en una
arquitectura desacoplada: la API confirma la recepción rápidamente y los
workers procesan los trabajos de forma independiente y escalable.

El proyecto sirve como referencia de portafolio técnico enterprise para
resiliencia, observabilidad, validación de contratos y ejecución distribuida.

## Casos de uso implementados

### `PROCESS_DOCUMENT`

Procesamiento asíncrono de archivos y documentos fuera del ciclo HTTP. El
worker analiza el contenido recibido y devuelve:

- identificador del documento;
- cantidad de caracteres y palabras;
- hash SHA-256 del contenido;
- fecha de finalización.

En un entorno productivo, los archivos grandes pueden persistirse en object
storage y el payload puede transportar únicamente una referencia al objeto.

### `SEND_NOTIFICATION`

Distribución masiva de notificaciones y correos mediante la cola y los
workers. El contrato valida destinatario, asunto y mensaje, y el worker
registra el resultado procesado. La implementación actual es un adaptador
demostrativo: no conecta todavía con un proveedor externo de email o
notificaciones. Esa integración debe añadir idempotencia, límites del
proveedor y tratamiento de errores específicos.

### `PROCESS_INVOICE`

Simulación de facturación asíncrona con un servicio externo. El servicio mock:

- introduce un retardo aleatorio de entre **1.5 y 3 segundos**;
- responde con una factura `PAID`, UUID y URL simulada de PDF cuando tiene
  éxito;
- genera un error transitorio **503** en el **20%** de las llamadas;
- permite demostrar la resiliencia y los reintentos automáticos de BullMQ.

La cola configura hasta cinco intentos con **exponential backoff** desde 2
segundos. Cuando el servicio simulado falla, el worker propaga el error para
que BullMQ reprograme el trabajo. No se realiza un cobro real ni se invoca un
proveedor de facturación externo.

## Arquitectura y tecnologías

El flujo de una solicitud es:

```text
HTTP Router → Controller → Zod → Service → BullMQ Queue → Redis → Worker
```

- **Node.js + TypeScript**: runtime ESM y tipado estricto para contratos
  mantenibles.
- **Express**: API HTTP, middleware, manejo de errores y composición de rutas.
- **Zod**: validación runtime estricta mediante una unión discriminada por
  `type` antes de encolar cada tarea.
- **BullMQ**: encolamiento, consumo, concurrencia, reintentos, exponential
  backoff y persistencia de estados.
- **Redis**: coordinación y almacenamiento distribuido de BullMQ mediante
  `ioredis`. Se admite `REDIS_URL`, incluyendo conexiones TLS `rediss://`.
- **Pino**: logs estructurados para eventos de Redis, workers, intentos,
  resultados y errores.
- **Bull-Board**: interfaz gráfica de monitoreo disponible en
  `/admin/queues`, con el estado de la cola `tasks` y sus trabajos activos,
  completados, diferidos y fallidos.
- **Vitest + Supertest**: suite de pruebas unitarias y de integración HTTP.

Los trabajos completados se conservan hasta 24 horas y los fallidos hasta
7 días, sujetos también a los límites de cantidad configurados en la cola.
La concurrencia del worker se controla con `WORKER_CONCURRENCY` y el proceso
implementa cierre ordenado ante `SIGTERM` y `SIGINT`.

> **Seguridad del dashboard:** `/admin/queues` no incorpora autenticación ni
> autorización. En producción debe exponerse únicamente detrás de controles
> de acceso, una red privada o un proxy autenticado.

## Despliegue y ejecución con Docker

El entorno completo —aplicación Node.js y Redis persistente— se levanta con
Docker Compose:

```bash
docker compose up --build
```

Compose inicia Redis 7, espera su healthcheck y después arranca la aplicación.
La API queda disponible en `http://localhost:3000` y los datos de Redis se
conservan en el volumen `redis_data`. Para detener los servicios:

```bash
docker compose down
```

El [Dockerfile](./Dockerfile) utiliza un **multi-stage build**:

1. instala todas las dependencias necesarias para compilar;
2. compila TypeScript a `dist/`;
3. instala únicamente dependencias de producción con `npm ci --omit=dev`;
4. crea una imagen runtime mínima basada en Node.js Alpine, ejecutada con el
   usuario sin privilegios `node`.

Este enfoque reduce el tamaño de la imagen final y evita incluir herramientas
de desarrollo en producción. Las variables `PORT` y `WORKER_CONCURRENCY`
pueden sobrescribirse al iniciar Compose.

## Desarrollo local y pruebas

### Requisitos e instalación

Se requiere Node.js 22 o una versión compatible con las dependencias y una
instancia de Redis accesible. Para instalar dependencias:

```bash
npm install
```

Configura un archivo `.env` en la raíz (no incluyas credenciales reales):

```dotenv
PORT=3000
REDIS_URL=redis://127.0.0.1:6379
WORKER_CONCURRENCY=10
LOG_LEVEL=info
SERVICE_NAME=distributed-task-queue
```

Para Redis administrado con TLS, utiliza una URL `rediss://` y almacena el
token como secreto. También se admiten `REDIS_HOST`, `REDIS_PORT` y
`REDIS_PASSWORD` para una configuración local alternativa.

### Scripts de desarrollo y validación

Arrancar la aplicación en modo desarrollo:

```bash
npm run dev
```

Ejecutar pruebas unitarias y de integración:

```bash
npm test
```

Verificar tipos sin emitir archivos:

```bash
npx tsc --noEmit
```

## API: envío de tareas

### `POST /api/v1/tasks/process`

El endpoint valida el payload con Zod y devuelve `202 Accepted` cuando BullMQ
acepta el trabajo. La respuesta contiene los campos `taskId`, `taskType`,
`status` (con valor `accepted`) y `acceptedAt`.

Ejemplo con `PROCESS_DOCUMENT`:

```bash
curl -X POST http://localhost:3000/api/v1/tasks/process \
  -H "Content-Type: application/json" \
  -d '{
    "type": "PROCESS_DOCUMENT",
    "documentId": "document-42",
    "content": "Contenido del documento para procesar."
  }'
```

Ejemplo con `SEND_NOTIFICATION`:

```bash
curl -X POST http://localhost:3000/api/v1/tasks/process \
  -H "Content-Type: application/json" \
  -d '{
    "type": "SEND_NOTIFICATION",
    "recipient": "persona@example.com",
    "subject": "Actualización",
    "message": "El procesamiento terminó."
  }'
```

Ejemplo completo con `PROCESS_INVOICE`:

```bash
curl -X POST http://localhost:3000/api/v1/tasks/process \
  -H "Content-Type: application/json" \
  -d '{
    "type": "PROCESS_INVOICE",
    "customerId": "customer-42",
    "amount": 125.50,
    "currency": "USD",
    "taxId": "TAX-123",
    "items": [
      {
        "description": "Consulting services",
        "quantity": 2,
        "unitPrice": 62.75
      }
    ]
  }'
```

El payload de factura requiere `customerId`, `amount` positivo, `currency`,
`taxId` y al menos un elemento en `items`, cada uno con `description`,
`quantity` entera positiva y `unitPrice` positivo.

Un payload inválido devuelve `400 Bad Request` con `error` e `issues`, que
identifican la ruta y el mensaje de cada incumplimiento. Los errores de
encolamiento se registran con Pino y pasan al middleware HTTP de errores.

## Monitoreo

Con la aplicación ejecutándose, abre:

```text
http://localhost:3000/admin/queues
```

Bull-Board permite inspeccionar la cola `tasks`, sus estados, intentos y
errores. En producción, protege este endpoint antes de exponerlo fuera de
una red de confianza.

## Estructura del proyecto

```text
.
├── src/
│   ├── config/redis.ts
│   ├── controllers/task.controller.ts
│   ├── queues/task.queue.ts
│   ├── routes/
│   │   ├── board.routes.ts
│   │   └── task.routes.ts
│   ├── services/
│   │   ├── billing.mock.service.ts
│   │   └── task.service.ts
│   ├── types/task.types.ts
│   ├── workers/task.worker.ts
│   ├── app.ts
│   └── index.ts
├── tests/task.test.ts
├── Dockerfile
├── docker-compose.yml
├── package.json
├── tsconfig.json
└── vitest.config.ts
```
