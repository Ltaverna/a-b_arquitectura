# Arquitectura de Autenticación OpenVPN + PAM + Authelia + Google Workspace (Community)

Este documento describe cómo desplegar un servidor **OpenVPN** sin IP pública, protegido detrás de un **Load Balancer**, utilizando autenticación **PAM → Authelia → Google Workspace OAuth2**.

---

## 📌 Objetivo

- Cumplimiento sin exponer IP pública.  
- MFA con Authelia + Google Workspace.  
- Validación de identidad sin AD.  
- Servidor VPN en subnet de management dentro del Hub.

---

## 🧩 Flujo de Autenticación

```mermaid
flowchart TD
  User[(Cliente VPN)] -->|OpenVPN| OVPN[OpenVPN Server]
  OVPN -->|PAM| PAM[PAM Module]
  PAM --> AUTHELIA[Authelia]
  AUTHELIA --> GOOGLE[Google Workspace OAuth2]
  GOOGLE --> AUTHELIA
  AUTHELIA --> PAM
  PAM --> OVPN
  OVPN --> User
```

---

## 🏗️ Infraestructura

- VM en **subnet de management** dentro del Hub (EU).
- Sin IP pública.
- Acceso expuesto únicamente vía Load Balancer.
- Docker Compose con:
  - openvpn-as
  - authelia
  - pam_oauth2

---

## 📦 docker-compose.yml

```yaml
version: "3.8"

services:
  openvpn:
    image: ghcr.io/linuxserver/openvpn-as:latest
    container_name: openvpn
    network_mode: host
    cap_add:
      - NET_ADMIN
    environment:
      - TZ=Etc/UTC
    volumes:
      - ./openvpn/config:/config
    restart: always

  authelia:
    image: authelia/authelia:latest
    container_name: authelia
    volumes:
      - ./authelia:/config
    ports:
      - "9091:9091"
    restart: unless-stopped

  pam_oauth2:
    image: thde/openvpn-pam-oauth2:latest
    container_name: pam-oauth2
    volumes:
      - ./pam_oauth2:/config
    restart: always
```

---

## 🛠️ Configuración de Authelia

**authelia/configuration.yml**

```yaml
jwt_secret: "super-secret"
default_redirection_url: https://vpn.example.com

authentication_backend:
  file:
    path: /config/users_database.yml

access_control:
  default_policy: deny
  rules:
    - domain: "*.vpn.example.com"
      policy: one_factor

identity_providers:
  oidc:
    hmac_secret: "hmac-secret"
    issuer_private_key: |
      -----BEGIN PRIVATE KEY-----
      ...
      -----END PRIVATE KEY-----
    clients:
      - id: openvpn
        description: "OpenVPN Login"
        secret: "client-secret"
        redirect_uris:
          - https://vpn.example.com/oauth2/callback
        scopes:
          - openid
          - email
          - profile
```

---

## 🔐 PAM

`/etc/pam.d/openvpn`

```bash
auth required pam_exec.so /usr/local/bin/pam-authelia.sh
account required pam_permit.so
```

---

## 🧪 Script PAM → Authelia

`/usr/local/bin/pam-authelia.sh`

```bash
#!/bin/bash
USERNAME="$PAM_USER"
PASSWORD="$PAM_AUTHTOK"

RESPONSE=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -d "{\"username\": \"$USERNAME\", \"password\": \"$PASSWORD\"}" \
  http://127.0.0.1:9091/api/verify)

echo "$RESPONSE" | grep -q '"status":"OK"' && exit 0
exit 1
```

---

## 🌐 Google Workspace OAuth2

1. Ir a **Admin Console**  
   → Apps  
   → Web & mobile apps  
2. Crear app OAuth.  
3. Redirect URI:
   ```
   https://vpn.example.com/oauth2/callback
   ```
4. Copiar:
   - CLIENT_ID  
   - CLIENT_SECRET  
5. Insertar en Authelia.

---

## 🛡️ Load Balancer sin IP pública

### OpenVPN escucha en:
- UDP 1194  
- TCP 443 (opcional para fallback)

### Configuración recomendada:

```
Cliente VPN
   ↓
External/Internal Load Balancer
   ↓
VM OpenVPN (sin IP pública)
```

Firewall:
- LB → VM (UDP/TCP 1194)
- Admin → Authelia 9091 (solo rangos internos)

---

## ✔️ Beneficios para Compliance

- No hay IP pública en la VM.  
- Autenticación delegada al IdP corporativo.  
- MFA integrado.  
- Acceso controlado y auditado.  
- Subnet de management aislada.

---

# 📄 Archivo listo para descargar

