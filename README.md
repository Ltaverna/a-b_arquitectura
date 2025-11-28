Arquitectura Hub & Spoke Multi-Región + OpenVPN con MFA vía Authelia (Google Workspace)
📌 Objetivos

Separar ambientes por proyecto (dev / qa / prod).

Aislar aplicaciones por región:

App1 → Montreal (northamerica-northeast1)

App2 → Europa (europe-westX)

Cumplir políticas de residencia de datos en la UE.

Usar Hub & Spoke con Shared VPC.

Añadir subnet de Management con:

Jumpbox/Bastion

OpenVPN Server sin IP pública (ingreso sólo por Internal Load Balancer)

Autenticación para VPN con Google Workspace usando:

OpenVPN → PAM → Authelia → Google Workspace

1. Arquitectura General (Hub & Spoke Multi-Región)
flowchart TB

  subgraph ORG["Organización GCP"]
  
    %%================== HUB NA ==================
    subgraph HUB_NA["Hub NA - Shared VPC (northamerica-northeast1)"]
      NA_VPC[(VPC vpc-hub-na)]
      NA_APP_SUBNET[[subnet-na-app]]
      NA_MGMT_SUBNET[[subnet-na-mgmt\nJumpbox / Optional VPN]]
    end

    %%================== HUB EU ==================
    subgraph HUB_EU["Hub EU - Shared VPC (europe-westX)"]
      EU_VPC[(VPC vpc-hub-eu)]
      EU_APP_SUBNET[[subnet-eu-app]]
      EU_MGMT_SUBNET[[subnet-eu-mgmt\nJumpbox/Bastion\nOpenVPN Server]]
    end

    %%================== SPOKES NA ==================
    subgraph SPOKES_NA["Spokes App1 NA"]
      NA_DEV[(app1-na-dev)]
      NA_QA[(app1-na-qa)]
      NA_PROD[(app1-na-prod)]
    end

    %%================== SPOKES EU ==================
    subgraph SPOKES_EU["Spokes App2 EU"]
      EU_DEV[(app2-eu-dev)]
      EU_QA[(app2-eu-qa)]
      EU_PROD[(app2-eu-prod)]
    end

    %%================== GESTIÓN ==================
    DEV_PC[[Laptops Devs]]
    LB_INT[(Internal TCP LB)]
    OPENVPN[(VM OpenVPN\nsin IP Pública)]
    AUTHELIA[(Authelia Container)]
    
  end
  
  %% RELACIONES HUB-SPOKE
  NA_VPC --- NA_DEV
  NA_VPC --- NA_QA
  NA_VPC --- NA_PROD

  EU_VPC --- EU_DEV
  EU_VPC --- EU_QA
  EU_VPC --- EU_PROD

  %% MANAGEMENT
  DEV_PC --> LB_INT --> OPENVPN
  OPENVPN --> AUTHELIA

  classDef eu fill:#e3ffe3,stroke:#008000;
  class EU_VPC,EU_APP_SUBNET,EU_MGMT_SUBNET,EU_DEV,EU_QA,EU_PROD eu;

2. Subnet de Management

Cada región tiene su propia subnet de gestión:

Región	Subnet	Contiene
NA	subnet-na-mgmt	Jumpbox/Bastion
EU	subnet-eu-mgmt	Bastion + OpenVPN Server sin IP pública

La de Europa es crítica por las políticas de residencia de datos.

3. OpenVPN sin IP Pública usando Internal Load Balancer
Cómo funciona

OpenVPN está en una VM privada dentro de subnet-eu-mgmt.

No tiene IP pública por compliance.

Se expone usando un Internal TCP Load Balancer.

Los desarrolladores acceden vía:

Cloud VPN On-Prem → VPC EU

o

Cloud Interconnect

o

Otra VPN corporativa

El LB redirige el tráfico a la VM OpenVPN.

4. Autenticación: OpenVPN → PAM → Authelia → Google Workspace

Este pipeline permite:

MFA (TOTP o Push)

Login con credenciales de Google Workspace

Sin necesidad de Active Directory

User management centralizado en Google

Flujo de autenticación
Client → OpenVPN → PAM → Authelia API → OAuth (Google) → MFA → Authelia → OpenVPN

5. Docker Compose para Authelia dentro de la VM (EU Management)
version: "3.8"

services:
  authelia:
    image: authelia/authelia:latest
    container_name: authelia
    volumes:
      - ./authelia:/config
    ports:
      - "9091:9091"
    restart: unless-stopped

  redis:
    image: redis:6
    container_name: redis
    restart: unless-stopped

6. Archivo configuration.yml de Authelia (Google Workspace OAuth)
server:
  host: 0.0.0.0
  port: 9091

authentication_backend:
  password:
    algorithm: argon2id

session:
  name: authelia_session
  secret: "REPLACE_ME"
  expiration: 3600
  domain: example.com

identity_providers:
  oidc:
    enabled: true
    clients:
      - id: "openvpn"
        description: "OpenVPN Login"
        secret: "REPLACE_ME"
        redirect_uris:
          - "http://openvpn.local/auth/callback"
    providers:
      google:
        client_id: "GOOGLE_CLIENT_ID"
        client_secret: "GOOGLE_CLIENT_SECRET"
        scope: "openid email profile"

access_control:
  default_policy: deny
  rules:
    - domain: "*.vpn.example.com"
      policy: two_factor

7. PAM + Authelia (OpenVPN)

Archivo /etc/pam.d/openvpn:

auth     requisite pam_exec.so debug stdout /etc/openvpn/scripts/pam-authelia.sh
account  required  pam_permit.so


Archivo /etc/openvpn/scripts/pam-authelia.sh:

#!/bin/bash

USERNAME=$PAM_USER
PASSWORD=$(cat -)

RESULT=$(curl -s -X POST http://127.0.0.1:9091/api/auth/verify \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"$USERNAME\",\"password\":\"$PASSWORD\"}")

echo $RESULT | grep -q '"status":"OK"'
exit $?

8. Configuración del Internal Load Balancer (ILB)
Backend (OpenVPN VM)
Port: 1194 (TCP)
Health check: TCP 1194

Frontend (ILB)
IP: internal (RFC1918)
Region: europe-westX
Subnet: subnet-eu-mgmt


Los devs deben llegar a esta red mediante:

VPN corporativa

Interconnect

Otra red autorizada

9. Configuración de OpenVPN Server

Archivo /etc/openvpn/server.conf:

port 1194
proto tcp
dev tun
user nobody
group nogroup

auth-user-pass-verify /etc/openvpn/scripts/check-pam.sh via-env
plugin /usr/lib/openvpn/openvpn-plugin-auth-pam.so openvpn

client-cert-not-required
username-as-common-name

server 10.9.0.0 255.255.255.0
push "dhcp-option DNS 10.20.0.2"
push "redirect-gateway def1 bypass-dhcp"

persist-key
persist-tun
keepalive 10 120
cipher AES-256-GCM

10. Diagrama de Flujo de Autenticación VPN
sequenceDiagram
  participant DEV as Usuario
  participant LB as Internal LB
  participant VPN as OpenVPN Server
  participant PAM as PAM
  participant AUT as Authelia
  participant GWS as Google Workspace

  DEV->>LB: Conexión VPN (tcp/1194)
  LB->>VPN: Traffic forwarding
  VPN->>PAM: auth-user-pass
  PAM->>AUT: verify(username, password)
  AUT->>GWS: OAuth login
  GWS-->>AUT: Token + MFA OK
  AUT-->>PAM: OK
  PAM-->>VPN: success
  VPN-->>DEV: Tunnel established

11. Resumen Final

✔ Hub & Spoke en NA y EU
✔ Subnets separadas por región
✔ Subnet de Management con Jumpbox + OpenVPN
✔ OpenVPN sin IP pública usando Internal Load Balancer
✔ Autenticación segura con Google Workspace vía Authelia
✔ MFA obligatorio
✔ Cumplimiento con normas de residencia de datos en la UE
