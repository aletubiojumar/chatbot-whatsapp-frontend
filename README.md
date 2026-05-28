# Panel de Administración — Chatbot WhatsApp Jumar

Panel web de administración para el bot de WhatsApp de gestión de siniestros de seguros. Permite supervisar conversaciones en tiempo real, gestionar expedientes, visualizar multimedia y cerrar siniestros sincronizando con PeritoLine.

> **Repo del bot (backend):** [github.com/certa-innovacion/chatbot_whatsapp](https://github.com/certa-innovacion/chatbot_whatsapp)

---

## Índice

1. [¿Qué es?](#qué-es)
2. [Funcionalidades](#funcionalidades)
3. [Acceso](#acceso)
4. [Estructura de archivos](#estructura-de-archivos)
5. [Autenticación](#autenticación)
6. [Pantalla de login](#pantalla-de-login)
7. [Panel principal](#panel-principal)
8. [Visor de media](#visor-de-media)
9. [Formulario de cierre](#formulario-de-cierre)
10. [Despliegue](#despliegue)

---

## ¿Qué es?

Interfaz web de una sola página (SPA) servida directamente por el servidor Express del bot. No requiere build step ni framework: es HTML + CSS + JavaScript vanilla, lo que facilita el despliegue y el mantenimiento.

El servidor Express (`src/bot/index.js`) sirve estos archivos como estáticos en la raíz del dominio. Los endpoints de la API de administración (`/admin/*`) están protegidos con JWT.

---

## Funcionalidades

### Lista de conversaciones (sidebar)
- **Búsqueda** en tiempo real por número de teléfono, expediente o estado
- **Modos de orden:** Reciente · Agrupar por Estado · Pausados primero
- **★ Favoritos** — sección fija no plegable en la parte superior, persiste en `localStorage` (no afecta a la base de datos ni a la máquina de estados del bot)
- **Campana de notificación** animada cuando llega un mensaje nuevo sin leer
- **Grupos colapsables** por estado (Cerrado, Valoración, Agendando, Escalado, Consentimiento)
- **"Conversaciones pendientes"** — grupo no plegable para conversaciones con bot pausado

### Chat en tiempo real
- Polling cada 5 s con detección de cambios (evita re-renderizar el DOM si no hay mensajes nuevos, preservando el estado de reproducción de vídeos)
- Renderiza: texto, imágenes (clickables), vídeo (con soporte HTTP Range para seek), audio (player nativo), documentos (descarga), tarjetas de contacto vCard
- Envío de mensajes del administrador directamente al asegurado
- Toggle para **pausar / reactivar el bot** por conversación

### Cabecera del chat
- Número de teléfono y expediente del asegurado
- Botón estrella ⭐ para marcar/desmarcar como favorito
- Badge de estado del bot (Activo / Pausado / Cerrado)

### Gestión de conversaciones
- **Cambio de estado manual** — mueve la conversación entre estados del flujo
- **Cierre con formulario** — modal split (formulario + visor de la conversación) que permite rellenar todos los campos del expediente antes de cerrar y sincronizar con PeritoLine
- **Eliminar de la vista** — oculta la conversación sin borrarla de la base de datos
- **Visor de media** — modal con galería de imágenes enviadas en la conversación

### Administración
- **Cambio de contraseña** — modal en la cabecera, fuerza re-login al completar
- **Recuperación de contraseña** — genera contraseña aleatoria y la envía por email (SMTP Office 365)
- **Modo claro / oscuro** — disponible en login y en el panel, persiste en `localStorage`
- **Toasts** de confirmación / error en la esquina inferior derecha

---

## Acceso

```
https://botjumar.com/login.html   →  Login
https://botjumar.com/admin-panel.html  →  Panel (redirige a login si no hay sesión)
https://botjumar.com              →  Raíz (redirige al panel)
```

La sesión dura **7 días** mediante JWT firmado con HMAC-SHA256. El secreto se genera automáticamente en `.jwt-secret` si no existe.

---

## Estructura de archivos

```
chatbot-whatsapp-frontend/
├── admin-panel.html        # SPA principal del panel de administración
├── login.html              # Pantalla de login con recuperación de contraseña
├── auth.json               # Credenciales de usuario (hash scrypt + salt, NO texto plano)
├── .jwt-secret             # Secreto JWT generado automáticamente (no commitear)
└── README.md
```

> `auth.json` y `.jwt-secret` son generados y gestionados por el servidor. No se incluyen en el repositorio.

---

## Autenticación

El backend expone tres endpoints de autenticación:

| Endpoint | Método | Descripción |
|---|---|---|
| `/auth/login` | POST | Verifica usuario y contraseña (scrypt + salt), devuelve JWT de 7 días |
| `/auth/change-password` | POST | Cambia la contraseña del usuario autenticado |
| `/auth/forgot-password` | POST | Genera contraseña aleatoria y la envía al email indicado |

Las contraseñas se almacenan con **scrypt** (derivación de clave + salt de 16 bytes) en `auth.json`. Nunca se guardan en texto plano.

La recuperación de contraseña requiere que el usuario indique su email. El sistema envía un correo vía SMTP (Office 365) con la nueva contraseña y recomienda cambiarla nada más entrar.

---

## Pantalla de login

`login.html` es una página independiente con:

- Formulario de usuario + contraseña
- Toggle modo claro/oscuro (esquina superior derecha)
- Enlace **"¿Contraseña olvidada?"** que abre un modal:
  1. El usuario escribe su correo electrónico
  2. El backend genera una contraseña aleatoria de 10 caracteres
  3. La guarda hasheada en `auth.json`
  4. Envía un email con la nueva contraseña al correo indicado
  5. Muestra confirmación o error en la pantalla (sin cerrar el login)

---

## Panel principal

`admin-panel.html` implementa toda la lógica en JavaScript vanilla sin dependencias externas. Los puntos más relevantes de la implementación:

### Estado global (`STATE`)
```js
STATE = {
  serverUrl,        // origin de la petición (automático)
  conversations,    // lista completa cargada del backend
  activeConvId,     // waId de la conversación abierta
  botPaused,        // mapa waId → boolean (estado local)
  favorites,        // Set de waIds guardados en localStorage
  unreadConvs,      // Set de waIds con mensajes sin leer
  sortMode,         // 'recent' | 'stage' | 'paused'
  collapsedGroups,  // Set de claves de grupos plegados
  pollTimer,        // referencia al setInterval del polling
}
```

### Polling
Cada 5 s llama a `loadConversations()` y `loadMessages()`. El render de mensajes compara una firma (`waId + nº mensajes + timestamp + texto del último`) antes de modificar el DOM, evitando interrumpir la reproducción de vídeo o audio.

### Favoritos
Solo existen en `localStorage` bajo la clave `panel_favorites`. No generan ninguna llamada al backend ni modifican el estado del bot.

### Detección de mensajes nuevos
El polling compara `conv.lastMessageAt` con el valor previo almacenado en `STATE.prevLastMessageAt`. Si cambia, añade el `waId` a `STATE.unreadConvs` y muestra la campana animada en la fila de la conversación. Al abrir la conversación (`selectConv`) se limpia automáticamente.

---

## Visor de media

Los archivos enviados por el asegurado se sirven a través del endpoint proxy del bot:

```
GET /media/s3/<s3-key>
```

El proxy añade soporte de **HTTP Range requests** (necesario para que `<video>` pueda hacer seek sin descargar el archivo completo). Los archivos se almacenan temporalmente en S3 y se eliminan al subirse a PeritoLine al cierre de la conversación.

**Tipos renderizados:**

| Tipo | Renderizado |
|---|---|
| Imágenes (jpg, png, webp, heic…) | `<img>` clickable que abre en pestaña nueva |
| Vídeo (mp4, webm, mov, 3gp…) | `<video controls>` con HTTP Range |
| Audio (mp3, ogg, opus, aac, wav…) | `<audio controls>` + botón de descarga |
| Contactos vCard | Tarjeta 📇 con nombre y teléfono clicable (`tel:`) |
| PDF / DOCX / XLSX / ZIP… | Botón de descarga con icono según extensión |
| Extensión desconocida (`/media/…`) | Botón de descarga genérico |

---

## Formulario de cierre

Al cambiar el estado de una conversación a "Cerrado" desde el panel, se abre un modal split:

- **Izquierda:** formulario con todos los campos del expediente (dirección, CP, municipio, coordenadas GPS, datos del perito, daños, causantes, perjudicados, modalidad, anotación)
- **Derecha:** visor read-only de la conversación completa para poder copiar datos

Los datos del formulario se cargan previamente desde el Excel (`GET /admin/conversations/:waId/details`). Al enviar:

1. Se escriben en el Excel (`POST /admin/conversations/:waId/close-form`)
2. Se genera el PDF de transcripción
3. Se dispara la sincronización con PeritoLine con anotación prefijada `[ADMIN]`

El cierre con formulario no cierra el modal hasta recibir HTTP 200 del servidor.

---

## Despliegue

El panel no requiere compilación. Para actualizar:

```bash
# En el servidor EC2
cd /home/ubuntu/chatbot-whatsapp-frontend
git pull origin main

# No es necesario reiniciar el bot (Express sirve los archivos directamente desde disco)
# Basta con que el usuario haga hard refresh en el navegador (Ctrl+Shift+R)
```

Para cambios en el backend que afecten a la API de administración, sí hay que reiniciar el bot:

```bash
pm2 restart chatbot_whatsapp --update-env
```

---

*Documentación generada a partir del código fuente — 2026-05-28*
