# Manual de Administrador — Sistema Corporativo Zero Trust

**Proyecto:** Zero Trust Corporate System
**Autor:** Asier Barranco
**Fecha:** 12/05/2026
**Versión:** 1.0
**Destinatarios:** Administradores de sistemas

---

## 1. Visión General del Sistema

El Sistema Corporativo Zero Trust es una infraestructura híbrida que implementa el modelo de seguridad Zero Trust. Ningún usuario, dispositivo o red es de confianza por defecto — cada solicitud de acceso se autentica, autoriza y cifra independientemente de su origen.

### 1.1 Resumen de la Arquitectura

El sistema se organiza en tres capas:

| Capa | Función | Componentes |
|---|---|---|
| Capa 1 — Identidad y Control | Directorio de identidad centralizado | Active Directory (DC01) |
| Capa 2 — Servicios y Datos | Aplicaciones corporativas y bases de datos | Nextcloud, Mattermost, MariaDB |
| Capa 3 — Perímetro | Punto de entrada único, pasarela de autenticación | Nginx, Keycloak, Fail2ban, UFW |

**Infraestructura:**
- **On-premise:** Windows Server 2022 con Active Directory — alojado en VirtualBox
- **Nube (AWS):** dos instancias EC2 en una VPC con segmentación de subredes pública/privada
- **Conectividad:** túnel VPN WireGuard que conecta el perímetro AWS con el DC01 on-premise

### 1.2 Topología de Red

| Host | Función | IP Privada | IP Pública |
|---|---|---|---|
| DC01 | Active Directory, DNS, DHCP | 192.168.56.10 | Ninguna |
| ztcs-perimeter | Nginx, Keycloak, Fail2ban, UFW | 10.0.1.9 | 32.197.108.153 |
| ztcs-services | Nextcloud, Mattermost, MariaDB | 10.0.2.220 | Ninguna |

**Dominio:** `abb-ztcs.com` → `32.197.108.153` (DNS IONOS)
**Subred VPN:** `10.10.0.0/24` (WireGuard)
**Dominio AD:** `corp.zerotrust.local`

### 1.3 Estructura del Repositorio

```
zero-trust-corporate-system/
├── architecture/          # Diagramas de red
├── docs/                  # Documentación del proyecto (en + es)
├── infrastructure/        # Guías de despliegue por componente
│   ├── aws/
│   ├── domain-controller/
│   ├── identity-provider/
│   ├── reverse-proxy/
│   ├── services/
│   └── vpn/
├── media/                 # Capturas de pantalla y evidencias
├── security/              # Ejercicios Purple Team
├── sprints/               # Documentación de sprints SCRUM
└── tests/                 # Planes de pruebas y resultados
```

---

## 2. Referencia de Componentes

### 2.1 Active Directory (DC01)

**Guía de despliegue:** `infrastructure/domain-controller/dc-setup.md`

| Parámetro | Valor |
|---|---|
| SO | Windows Server 2022 |
| Dominio | `corp.zerotrust.local` |
| IP | `192.168.56.10` |
| Administrador | `CORP\zt-administrator` |
| Puerto LDAP | 389 |

**Unidades Organizativas:**
- `OU=ZeroTrust Users` — cuentas de usuario corporativas
- `OU=ZeroTrust Computers` — estaciones de trabajo
- `OU=ZeroTrust Groups` — grupos de seguridad

**Usuarios principales:**

| Usuario | Función |
|---|---|
| alice.smith | Usuario corporativo estándar |
| bob.jones | Usuario corporativo estándar |
| zt.admin | Administrador de TI |
| svc.keycloak | Cuenta de servicio LDAP de Keycloak (solo lectura) |

**Políticas GPO aplicadas:**
- Política de contraseñas: mínimo 12 caracteres, complejidad requerida, caducidad 90 días
- Bloqueo de cuenta: 5 intentos fallidos → bloqueo 30 minutos
- Auditoría de seguridad: eventos de inicio de sesión, gestión de cuentas, uso de privilegios

**Iniciar/detener DC01:**
- Iniciar la VM `DC01` en VirtualBox
- Iniciar sesión como `CORP\zt-administrator`
- Verificar servicios AD: `Get-Service NTDS, DNS, Netlogon`

---

### 2.2 VPN WireGuard

**Guía de despliegue:** `infrastructure/vpn/wireguard-setup.md`

El túnel WireGuard conecta `ztcs-perimeter` (AWS) con `DC01` (on-premise), permitiendo que Keycloak acceda a Active Directory vía LDAP.

**Configuración perímetro:** `infrastructure/vpn/wg0-perimeter.conf`
**Configuración DC01:** `infrastructure/vpn/wg0-dc01.conf`

**Verificar estado del túnel (ztcs-perimeter):**
```bash
sudo wg show
```

**Verificar accesibilidad LDAP:**
```bash
nc -zv 192.168.56.10 389
```

**Reiniciar WireGuard (ztcs-perimeter):**
```bash
sudo systemctl restart wg-quick@wg0
```

> **Importante:** el túnel WireGuard debe estar activo en todo momento para que la autenticación funcione. Si el túnel está caído, Keycloak no puede validar credenciales contra AD y todos los inicios de sesión fallarán.

---

### 2.3 Infraestructura AWS

**Guía de despliegue:** `infrastructure/aws/aws-setup.md`

**VPC:** `10.0.0.0/16`

| Subred | CIDR | Tipo | Hosts |
|---|---|---|---|
| Subred pública | `10.0.1.0/24` | Pública (Internet Gateway) | ztcs-perimeter |
| Subred privada | `10.0.2.0/24` | Privada (sin internet) | ztcs-services |

**Security Groups:**

| Grupo | Reglas de entrada |
|---|---|
| sg-perimeter | 22/tcp, 80/tcp, 443/tcp, 51820/udp desde 0.0.0.0/0 |
| sg-services | Todo el tráfico solo desde sg-perimeter |

**Acceso SSH a ztcs-perimeter:**
```bash
ssh -i /ruta/labsuser.pem ubuntu@32.197.108.153
```

**Acceso SSH a ztcs-services (via jump host):**
```bash
eval $(ssh-agent) && ssh-add /ruta/labsuser.pem
ssh -A -J ubuntu@32.197.108.153 ubuntu@10.0.2.220
```

> **Gestión de costes:** detener las instancias EC2 cuando no estén en uso para preservar los créditos AWS. Detener siempre `ztcs-services` primero, luego `ztcs-perimeter`.

---

### 2.4 Nginx — Proxy Inverso

**Guía de despliegue:** `infrastructure/reverse-proxy/nginx-setup.md`
**Fichero de configuración:** `infrastructure/reverse-proxy/abb-ztcs.com.conf`

Nginx es el único punto de entrada para todo el tráfico externo. Termina TLS y redirige a los servicios internos.

**Tabla de enrutamiento:**

| Ruta | Redirige a | Servicio |
|---|---|---|
| `https://abb-ztcs.com/` | `http://10.0.2.220:8080` | Nextcloud |
| `https://abb-ztcs.com/auth/` | `http://localhost:8080` | Keycloak |
| `https://abb-ztcs.com/mattermost/` | `http://10.0.2.220:8065` | Mattermost |

**Comandos principales:**
```bash
sudo systemctl status nginx
sudo systemctl restart nginx
sudo nginx -t                          # Verificar sintaxis de configuración
sudo certbot renew --dry-run           # Probar renovación automática del certificado
```

**Certificado TLS:**
- Proveedor: Let's Encrypt (Certbot)
- Dominio: `abb-ztcs.com`
- Caducidad: 2026-08-07
- Renovación automática: configurada mediante temporizador systemd

---

### 2.5 Keycloak — Proveedor de Identidad

**Guía de despliegue:** `infrastructure/identity-provider/keycloak-setup.md`
**Docker Compose:** `infrastructure/identity-provider/docker-compose.yml`

| Parámetro | Valor |
|---|---|
| Versión | 24.0.4 |
| URL de administración | `https://abb-ztcs.com/auth/` (navegador) o `http://localhost:8080/auth/` (túnel SSH) |
| Credenciales admin | `admin` / `KeycloakAdmin2026!` |
| Realm | `zerotrust` |
| Proveedor LDAP | `corp.zerotrust.local` vía `192.168.56.10:389` |
| Cliente SAML | `https://abb-ztcs.com` (Nextcloud) |

**Comandos principales (ztcs-perimeter):**
```bash
docker ps | grep keycloak
docker logs keycloak --tail 50
docker restart keycloak
cd ~/keycloak && docker compose up -d
```

**Acceso a la consola admin vía túnel SSH:**
```bash
ssh -i /ruta/labsuser.pem -L 8080:localhost:8080 ubuntu@32.197.108.153
# Luego abrir http://localhost:8080/auth/ en el navegador
```

**Sincronizar usuarios LDAP manualmente:**
Consola admin → realm `zerotrust` → Federación de usuarios → `ldap` → **Sincronizar todos los usuarios**

**Restablecer MFA de un usuario:**
Consola admin → realm `zerotrust` → Usuarios → seleccionar usuario → Credenciales → eliminar entrada OTP

---

### 2.6 Fail2ban y UFW — Protección Perimetral

**Guía de despliegue:** `infrastructure/reverse-proxy/perimeter-protection-setup.md`
**Configuración:** `infrastructure/reverse-proxy/jail.local` y `infrastructure/reverse-proxy/keycloak.conf`

**Jails activos:**

| Jail | Disparador | Duración del ban |
|---|---|---|
| sshd | 3 intentos SSH fallidos | 1 hora |
| keycloak | 5 eventos LOGIN_ERROR via journald | 30 minutos |
| nginx-http-auth | Autenticación HTTP fallida | 1 hora |
| nginx-botsearch | Patrones de escaneo bot | 1 hora |

**Comandos principales:**
```bash
sudo fail2ban-client status
sudo fail2ban-client status keycloak
sudo fail2ban-client set keycloak unbanip <IP>
sudo grep "Ban" /var/log/fail2ban.log | tail -20
```

**Puertos abiertos en UFW:**
```
22/tcp    SSH
80/tcp    HTTP (redirigido a HTTPS)
443/tcp   HTTPS
51820/udp WireGuard VPN
```

---

### 2.7 Nextcloud y Mattermost — Servicios Corporativos

**Guía de despliegue:** `infrastructure/services/services-setup.md`
**Guía SSO:** `infrastructure/services/sso-nextcloud-setup.md`
**Docker Compose:** `infrastructure/services/docker-compose.yml`

Todos los servicios se ejecutan como contenedores Docker en `ztcs-services`.

**Comandos principales (ztcs-services):**
```bash
docker ps
docker compose -f ~/services/docker-compose.yml up -d
docker compose -f ~/services/docker-compose.yml down
docker logs nextcloud --tail 50
docker logs mattermost --tail 50
docker logs mariadb --tail 50
```

**Comandos occ de Nextcloud:**
```bash
docker exec -u 33 -it nextcloud php occ user:list
docker exec -u 33 -it nextcloud php occ app:list
docker exec -u 33 -it nextcloud php occ saml:config:get --providerId=1
```

**Emergencia — deshabilitar SAML (si se pierde el acceso):**
```bash
docker exec -u 33 -it nextcloud php occ app:disable user_saml
# Acceso restaurado en https://abb-ztcs.com/login?direct=1
```

**Acceso admin local (sin SSO):**
```
https://abb-ztcs.com/login?direct=1
Credenciales: nc_admin / AdminZeroTrust2026!
```

---

### 2.8 MariaDB — Base de Datos

MariaDB se ejecuta como contenedor Docker en `ztcs-services` y es utilizado exclusivamente como backend por Nextcloud y Mattermost. No tiene exposición pública.

```bash
# Acceder a la shell de MariaDB
docker exec -it mariadb mariadb -u root -p
# Contraseña: MariaZeroTrust2026!
```

**Bases de datos:**

| Base de datos | Usuario | Utilizada por |
|---|---|---|
| nextcloud | nextcloud_user | Nextcloud |
| mattermost | mattermost_user | Mattermost |

---

## 3. Procedimientos Operativos

### 3.1 Arranque Completo del Sistema

Sigue este orden para iniciar el sistema desde cero:

1. **Iniciar DC01** en VirtualBox — esperar a que los servicios AD se inicialicen (~2 minutos)
2. **Iniciar ztcs-perimeter** en la consola AWS
3. **Iniciar ztcs-services** en la consola AWS
4. **Verificar túnel WireGuard** desde ztcs-perimeter: `nc -zv 192.168.56.10 389`
5. **Verificar todos los servicios:** `docker ps` en ambas instancias
6. **Verificar endpoint público:** `curl -s -o /dev/null -w "%{http_code}\n" https://abb-ztcs.com`

### 3.2 Parada Completa del Sistema

1. Detener la instancia EC2 ztcs-services
2. Detener la instancia EC2 ztcs-perimeter
3. Apagar la VM DC01 en VirtualBox

### 3.3 Añadir un Nuevo Usuario

1. Crear usuario en Active Directory (DC01):
```powershell
New-ADUser -Name "Juan García" -SamAccountName "juan.garcia" `
  -UserPrincipalName "juan.garcia@corp.zerotrust.local" `
  -AccountPassword (ConvertTo-SecureString "Contraseña123!" -AsPlainText -Force) `
  -Enabled $true -Path "OU=ZeroTrust Users,DC=corp,DC=zerotrust,DC=local"
```

2. Sincronizar usuarios en Keycloak: Consola admin → Federación de usuarios → `ldap` → **Sincronizar todos los usuarios**

3. El usuario ya puede iniciar sesión en `https://abb-ztcs.com` — la cuenta de Nextcloud se crea automáticamente en el primer inicio de sesión.

### 3.4 Renovación del Certificado TLS

Certbot renueva automáticamente mediante temporizador systemd. Para renovar manualmente:

```bash
sudo certbot renew
sudo systemctl reload nginx
```

Verificar caducidad:
```bash
echo | openssl s_client -connect abb-ztcs.com:443 2>/dev/null | openssl x509 -noout -dates
```

### 3.5 Consulta de Logs

| Componente | Comando |
|---|---|
| Nginx acceso | `sudo tail -f /var/log/nginx/access.log` |
| Nginx errores | `sudo tail -f /var/log/nginx/error.log` |
| Keycloak | `docker logs keycloak -f` o `journalctl CONTAINER_TAG=keycloak -f` |
| Fail2ban | `sudo tail -f /var/log/fail2ban.log` |
| Nextcloud | `docker exec -it nextcloud cat data/nextcloud.log \| tail -50` |
| MariaDB | `docker logs mariadb --tail 50` |

### 3.6 Copias de Seguridad

**Datos de Nextcloud:**
```bash
docker exec -it nextcloud tar czf /tmp/nextcloud-backup.tar.gz /var/www/html/data
docker cp nextcloud:/tmp/nextcloud-backup.tar.gz ~/backups/
```

**MariaDB:**
```bash
docker exec mariadb mariadb-dump -u root -pMariaZeroTrust2026! --all-databases > ~/backups/mariadb-$(date +%Y%m%d).sql
```

**Configuración del realm Keycloak:**
```bash
TOKEN=$(curl -s -X POST http://localhost:8080/auth/realms/master/protocol/openid-connect/token \
  -d "username=admin&password=KeycloakAdmin2026%21&grant_type=password&client_id=admin-cli" \
  | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/auth/admin/realms/zerotrust > ~/backups/keycloak-realm-$(date +%Y%m%d).json
```

---

## 4. Resolución de Problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| Login falla con "Error inesperado" | DC01 apagado o túnel WireGuard caído | Iniciar DC01, verificar `nc -zv 192.168.56.10 389` |
| `https://abb-ztcs.com` inaccesible | Nginx parado o instancia EC2 apagada | `sudo systemctl restart nginx` o iniciar EC2 |
| Keycloak 502 Bad Gateway | Contenedor Keycloak no arrancado | `docker ps` → `docker compose up -d` en `~/keycloak/` |
| Nextcloud muestra página en blanco | Contenedor caído o sin memoria | `docker restart nextcloud` |
| IP bloqueada por Fail2ban | Demasiados intentos fallidos | `sudo fail2ban-client set <jail> unbanip <IP>` |
| Usuario no provisionado en Nextcloud | Problema de mapeo SAML uid | Verificar `occ saml:config:get --providerId=1` |
| Usuario bloqueado en AD | Demasiados intentos de inicio de sesión fallidos | `Unlock-ADAccount -Identity <usuario>` en DC01 |
| Certificado caducado | Renovación automática fallida | `sudo certbot renew && sudo systemctl reload nginx` |

---

## 5. Consideraciones de Seguridad

| Área | Implementación | Notas |
|---|---|---|
| Autenticación SSH | Solo clave — autenticación por contraseña deshabilitada | Previene fuerza bruta a nivel de protocolo |
| Aislamiento de red | ztcs-services sin IP pública ni internet | Servicios solo accesibles vía perímetro |
| TLS | Let's Encrypt — renovación automática | Todo el tráfico cifrado en tránsito |
| MFA | TOTP obligatorio para todos los usuarios | Las credenciales robadas solas son insuficientes |
| Prevención de intrusiones | Fail2ban con 4 jails | Baneo dinámico de IPs vía UFW |
| Tráfico LDAP | Transmitido sobre túnel WireGuard | VPN cifrada punto a punto |
| Seguridad de sesión | Cookies HttpOnly, Secure, SameSite | Mitiga ataques XSS y CSRF |

---

## 6. Índice de Documentación

| Documento | Ruta | Contenido |
|---|---|---|
| Configuración AWS | `infrastructure/aws/aws-setup.md` | VPC, subredes, EC2, Security Groups |
| DNS y Elastic IP | `infrastructure/aws/dns-elastic-ip-setup.md` | Configuración del dominio |
| Controlador de dominio | `infrastructure/domain-controller/dc-setup.md` | AD, GPO, usuarios, grupos |
| VPN WireGuard | `infrastructure/vpn/wireguard-setup.md` | Configuración del túnel VPN |
| Keycloak | `infrastructure/identity-provider/keycloak-setup.md` | IdP, federación LDAP, MFA |
| Nginx | `infrastructure/reverse-proxy/nginx-setup.md` | Proxy inverso, TLS |
| Protección perimetral | `infrastructure/reverse-proxy/perimeter-protection-setup.md` | Fail2ban, UFW |
| Despliegue de servicios | `infrastructure/services/services-setup.md` | Nextcloud, Mattermost, MariaDB |
| Integración SSO | `infrastructure/services/sso-nextcloud-setup.md` | Configuración SAML SSO |
| Pruebas de red | `tests/connectivity-tests.md` | Validación de aislamiento y enrutamiento |
| Pruebas SSO | `tests/sso-validation.md` | Validación de autenticación |
| Plan de pruebas | `tests/test-plan.md` | Resumen completo de pruebas |
| Purple Team | `security/purple-team.md` | Simulaciones de ataque y respuesta a incidentes |
| Manual de usuario | `docs/es/manual-usuario.md` | Guía para usuarios finales |