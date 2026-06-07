## Explicación del Problema

En entornos institucionales como oficinas, universidades o grandes centros de trabajo, la gestión de pedidos de cafetería suele ser ineficiente, provocando filas largas y errores en la toma de pedidos manual. La falta de un sistema digitalizado impide que el personal de cocina organice sus tareas y que los usuarios conozcan el tiempo real de entrega de sus productos.

**DeliveryBot** es una solución de automatización basada en n8n que convierte a Telegram en una terminal de pedidos inteligente. El sistema permite a los empleados o estudiantes:

- Consultar el menú por categorías mediante un flujo guiado por números, sin necesidad de recordar comandos.
- Armar su carrito seleccionando productos numerados y especificando cantidades.
- Recibir notificaciones push en tiempo real sobre el estado del pedido: Recibido → Preparación → En camino → Entregado.
- Consultar el historial de sus últimos 5 pedidos con estado actualizado.
- Acumular puntos de lealtad y canjearlos como cupones de descuento del 10%.

Mientras tanto, el personal de cocina recibe una alerta inmediata de cada nueva orden y el administrador obtiene un reporte diario automático con 6 métricas clave.

### Objetivos del sistema

- Implementar un sistema de pedidos digital mediante una interfaz conversacional guiada (Wizard) en Telegram.
- Automatizar el cálculo de totales con IVA del 19% y la generación de números de orden únicos.
- Gestionar el ciclo de vida del pedido a través de estados dinámicos.
- Centralizar el inventario y menú en Google Sheets para actualización fácil por parte del administrador.
- Validar el stock disponible antes de confirmar cualquier pedido.
- Implementar un sistema de puntos de lealtad con cupones de descuento automáticos.
- Generar reportes diarios de ventas con 6 métricas automáticamente.
- Optimizar la comunicación entre la cocina y el cliente mediante notificaciones push automáticas.

---

## Investigación Realizada

### ¿Qué es n8n y por qué usarlo?

n8n es una plataforma de automatización de flujos de trabajo open-source que permite conectar aplicaciones, APIs y servicios mediante una interfaz visual de nodos. A diferencia de herramientas como Zapier o Make, n8n puede instalarse en servidor propio (self-hosted), garantizando privacidad total de los datos y sin límites de ejecuciones en la versión gratuita.

### ¿Qué es un bot de Telegram y cómo funciona?

Un bot de Telegram es una cuenta automatizada que responde a mensajes enviados por usuarios. Se comunica mediante la **Telegram Bot API**, que expone endpoints REST para enviar y recibir mensajes, botones inline y notificaciones push. En n8n, el nodo `Telegram Trigger` escucha los mensajes entrantes y el nodo `Telegram` los envía.

### ¿Qué es Google Sheets como base de datos?

Google Sheets funciona como una base de datos liviana y accesible visualmente para proyectos de automatización. Su API permite leer, escribir y actualizar filas de forma programática. En este proyecto actúa como capa de persistencia para el menú, pedidos, usuarios, sesiones y cupones.

### ¿Qué es un flujo Wizard (guiado)?

Un flujo Wizard es una experiencia conversacional donde el bot lleva al usuario paso a paso, presentando opciones numeradas en cada etapa y esperando una respuesta numérica. Esto elimina la necesidad de recordar comandos y reduce los errores de entrada. El estado de cada paso se persiste en la hoja SESSIONS mediante el campo `pantalla_actual`.

### ¿Qué es la validación de stock?

La validación de stock verifica que la cantidad solicitada no exceda las unidades disponibles en inventario **antes** de confirmar el pedido. En DeliveryBot esta validación ocurre en dos puntos: al agregar al carrito (nodo `Agregar Producto al Carrito`) y al confirmar (nodo `Validar Stock Carrito`), garantizando consistencia del inventario.

### ¿Qué es el IVA y cómo se aplica?

El IVA (Impuesto al Valor Agregado) del 19% se calcula sobre el subtotal después de aplicar cualquier descuento de cupón. Se muestra desglosado tanto al agregar cada producto al carrito como en la confirmación final del pedido.

### ¿Qué son los puntos de lealtad?

Los puntos de lealtad son un sistema de recompensas donde el usuario acumula **1 punto por cada $1.000 COP gastados** (calculado sobre el total con IVA). Al acumular 100 puntos se genera automáticamente un cupón de descuento del 10% que se aplica en el siguiente pedido.


## Referencias

- [Documentación n8n](https://docs.n8n.io/)
- [n8n en Docker](https://docs.n8n.io/hosting/installation/docker/)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Google Sheets API](https://developers.google.com/sheets/api)
- [Nodo Switch n8n](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/)
- [Nodo Code n8n](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/)