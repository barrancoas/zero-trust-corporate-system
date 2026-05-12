# Manual de Usuario — Sistema Corporativo Zero Trust

**Proyecto:** Zero Trust Corporate System
**Autor:** Asier Barranco
**Fecha:** 12/05/2026
**Versión:** 1.0
**Destinatarios:** Usuarios finales corporativos

---

## 1. Introducción

Este manual describe cómo acceder y utilizar los servicios corporativos del Sistema Corporativo Zero Trust. Todos los servicios son accesibles exclusivamente a través del portal corporativo en `https://abb-ztcs.com`.

La autenticación es centralizada — inicias sesión una sola vez y accedes a todos los servicios. Cada inicio de sesión requiere dos factores:

1. **Tu contraseña de Active Directory** — la misma contraseña de tu cuenta corporativa
2. **Un código de un solo uso** — generado por una aplicación autenticadora en tu dispositivo móvil (Google Authenticator, Microsoft Authenticator o similar)

---

## 2. Primer Acceso — Configuración del MFA

La primera vez que inicias sesión, el sistema te pedirá que registres tu dispositivo móvil como segundo factor de autenticación. Esto solo se hace una vez.

1. Abre un navegador y ve a `https://abb-ztcs.com`
2. Serás redirigido a la página de inicio de sesión corporativa
3. Introduce tu nombre de usuario y contraseña
4. El sistema muestra un **código QR** — abre tu aplicación autenticadora y escanéalo
5. Introduce el código de 6 dígitos que muestra la app para confirmar el registro
6. Asigna un nombre a tu dispositivo (por ejemplo, `iphone-trabajo`) y haz clic en **Enviar**
7. Serás redirigido al panel de control de Nextcloud

> **Importante:** mantén la aplicación autenticadora instalada y accesible. Si pierdes acceso a tu dispositivo, contacta con el administrador del sistema para restablecer tu MFA.

---

## 3. Inicio de Sesión Diario

1. Abre un navegador y ve a `https://abb-ztcs.com`
2. Serás redirigido a la página de inicio de sesión — introduce tu usuario y contraseña
3. Introduce el código TOTP de 6 dígitos de tu aplicación autenticadora
4. Serás redirigido al **panel de control de Nextcloud**

![Portal corporativo — panel de Nextcloud tras el inicio de sesión](../../media/sso-config-01-dashboard.png)

> **Consejo:** utiliza siempre un navegador compatible (Firefox, Chrome, Edge). El modo privado/incógnito funciona, pero requerirá inicio de sesión completo cada vez.

---

## 4. Nextcloud — Gestión Documental

Nextcloud es la plataforma corporativa de gestión de documentos. Úsala para almacenar, organizar y compartir archivos de trabajo de forma segura.

### 4.1 Acceder a tus archivos

Tras iniciar sesión, haz clic en **Archivos** en la barra lateral izquierda o en el icono de archivos de la barra de navegación superior. Se muestra tu almacenamiento personal.

### 4.2 Subir un archivo

1. Navega a la carpeta donde quieres subir el archivo
2. Haz clic en el botón **+** (arriba a la izquierda) → **Subir archivo**
3. Selecciona el archivo desde tu ordenador
4. El archivo aparece en tu lista de archivos una vez subido

### 4.3 Crear una carpeta

1. Haz clic en **+** → **Nueva carpeta**
2. Introduce un nombre y pulsa **Intro**

### 4.4 Descargar un archivo

1. Pasa el cursor por encima del nombre del archivo
2. Haz clic en el **menú de tres puntos** (⋮) → **Descargar**

### 4.5 Compartir un archivo con otro usuario

1. Pasa el cursor por el archivo → haz clic en el icono **Compartir** (persona con +)
2. Escribe el nombre de usuario del destinatario (por ejemplo, `bob.jones`)
3. Selecciona el usuario en el desplegable
4. Elige los permisos (puede editar / solo lectura)
5. Haz clic en **Compartir**

El destinatario verá el archivo compartido en su sección **Compartido conmigo**.

### 4.6 Ver archivos compartidos

Haz clic en **Compartido** en la barra lateral izquierda para ver los archivos compartidos contigo o por ti.

---

## 5. Mattermost — Mensajería Corporativa

Mattermost es la plataforma de mensajería corporativa. Úsala para la comunicación en equipo, mensajes directos y compartir archivos.

### 5.1 Acceder a Mattermost

Navega a `https://abb-ztcs.com/mattermost/` en tu navegador.

> **Nota:** Mattermost utiliza una cuenta local independiente. Contacta con tu administrador para obtener tus credenciales de Mattermost si aún no las tienes.

### 5.2 Uso básico

- **Canales** — haz clic en el nombre de un canal en la barra lateral izquierda para abrirlo y leer o publicar mensajes
- **Mensajes directos** — haz clic en **+** junto a **Mensajes Directos** → busca a un compañero → inicia la conversación
- **Enviar un mensaje** — escribe en el cuadro de mensaje en la parte inferior y pulsa **Intro**
- **Compartir un archivo** — haz clic en el icono del **clip** en el cuadro de mensaje para adjuntar un archivo

---

## 6. Cerrar Sesión

### 6.1 Cerrar sesión en Nextcloud

1. Haz clic en tu **avatar** (esquina superior derecha)
2. Haz clic en **Cerrar sesión**
3. El navegador te redirige a la página de cierre de sesión de Keycloak — tu sesión queda terminada en todos los servicios

### 6.2 Expiración de sesión

Si dejas la sesión inactiva durante un tiempo prolongado, el sistema cerrará la sesión automáticamente. Ve a `https://abb-ztcs.com` para volver a iniciar sesión.

---

## 7. Resolución de Problemas

| Problema | Solución |
|---|---|
| "Error inesperado al gestionar la solicitud de autenticación" | DC01 puede estar apagado — contacta con tu administrador |
| Has perdido acceso a la aplicación autenticadora | Contacta con tu administrador para restablecer el MFA |
| No puedes acceder a `https://abb-ztcs.com` | Comprueba tu conexión a internet — el servicio requiere HTTPS en el puerto 443 |
| La subida de archivos falla | Comprueba el tamaño del archivo — los archivos grandes pueden agotar el tiempo. Inténtalo de nuevo o divídelo en partes más pequeñas |
| Mattermost muestra una página en blanco | Limpia la caché del navegador y recarga |

---

## 8. Pautas de Seguridad

- **Nunca compartas tu contraseña** con nadie, incluido el personal de TI
- **Nunca compartas tus códigos TOTP** — son de un solo uso y caducan en 30 segundos
- **Cierra siempre la sesión** cuando uses un ordenador compartido o público
- **Notifica cualquier actividad sospechosa** a tu administrador de inmediato — solicitudes de inicio de sesión inesperadas, dispositivos desconocidos en tu aplicación autenticadora o archivos que no hayas creado

---

## 9. Referencia Rápida

| Acción | URL / Ubicación |
|---|---|
| Portal corporativo | `https://abb-ztcs.com` |
| Archivos Nextcloud | `https://abb-ztcs.com/apps/files/` |
| Mattermost | `https://abb-ztcs.com/mattermost/` |
| Cerrar sesión | Avatar (esquina superior derecha) → Cerrar sesión |