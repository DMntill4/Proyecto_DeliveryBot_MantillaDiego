# Flujos n8n — DeliveryBot

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

### Nodos

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Telegram Trigger | `telegramTrigger` | Escucha updates de tipo `message`. Credential: `DeliveryBot Telegram` |
| Leer Sesión Usuario | `googleSheets` (get) | Busca en SESSIONS (`149368348`) por `telegram_id` = `{{ $json.message.from.id }}` |
| Verificar Sesión Activa | `code` — Run Once for All Items | Evalúa si el usuario puede continuar. `/start` siempre pasa aunque no tenga sesión, para evitar ciclo vicioso de registro. Retorna: `puedeAcceder`, `textoUsuario`, `telegramId`, `chatId`, `pantalla_actual` |
| IF Sesión Activa | `if` | `{{ $input.first().json.puedeAcceder }}` is `true` → Switch Navegación Principal \| FALSE → Msg Sin Sesión Activa |
| Switch Navegación Principal | `switch` | 7 salidas evaluadas en orden (ver tabla de salidas) |
| Msg Sin Sesión Activa | `telegram` | Mensaje al usuario indicando que no tiene sesión activa |

> **Por qué Run Once for All Items en Verificar Sesión Activa:** Si SESSIONS devuelve varias filas (duplicados), el modo por defecto generaría múltiples items y el IF siguiente daría `true` y `false` simultáneamente.

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

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Leer Usuarios | `googleSheets` (get) | Busca al usuario en hoja USUARIOS por `telegram_id` |
| IF Usuario Nuevo | `if` | `{{ $input.all().length }}` equal to `0` → TRUE: usuario nuevo \| FALSE: ya existe |
| Crear Nuevo Usuario | `googleSheets` (append) | Inserta en USUARIOS: `telegram_id`, `nombre_completo`, `departamento=Sin asignar`, `puntos_lealtad=0` |
| Crear Nueva Sesión | `googleSheets` (append) | Inserta en SESSIONS registro inicial para el usuario |
| Actualizar Sesión Start | `googleSheets` (update) | Match por `telegram_id`. Escribe: `pantalla_actual=inicio`, `carrito_temporal=[]`, `categoria_seleccionada=''`, `producto_seleccionado=''`, `ultimo_cambio={{ new Date().toISOString() }}` |
| Msg Bienvenida Nuevo | `telegram` | Mensaje de bienvenida para usuario registrado por primera vez |
| Msg Bienvenida Existente | `telegram` | Mensaje de bienvenida para usuario que ya tenía cuenta |
| Limit Bienvenida | `limit` (valor=1) | Evita duplicación de items en rama terminal |
| Limit Usuario Nuevo | `limit` (valor=1) | Evita duplicación de items en rama terminal |

---

### Salida 1 — Historial + Puntos

```
Leer Historial Pedidos → Leer Todos Usuarios → Leer Todos Cupones
  → IF Sin Pedidos
      ├── TRUE  → Msg Sin Pedidos Aún
      └── FALSE → Construir Historial Pedidos → Msg Historial Pedidos
```

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Leer Historial Pedidos | `googleSheets` (get) | Lee pedidos del usuario en hoja PEDIDOS |
| Leer Todos Usuarios | `googleSheets` (get) | Lee hoja USUARIOS completa (para obtener puntos del usuario) |
| Leer Todos Cupones | `googleSheets` (get) | Lee hoja CUPONES completa (para listar cupones disponibles) |
| IF Sin Pedidos | `if` | Evalúa si el usuario tiene pedidos previos |
| Construir Historial Pedidos | `code` — Run Once for All Items | Muestra últimos 5 pedidos (más reciente primero). Agrega sección de fidelización con puntos actuales y cupones disponibles. Retorna: `chatId` (String), `mensaje` (String Markdown) |
| Msg Sin Pedidos Aún | `telegram` | Mensaje: "Aún no has realizado ningún pedido." |
| Msg Historial Pedidos | `telegram` | Envía el mensaje construido con historial y puntos |
| Limit Sin Pedidos | `limit` (valor=1) | Evita duplicación de items en rama terminal |

```javascript
const ultimos = pedidos.slice(-5).reverse();
// Formatea cada pedido: id_pedido, estado, productos, total, fecha
// Sección de fidelización:
const puntosParaSiguiente = 100 - (puntosActuales % 100);
// Si hay cupones disponibles → lista códigos; si no → pts faltantes
```

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

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Resolver Categoría | `code` | Mapea `1→Bebidas`, `2→Comida`, `3→Aperitivos`, `4→Postres`. Si no hay match → `error: true` |
| IF Error Categoría | `if` | Evalúa `error` del nodo anterior → TRUE: Msg Error \| FALSE: continúa |
| Normalizar Categoría | `set` | Limpia el nombre de categoría para usarlo como filtro |
| Leer Menú por Categoría | `googleSheets` (get) | Filter Column: `categorias` / Filter Value: `{{ $('Normalizar Categoría').item.json.categoria_limpia.trim() }}` |
| Construir Menú Categoría | `code` — Run Once for All Items | Filtra `stock > 0`, genera lista numerada con padding (`[ 01 ]`, `[ 02 ]`...), guarda array de productos en SESSIONS |
| Actualizar Sesión Categoría | `googleSheets` (update) | Escribe: `pantalla_actual=categoria`, `categoria_seleccionada={{ categoria }}`, `producto_seleccionado={{ JSON.stringify(listaProductos) }}` |
| Msg Menú Categoría | `telegram` | Envía lista de productos disponibles en la categoría seleccionada |
| Msg Error Categoría | `telegram` | Mensaje de error por categoría inválida |

```javascript
const listaProductos = productosDisponibles.map((item, idx) => ({
  num: idx + 1,
  id: prod.id_producto,
  nombre: prod.nombre,
  precio: prod.precio,
  stock: prod.stock
}));
```

**Cadena selección de producto (Salida 2):**

```
Seleccionar Producto (Code) → IF Error Selección
  ├── TRUE  → Msg Error Selección
  └── FALSE → Actualizar Sesión Producto → Msg Selección Producto
```

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Seleccionar Producto | `code` | Recibe el número digitado, busca el producto en el array guardado en `producto_seleccionado` de SESSIONS. Si el índice no existe → `error: true`. Si el usuario escribe `0` → regresa al inicio |
| IF Error Selección | `if` | Evalúa `error` → TRUE: Msg Error \| FALSE: continúa |
| Actualizar Sesión Producto | `googleSheets` (update) | Guarda el producto seleccionado en sesión y cambia `pantalla_actual=producto` |
| Msg Selección Producto | `telegram` | Solicita al usuario que ingrese la cantidad deseada |
| Msg Error Selección | `telegram` | Mensaje de opción inválida con rango permitido |

---

### Salida 3 — Agregar al Carrito

```
Agregar Producto al Carrito (Code) → IF Error Agregar Carrito
  ├── TRUE  → Msg Error Agregar Carrito
  └── FALSE → Guardar Carrito en Sesión → Actualizar Sesión Producto1
              → Msg Producto Agregado Carrito
```

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Agregar Producto al Carrito | `code` — Run Once for All Items | Valida sesión, producto seleccionado, cantidad numérica ≥ 1 y stock suficiente. Si pasa todas → actualiza `carrito[]` (suma si ya existe, agrega si es nuevo) y calcula subtotal + IVA (`subtotal * 0.19`) por línea |
| IF Error Agregar Carrito | `if` | Evalúa `error` → TRUE: Msg Error \| FALSE: continúa |
| Guardar Carrito en Sesión | `googleSheets` (update) | Escribe: `carrito_temporal={{ carritoJSON }}`, `pantalla_actual=carrito`, `producto_seleccionado=''` |
| Actualizar Sesión Producto1 | `googleSheets` (update) | Limpia el producto seleccionado de la sesión |
| Msg Producto Agregado Carrito | `telegram` | Confirma al usuario que el producto fue agregado al carrito |
| Msg Error Agregar Carrito | `telegram` | Mensaje con el error específico de validación |

**Validaciones de Agregar Producto al Carrito — en orden:**

| # | Validación | Error |
|---|-----------|-------|
| 1 | Sesión existe | "No encontramos tu sesión. Escribe /start." |
| 2 | `producto_seleccionado` no vacío | "Selecciona un producto del menú antes de indicar la cantidad." |
| 3 | Texto es número ≥ 1 | "Escribe un número válido mayor a 0." |
| 4 | Cantidad ≤ stock disponible | "Stock insuficiente. Disponible: X unidades." |

```javascript
const subtotal = Number(p.precio) * cantidad;
const iva = Math.round(subtotal * 0.19);
const totalConIva = subtotal + iva;
// Retorna carritoJSON (serializado) para guardar en SESSIONS
```

---

### Salida 4 — Ver Carrito

```
Leer Sesión Usuario → Leer Usuarios para Carrito → Leer Cupones para Carrito
  → Actualizar Sesión Ver Carrito → Construir Vista Carrito → Msg Vista Carrito
```

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Leer Sesión Usuario | `googleSheets` (get) | Re-lee la sesión actualizada del usuario |
| Leer Usuarios para Carrito | `googleSheets` (get) | Obtiene puntos actuales del usuario desde USUARIOS |
| Leer Cupones para Carrito | `googleSheets` (get) | Obtiene cupones disponibles del usuario desde CUPONES |
| Actualizar Sesión Ver Carrito | `googleSheets` (update) | Actualiza `pantalla_actual` en sesión |
| Construir Vista Carrito | `code` — Run Once for All Items | Lee carrito de SESSIONS, puntos de USUARIOS, cupones de CUPONES. Calcula `totalBase → valorIVA = round(totalBase * 0.19) → totalConIVA`. Si hay cupón: muestra aviso de aplicación automática. Si no: muestra `puntosParaSiguiente = 100 - (puntosActuales % 100)` |
| Msg Vista Carrito | `telegram` | Envía resumen del carrito con totales e IVA al usuario |

---

### Nodos auxiliares del Flujo 1

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Limit Opción Inválida | `limit` (valor=1) | Después de Msg Error Opción Inválida — evita duplicación de items |
| Limit Sin Pedidos | `limit` (valor=1) | Después de Msg Sin Pedidos Aún — evita duplicación de items |
| Limit Usuario Nuevo | `limit` (valor=1) | Después de Msg Bienvenida Nuevo — evita duplicación de items |
| Limit Sin Sesión | `limit` (valor=1) | Después de Msg Sin Sesión Activa — evita duplicación de items |
| Limit Bienvenida | `limit` (valor=1) | Después de Msg Bienvenida Existente — evita duplicación de items |
| Wait Sesión 1 | `wait` (1-2 s) | Entre escritura y lectura de SESSIONS |
| Wait Sesión 2 | `wait` (1-2 s) | Entre actualización de puntos y lectura de cupones |
| Wait Sesión 3 | `wait` (1-2 s) | Antes de leer sesión actualizada en confirmar |

---

## Flujo 2 — Procesamiento del Pedido

**Trigger:** Salida 5 del Switch (texto `confirmar` / `SI`).

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

### Nodos

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Leer Sesión para Carrito | `googleSheets` (get) | Lee sesión actual del usuario en SESSIONS |
| Leer Menú Completo Stock | `googleSheets` (get) | Lee hoja MENU completa para validar stock en tiempo real |
| Leer Usuarios para Pedido | `googleSheets` (get) | Lee hoja USUARIOS para obtener `puntos_lealtad` actuales |
| Leer Cupones para Pedido | `googleSheets` (get) | Lee hoja CUPONES para detectar cupones con `estado = 'disponible'` |
| Validar Stock Carrito | `code` — Run Once for All Items | Cruza cada item del carrito con MENU. Si algún producto tiene stock insuficiente o fue eliminado → `stockOk: false`. Retorna: `stockOk`, `carrito` (array validado), `mensajeError` |
| IF Stock Suficiente | `if` | `stockOk = true` → continúa \| FALSE → Msg Error Stock |
| Procesar Confirmación Pedido | `code` — Run Once for All Items | Nodo más complejo del sistema. Ejecuta los 12 pasos de cálculo (ver tabla de pasos). Genera `idPedido = PED-{Date.now()}-{telegramId}` |
| Actualizar Puntos Usuario | `googleSheets` (update) | Sheet USUARIOS (`1950246923`). Match por `telegram_id`. Escribe `puntos_lealtad={{ puntosNuevos }}` |
| Guardar Pedido en PEDIDOS | `googleSheets` (appendOrUpdate) | Escribe: `id_pedido`, `telegram_id`, `detalles_pedido`, `total_pago`, `estado=Recibido`, `fecha`, `hora` |
| IF Tiene Cupón Activo | `if` | `{{ $('Procesar Confirmación Pedido').item.json.codigoCuponUsado }}` is not empty → Marcar Cupón Usado |
| Marcar Cupón Usado | `googleSheets` (update) | Match por `codigo_cupon`. Escribe: `estado=usado`, `fecha_uso`, `pedido_id` |
| IF Genera Cupón Nuevo | `if` | `{{ $('Procesar Confirmación Pedido').item.json.generaCupon }}` equal to `true` → Guardar Cupón Nuevo |
| Guardar Cupón Nuevo | `googleSheets` (appendOrUpdate) | Escribe en CUPONES: `codigo_cupon`, `telegram_id`, `descuento_pct=10`, `estado=disponible`, `fecha_creacion` |
| Distribuir Items Pedido | `code` | Convierte el array del carrito en items individuales para que n8n itere `Descontar Stock Vendido` una vez por producto |
| Descontar Stock Vendido | `googleSheets` (update) — N veces | Match por `id_producto`. Escribe `stock = {{ stockActual - qty_vendida }}`. Se ejecuta una vez por cada producto del carrito |
| Limpiar Sesión Tras Pedido | `googleSheets` (update) | Escribe: `pantalla_actual=inicio`, `carrito_temporal=[]`, `producto_seleccionado=''`, `categoria_seleccionada=''` |
| Msg Pedido Confirmado Usuario | `telegram` | Envía resumen del pedido confirmado al usuario |
| Msg Nuevo Pedido Cocina | `telegram` | Envía alerta al grupo de cocina `COCINA_CHAT_ID = -5140257658` |
| Msg Error Stock | `telegram` | Detalla qué producto tiene stock insuficiente |

### Pasos de Procesar Confirmación Pedido

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

### Output de Procesar Confirmación Pedido

| Campo | Consumidor |
|-------|-----------|
| `idPedido`, `detallesJSON`, `totalFinal`, `fecha`, `hora` | Guardar Pedido en PEDIDOS |
| `puntosNuevos` | Actualizar Puntos Usuario |
| `codigoCuponUsado` | IF Tiene Cupón Activo → Marcar Cupón Usado |
| `generaCupon`, `codigoCuponNuevo` | IF Genera Cupón Nuevo → Guardar Cupón Nuevo |
| `mensajeUsuario` | Msg Pedido Confirmado Usuario |
| `mensajeCocina` | Msg Nuevo Pedido Cocina |

### Ejemplo — Msg Nuevo Pedido Cocina

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

### Nodos

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Procesar Comando Estado | `code` — Run Once for All Items | Valida en orden: origen del grupo de cocina, formato del comando, estado reconocido. Constantes: `ADMIN_ID = 8900800374`, `COCINA_CHAT_ID = "-5140257658"`. Normaliza tildes/mayúsculas con `.normalize("NFD")`. Retorna: `idPedido`, `nuevoEstado`, `esAdmin`, `emoji` |
| IF Es Admin | `if` | `$json.esAdmin` = `true` → IF Comando Válido \| FALSE → Msg Acceso Denegado |
| IF Comando Válido | `if` | `$json.comandoValido` = `true` → IF Estado Válido \| FALSE → Msg Error Formato |
| IF Estado Válido | `if` | `$json.esEstadoValido` = `true` → Leer Pedido por ID \| FALSE → Msg Estado Inválido |
| Leer Pedido por ID | `googleSheets` (get) | Busca en PEDIDOS por `id_pedido` |
| IF Pedido Encontrado | `if` | `$input.all().length` > `0` → Actualizar Estado Pedido \| FALSE → Msg Pedido No Encontrado |
| Actualizar Estado Pedido | `googleSheets` (update) | Sheet PEDIDOS (`1815407898`). Match por `id_pedido`. Escribe `estado={{ nuevoEstado }}` |
| Msg Actualización Estado Cliente | `telegram` | Notificación push al `telegram_id` del cliente leído desde la fila de PEDIDOS |
| Msg Confirmación Admin | `telegram` | Confirmación al grupo de cocina de que el estado fue actualizado |
| Msg Acceso Denegado | `telegram` | "Los comandos admin solo funcionan en el grupo de cocina." |
| Msg Error Formato | `telegram` | Indica el formato correcto del comando |
| Msg Estado Inválido | `telegram` | Lista los estados válidos con sus emojis |
| Msg Pedido No Encontrado | `telegram` | "Pedido [ID] no encontrado en el sistema." |

### Estados válidos

| Estado | Emoji |
|--------|-------|
| Recibido | 📋 |
| Preparación | 👨‍🍳 |
| En camino | 🛵 |
| Entregado | ✅ |

---

## Flujo 4 — Reporte Diario Automático

**Trigger:** Schedule — todos los días a las 23:50 hora Bogotá.

```
Schedule Trigger (23:50 America/Bogota)
→ Leer Pedidos del Día (fecha = hoy)
→ Leer Menú Reporte Diario
→ Generar Reporte Diario (Code)
→ Msg Reporte Diario (adminId: 8900800374)
```

### Nodos

| Nodo | Tipo | Descripción |
|------|------|-------------|
| Schedule Trigger | `scheduleTrigger` | Todos los días a las 23:50. Timezone: `America/Bogota` |
| Leer Pedidos del Día | `googleSheets` (get) | Filter Column: `fecha` / Filter Value: `{{ new Date().toLocaleDateString('es-CO') }}` |
| Leer Menú Reporte Diario | `googleSheets` (get) | Lee hoja MENU completa para cruzar nombres de productos |
| Generar Reporte Diario | `code` — Run Once for All Items | Cruza pedidos del día con MENU para resolver nombre del producto estrella. Calcula 6 métricas (ver tabla) |
| Msg Reporte Diario | `telegram` | Envía el reporte al admin (`adminId: 8900800374`) |

### Métricas de Generar Reporte Diario

| Métrica | Cálculo |
|---------|---------|
| Total pedidos | `listaPedidos.length` |
| Total vendido | `Σ total_pago` |
| Ticket promedio | `totalVendido / totalPedidos` |
| Producto estrella | ID más frecuente en `detalles_pedido` → cruzado con MENU para obtener nombre |
| Hora pico | Hora con mayor frecuencia de pedidos |
| Usuarios únicos | `new Set(telegram_ids).size` |

```javascript
const topProductoEntry = Object.entries(conteoProductos).sort((a, b) => b[1] - a[1])[0];
const productoMenu = menuItems.find(m => String(m.json.id_producto) === String(idEstrella));
productoEstrella = `${productoMenu.json.nombre} (${cantidadEstrella} uds.)`;
```

### Ejemplo de reporte

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