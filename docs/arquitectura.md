# Arquitectura del Sistema — DeliveryBot

---

## Diagrama general

```
┌──────────────────────────────────────────────────────────┐
│                  FLUJO 1 — CONVERSACIÓN                  │
│  Telegram Trigger → Verificar Sesión → Switch Principal  │
│    ├── /start         → Registro / Bienvenida            │
│    ├── mi historial   → PEDIDOS → Historial + puntos     │
│    ├── pantalla=categoria → Selección de producto        │
│    ├── pantalla=producto  → Ingreso de cantidad → Carrito│
│    ├── ver carrito    → Resumen con IVA y puntos         │
│    └── confirmar      → [Flujo 2]                        │
├──────────────────────────────────────────────────────────┤
│                 FLUJO 2 — PROCESAMIENTO                  │
│  Validar Stock → Procesar Pedido → Guardar PEDIDOS       │
│  → Descontar Stock → Actualizar Puntos → Gestionar Cupón │
│  → Limpiar Sesión → Notificar Usuario + Cocina           │
├──────────────────────────────────────────────────────────┤
│              FLUJO 3 — ESTADOS (ADMIN)                   │
│  /estado ID ESTADO → Verificar Admin + Grupo Cocina      │
│  → Validar Estado → Actualizar PEDIDOS → Notificar       │
├──────────────────────────────────────────────────────────┤
│            FLUJO 4 — REPORTE DIARIO                      │
│  Schedule (11:50 PM) → PEDIDOS del día + MENU            │
│  → 6 métricas → Reporte al admin                         │
└──────────────────────────────────────────────────────────┘
```

---

## Interacción entre servicios

```
Usuario (Telegram)
       │  mensaje
       ▼
  Telegram Bot API
       │  webhook POST
       ▼
  n8n (Docker:5678) ◄── ngrok (túnel público)
       │
       ├── Lee/escribe ──► Google Sheets (DeliveryBot_DB)
       │                        ├── MENU
       │                        ├── PEDIDOS
       │                        ├── USUARIOS
       │                        ├── SESSIONS
       │                        └── CUPONES
       │
       └── Envía mensajes ──► Telegram Bot API ──► Usuario / Cocina
```

---

## Modelo de datos — Google Sheets (`DeliveryBot_DB`)

### MENU

| Columna | Tipo | Descripción |
|---|---|---|
| `id_producto` | Texto | ID único (ej: `BEB001`) |
| `nombre` | Texto | Nombre del producto |
| `descripcion` | Texto | Descripción breve |
| `precio` | Número | Precio base COP (sin IVA) |
| `categorias` | Texto | `Bebidas`, `Comida`, `Aperitivos`, `Postres` |
| `stock` | Número | Unidades disponibles |

**Datos de prueba:**

| id_producto | nombre | precio | categorias | stock |
|---|---|---|---|---|
| BEB001 | Café Americano | 3500 | Bebidas | 50 |
| BEB002 | Jugo Hit de Lulo | 3500 | Bebidas | 180 |
| BEB003 | Speed Max | 3000 | Bebidas | 150 |
| COM001 | Almuerzo del Día | 12000 | Comida | 20 |
| APE001 | Almojábana Costeña | 2500 | Aperitivos | 40 |

> La columna se llama `categorias` (con s). El filtro del nodo `Leer Menú por Categoría` apunta a este nombre exacto.

---

### PEDIDOS

| Columna | Tipo | Descripción |
|---|---|---|
| `id_pedido` | Texto | `PED-{timestamp}-{telegram_id}` |
| `telegram_id` | Texto | ID del usuario |
| `detalles_pedido` | JSON | Array con productos y cantidades |
| `total_pago` | Número | Total final (IVA + descuentos) en COP |
| `estado` | Texto | `Recibido` / `Preparación` / `En camino` / `Entregado` |
| `fecha` | Texto | `DD/MM/YYYY` |
| `hora` | Texto | `HH:MM` |

---

### USUARIOS

| Columna | Tipo | Descripción |
|---|---|---|
| `telegram_id` | Texto | ID único de Telegram |
| `nombre_completo` | Texto | `message.from.first_name` |
| `departamento` | Texto | Default: `Sin asignar` |
| `puntos_lealtad` | Número | 1 pto por cada $1.000 COP gastados |

---

### SESSIONS

| Columna | Tipo | Descripción |
|---|---|---|
| `telegram_id` | Texto | ID del usuario |
| `pantalla_actual` | Texto | `inicio` / `categoria` / `producto` / `carrito` |
| `categoria_seleccionada` | Texto | Categoría activa (ej: `Bebidas`) |
| `producto_seleccionado` | JSON | Objeto del producto esperando cantidad |
| `carrito_temporal` | JSON | Array con productos del carrito |
| `ultimo_cambio` | Texto | Timestamp ISO |

> `pantalla_actual` es el campo clave del Wizard. El Switch principal lo lee para enrutar el mismo número `1` de forma diferente según el contexto (categoría vs. producto).

---

### CUPONES

| Columna | Tipo | Descripción |
|---|---|---|
| `codigo_cupon` | Texto | `CUP-{telegram_id}-{timestamp}` |
| `telegram_id` | Texto | Propietario |
| `descuento_pct` | Número | `10` |
| `estado` | Texto | `disponible` / `usado` |
| `fecha_creacion` | Texto | Fecha de generación |
| `fecha_uso` | Texto | Fecha de canje |
| `pedido_id` | Texto | ID del pedido en que se usó |

---

## Configuración de infraestructura

### docker-compose.yml (desarrollo local)

```yaml
version: "3.8"

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n_deliverybot
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=false
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - GENERIC_TIMEZONE=America/Bogota
      - TZ=America/Bogota
      - WEBHOOK_URL=https://grouped-omission-scant.ngrok-free.dev/
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

### Variables adicionales para producción

```yaml
- N8N_BASIC_AUTH_ACTIVE=true
- N8N_BASIC_AUTH_USER=admin_deliverybot
- N8N_BASIC_AUTH_PASSWORD=ContraseñaSegura2025!
- N8N_HOST=tudominio.com
- N8N_PROTOCOL=https
```

### Comandos Docker

```bash
docker compose up -d            # Levantar
docker ps                       # Ver estado
docker compose down             # Detener
docker logs -f n8n_deliverybot  # Ver logs
```

---

## Configuración de credenciales en n8n

### Bot de Telegram
1. Telegram → `@BotFather` → `/newbot` → copiar token.
2. n8n → Settings → Credentials → Add → **Telegram API** → pegar token → guardar como `DeliveryBot Telegram`.

### Google Sheets
1. n8n → Settings → Credentials → Add → **Google Sheets OAuth2**.
2. Seguir el flujo OAuth con la cuenta que tiene el Sheets.
3. Guardar como `DeliveryBot Sheets`.

---

## Constantes del sistema

| Constante | Valor | Uso |
|---|---|---|
| `ADMIN_ID` | `8900800374` | Verificación de admin en Flujo 3 |
| `COCINA_CHAT_ID` | `-5140257658` | Grupo de cocina en Telegram |
| `IVA_PCT` | `0.19` | Aplicado sobre subtotal con descuento |
| `PUNTOS_PARA_CUPON` | `100` | Umbral de puntos para generar cupón |
| `DESCUENTO_CUPON` | `10%` | Sobre subtotal base |
| `PUNTOS_POR_MIL` | `1 pto / $1.000 COP` | Calculado sobre `totalFinal` |