# Arquitectura Hub & Spoke multi-región en GCP (NA + EU)

## Objetivos

- Separar ambientes por proyecto: **dev / qa / prod** para cada aplicación.
- Separar redes por **región** y **zona geográfica**:
  - App1 en **northamerica-northeast1 (Montreal)**.
  - App2 en alguna región de **Europa** (ej. `europe-west1`).
- Cumplir políticas de **residencia de datos en Europa**:  
  - Los datos de App2 (EU) **no pueden almacenarse** ni procesarse de forma persistente en Norteamérica.
- Estandarizar el modelo de red con **Hub & Spoke** usando **Shared VPC**.
- Centralizar:
  - Seguridad de red (firewall, Cloud NAT, Cloud VPN/Interconnect).
  - Gestión / administración (**management subnet + jumpbox/bastion/OpenVPN**).

---

## Topología de proyectos (ejemplo)

> Ajustá los IDs según tu naming convention.

### Región Norteamérica (Montreal)

- **Hub NA**
  - `prj-net-na-hub` → Host de **Shared VPC** para NA.
- **Spokes App1 NA**
  - `prj-app1-na-dev`
  - `prj-app1-na-qa`
  - `prj-app1-na-prod`

### Región Europa

- **Hub EU**
  - `prj-net-eu-hub` → Host de **Shared VPC** para EU.
- **Spokes App2 EU**
  - `prj-app2-eu-dev`
  - `prj-app2-eu-qa`
  - `prj-app2-eu-prod`

Opcionalmente podés tener proyectos dedicados para **logging / seguridad**, pero en muchos casos se comparte el hub.

---

## Topología de red (alto nivel)

- **Hub NA**
  - VPC `vpc-hub-na`
    - Subnet `subnet-na-app` (rango grande, reserveado para workloads App1).
    - Subnet `subnet-na-mgmt` (**management**: jumpbox/bastion/OpenVPN).
- **Hub EU**
  - VPC `vpc-hub-eu`
    - Subnet `subnet-eu-app` (workloads App2).
    - Subnet `subnet-eu-mgmt` (**management**: jumpbox/bastion/OpenVPN, sólo en EU).
- **Spokes** (service projects) no crean VPC propia, consumen la Shared VPC del hub.

> **Residencia de datos EU**  
> - App2 y sus bases de datos/logs se despliegan SOLO en `vpc-hub-eu` + proyectos EU.  
> - No se permite escribir datos EU en recursos NA.  
> - Sólo se permite:
>   - Tráfico de **control / gestión** global muy controlado (por ejemplo, SCC, IAM, métricas agregadas).
>   - Tráfico opcional de lectura desde EU hacia NA si cumple política (por ej. sólo metadatos).

---

## Diagrama lógico (Mermaid)

Podés pegar esto en un visor Mermaid.

```mermaid
flowchart LR
  %% ===================== ORGANIZACION =====================
  subgraph ORG["Org GCP"]

    %% ---------- HUB NA ----------
    subgraph HUB_NA["Hub NA (prj-net-na-hub, northamerica-northeast1)"]
      NA_VPC[(VPC vpc-hub-na)]
      subgraph NA_SUBNETS["Subnets NA"]
        NA_APP_SUBNET[[subnet-na-app\nWorkloads App1]]
        NA_MGMT_SUBNET[[subnet-na-mgmt\nJumpbox/Bastion\n(OpenVPN opcional)]]
      end
    end

    %% ---------- HUB EU ----------
    subgraph HUB_EU["Hub EU (prj-net-eu-hub, europe-westX)"]
      EU_VPC[(VPC vpc-hub-eu)]
      subgraph EU_SUBNETS["Subnets EU"]
        EU_APP_SUBNET[[subnet-eu-app\nWorkloads App2]]
        EU_MGMT_SUBNET[[subnet-eu-mgmt\nJumpbox/Bastion\nOpenVPN Server]]
      end
    end

    %% ---------- SPOKES NA ----------
    subgraph SPOKES_NA["Spokes NA - Shared VPC"]
      subgraph APP1_DEV["prj-app1-na-dev"]
        APP1_DEV_SA[(GKE/Run/VMs)]
      end
      subgraph APP1_QA["prj-app1-na-qa"]
        APP1_QA_SA[(GKE/Run/VMs)]
      end
      subgraph APP1_PROD["prj-app1-na-prod"]
        APP1_PROD_SA[(GKE/Run/VMs)]
      end
    end

    %% ---------- SPOKES EU ----------
    subgraph SPOKES_EU["Spokes EU - Shared VPC"]
      subgraph APP2_DEV["prj-app2-eu-dev"]
        APP2_DEV_SA[(GKE/Run/VMs)]
      end
      subgraph APP2_QA["prj-app2-eu-qa"]
        APP2_QA_SA[(GKE/Run/VMs)]
      end
      subgraph APP2_PROD["prj-app2-eu-prod"]
        APP2_PROD_SA[(GKE/Run/VMs)]
      end
    end

    %% ---------- MANAGEMENT ACCESS ----------
    DEV_PC[[Desarrolladores\nLaptop]]
    ONPREM_DC[(OnPrem DC\n(optional))]

  end

  %% ----- ENLACES HUB <-> SPOKES -----
  NA_VPC --- APP1_DEV_SA
  NA_VPC --- APP1_QA_SA
  NA_VPC --- APP1_PROD_SA

  EU_VPC --- APP2_DEV_SA
  EU_VPC --- APP2_QA_SA
  EU_VPC --- APP2_PROD_SA

  %% ----- MANAGEMENT/JUMPBOX/OPENVPN -----
  NA_MGMT_SUBNET --- DEV_PC
  EU_MGMT_SUBNET --- DEV_PC
  EU_MGMT_SUBNET --- ONPREM_DC

  %% ----- NOTA DE CUMPLIMIENTO -----
  classDef euData fill=#e3ffe3,stroke=#008000,stroke-width=1px;
  class EU_VPC,EU_APP_SUBNET,EU_MGMT_SUBNET,APP2_DEV_SA,APP2_QA_SA,APP2_PROD_SA euData;
```

---

## Management: Jumpbox/Bastion vs OpenVPN

### Subnet de Management

- Crear una subnet dedicada por región:
  - `subnet-na-mgmt` en `vpc-hub-na` (Montreal).
  - `subnet-eu-mgmt` en `vpc-hub-eu` (Europa).
- En esta subnet viven:
  - **Jumpbox/Bastion** (acceso SSH/RDP vía IAP o VPN).
  - Opcionalmente, una **VM OpenVPN / WireGuard** si querés que los devs tengan IP interna y DNS corporativo.

### ¿Jumpbox/Bastion o OpenVPN?

- **Solo Jumpbox/Bastion (recomendado como base):**
  - Devs se conectan vía:
    - IAP TCP → SSH al bastion.
    - O Cloud VPN desde on-prem al hub.
  - Desde bastion hacen:
    - SSH a VMs internas.
    - Port-forwarding a servicios internos (p.ej. GKE dashboards internos).
  - DNS interno: bastion puede usar **Cloud DNS** (private zones) y los devs usan `ssh -D` o `sshuttle` para resolver a través del bastion.

- **OpenVPN/WireGuard (cuando devs necesitan IP interna + DNS):**
  - VM `openvpn-eu` en `subnet-eu-mgmt`.
  - Asigna IPs a los clientes desde un rango `10.x.y.0/24` exclusivo.
  - El servidor VPN configura el **DNS de los clientes** hacia Cloud DNS (resolviendo nombres internos).
  - Útil si los devs necesitan consumir APIs internas desde su laptop como si estuvieran “dentro de la VPC”.

> Para cumplir con políticas EU, el **VPN Server para App2** debería estar en **Europa** (`subnet-eu-mgmt`), de modo que el tráfico de administración entre devs y App2 se termine en Europa.

---

## Seguridad y cumplimiento (GCP Security Command Center / políticas EU)

Este diseño te ayuda a aprobar revisiones de seguridad porque:

1. **Separación clara NA / EU**
   - Distintos proyectos, distintas VPCs, distintas subnets.
   - No hay bases de datos/configs de App2 en NA.
2. **Shared VPC por región**
   - Seguridad de red centralizada en `prj-net-na-hub` y `prj-net-eu-hub`.
   - Reglas de firewall, Cloud NAT y Cloud VPN centralizadas.
3. **Residencia de datos EU**
   - Todos los recursos de datos de App2 (DB, buckets, Pub/Sub, BigQuery, etc.) se crean en regiones EU.
   - Logs de App2 se envían a **Cloud Logging buckets regionales en EU**.
4. **Gestión y acceso remoto**
   - Subnet de `management` separada con reglas muy estrictas.
   - Bastion / OpenVPN con MFA + IAM + short-lived credentials.
5. **SCC (Security Command Center)**
   - Habilitás SCC a nivel organización.
   - Configurás **scanners** para:
     - Exposición pública de recursos.
     - Buckets sin cifrado.
     - Reglas de firewall muy abiertas.
   - Para cumplir políticas EU, documentás que **cualquier hallazgo relacionado a App2** se remedia en **infraestructura exclusivamente europea**.

---

## Terraform de ejemplo

> Nota: Es un **ejemplo simplificado** para que tengas el esqueleto.  
> Ajustá rangos, nombres, etiquetas y módulos según tu estándar.

### 1. Variables

```hcl
variable "org_id" {
  type        = string
  description = "Organization ID"
}

variable "na_hub_project_id" {
  type        = string
  description = "Hub (Shared VPC host) project for North America"
}

variable "eu_hub_project_id" {
  type        = string
  description = "Hub (Shared VPC host) project for Europe"
}

variable "app1_na_projects" {
  description = "Service projects for App1 NA (dev/qa/prod)"
  type        = list(string)
}

variable "app2_eu_projects" {
  description = "Service projects for App2 EU (dev/qa/prod)"
  type        = list(string)
}
```

---

### 2. Redes Hub (NA y EU)

```hcl
resource "google_compute_network" "vpc_hub_na" {
  project                 = var.na_hub_project_id
  name                    = "vpc-hub-na"
  auto_create_subnetworks = false
}

resource "google_compute_network" "vpc_hub_eu" {
  project                 = var.eu_hub_project_id
  name                    = "vpc-hub-eu"
  auto_create_subnetworks = false
}

# Subnets NA
resource "google_compute_subnetwork" "na_app" {
  project                  = var.na_hub_project_id
  name                     = "subnet-na-app"
  ip_cidr_range            = "10.10.0.0/20"
  region                   = "northamerica-northeast1"
  network                  = google_compute_network.vpc_hub_na.id
  private_ip_google_access = true
}

resource "google_compute_subnetwork" "na_mgmt" {
  project                  = var.na_hub_project_id
  name                     = "subnet-na-mgmt"
  ip_cidr_range            = "10.10.16.0/24"
  region                   = "northamerica-northeast1"
  network                  = google_compute_network.vpc_hub_na.id
  private_ip_google_access = true
}

# Subnets EU
resource "google_compute_subnetwork" "eu_app" {
  project                  = var.eu_hub_project_id
  name                     = "subnet-eu-app"
  ip_cidr_range            = "10.20.0.0/20"
  region                   = "europe-west1"
  network                  = google_compute_network.vpc_hub_eu.id
  private_ip_google_access = true
}

resource "google_compute_subnetwork" "eu_mgmt" {
  project                  = var.eu_hub_project_id
  name                     = "subnet-eu-mgmt"
  ip_cidr_range            = "10.20.16.0/24"
  region                   = "europe-west1"
  network                  = google_compute_network.vpc_hub_eu.id
  private_ip_google_access = true
}
```

---

### 3. Shared VPC Host y Service Projects

```hcl
# Hub NA como host
resource "google_compute_shared_vpc_host_project" "na_host" {
  project = var.na_hub_project_id
}

# Hub EU como host
resource "google_compute_shared_vpc_host_project" "eu_host" {
  project = var.eu_hub_project_id
}

# Service projects App1 NA
resource "google_compute_shared_vpc_service_project" "app1_na" {
  for_each = toset(var.app1_na_projects)

  host_project    = var.na_hub_project_id
  service_project = each.value
}

# Service projects App2 EU
resource "google_compute_shared_vpc_service_project" "app2_eu" {
  for_each = toset(var.app2_eu_projects)

  host_project    = var.eu_hub_project_id
  service_project = each.value
}
```

---

### 4. Firewall básico (ejemplo)

```hcl
# SSH/IAP desde internet al bastion NA (si lo usás)
resource "google_compute_firewall" "na_allow_ssh_bastion" {
  project = var.na_hub_project_id
  name    = "fw-na-allow-ssh-bastion"
  network = google_compute_network.vpc_hub_na.name

  direction = "INGRESS"
  priority  = 1000

  target_tags = ["bastion-na"]

  source_ranges = ["35.235.240.0/20"] # rangos IAP o tu rango corporativo

  allows {
    protocol = "tcp"
    ports    = ["22"]
  }
}

# SSH/VPN a bastion/OpenVPN EU (desde on-prem o devs)
resource "google_compute_firewall" "eu_allow_mgmt" {
  project = var.eu_hub_project_id
  name    = "fw-eu-allow-mgmt"
  network = google_compute_network.vpc_hub_eu.name

  direction = "INGRESS"
  priority  = 1000

  target_tags = ["bastion-eu", "openvpn-eu"]

  source_ranges = [
    "x.x.x.x/32", # IPs públicas corporativas
  ]

  allows {
    protocol = "tcp"
    ports    = ["22", "1194"] # 1194 UDP/TCP para OpenVPN (ajustar)
  }
}
```

---

### 5. VM Bastion / Jumpbox (ejemplo en EU)

```hcl
resource "google_compute_instance" "bastion_eu" {
  project      = var.eu_hub_project_id
  name         = "bastion-eu"
  machine_type = "e2-medium"
  zone         = "europe-west1-b"

  tags = ["bastion-eu"]

  boot_disk {
    initialize_params {
      image = "projects/debian-cloud/global/images/family/debian-12"
    }
  }

  network_interface {
    network    = google_compute_network.vpc_hub_eu.id
    subnetwork = google_compute_subnetwork.eu_mgmt.id

    # Sin IP externa si vas a usar IAP/VPN
    access_config {} # quitalo si no querés IP pública
  }

  metadata = {
    enable-oslogin = "TRUE"
  }

  service_account {
    email  = "bastion-sa@${var.eu_hub_project_id}.iam.gserviceaccount.com"
    scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
}
```

---

### 6. VM OpenVPN (esqueleto)

La VM OpenVPN/wireguard también debería ir a `subnet-eu-mgmt` (u otra subnet de mgmt en EU).  
Ejemplo mínimo:

```hcl
resource "google_compute_instance" "openvpn_eu" {
  project      = var.eu_hub_project_id
  name         = "openvpn-eu"
  machine_type = "e2-small"
  zone         = "europe-west1-b"

  tags = ["openvpn-eu"]

  boot_disk {
    initialize_params {
      image = "projects/debian-cloud/global/images/family/debian-12"
    }
  }

  network_interface {
    network    = google_compute_network.vpc_hub_eu.id
    subnetwork = google_compute_subnetwork.eu_mgmt.id

    access_config {} # IP pública para que los devs se conecten
  }

  metadata_startup_script = <<-EOT
    #!/bin/bash
    # Instalar y configurar OpenVPN / WireGuard acá
    # Configurar DNS de los clientes hacia Cloud DNS interno
  EOT
}
```

---

## Resumen

- **NA** y **EU** tienen cada uno su **hub** con Shared VPC y subnets de `app` + `mgmt`.
- Cada app/ambiente (App1 NA, App2 EU) vive en **service projects** que consumen la Shared VPC del hub regional.
- La **subnet de management** vive en el **hub** de cada región y aloja bastion/jumpbox y, si se requiere, el **servidor OpenVPN**.
- Las políticas de residencia de datos EU se cumplen al:
  - Mantener todos los recursos de datos de App2 en EU.
  - Limitar accesos trans-región a control/telemetría estrictamente necesaria.
- SCC puede usar esta segmentación para revisar exposición, firewall, IAM y ayudarte a documentar cumplimiento.

Podés copiar este archivo como `README.md` directamente.
