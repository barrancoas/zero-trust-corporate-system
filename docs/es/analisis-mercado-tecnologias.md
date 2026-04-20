# Análisis de Mercado y Estudio de Tecnologías

**Proyecto:** Zero Trust Corporate System  
**Autor:** Asier Barranco  
**Fecha:** 20/04/2026  
**Versión:** 1.0

---

## 1. Contexto y Necesidad de Mercado

La consolidación del trabajo remoto y la migración masiva hacia infraestructuras cloud han dejado obsoleto el modelo de seguridad perimetral clásico. En ese modelo, los usuarios y dispositivos dentro de la red corporativa reciben confianza implícita. Las amenazas modernas, como el robo de credenciales, movimiento lateral y ataques internos explotan precisamente esa suposición.

El modelo Zero Trust responde a este problema tratando cada solicitud de acceso como no confiable por defecto, independientemente de su origen. La identidad se convierte en el nuevo perímetro: cada usuario debe ser verificado explícitamente, cada acceso debe ser autorizado y todo el tráfico debe ir cifrado de extremo a extremo.

Esta tendencia tiene un reflejo claro en el mercado. El mercado global de seguridad Zero Trust superó los 30.000 millones de dólares en 2023 y se proyecta un crecimiento anual compuesto superior al 17% hasta 2030, impulsado por la presión regulatoria, la adopción cloud y la generalización del trabajo híbrido.

---

## 2. Soluciones Comerciales Existentes

Antes de seleccionar el stack tecnológico, se realizó un análisis de las principales plataformas Zero Trust comerciales disponibles en el mercado.

### 2.1 Okta Workforce Identity

Okta es uno de los principales proveedores de Identity-as-a-Service. Ofrece SSO, MFA, gestión del ciclo de vida de usuarios e integraciones con miles de aplicaciones cloud. Sus puntos fuertes son la facilidad de uso y el extenso catálogo de conectores. Sin embargo, es un producto SaaS sin opción de despliegue propio, lo que lo hace inadecuado para un proyecto educativo que requiere despliegue de infraestructura real. El precio parte de aproximadamente 6$ por usuario al mes.

### 2.2 Microsoft Azure Active Directory

La plataforma de identidad cloud de Microsoft se integra de forma nativa con entornos Windows y Microsoft 365. Soporta SSO, MFA y políticas de acceso condicional. Aunque es una solución potente para organizaciones dentro del ecosistema Microsoft, es una plataforma propietaria de código cerrado. Las funcionalidades completas de Zero Trust requieren licencias Azure AD Premium P2 de coste elevado.

### 2.3 Zscaler Zero Trust Exchange

Zscaler ofrece una plataforma ZTNA cloud-native completa. Enruta todo el tráfico corporativo a través de su infraestructura global distribuida. Es técnicamente maduro, pero es un servicio totalmente gestionado sin componente on-premise, y su modelo de licencias está orientado a grandes empresas.

### 2.4 Cloudflare Zero Trust

Cloudflare ofrece una solución ZTNA construida sobre su red de edge global, incluyendo proxy de acceso, federación de identidad e inteligencia de amenazas. Dispone de un nivel gratuito generoso, pero su arquitectura no permite el nivel de trabajo práctico sobre infraestructura que este proyecto requiere.

### 2.5 Resumen de Soluciones Comerciales

| Solución | Despliegue | Código abierto | Coste | Adecuada para este proyecto |
|----------|-----------|----------------|-------|---------------------------|
| Okta | SaaS | No | Alto | No |
| Azure AD | Cloud + híbrido | No | Medio-Alto | No |
| Zscaler | SaaS | No | Alto | No |
| Cloudflare Zero Trust | SaaS | No | Bajo-Medio | No |

Ninguna de las soluciones comerciales líderes es adecuada para este proyecto. Son servicios gestionados que eliminan el componente de despliegue de infraestructura, son propietarias o tienen un coste prohibitivo. Esto justifica la selección de un stack open source que replica los mismos principios arquitectónicos con control total sobre cada capa.

---

## 3. Análisis de Alternativas Open Source

Para cada componente de la arquitectura se evaluaron múltiples opciones open source antes de seleccionar la herramienta definitiva.

### 3.1 Proveedor de Identidad

El proveedor de identidad es el componente más crítico del stack Zero Trust. Debe soportar federación LDAP con Active Directory, SSO mediante OIDC o SAML 2.0, y MFA obligatorio en cada autenticación.

| Herramienta | Protocolos | Federación AD | MFA | Madurez | Seleccionada |
|-------------|-----------|---------------|-----|---------|--------------|
| **Keycloak** | OIDC, SAML 2.0, OAuth 2.0 | Sí (LDAP) | Sí (TOTP, WebAuthn) | Alta | **Sí** |
| Authentik | OIDC, SAML 2.0 | Sí (LDAP) | Sí | Media | No |
| Authelia | OIDC | Limitada | Sí | Media | No |
| Dex | OIDC | Sí | No (nativo) | Media | No |

**Keycloak** fue seleccionado por ser la plataforma de identidad open source más madura y ampliamente adoptada. Mantenido por Red Hat y el ecosistema CNCF, dispone de documentación oficial exhaustiva para federación LDAP, integración SAML y MFA basado en TOTP. Nextcloud y Mattermost tienen guías de integración documentadas y probadas específicamente para Keycloak.

### 3.2 Proxy Inverso

El proxy inverso actúa como único punto de entrada para todo el tráfico externo. Debe gestionar la terminación TLS, enrutar las peticiones autenticadas hacia los servicios internos y aplicar reglas de control de acceso.

| Herramienta | TLS automático | Proxy inverso | Rendimiento | Calidad documentación | Seleccionada |
|-------------|---------------|---------------|-------------|----------------------|--------------|
| **Nginx** | Sí (con Certbot) | Sí | Alto | Excelente | **Sí** |
| Apache HTTP Server | Sí (con Certbot) | Sí | Medio | Excelente | No |
| Caddy | Sí (automático) | Sí | Alto | Buena | No |
| Traefik | Sí (automático) | Sí | Alto | Buena | No |

**Nginx** fue seleccionado sobre Apache pese a la mayor familiaridad previa con este último. El factor determinante es que Keycloak, Nextcloud y Mattermost proporcionan ejemplos de configuración de proxy inverso escritos para Nginx. Usar Apache requeriría traducir cada bloque de configuración a su sintaxis, introduciendo riesgo innecesario dado el tiempo disponible. Nginx es además el estándar del sector para proxy inverso en entornos cloud y contenedorizados.

### 3.3 Servicio de Gestión Documental

| Herramienta | SAML / OIDC | Auto-hospedado | Comunidad | Seleccionada |
|-------------|------------|----------------|-----------|--------------|
| **Nextcloud** | Sí (app SAML) | Sí | Muy activa | **Sí** |
| Seafile | Sí | Sí | Activa | No |
| OnlyOffice | Parcial | Sí | Activa | No |

**Nextcloud** fue seleccionado por ser la plataforma de gestión documental auto-hospedada más ampliamente desplegada, con una aplicación SAML dedicada que se integra directamente con Keycloak. Su despliegue Docker está bien documentado y mantenido activamente.

### 3.4 Servicio de Comunicaciones

| Herramienta | SAML / OIDC | Auto-hospedado | Consumo de recursos | Seleccionada |
|-------------|------------|----------------|---------------------|--------------|
| **Mattermost** | Sí (SAML, LDAP) | Sí | Bajo | **Sí** |
| Rocket.Chat | Sí | Sí | Alto | No |
| Matrix / Element | Sí | Sí | Medio | No |

**Mattermost** fue seleccionado por su bajo consumo de recursos y su integración SAML directa con Keycloak. Rocket.Chat ofrece funcionalidad similar pero es significativamente más pesado, lo que es relevante dado el presupuesto AWS disponible (~40$).

### 3.5 Base de Datos Relacional

| Herramienta | Compatibilidad | Conocida por el desarrollador | Seleccionada |
|-------------|--------------|-------------------------------|--------------|
| **MariaDB** | Nextcloud, Mattermost | Sí | **Sí** |
| PostgreSQL | Nextcloud, Mattermost | Parcialmente | No |
| MySQL | Nextcloud, Mattermost | Sí | No |

**MariaDB** fue seleccionada por experiencia personal previa con ella durante el ciclo formativo, incluyendo diseño de esquemas, operaciones CRUD, gestión de usuarios y procedimientos de backup. Es compatible con ambas plataformas y es la base de datos recomendada en la documentación oficial de Nextcloud y Mattermost.

### 3.6 Gestión de Certificados TLS

| Herramienta | Automatización | Coste | Integración con Nginx | Seleccionada |
|-------------|---------------|-------|----------------------|--------------|
| **Let's Encrypt + Certbot** | Sí (renovación automática) | Gratuito | Sí | **Sí** |
| Certificados autofirmados | Manual | Gratuito | Sí | No |
| AWS Certificate Manager | Sí | Gratuito (con ALB) | Parcial | No |

**Let's Encrypt con Certbot** fue seleccionado para el aprovisionamiento y renovación automática de certificados. Se integra directamente con Nginx y requiere configuración mínima. Los certificados autofirmados fueron descartados porque generan avisos en el navegador y no pueden usarse de forma transparente en un entorno de producción real.

### 3.7 Túnel VPN (On-premise ↔ AWS)

| Herramienta | Rendimiento | Complejidad de configuración | Consumo de recursos | Seleccionada |
|-------------|------------|------------------------------|---------------------|--------------|
| **WireGuard** | Alto | Baja | Muy bajo | **Sí** |
| OpenVPN | Medio | Media | Medio | No |
| AWS Site-to-Site VPN | Alto | Baja | N/A (gestionado) | No |

**WireGuard** fue seleccionado para el túnel cifrado entre el Active Directory on-premise y la VPC de AWS. Es significativamente más sencillo de configurar que OpenVPN, usa criptografía moderna y tiene un consumo de recursos mínimo. AWS Site-to-Site VPN fue descartado por su coste por hora (~0,05$/h) que consumiría una parte importante del presupuesto disponible.

### 3.8 Firewall y Protección Activa

| Herramienta | Función | Conocida por el desarrollador | Seleccionada |
|-------------|---------|-------------------------------|--------------|
| **Fail2ban** | Bloqueo dinámico de IPs | Sí | **Sí** |
| **AWS Security Groups** | Firewall perimetral cloud | Sí | **Sí** |
| UFW | Firewall de host | Sí | **Sí** (complementario) |

Se adoptó un enfoque de firewall por capas. Los Security Groups de AWS controlan el tráfico en el perímetro cloud. UFW gestiona las reglas de firewall a nivel de host en cada instancia EC2. Fail2ban monitoriza los logs de autenticación y bloquea dinámicamente las IPs tras intentos fallidos repetidos — una herramienta que el desarrollador ya ha desplegado en ejercicios prácticos anteriores.

### 3.9 Contenedorización

**Docker + Docker Compose** fue seleccionado para el despliegue de los servicios de la Capa 2 (Nextcloud, Mattermost, MariaDB). Simplifica la orquestación multi-contenedor, hace el entorno reproducible y se alinea con la práctica habitual en despliegue de servicios auto-hospedados. El desarrollador tiene experiencia previa con Docker desde el ciclo formativo.

---

## 4. Stack Tecnológico Final

| Componente | Tecnología | Capa | Entorno |
|------------|-----------|------|---------|
| Controlador de dominio | Windows Server 2022 + AD DS | Capa 1 | On-premise (VirtualBox) |
| Proveedor de identidad | Keycloak | Capa 3 | AWS EC2 (subred pública) |
| Proxy inverso | Nginx + Certbot | Capa 3 | AWS EC2 (subred pública) |
| Protección activa | Fail2ban + AWS Security Groups + UFW | Capa 3 | AWS + instancias EC2 |
| Gestión documental | Nextcloud (Docker) | Capa 2 | AWS EC2 (subred privada) |
| Comunicaciones | Mattermost (Docker) | Capa 2 | AWS EC2 (subred privada) |
| Base de datos relacional | MariaDB (Docker) | Capa 2 | AWS EC2 (subred privada) |
| Certificados TLS | Let's Encrypt + Certbot | Capa 3 | AWS EC2 (subred pública) |
| Túnel VPN | WireGuard | Infraestructura | On-premise ↔ AWS |
| Infraestructura cloud | AWS (VPC, EC2, Security Groups) | Infraestructura | AWS |
| Virtualización on-premise | VirtualBox | Infraestructura | On-premise |
| Scripting | Bash + PowerShell | Automatización | Linux + Windows |
| Contenedorización | Docker + Docker Compose | Despliegue | AWS EC2 (subred privada) |
| Control de versiones | GitHub | Gestión del proyecto | Cloud |
| Gestión de proyecto | ProofHub | Gestión del proyecto | Cloud |

---

## 5. Conclusiones

El stack seleccionado está compuesto íntegramente por herramientas open source de grado productivo que, en conjunto, implementan una arquitectura Zero Trust funcional. Cada herramienta fue elegida en base a tres criterios: adecuación técnica a los requisitos del proyecto, compatibilidad de integración con el resto de componentes y viabilidad dentro del tiempo y presupuesto disponibles.

Las alternativas comerciales analizadas — Okta, Azure AD, Zscaler y Cloudflare Zero Trust — ofrecen funcionalidad comparable o superior, pero son incompatibles con los objetivos de despliegue práctico de este proyecto. Son servicios gestionados, plataformas propietarias o requieren costes de licencia fuera del alcance del proyecto.

El stack open source seleccionado replica los principios fundamentales de las implementaciones Zero Trust empresariales: identidad centralizada, autenticación federada, MFA obligatorio, único punto de entrada y segmentación de red. Todos los componentes se integran mediante protocolos abiertos y documentados (LDAP, SAML 2.0, OIDC), garantizando un sistema coherente y auditable.