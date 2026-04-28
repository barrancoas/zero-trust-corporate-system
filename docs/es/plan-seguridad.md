# Plan de Seguridad — Red Team y Blue Team

**Proyecto:** Zero Trust Corporate System  
**Autor:** Asier Barranco  
**Fecha:** 27/04/2026  
**Versión:** 1.0  

---

## 1. Propósito

Este documento define las actividades de seguridad ofensivas y defensivas planificadas para el Zero Trust Corporate System. Establece qué ataques se simularán, qué mitigaciones se aplicarán y verificarán, y los estándares mínimos de seguridad que la arquitectura debe cumplir para considerarse sólida.

El enfoque Purple Team significa que tanto las actividades ofensivas (Red Team) como las defensivas (Blue Team) las realiza la misma persona, con el objetivo explícito de demostrar que la arquitectura puede detectar, resistir y recuperarse de escenarios de ataque realistas.

---

## 2. Alcance

La validación de seguridad cubre el perímetro expuesto externamente y la capa de autenticación. No incluye movimiento lateral interno entre servicios de la subred privada, ya que el alcance del proyecto no contempla un escenario de host interno comprometido.

| En alcance | Fuera de alcance |
|---|---|
| Fuerza bruta contra el portal SSO | Movimiento lateral interno |
| Simulación de robo de credenciales (phishing) | Ataques de acceso físico |
| Intentos de secuestro de sesión | DDoS a escala |
| Bloqueo de IPs y repudio MFA | Ingeniería social más allá de la simulación de phishing |
| Análisis de logs y respuesta al incidente | Simulación de exploits de día cero |

---

## 3. Red Team — Simulaciones de Ataque

### 3.1 Ataque 1 — Fuerza Bruta

**Objetivo:** Verificar que el portal de autenticación y el acceso SSH están protegidos contra la adivinación automatizada de credenciales.

**Método:**
- Usar `hydra` o `medusa` desde una instancia Kali Linux para lanzar un ataque de diccionario contra el endpoint de login de Keycloak
- Lanzar por separado un ataque de fuerza bruta SSH contra la instancia EC2 pública

**Condición de éxito para el atacante:** Obtener credenciales válidas antes de ser bloqueado.

**Resultado esperado:** El ataque queda bloqueado tras 5 intentos fallidos. Fail2ban banea la IP origen. La cuenta queda bloqueada en Keycloak. No se obtiene ninguna sesión válida.

**Evidencia a recopilar:**
- Captura de pantalla de la lista de baneos de Fail2ban tras el ataque
- Consola de administración de Keycloak mostrando la cuenta bloqueada
- Log de acceso de Nginx con los intentos fallidos y los drops de conexión posteriores

---

### 3.2 Ataque 2 — Robo de Credenciales (simulación de phishing)

**Objetivo:** Verificar que las credenciales robadas por sí solas no son suficientes para acceder a los servicios corporativos — el MFA debe repudiar el acceso.

**Método:**
- Simular el compromiso de credenciales usando deliberadamente un usuario y contraseña válidos (`alice.smith`) para intentar el login sin el código TOTP
- Intentar el login con credenciales válidas pero un código TOTP incorrecto
- Intentar el login con credenciales válidas pero un código TOTP caducado

**Condición de éxito para el atacante:** Acceder a un servicio corporativo usando únicamente la contraseña robada.

**Resultado esperado:** Los tres intentos son rechazados por Keycloak. El usuario nunca llega a Nextcloud. Los eventos quedan registrados en el log de eventos de administración de Keycloak.

**Evidencia a recopilar:**
- Captura de pantalla de Keycloak rechazando el login en el paso de MFA
- Log de eventos de Keycloak mostrando los eventos de autenticación fallida
- Captura del navegador confirmando que no se estableció ninguna sesión

---

### 3.3 Ataque 3 — Secuestro de Sesión

**Objetivo:** Verificar que los tokens de sesión no pueden reutilizarse tras el logout o su expiración, y que el robo de token no otorga acceso persistente.

**Método:**
- Autenticarse legítimamente como `alice.smith` y capturar la cookie de sesión o el bearer token desde el navegador (DevTools → Aplicación → Cookies / pestaña Red)
- Cerrar la sesión
- Intentar reutilizar el token capturado para acceder a Nextcloud directamente, saltando el flujo de login de Keycloak
- Por separado, intentar reproducir el token desde una sesión de navegador diferente

**Condición de éxito para el atacante:** Acceder a Nextcloud usando un token de una sesión terminada.

**Resultado esperado:** El token reproducido es rechazado. Keycloak invalida los tokens al hacer logout. Nextcloud redirige a la página de login de Keycloak.

**Evidencia a recopilar:**
- Captura del navegador mostrando la captura del token (DevTools)
- Captura del intento de reproducción rechazado (redirección a la página de login)
- Consola de gestión de sesiones de Keycloak mostrando que no hay sesión activa para el usuario

---

## 4. Blue Team — Mitigaciones y Defensas

### 4.1 Defensas Activas (por diseño)

Estas defensas están integradas en la arquitectura y activas antes de que se simule ningún ataque:

| Defensa | Implementación | Valida |
|---|---|---|
| Bloqueo de cuenta | GPO de AD — 5 intentos fallidos, bloqueo 30 min | Resistencia a fuerza bruta |
| Baneo de IP | Fail2ban — jails para SSH y Nginx | Resistencia a fuerza bruta |
| MFA obligatorio | Keycloak — TOTP requerido para todos los usuarios | Repudio de robo de credenciales |
| TLS en todo el tráfico | Nginx + Let's Encrypt | Previene la interceptación de credenciales en tránsito |
| Expiración de token | Configuración de TTL de sesión y token en Keycloak | Resistencia a secuestro de sesión |
| Punto de entrada único | Proxy inverso Nginx — todo el tráfico enrutado por el proxy | Reducción de superficie de ataque |
| Security Groups | AWS — solo puertos 22 (solo IP admin) y 443 abiertos | Bastionado perimetral |

### 4.2 Respuesta al Incidente — Ataque 1 (Fuerza Bruta)

**Detección:** Alerta de Fail2ban en `/var/log/fail2ban.log` — respuestas 401 repetidas en el log de acceso de Nginx.

**Acciones de respuesta:**
1. Confirmar que el baneo está activo: `fail2ban-client status nginx-auth`
2. Identificar la IP origen y comprobar si hay reincidencia
3. Si la IP es persistente, añadirla permanentemente a UFW: `ufw deny from <IP>`
4. Revisar Keycloak por cuentas legítimamente bloqueadas y desbloquear si es necesario
5. Documentar el evento en el log de respuesta al incidente

**Recuperación:** No requiere recuperación si no se produjo ninguna autenticación exitosa. Si una cuenta quedó bloqueada legítimamente, desbloquear desde la consola de administración de Keycloak.

---

### 4.3 Respuesta al Incidente — Ataque 2 (Robo de Credenciales)

**Detección:** Log de eventos de Keycloak — eventos `LOGIN_ERROR` repetidos con tipo de error `INVALID_TOTP` para un usuario específico.

**Acciones de respuesta:**
1. Revisar los eventos de administración de Keycloak para el usuario afectado
2. Confirmar que no se estableció ninguna sesión exitosa
3. Si el patrón sospechoso continúa, deshabilitar temporalmente la cuenta en Keycloak
4. Notificar al usuario simulado (documentar la notificación en el log de incidente)
5. Forzar una rotación del secreto TOTP para la cuenta afectada

**Recuperación:** Reactivar la cuenta de usuario tras la rotación del secreto TOTP y la re-inscripción del usuario.

---

### 4.4 Respuesta al Incidente — Ataque 3 (Secuestro de Sesión)

**Detección:** Gestión de sesiones de Keycloak — un token presentado tras la invalidación de la sesión genera un evento de rechazo.

**Acciones de respuesta:**
1. Confirmar en Keycloak que la sesión fue correctamente invalidada al hacer logout
2. Verificar que el TTL del token está configurado a un valor que minimiza la ventana de reproducción (recomendado: TTL del access token ≤ 5 minutos)
3. Si se detectó una reproducción válida, invalidar inmediatamente todas las sesiones activas del usuario
4. Revisar los logs de acceso de Nginx para la IP origen del intento de reproducción

**Recuperación:** Forzar la reautenticación del usuario afectado. Revisar la configuración del TTL si la ventana de reproducción fue demasiado amplia.

---

## 5. Estándares Mínimos de Seguridad

Los siguientes estándares definen lo que la arquitectura debe ser capaz de soportar para considerarse sólida. No son aspiracionales — son la línea base.

| Estándar | Requisito |
|---|---|
| Resistencia a fuerza bruta | Ninguna cuenta queda comprometida tras un ataque de diccionario automatizado |
| Repudio MFA | Las credenciales válidas solas nunca otorgan acceso a ningún servicio corporativo |
| Invalidación de sesión | Los tokens son rechazados tras el logout sin ventana de reproducción |
| Integridad perimetral | Ningún servicio corporativo es accesible saltándose el proxy inverso |
| Trazabilidad de logs | Cada intento de ataque produce una entrada de log trazable |
| Bloqueo de IP | Las IPs atacantes son bloqueadas en menos de 30 segundos tras el 5º intento fallido |

---

## 6. Herramientas

| Herramienta | Propósito | Origen |
|---|---|---|
| Hydra | Simulación de fuerza bruta | Kali Linux |
| DevTools del navegador | Captura de token para prueba de secuestro de sesión | Integrado en navegador |
| Fail2ban | Bloqueo dinámico de IPs | EC2 Ubuntu |
| Consola de administración de Keycloak | Revisión de log de eventos, gestión de sesiones | Keycloak |
| Log de acceso de Nginx | Análisis de tráfico | EC2 Ubuntu |
| UFW | Bloqueo permanente de IPs | EC2 Ubuntu |

---

## 7. Entregables

Cada simulación de ataque produce la siguiente documentación, commiteada en `security/`:

| Archivo | Contenido |
|---|---|
| `security/red-team/attack-01-bruteforce.md` | Configuración del ataque, pasos de ejecución, resultados, evidencias |
| `security/red-team/attack-02-credential-theft.md` | Configuración del ataque, pasos de ejecución, resultados, evidencias |
| `security/red-team/attack-03-session-hijack.md` | Configuración del ataque, pasos de ejecución, resultados, evidencias |
| `security/blue-team/mitigation-01-bruteforce.md` | Logs de Fail2ban, lista de baneos, evidencia de bloqueo de cuenta |
| `security/blue-team/mitigation-02-mfa-repudiation.md` | Log de eventos de Keycloak, capturas de pantalla |
| `security/blue-team/incident-response-log.md` | Informe unificado de respuesta al incidente con línea temporal y lecciones aprendidas |