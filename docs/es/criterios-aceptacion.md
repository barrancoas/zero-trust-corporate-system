# Criterios de Aceptación y Rendimiento

**Proyecto:** Zero Trust Corporate System  
**Autor:** Asier Barranco  
**Fecha:** 27/04/2026  
**Versión:** 1.1  

---

## 1. Propósito

Este documento define los criterios de aceptación y los estándares mínimos de rendimiento para cada componente del sistema Zero Trust Corporate System. Un componente se considera funcional y aceptado cuando cumple todos sus criterios. Estos criterios sirven como referencia para la fase de validación (Sprint 2, Fase 9) y el plan de pruebas final.

### Niveles de prioridad

Cada criterio está etiquetado con uno de los siguientes niveles de prioridad:

- **[NÚCLEO]** — Obligatorio. El proyecto no se considera completo sin esto.
- **[OPCIONAL]** — Deseable. Aporta valor pero no se considerará un fallo si no se consigue dentro del tiempo disponible.

---

## 2. Criterios de Aceptación por Componente

### 2.1 Infraestructura AWS

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| AWS-01 | **[NÚCLEO]** | VPC y subredes creadas | Existen una subred pública y una privada con rangos CIDR y tablas de routing correctos |
| AWS-02 | **[NÚCLEO]** | Conectividad a internet | Las instancias EC2 en la subred pública tienen salida a internet |
| AWS-03 | **[NÚCLEO]** | Aislamiento de subred privada | Las instancias EC2 en la subred privada no tienen routing de entrada ni salida directa a internet |
| AWS-04 | **[NÚCLEO]** | Security Groups | Solo están abiertos los puertos explícitamente requeridos; el resto denegados por defecto |
| AWS-05 | **[NÚCLEO]** | Instancias accesibles | Las instancias públicas responden a SSH desde la IP del administrador y a HTTPS en el puerto 443 |

---

### 2.2 Servidor Windows on-premise / Active Directory

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| AD-01 | **[NÚCLEO]** | Controlador de dominio operativo | `Get-ADDomain` y `Get-ADDomainController` devuelven resultado válido sin errores |
| AD-02 | **[NÚCLEO]** | Resolución DNS | `Resolve-DnsName corp.zerotrust.local` resuelve correctamente desde dentro del dominio |
| AD-03 | **[NÚCLEO]** | Estructura de UOs | Las UOs `Users`, `Groups`, `Computers`, `ServiceAccounts` y `Admins` existen bajo `ZeroTrust` |
| AD-04 | **[NÚCLEO]** | Usuarios de prueba creados | `alice.smith`, `bob.jones` y `zt.admin` existen y pueden autenticarse contra el dominio |
| AD-05 | **[NÚCLEO]** | Cuenta de servicio operativa | `svc.keycloak` existe en `ServiceAccounts` con permisos de lectura LDAP sobre objetos de usuario |
| AD-06 | **[NÚCLEO]** | Grupos de seguridad | `GRP-Users`, `GRP-Admins`, `GRP-DocMgmt` y `GRP-Comms` existen con los miembros correctos |
| AD-07 | **[NÚCLEO]** | GPO — Política de contraseñas | Mínimo 12 caracteres y complejidad requerida; verificado mediante `gpresult /r` |
| AD-08 | **[NÚCLEO]** | GPO — Bloqueo de cuenta | La cuenta se bloquea tras 5 intentos fallidos; verificado mediante prueba deliberada |
| AD-09 | **[NÚCLEO]** | Conectividad LDAP | La cuenta de servicio puede hacer bind a `ldap://192.168.56.10:389` y consultar objetos de usuario |

---

### 2.3 Túnel VPN WireGuard

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| VPN-01 | **[NÚCLEO]** | Túnel establecido | `wg show` en el peer Linux reporta un handshake válido con el peer Windows |
| VPN-02 | **[NÚCLEO]** | Conectividad bidireccional | La subred privada de AWS puede hacer ping a `192.168.56.10` y viceversa |
| VPN-03 | **[OPCIONAL]** | Verificación de tráfico cifrado | Se confirma mediante captura de paquetes que el tráfico pasa por la interfaz WireGuard |

---

### 2.4 Proveedor de Identidad (Keycloak)

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| KC-01 | **[NÚCLEO]** | Keycloak operativo | La consola de administración es accesible en su URL y responde con HTTP 200 |
| KC-02 | **[NÚCLEO]** | Federación AD mediante LDAP | Los usuarios del AD son visibles en Keycloak tras la sincronización LDAP |
| KC-03 | **[NÚCLEO]** | Flujo SSO | Un usuario del AD accede a una aplicación protegida a través de Keycloak sin reintroducir credenciales |
| KC-04 | **[NÚCLEO]** | MFA obligatorio | Tras la autenticación con contraseña, se solicita un código TOTP antes de conceder acceso |
| KC-05 | **[NÚCLEO]** | Repudio MFA | Un intento con credenciales válidas pero TOTP incorrecto o ausente es rechazado |
| KC-06 | **[NÚCLEO]** | Emisión de token | Keycloak emite un token OIDC/SAML válido aceptado por la aplicación protegida |

---

### 2.5 Proxy Inverso (Nginx)

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| NX-01 | **[NÚCLEO]** | Punto de entrada único | Todos los servicios solo son accesibles a través del proxy; el acceso directo a IPs internas está bloqueado |
| NX-02 | **[NÚCLEO]** | TLS obligatorio | Todas las conexiones se sirven sobre HTTPS; las peticiones HTTP se redirigen con estado 301 |
| NX-03 | **[NÚCLEO]** | Certificado válido | El certificado está emitido por Let's Encrypt y no genera avisos en el navegador |
| NX-04 | **[NÚCLEO]** | Renovación automática | `certbot renew` se ejecuta sin errores; más de 60 días de validez tras la renovación |
| NX-05 | **[NÚCLEO]** | Cabeceras de seguridad | Las cabeceras incluyen como mínimo: `Strict-Transport-Security`, `X-Frame-Options`, `X-Content-Type-Options` |
| NX-06 | **[NÚCLEO]** | Routing | Las peticiones se enrutan correctamente al servicio interno correspondiente |

---

### 2.6 Protección Activa (Fail2ban + UFW + Security Groups)

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| FW-01 | **[NÚCLEO]** | UFW activo | `ufw status` reporta activo con reglas de allow solo para los puertos requeridos |
| FW-02 | **[NÚCLEO]** | Fail2ban activo | `fail2ban-client status` reporta jails activos para SSH y Nginx |
| FW-03 | **[NÚCLEO]** | Bloqueo de IP | Tras 5 intentos fallidos desde una IP de prueba, esa IP se añade a la lista de baneos |
| FW-04 | **[NÚCLEO]** | Verificación del baneo | Una IP baneada recibe timeout al intentar acceder al proxy |

---

### 2.7 Base de Datos Relacional (MariaDB)

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| DB-01 | **[NÚCLEO]** | Servicio en ejecución | `systemctl status mariadb` (o equivalente Docker) reporta activo |
| DB-02 | **[NÚCLEO]** | Sin exposición externa | El puerto 3306 no es accesible desde fuera de la subred privada |
| DB-03 | **[NÚCLEO]** | Usuarios dedicados | Los servicios se conectan con usuarios dedicados con permisos mínimos |
| DB-04 | **[NÚCLEO]** | Acceso remoto root deshabilitado | El login remoto como root es rechazado |
| DB-05 | **[OPCIONAL]** | Backup automatizado | El script de backup se ejecuta correctamente y produce un volcado válido |

---

### 2.8 Servicio de Gestión Documental (Nextcloud)

Nextcloud es el **servicio corporativo principal** del proyecto. La integración SSO completa con Keycloak es un requisito de núcleo.

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| NC-01 | **[NÚCLEO]** | Servicio accesible | Nextcloud es accesible a través del proxy en su URL designada |
| NC-02 | **[NÚCLEO]** | Autenticación SSO | Un usuario del AD puede iniciar sesión en Nextcloud vía Keycloak SSO sin contraseña separada |
| NC-03 | **[NÚCLEO]** | MFA obligatorio | El flujo SSO requiere TOTP antes de conceder acceso a Nextcloud |
| NC-04 | **[NÚCLEO]** | Sin exposición directa | Nextcloud no es accesible saltándose el proxy |
| NC-05 | **[NÚCLEO]** | Operaciones de ficheros | Un usuario autenticado puede subir, descargar y eliminar ficheros sin errores |

---

### 2.9 Servicio de Comunicaciones (Mattermost)

Mattermost es un **servicio corporativo secundario**. Su despliegue e integración SSO son deseables pero no críticos si el tiempo no lo permite.

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| MM-01 | **[OPCIONAL]** | Servicio accesible | Mattermost es accesible a través del proxy en su URL designada |
| MM-02 | **[OPCIONAL]** | Autenticación SSO | Un usuario del AD puede iniciar sesión en Mattermost vía Keycloak SSO sin contraseña separada |
| MM-03 | **[OPCIONAL]** | MFA obligatorio | El flujo SSO requiere TOTP antes de conceder acceso a Mattermost |
| MM-04 | **[OPCIONAL]** | Sin exposición directa | Mattermost no es accesible saltándose el proxy |
| MM-05 | **[OPCIONAL]** | Mensajería | Un usuario autenticado puede enviar y recibir mensajes sin errores |

---

### 2.10 Script de Monitorización de Logs

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| LOG-01 | **[OPCIONAL]** | El script se ejecuta | El script se ejecuta sin errores mediante `bash` o como tarea cron |
| LOG-02 | **[OPCIONAL]** | Análisis de logs | El script identifica y reporta intentos de autenticación fallidos de Nginx y SSH |
| LOG-03 | **[OPCIONAL]** | Ejecución programada | La entrada cron está activa y el script se ejecuta en el intervalo definido |

---

### 2.11 Validación End-to-End del Sistema

| ID | Prioridad | Criterio | Aceptado cuando |
|---|---|---|---|
| E2E-01 | **[NÚCLEO]** | Flujo de autenticación completo | Un usuario del AD se autentica vía Keycloak (contraseña + TOTP), pasa por el proxy y accede a Nextcloud sin reautenticarse |
| E2E-02 | **[NÚCLEO]** | Denegación de acceso | Un usuario que no pertenece a `GRP-DocMgmt` no puede acceder a Nextcloud |
| E2E-03 | **[NÚCLEO]** | Aislamiento de red | Ningún servicio corporativo responde a peticiones directas que eviten el proxy |
| E2E-04 | **[OPCIONAL]** | Dependencia del túnel VPN | Si el túnel WireGuard cae, Keycloak no puede sincronizar con AD y la autenticación falla |

---

## 3. Estándares Mínimos de Rendimiento

| Métrica | Estándar mínimo | Prioridad |
|---|---|---|
| Tiempo de respuesta de autenticación SSO | < 3 segundos desde el envío hasta el acceso al servicio | **[NÚCLEO]** |
| Tiempo de respuesta del proxy (HTTPS) | < 1 segundo para peticiones estáticas bajo carga normal | **[NÚCLEO]** |
| Tiempo de bloqueo tras login fallido | ≤ 30 segundos después del 5º intento fallido | **[NÚCLEO]** |
| Validez del certificado | > 60 días restantes en cualquier momento | **[NÚCLEO]** |
| Tiempo de ejecución del backup | Finaliza sin error en menos de 5 minutos | **[OPCIONAL]** |
| Recuperación del túnel VPN | El túnel se restablece en menos de 60 segundos tras un reinicio | **[OPCIONAL]** |

---

## 4. Fuera de Alcance

Los siguientes criterios están explícitamente fuera del alcance de este proyecto y no serán validados:

- Alta disponibilidad o balanceo de carga (despliegue de instancia única)
- Resistencia a ataques DDoS a escala
- Gestión de dispositivos móviles (MDM)
- Integración con servicio de correo electrónico