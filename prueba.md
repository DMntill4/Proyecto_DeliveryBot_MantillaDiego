# Flujos n8n — DeliveryBot

**Total:** 4 flujos / 97 nodos

---

## Flujo 1 — Menú y Carrito (Wizard)

**Trigger:** Cualquier mensaje de Telegram al bot.

### Cadena principal

```
Telegram Trigger
  → IF /estado (Flujo 3)
  → Leer Sesión Usuario (SESSIONS)
  → Verificar Sesión Activa (Code)
  → IF Sesión Activa
      ├── FALSE → Msg Sin Sesión Activa
      └── TRUE  → Switch Navegación Principal (7 salidas)
```

---

### Nodos de la cadena principal

| Nodo | Tipo | Parámetro | Valor |
|------|------|-----------|-------|
| Telegram Trigger | Trigger | Credential | `DeliveryBot Telegram` |
| Telegram Trigger | Trigger | Updates | `message` |
| Leer Sesión Usuario | Google Sheets — Get Row(s) | Sheet | SESSIONS (`149368348`) |
| Leer Sesión Usuario | Google Sheets — Get Row(s) | Filter Column | `telegram_id` |
| Leer Sesión Usuario | Google Sheets — Get Row(s) | Filter Value | `{{ $json.message.from.id }}` |
| IF Sesión Activa | IF | Condición TRUE | `{{ $input.first().json.puedeAcceder }}` is `true` → Switch Navegación Principal |
| IF Sesión Activa | IF | Condición FALSE | → Msg Sin Sesión Activa |

---

### Nodo — Verificar Sesión Activa (Code)

**Mode:** Run Once for All Items

Evalúa si el usuario puede continuar. `/start` siempre pasa aunque no tenga sesión (evita ciclo vicioso de registro).

```javascript
const trigger = $('Telegram Trigger').item.json;
const telegramId = String(trigger.message?.from?.id ?? "");
const textoUsuario = trigger.message?.text?.trim() ?? "";
const sesiones = $input.all();
const session = sesiones.find(s =>
  String(s.json.telegram_id ?? "").trim() === telegramId
)?.json;

const puedeAcceder = !!session || textoUsuario === '/start';

return [{
  json: {
    puedeAcceder,
    textoUsuario,
    telegramId,
    chatId: String(trigger.message?.chat?.id ?? ""),
    pantalla_actual: session ? session.pantalla_actual : 'inicio'
  }
}];
```

> **Por qué Run Once for All Items:** Si SESSIONS devuelve varias filas (duplicados), el modo por defecto generaría múltiples items y el IF siguiente daría `true` y `false` simultáneamente.

---

### Switch Navegación Principal — 7 salidas

> **Orden crítico:** Salidas 2 y 3 (por `pantalla_actual`) preceden a la salida 6 (por número). De lo contrario, el `1` de "seleccionar producto" caería siempre en "seleccionar categoría".

| Salida | Condición | Destino |
|--------|-----------|---------|
| 0 | Texto = `/start` | Registro / Bienvenida |
| 1 | Texto = `mi_historial` / `mi historial` / `📦 mi historial` | Historial + puntos |
| 2 | `pantalla_actual` = `categoria` | Selección de producto |
| 3 | `pantalla_actual` = `producto` | Ingreso de cantidad |
| 4 | Texto = `ver_carrito` / `ver carrito` / `🛒 ver carrito` | Vista del carrito |
| 5 | Texto = `confirmar` / `SI` | → Flujo 2 |
| 6 | Texto = `1` / `2` / `3` / `4` | Selección de categoría |
| Fallback | Cualquier otro | Msg Error Opción Inválida |

---

### Salida 0 — `/start`

```
Leer Usuarios (USUARIOS) → IF Usuario Nuevo
  ├── TRUE  → Crear Nuevo Usuario → Crear Nueva Sesión → Msg Bienvenida Nuevo
  └── FALSE → Actualizar Sesión Start → Msg Bienvenida Existente
```

**IF Usuario Nuevo:** `{{ $input.all().length }}` equal to `0`

| Nodo | Operación | Campo | Valor |
|------|-----------|-------|-------|
| Crear Nuevo Usuario | Sheets — Append | `telegram_id` | `{{ $('Telegram Trigger').item.json.message.from.id }}` |
| Crear Nuevo Usuario | Sheets — Append | `nombre_completo` | `{{ $('Telegram Trigger').item.json.message.from.first_name }}` |
| Crear Nuevo Usuario | Sheets — Append | `departamento` | `Sin asignar` |
| Crear Nuevo Usuario | Sheets — Append | `puntos_lealtad` | `0` |
| Actualizar Sesión Start | Sheets — Update | Match Column | `telegram_id` |
| Actualizar Sesión Start | Sheets — Update | `pantalla_actual` | `inicio` |
| Actualizar Sesión Start | Sheets — Update | `carrito_temporal` | `[]` |
| Actualizar Sesión Start | Sheets — Update | `categoria_seleccionada` | *(vacío)* |
| Actualizar Sesión Start | Sheets — Update | `producto_seleccionado` | *(vacío)* |
| Actualizar Sesión Start | Sheets — Update | `ultimo_cambio` | `{{ new Date().toISOString() }}` |

---

### Salida 1 — Historial + Puntos

```
Leer Historial Pedidos → Leer Todos Usuarios → Leer Todos Cupones
  → IF Sin Pedidos
      ├── TRUE  → Msg Sin Pedidos Aún
      └── FALSE → Construir Historial Pedidos → Msg Historial Pedidos
```

**Construir Historial Pedidos (Code) — Run Once for All Items:**

| Detalle | Descripción |
|---------|-------------|
| Fuentes | `Leer Historial Pedidos`, `Leer Todos Usuarios`, `Leer Todos Cupones` |
| Pedidos mostrados | Últimos 5 (más reciente primero) |
| Fidelización | Agrega sección con puntos actuales y cupones disponibles |

```javascript
const ultimos = pedidos.slice(-5).reverse();
// Formatea cada pedido: id_pedido, estado, productos, total, fecha
// Sección de fidelización:
const puntosParaSiguiente = 100 - (puntosActuales % 100);
// Si hay cupones disponibles → lista códigos; si no → pts faltantes
```

**Output:**

| Campo | Tipo |
|-------|------|
| `chatId` | String |
| `mensaje` | String (Markdown) |

---

### Salida 6 → Salida 2 — Categoría y Productos

**Cadena selección de categoría:**

```
Resolver Categoría → IF Error Categoría
  ├── TRUE  → Msg Error Categoría
  └── FALSE → Normalizar Categoría → Leer Menú por Categoría
              → Construir Menú Categoría → Actualizar Sesión Categoría
              → Msg Menú Categoría
```

| Nodo | Detalle |
|------|---------|
| Resolver Categoría (Code) | Mapea `1→Bebidas`, `2→Comida`, `3→Aperitivos`, `4→Postres`. Si no hay match → `error: true` |
| Leer Menú por Categoría (Sheets) | Filter Column: `categorias` / Filter Value: `{{ $('Normalizar Categoría').item.json.categoria_limpia.trim() }}` |
| Construir Menú Categoría (Code) — Run Once for All Items | Filtra `stock > 0`, genera lista numerada con padding (`[ 01 ]`, `[ 02 ]`...), guarda array de productos en SESSIONS |

```javascript
const listaProductos = productosDisponibles.map((item, idx) => ({
  num: idx + 1,
  id: prod.id_producto,
  nombre: prod.nombre,
  precio: prod.precio,
  stock: prod.stock
}));
```

**Actualizar Sesión Categoría (Sheets — Update):**

| Campo | Valor |
|-------|-------|
| `pantalla_actual` | `categoria` |
| `categoria_seleccionada` | `{{ categoria }}` |
| `producto_seleccionado` | `{{ JSON.stringify(listaProductos) }}` |

---

**Cadena selección de producto (Salida 2):**

```
Seleccionar Producto (Code) → IF Error Selección
  ├── TRUE  → Msg Error Selección
  └── FALSE → Actualizar Sesión Producto → Msg Selección Producto
```

**Seleccionar Producto (Code):** Recibe el número digitado, busca el producto en el array guardado en `producto_seleccionado` de SESSIONS. Si el índice no existe → `error: true`. Si el usuario escribe `0` → regresa al inicio.

---

### Salida 3 — Agregar al Carrito

```
Agregar Producto al Carrito (Code) → IF Error Agregar Carrito
  ├── TRUE  → Msg Error Agregar Carrito
  └── FALSE → Guardar Carrito en Sesión → Actualizar Sesión Producto1
              → Msg Producto Agregado Carrito
```

**Agregar Producto al Carrito (Code) — Run Once for All Items:**

Validaciones en orden:

| # | Validación | Error |
|---|-----------|-------|
| 1 | Sesión existe | "No encontramos tu sesión. Escribe /start." |
| 2 | `producto_seleccionado` no vacío | "Selecciona un producto del menú antes de indicar la cantidad." |
| 3 | Texto es número ≥ 1 | "Escribe un número válido mayor a 0." |
| 4 | Cantidad ≤ stock disponible | "Stock insuficiente. Disponible: X unidades." |

Si pasa todas → actualiza `carrito[]` (suma si el producto ya existe, agrega si es nuevo) y calcula subtotal + IVA de la línea:

```javascript
const subtotal = Number(p.precio) * cantidad;
const iva = Math.round(subtotal * 0.19);
const totalConIva = subtotal + iva;
// Retorna carritoJSON (serializado) para guardar en SESSIONS
```

**Guardar Carrito en Sesión (Sheets — Update):**

| Campo | Valor |
|-------|-------|
| `carrito_temporal` | `{{ $('Agregar Producto al Carrito').item.json.carritoJSON }}` |
| `pantalla_actual` | `carrito` |
| `producto_seleccionado` | *(vacío)* |

---

### Salida 4 — Ver Carrito

```
Leer Sesión Usuario → Leer Usuarios para Carrito → Leer Cupones para Carrito
  → Actualizar Sesión Ver Carrito → Construir Vista Carrito → Msg Vista Carrito
```

**Construir Vista Carrito (Code) — Run Once for All Items:**

| Detalle | Descripción |
|---------|-------------|
| Fuentes | Carrito de SESSIONS, puntos de USUARIOS, cupones de CUPONES |
| Cálculo | `totalBase` → `valorIVA = round(totalBase * 0.19)` → `totalConIVA` |
| Con cupón disponible | Muestra aviso de aplicación automática |
| Sin cupón | Muestra `puntosParaSiguiente = 100 - (puntosActuales % 100)` |

---

### Nodos auxiliares del Flujo 1

**Nodos Limit (valor=1):** Evitan duplicación de items en ramas terminales.

| Nodo | Posición |
|------|----------|
| Limit Opción Inválida | Después de Msg Error Opción Inválida |
| Limit Sin Pedidos | Después de Msg Sin Pedidos Aún |
| Limit Usuario Nuevo | Después de Msg Bienvenida Nuevo |
| Limit Sin Sesión | Después de Msg Sin Sesión Activa |
| Limit Bienvenida | Después de Msg Bienvenida Existente |

**Nodos Wait (1-2 s):** Sincronizan escritura en Sheets antes de lecturas posteriores.

| Nodo | Propósito |
|------|-----------|
| Wait Sesión 1 | Entre escritura y lectura de SESSIONS |
| Wait Sesión 2 | Entre actualización de puntos y lectura de cupones |
| Wait Sesión 3 | Antes de leer sesión actualizada en confirmar |

---

## Flujo 2 — Procesamiento del Pedido

**Trigger:** Salida 5 del Switch (texto `confirmar` / `SI`).

### Diagrama

```
Leer Sesión para Carrito
→ Leer Menú Completo Stock → Leer Usuarios para Pedido → Leer Cupones para Pedido
→ Validar Stock Carrito (Code)
    │
    ├── IF Stock Suficiente = FALSE → Msg Error Stock → Fin
    │
    └── IF Stock Suficiente = TRUE
          → Procesar Confirmación Pedido (Code)
          → Actualizar Puntos Usuario (USUARIOS)
          → Guardar Pedido en PEDIDOS
          → IF Tiene Cupón Activo → Marcar Cupón Usado
          → IF Genera Cupón Nuevo → Guardar Cupón Nuevo
          → Distribuir Items Pedido (Code)
          → Descontar Stock Vendido (itera por producto)
          → Limpiar Sesión Tras Pedido
          → Msg Pedido Confirmado Usuario
          → Msg Nuevo Pedido Cocina
```

---

### Nodo — Validar Stock Carrito (Code)

**Mode:** Run Once for All Items

Lee el carrito de SESSIONS y lo cruza con el MENU en tiempo real. Si cualquier producto tiene stock insuficiente o fue eliminado → `stockOk: false`.

```javascript
for (const item of carrito) {
  const productoMenu = menuItems.find(m => String(m.json.id_producto) === String(item.id));
  if (!productoMenu) { problemas.push(...); continue; }
  if (Number(productoMenu.json.stock) < Number(item.qty)) { problemas.push(...); }
}
// stockOk: true solo si problemas.length === 0
```

**Output:**

| Campo | Valor si OK | Valor si error |
|-------|-------------|----------------|
| `stockOk` | `true` | `false` |
| `carrito` | Array validado | — |
| `mensajeError` | — | Detalle del problema |

---

### Nodo — Procesar Confirmación Pedido (Code)

**Mode:** Run Once for All Items — nodo más complejo del sistema.

**Secuencia de cálculo:**

| Paso | Acción |
|------|--------|
| 1 | Lee sesión más reciente del usuario |
| 2 | Parsea `carrito_temporal` |
| 3 | Lee `puntos_lealtad` de USUARIOS |
| 4 | Lee cupones con `estado = 'disponible'` de CUPONES |
| 5 | Calcula `subtotalBase = Σ(precio × qty)` |
| 6 | Aplica descuento: `descuento = round(subtotalBase * 0.10)` si hay cupón |
| 7 | Calcula IVA: `valorIVA = round((subtotalBase - descuento) * 0.19)` |
| 8 | Calcula total: `totalFinal = (subtotalBase - descuento) + valorIVA` |
| 9 | Calcula puntos: `puntosGanados = floor(totalFinal / 1000)` |
| 10 | Genera cupón nuevo si `floor(puntosNuevos/100) > floor(puntosActuales/100)` |
| 11 | Genera `idPedido = PED-{Date.now()}-{telegramId}` |
| 12 | Construye `mensajeUsuario` y `mensajeCocina` |

**Campos del output y nodos que los consumen:**

| Campo | Consumidor |
|-------|-----------|
| `idPedido`, `detallesJSON`, `totalFinal`, `fecha`, `hora` | Guardar Pedido en PEDIDOS |
| `puntosNuevos` | Actualizar Puntos Usuario |
| `codigoCuponUsado` | IF Tiene Cupón Activo → Marcar Cupón Usado |
| `generaCupon`, `codigoCuponNuevo` | IF Genera Cupón Nuevo → Guardar Cupón Nuevo |
| `mensajeUsuario` | Msg Pedido Confirmado Usuario |
| `mensajeCocina` | Msg Nuevo Pedido Cocina |

---

### Nodos de escritura del Flujo 2

| Nodo | Operación | Campo | Valor |
|------|-----------|-------|-------|
| Guardar Pedido en PEDIDOS | Sheets — appendOrUpdate | `id_pedido` | `{{ $json.idPedido }}` |
| Guardar Pedido en PEDIDOS | Sheets — appendOrUpdate | `telegram_id` | `{{ $json.telegramId }}` |
| Guardar Pedido en PEDIDOS | Sheets — appendOrUpdate | `detalles_pedido` | `{{ $json.detallesJSON }}` |
| Guardar Pedido en PEDIDOS | Sheets — appendOrUpdate | `total_pago` | `{{ $json.totalFinal }}` |
| Guardar Pedido en PEDIDOS | Sheets — appendOrUpdate | `estado` | `Recibido` |
| Guardar Pedido en PEDIDOS | Sheets — appendOrUpdate | `fecha` | `{{ $json.fecha }}` |
| Guardar Pedido en PEDIDOS | Sheets — appendOrUpdate | `hora` | `{{ $json.hora }}` |
| Actualizar Puntos Usuario | Sheets — Update | Sheet | USUARIOS (`1950246923`) |
| Actualizar Puntos Usuario | Sheets — Update | Match Column | `telegram_id` |
| Actualizar Puntos Usuario | Sheets — Update | `puntos_lealtad` | `{{ $('Procesar Confirmación Pedido').item.json.puntosNuevos }}` |
| Descontar Stock Vendido | Sheets — Update (N veces, una por producto) | Match Column | `id_producto` |
| Descontar Stock Vendido | Sheets — Update (N veces, una por producto) | Match Value | `{{ $json.id_producto }}` |
| Descontar Stock Vendido | Sheets — Update (N veces, una por producto) | `stock` | `{{ stockActual - qty_vendida }}` |
| Limpiar Sesión Tras Pedido | Sheets — Update | `pantalla_actual` | `inicio` |
| Limpiar Sesión Tras Pedido | Sheets — Update | `carrito_temporal` | `[]` |
| Limpiar Sesión Tras Pedido | Sheets — Update | `producto_seleccionado` | *(vacío)* |
| Limpiar Sesión Tras Pedido | Sheets — Update | `categoria_seleccionada` | *(vacío)* |

---

### Nodo — Distribuir Items Pedido (Code)

Convierte el array del carrito en items individuales. n8n ejecuta `Descontar Stock Vendido` una vez por cada item devuelto (iteración nativa).

```javascript
return carrito.map(item => ({
  json: {
    id_producto: item.id,
    qty_vendida: item.qty,
    // + mensajeUsuario, mensajeCocina, chatId, idPedido
  }
}));
```

---

### Nodos IF y cupones del Flujo 2

| Nodo | Condición | Acción TRUE |
|------|-----------|-------------|
| IF Tiene Cupón Activo | `{{ $('Procesar Confirmación Pedido').item.json.codigoCuponUsado }}` is not empty | → Marcar Cupón Usado |
| IF Genera Cupón Nuevo | `{{ $('Procesar Confirmación Pedido').item.json.generaCupon }}` equal to `true` | → Guardar Cupón Nuevo |

**Marcar Cupón Usado (Sheets — Update):**

| Campo | Valor |
|-------|-------|
| Match Column | `codigo_cupon` |
| `estado` | `usado` |
| `fecha_uso` | `{{ $json.fecha }}` |
| `pedido_id` | `{{ $json.idPedido }}` |

**Guardar Cupón Nuevo (Sheets — appendOrUpdate):**

| Campo | Valor |
|-------|-------|
| `codigo_cupon` | `{{ $json.codigoCuponNuevo }}` |
| `telegram_id` | `{{ $json.telegramId }}` |
| `descuento_pct` | `10` |
| `estado` | `disponible` |
| `fecha_creacion` | `{{ $json.fecha }}` |

---

### Msg Nuevo Pedido Cocina

Destino: `COCINA_CHAT_ID = -5140257658`

```
🔔 NUEVO PEDIDO
📋 Orden: PED-1720950000-8900800374
👤 Usuario: 8900800374

Detalle:
  • Café Americano x2 = $7.000 COP

💰 Total: $7.497 COP
🕐 14/07/2025 08:32
Para ajustar el estado use /estado.
```

---

## Flujo 3 — Gestión de Estados

**Trigger:** Mensaje que empieza con `/estado` (detectado en Flujo 1, IF verificación de comando).

**Comando:** `/estado PED-1720950000-8900800374 En camino`

### Diagrama

```
Procesar Comando Estado (Code)
  → IF Es Admin
      ├── FALSE → Msg Acceso Denegado
      └── TRUE  → IF Comando Válido
                    ├── FALSE → Msg Error Formato
                    └── TRUE  → IF Estado Válido
                                  ├── FALSE → Msg Estado Inválido
                                  └── TRUE  → Leer Pedido por ID
                                              → IF Pedido Encontrado
                                                  ├── FALSE → Msg Pedido No Encontrado
                                                  └── TRUE  → Actualizar Estado Pedido
                                                              → Msg Estado Cliente
                                                              → Msg Confirmación Admin
```

---

### Nodo — Procesar Comando Estado (Code)

**Mode:** Run Once for All Items

**Constantes hardcodeadas:**

```javascript
const ADMIN_ID = 8900800374;
const COCINA_CHAT_ID = "-5140257658";
```

**Validaciones en orden:**

| # | Condición | Resultado |
|---|-----------|-----------|
| 1 | `chatId !== COCINA_CHAT_ID` | Error acceso denegado |
| 2 | `partes.length < 2` | Error formato |
| 3 | Estado no reconocido | Error estado inválido (normaliza tildes/mayúsculas con `.normalize("NFD")`) |
| 4 | Todo OK | Output con `idPedido`, `nuevoEstado`, `esAdmin`, `emoji` |

**Estados válidos:** `Recibido` 📋 / `Preparación` 👨‍🍳 / `En camino` 🛵 / `Entregado` ✅

---

### Nodos IF del Flujo 3

| Nodo | Condición | TRUE | FALSE |
|------|-----------|------|-------|
| IF Es Admin | `$json.esAdmin` = `true` | → IF Comando Válido | → Msg Acceso Denegado |
| IF Comando Válido | `$json.comandoValido` = `true` | → IF Estado Válido | → Msg Error Formato |
| IF Estado Válido | `$json.esEstadoValido` = `true` | → Leer Pedido por ID | → Msg Estado Inválido |
| IF Pedido Encontrado | `$input.all().length` > `0` | → Actualizar Estado | → Msg Pedido No Encontrado |

---

### Nodo — Actualizar Estado Pedido (Sheets — Update)

| Parámetro | Valor |
|-----------|-------|
| Sheet | PEDIDOS (`1815407898`) |
| Match Column | `id_pedido` |
| Match Value | `{{ $json.idPedido }}` |
| `estado` | `{{ $json.nuevoEstado }}` |

El nodo `Msg Actualización Estado Cliente` envía la notificación push directamente al `telegram_id` leído desde la fila de PEDIDOS.

---

## Flujo 4 — Reporte Diario Automático

**Trigger:** Schedule — todos los días a las 23:50 hora Bogotá.

### Diagrama

```
Schedule Trigger (23:50 America/Bogota)
→ Leer Pedidos del Día (fecha = hoy)
→ Leer Menú Reporte Diario
→ Generar Reporte Diario (Code)
→ Msg Reporte Diario (adminId: 8900800374)
```

---

### Nodos del Flujo 4

| Nodo | Parámetro | Valor |
|------|-----------|-------|
| Schedule Trigger | Trigger Interval | Days |
| Schedule Trigger | Trigger at Hour | 23 |
| Schedule Trigger | Trigger at Minute | 50 |
| Schedule Trigger | Timezone | `America/Bogota` |
| Leer Pedidos del Día (Sheets) | Filter Column | `fecha` |
| Leer Pedidos del Día (Sheets) | Filter Value | `{{ new Date().toLocaleDateString('es-CO') }}` |

---

### Nodo — Generar Reporte Diario (Code)

**Mode:** Run Once for All Items

Cruza `Leer Pedidos del Día` con `Leer Menú Reporte Diario` para mostrar el nombre real del producto estrella.

**6 métricas calculadas:**

| Métrica | Cálculo |
|---------|---------|
| Total pedidos | `listaPedidos.length` |
| Total vendido | `Σ total_pago` |
| Ticket promedio | `totalVendido / totalPedidos` |
| Producto estrella | ID más frecuente en `detalles_pedido` → cruzado con MENU |
| Hora pico | Hora con mayor frecuencia de pedidos |
| Usuarios únicos | `new Set(telegram_ids).size` |

```javascript
const topProductoEntry = Object.entries(conteoProductos).sort((a, b) => b[1] - a[1])[0];
const productoMenu = menuItems.find(m => String(m.json.id_producto) === String(idEstrella));
productoEstrella = `${productoMenu.json.nombre} (${cantidadEstrella} uds.)`;
```

**Ejemplo de reporte:**

```
📊 Reporte diario — 14/07/2025
━━━━━━━━━━━━━━━━━━━━━━
📦 Total de pedidos:    18
💰 Total vendido:       $215.500 COP
🧾 Ticket promedio:     $11.972 COP
⭐ Producto estrella:   Café Americano (24 uds.)
🕐 Hora pico:           12:00 (7 pedidos)
👥 Usuarios únicos:     12
━━━━━━━━━━━━━━━━━━━━━━
🤖 Generado automáticamente por DeliveryBot
```

---

## Sistema de Puntos y Cupones

### Reglas

| Regla | Valor |
|-------|-------|
| Acumulación | 1 pto / $1.000 COP del `totalFinal` |
| Umbral | 100 puntos → 1 cupón |
| Descuento | 10% sobre subtotal base |
| Aplicación | Automática (1 cupón por pedido, el primero disponible) |
| Orden de cálculo | Descuento → IVA (IVA se calcula sobre el subtotal ya descontado) |

### Flujo de ejemplo

```
Pedido 1: $8.330 COP → +8 pts → total: 103 pts
  → floor(103/100) > floor(95/100) → genera CUP-8900800374-XXXXXX

Pedido 2 del mismo usuario: $12.000 COP
  → Sistema detecta cupón disponible
  → Descuento: $12.000 × 10% = $1.200
  → Subtotal con descuento: $10.800
  → IVA (19%): $2.052
  → Total final: $12.852
  → Cupón marcado como 'usado' en CUPONES
```

---

## Manejo de Errores

| Escenario | Nodo | Mensaje |
|-----------|------|---------|
| Texto no reconocido | Switch → fallback | "Opción no válida. Escribe /start para regresar." |
| Usuario sin sesión | IF Sesión Activa = FALSE | "No tienes una sesión activa. Escribe /start." |
| Categoría fuera de rango | IF Error Categoría | "Opción no válida. Elige entre 1 y 4." |
| Producto fuera de rango | Seleccionar Producto | "Opción no válida. Escribe un número entre 1 y N." |
| Sin producto seleccionado | Agregar Producto | "Selecciona un producto del menú antes de indicar la cantidad." |
| Cantidad no numérica | Agregar Producto | "Escribe un número válido mayor a 0." |
| Cantidad > stock | Agregar Producto | "Stock insuficiente. Disponible: X unidades." |
| Carrito vacío al confirmar | Validar Stock | "Tu carrito está vacío. Elige una categoría." |
| Stock insuficiente al confirmar | Validar Stock | "No podemos procesar tu pedido. [PRODUCTO]: solicitaste X pero hay Y." |
| Sin pedidos en historial | IF Sin Pedidos | "Aún no has realizado ningún pedido." |
| Comando admin fuera del grupo | Procesar Comando Estado | "Los comandos admin solo funcionan en el grupo de cocina." |
| Estado inválido en /estado | IF Estado Válido = FALSE | Lista de estados válidos con emojis |
| Pedido no encontrado | IF Pedido Encontrado = FALSE | "Pedido [ID] no encontrado en el sistema." |