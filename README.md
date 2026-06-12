# DeliveryBot — Sistema de Pedidos Automatizado

> Automatización conversacional de cafetería institucional mediante Telegram, n8n y Google Sheets

---

<div align="center">

| | |
|---|---|
| **Autor** | Diego Mantilla |
| **Curso** | Automatización de Procesos |
| **Plataforma** | n8n self-hosted (Docker) |
| **Repositorio** | `Proyecto_DeliveryBot_MantillaDiego` |

</div>

---

## ¿Qué es DeliveryBot?

Sistema de pedidos digital para cafeterías institucionales. Convierte Telegram en una terminal de pedidos con flujo guiado (Wizard), validación de stock en tiempo real, cálculo de IVA, fidelización por puntos y reportes automáticos. Google Sheets actúa como base de datos; n8n orquesta toda la lógica en **4 flujos / 97 nodos**.

---

## Navegación de la documentación

| Archivo | Contenido |
|---|---|
| [`docs/investigacion.md`](docs/investigacion.md) | Investigacion principal sobre n8n, bases de datos y otras cosas. |
| [`docs/arquitectura.md`](docs/arquitectura.md) | Diagrama de servicios, modelo de datos y configuración de infraestructura |
| [`docs/flujos_n8n.md`](docs/flujos_n8n.md) | Documentación técnica de los 4 flujos: nodos, lógica, código y parámetros |
| [`docs/casos_de_prueba.md`](docs/casos_de_prueba.md) | Escenarios de prueba, pasos de validación y resultados esperados |
| [`docs/evidencias.md`](docs/evidencias.md) | Registro de capturas de pantalla y resultados de ejecución |
| [`docs/Update_Examen_2.md`](docs/Update_Examen_2.md) Examen |
---

## Tecnologías Utilizadas

<div align="center">

| Tecnología | Rol en el proyecto |
|---|---|
| ![n8n](https://img.shields.io/badge/n8n-EA4B71?style=flat-square&logo=n8n&logoColor=white) | Motor de automatización — 97 nodos en 4 flujos |
| ![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white) | Contenedorización y despliegue del entorno n8n |
| ![Telegram](https://img.shields.io/badge/Telegram-26A5E4?style=flat-square&logo=telegram&logoColor=white) | Interfaz conversacional y canal de notificaciones |
| ![Google Sheets](https://img.shields.io/badge/Google_Sheets-34A853?style=flat-square&logo=google-sheets&logoColor=white) | Base de datos: MENU, PEDIDOS, USUARIOS, SESSIONS, CUPONES |
| ![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat-square&logo=javascript&logoColor=black) | Lógica de negocio en nodos Code |
| ![JSON](https://img.shields.io/badge/JSON-000000?style=flat-square&logo=json&logoColor=white) | Formato de comunicación entre nodos y almacenamiento del carrito |

</div>

---

## Estructura del repositorio

```
Proyecto_DeliveryBot_MantillaDiego/
├── README.md
├── docker-compose.yml
├── setting.json                 ← Workflow exportado de n8n (97 nodos)
├── screenshots/                 ← 20 capturas de evidencia
└── docs/
    ├── investigacion.md
    ├── arquitectura.md
    ├── flujos_n8n.md
    ├── casos_de_prueba.md
    └── evidencias.md
```

> **Importar workflow:** n8n → Settings → Workflows → Import from file → `setting.json`

---

## Base de datos — Google Sheets

El sistema usa un Spreadsheet de Google Sheets como base de datos. Puedes consultarlo directamente:

> 📊 **[Ver DeliveryBot_DB en Google Sheets](https://docs.google.com/spreadsheets/d/1u0Ur8gZeVgCLPy3YyscuHvKrzwtfaKfhXP0vQ7MT51w/edit?usp=sharing)**

| Hoja | Descripción |
|---|---|
| `MENU` | Productos, precios y stock |
| `PEDIDOS` | Historial de órdenes |
| `USUARIOS` | Registro y puntos de fidelización |
| `SESSIONS` | Estado del wizard por usuario |
| `CUPONES` | Códigos de descuento generados |

---

## Requisitos previos

| Herramienta | Versión requerida | Estado |
|---|---|---|
| Docker Desktop | ≥ 24.x | ✔ |
| Docker Compose | ≥ 2.x | ✔ |
| Cuenta Telegram | — | ✔ |
| Cuenta Google | — | ✔ |

---

## Inicio rápido

```bash
git clone https://github.com/DMntill4/Proyecto_DeliveryBot_MantillaDiego.git
cd Proyecto_DeliveryBot_MantillaDiego
docker compose up -d
ngrok http 5678
```

Acceder a `http://localhost:5678` → importar `setting.json` → configurar credenciales Telegram y Google Sheets.

---

## Referencias

- [Documentación n8n](https://docs.n8n.io/)
- [n8n en Docker](https://docs.n8n.io/hosting/installation/docker/)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Google Sheets API](https://developers.google.com/sheets/api)
- [Nodo Switch n8n](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/)
- [Nodo Code n8n](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/)