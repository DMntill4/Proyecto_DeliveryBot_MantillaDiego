# Casos de Prueba — DeliveryBot

**Convención de resultados:**
- ✅ PASS — El sistema responde exactamente como se espera
- ❌ FAIL — El sistema no responde o responde incorrectamente
- ⚠️ WARN — El sistema responde, pero con comportamiento inesperado menor

---

## Prerrequisitos de entorno

Antes de ejecutar cualquier prueba:

| Requisito | Verificación |
|---|---|
| n8n corriendo en Docker | `docker ps` → contenedor `n8n_deliverybot` en estado `Up` |
| ngrok activo | `ngrok http 5678` → URL visible en consola |
| Webhook configurado | n8n → Flujo 1 → Telegram Trigger → URL = URL de ngrok |
| Google Sheets con datos | MENU tiene ≥ 1 producto con `stock > 0` |
| Bot activo en Telegram | `@BotFather → /mybots` → bot listado como activo |
| Admin ID correcto | `ADMIN_ID = 8900800374` en los nodos Code de Flujo 3 |
| Cocina configurada | Usuario admin miembro del grupo `COCINA_CHAT_ID = -5140257658` |

---

## CP-01 — Registro de usuario nuevo

**Objetivo:** Verificar que un usuario sin registro previo queda registrado correctamente.

**Pasos:**

1. Abrir conversación con el bot desde una cuenta de Telegram NO registrada.
2. Enviar `/start`.

**Resultado esperado:**

| Verificación | Dónde |
|---|---|
| Bot responde con mensaje de bienvenida que incluye el nombre de usuario | Chat de Telegram |
| Nueva fila en SESSIONS con `pantalla_actual = inicio`, `carrito_temporal = []` | Hoja SESSIONS |
| Nueva fila en USUARIOS con `puntos_lealtad = 0` | Hoja USUARIOS |

---

## CP-02 — Registro de usuario existente

**Objetivo:** Verificar que `/start` no duplica un usuario ya registrado.

**Pasos:**

1. Enviar `/start` desde una cuenta YA registrada.

**Resultado esperado:**

| Verificación | Dónde |
|---|---|
| Bot responde con mensaje de bienvenida para usuario existente (distinto al de nuevo) | Chat de Telegram |
| Número de filas en USUARIOS no aumenta | Hoja USUARIOS |
| SESSIONS actualiza `pantalla_actual = inicio` y limpia el carrito | Hoja SESSIONS |

---

## CP-03 — Flujo completo de pedido (camino feliz)

**Objetivo:** Verificar el ciclo completo desde selección de categoría hasta confirmación.

**Datos de entrada:**

| Campo | Valor |
|---|---|
| Usuario | Registrado, `puntos_lealtad = 0`, sin cupones |
| Producto | `BEB001` — Café Americano — $3.500 COP — stock ≥ 2 |
| Cantidad | `2` |

**Pasos:**

1. Enviar `1` (Bebidas).
2. Enviar `1` (Café Americano o primer producto del menú).
3. Enviar `2` (cantidad).
4. Enviar `ver carrito`.
5. Enviar `confirmar`.

**Resultados esperados:**

| Paso | Verificación | Dónde |
|---|---|---|
| 1 | Bot muestra menú de Bebidas con lista numerada | Chat |
| 2 | Bot solicita cantidad para el producto seleccionado | Chat |
| 3 | Bot confirma el producto agregado al carrito | Chat |
| 4 | Bot muestra carrito con subtotal, IVA (19%) y total | Chat |
| 5 | Bot responde con confirmación de pedido incluyendo `idPedido` | Chat |
| 5 | Nueva fila en PEDIDOS con `estado = Recibido` | Hoja PEDIDOS |
| 5 | Stock de `BEB001` decrementado en 2 | Hoja MENU |
| 5 | `puntos_lealtad` actualizado en USUARIOS | Hoja USUARIOS |
| 5 | SESSIONS limpia (`pantalla_actual = inicio`, `carrito_temporal = []`) | Hoja SESSIONS |
| 5 | Grupo de cocina recibe mensaje de nuevo pedido | Telegram grupo cocina |

**Cálculo esperado:**

| Concepto | Valor |
|---|---|
| Subtotal | $7.000 COP (2 × $3.500) |
| IVA (19%) | $1.330 COP |
| Total | $8.330 COP |
| Puntos ganados | 8 pts |

---

## CP-04 — Pedido con cupón aplicado

**Objetivo:** Verificar descuento del 10% y marcado de cupón como usado.

**Datos de entrada:**

| Campo | Valor |
|---|---|
| Usuario | Tiene cupón `CUP-XXX-XXXXXX` con `estado = disponible` en CUPONES |
| Producto | `COM001` — Almuerzo del Día — $12.000 COP |
| Cantidad | `1` |

**Pasos:** Flujo completo como CP-03.

**Resultados esperados:**

| Verificación | Dónde |
|---|---|
| Mensaje de carrito muestra la línea de descuento (-10%) | Chat |
| Mensaje de confirmación incluye código de cupón usado y ahorro | Chat |
| Fila del cupón en CUPONES actualiza `estado = usado`, `fecha_uso` y `pedido_id` | Hoja CUPONES |

**Cálculo esperado:**

| Concepto | Valor |
|---|---|
| Subtotal base | $12.000 COP |
| Descuento 10% | −$1.200 COP |
| Subtotal con dcto | $10.800 COP |
| IVA (19%) | $2.052 COP |
| Total final | $12.852 COP |

---

## CP-05 — Generación automática de cupón

**Objetivo:** Verificar que al superar el umbral de 100 puntos se genera un cupón nuevo.

**Datos de entrada:**

| Campo | Valor |
|---|---|
| Usuario | `puntos_lealtad = 95` |
| Pedido | Total ≥ $5.000 COP (genera ≥ 5 puntos → supera 100) |

**Resultado esperado:**

| Verificación | Dónde |
|---|---|
| Mensaje de confirmación incluye sección "¡Desbloqueaste un cupón!" con código | Chat |
| Nueva fila en CUPONES con `estado = disponible` y `descuento_pct = 10` | Hoja CUPONES |
| `puntos_lealtad` actualizado (ej: `95 + 8 = 103`) | Hoja USUARIOS |

---

## CP-06 — Stock insuficiente al agregar producto

**Objetivo:** Verificar bloqueo cuando la cantidad pedida supera el stock disponible.

**Datos de entrada:** Producto con `stock = 1`. Cantidad ingresada: `5`.

**Resultado esperado:**

| Verificación | Dónde |
|---|---|
| Bot responde con error de stock indicando las unidades disponibles | Chat |
| SESSIONS no es modificado (carrito permanece intacto) | Hoja SESSIONS |
| PEDIDOS no recibe ninguna fila nueva | Hoja PEDIDOS |

---

## CP-07 — Stock se agota entre carrito y confirmación

**Objetivo:** Verificar que la validación de stock en tiempo real (Flujo 2) detecta cambios ocurridos después de agregar el producto al carrito.

**Setup:** Reducir manualmente el stock de un producto en MENU a 0 después de que el usuario lo haya agregado al carrito.

**Pasos:**

1. Agregar producto al carrito (stock disponible).
2. Reducir stock a 0 directamente en MENU.
3. Enviar `confirmar`.

**Resultado esperado:**

| Verificación | Dónde |
|---|---|
| Bot responde con mensaje de error de stock (nodo Validar Stock Carrito) | Chat |
| PEDIDOS no recibe fila nueva | Hoja PEDIDOS |
| SESSIONS no es limpiado (el carrito se conserva) | Hoja SESSIONS |

---

## CP-08 — Categoría o producto fuera de rango

**Objetivo:** Verificar manejo de inputs inválidos en el wizard.

**Sub-casos:**

| Sub-caso | Input | Resultado esperado |
|---|---|---|
| CP-08a | `5` en selección de categoría | "Opción no válida. Elige entre 1 y 4." |
| CP-08b | `0` en selección de producto | Regresa a pantalla de inicio |
| CP-08c | Número mayor al total de productos disponibles | "Opción no válida. Escribe un número entre 1 y N." |
| CP-08d | Texto no numérico en selección de cantidad | "Escribe un número válido mayor a 0." |

---

## CP-09 — Historial de pedidos y puntos

**Objetivo:** Verificar que el historial muestra los últimos 5 pedidos y el estado de fidelización.

**Pasos:**

1. Usuario con ≥ 1 pedido previo envía `mi historial`.

**Resultado esperado:**

| Verificación |
|---|
| Mensaje muestra hasta 5 pedidos, ordenados del más reciente al más antiguo |
| Cada pedido incluye: `id_pedido`, `estado`, productos, total, fecha |
| Sección de fidelización muestra puntos actuales y puntos para siguiente cupón |
| Si tiene cupones disponibles, lista los códigos |

---

## CP-10 — Cambio de estado por admin (Flujo 3)

**Objetivo:** Verificar actualización de estado y notificación push al cliente.

**Pasos:**

1. Desde el **grupo de cocina**, enviar:
   ```
   /estado PED-1720950000-8900800374 En camino
   ```

**Resultado esperado:**

| Verificación | Dónde |
|---|---|
| PEDIDOS actualiza `estado = En camino` en la fila correspondiente | Hoja PEDIDOS |
| Cliente recibe notificación push: "Tu pedido [ID] está: 🛵 En camino" | Chat usuario |
| Admin recibe confirmación de actualización | Grupo cocina |

---

## CP-11 — Validaciones de seguridad del Flujo 3

| Sub-caso | Acción | Resultado esperado |
|---|---|---|
| CP-11a | `/estado` desde chat privado (no grupo cocina) | "Los comandos admin solo funcionan en el grupo de cocina." |
| CP-11b | `/estado` sin `idPedido` ni estado | "Formato incorrecto. Uso: /estado [ID_PEDIDO] [ESTADO]" |
| CP-11c | Estado inválido: `/estado PED-XXX Enviado` | Mensaje con lista de estados válidos |
| CP-11d | ID de pedido inexistente | "Pedido [ID] no encontrado en el sistema." |
| CP-11e | Usuario no admin en grupo cocina | "Acceso denegado." |

---

## CP-12 — Reporte diario automático (Flujo 4)

**Objetivo:** Verificar generación y envío del reporte a las 23:50.

**Método de prueba rápida (sin esperar la hora):**

1. En n8n, abrir el Flujo 4.
2. Clic en `Test workflow` (ejecutar manualmente).

**Resultado esperado:**

| Verificación | Condición |
|---|---|
| Admin recibe reporte con 6 métricas | Si hay pedidos del día |
| Reporte indica "Sin pedidos registrados hoy." | Si no hay pedidos del día |
| `fecha` del reporte coincide con la fecha actual | Siempre |
| `Producto estrella` muestra nombre real (no ID) | Si hay pedidos |

---

## CP-13 — Concurrencia básica

**Objetivo:** Verificar que dos usuarios distintos pueden realizar pedidos simultáneos sin interferencia.

**Pasos:**

1. Con dos cuentas de Telegram distintas, iniciar pedidos al mismo tiempo.
2. Ambos confirman el pedido en un intervalo < 30 segundos.

**Resultado esperado:**

| Verificación |
|---|
| PEDIDOS contiene dos filas distintas, una por usuario |
| Stock descontado correctamente para ambos pedidos (sin sobreescritura) |
| Cada usuario recibe su propia confirmación |
| El grupo de cocina recibe dos notificaciones separadas |

---

## Matriz de cobertura

| Flujo | CPs asociados | Funcionalidades cubiertas |
|---|---|---|
| Flujo 1 | CP-01, CP-02, CP-08, CP-09 | Registro, wizard, errores de input, historial |
| Flujo 2 | CP-03, CP-04, CP-05, CP-06, CP-07 | Pedido completo, cupones, stock, puntos |
| Flujo 3 | CP-10, CP-11 | Estados, validaciones admin |
| Flujo 4 | CP-12 | Reporte automático |
| Transversal | CP-13 | Concurrencia |