# Arquitectura Hub & Spoke multi-región

## NA HUB (Montreal/Toronto)

- Proyecto: `hub-na-network`
- Región principal: `northamerica-northeast1`
- VPC: `vpc-hub-na (10.0.0.0/16)`
  - `10.0.0.0/24`  → `subnet-hub-montreal`
  - `10.10.0.0/24` → `subnet-hub-toronto` (secundaria / DR)
- Servicios centralizados:
  - Shared VPC **Host**
  - Cloud NAT + Cloud Router
  - Cloud VPN (si necesitas site-to-site on-prem)
  - Bastion / Jumpbox (opcional OpenVPN server)
  - KMS keys NA (solo App1)
  - Logs de red NA → BQ dataset regional NA

### Spokes NA (App1)

> Todos son **service projects** de la Shared VPC `hub-na-network`.

- `app1-dev-na`
  - Subnet: `172.16.200.0/24` (DEV)
  - Cloud Run (privado, con Serverless VPC Connector)
  - Cloud SQL / Memorystore / etc. (región NA)
- `app1-qa-na`
  - Subnet: `172.16.230.0/24` (QA)
  - Mismos componentes que DEV
- `app1-prod-na`
  - Subnet: `172.16.100.0/24` (PROD)
  - Cloud Run, Cloud SQL, etc.
  - Firewalls más cerrados

---

## EU HUB (Europa)

- Proyecto: `hub-eu-network`
- Región principal: `europe-west1` (o la que te exija el regulador)
- VPC: `vpc-hub-eu (10.20.0.0/16)`
  - `10.20.0.0/24` → `subnet-hub-eu-primary`
  - `10.30.0.0/24` → `subnet-hub-eu-dr` (opcional)
- Servicios centralizados:
  - Shared VPC **Host**
  - Cloud NAT + Cloud Router
  - Bastion / Jumpbox EU (puede tener OpenVPN server para devs europeos)
  - KMS keys EU (App2)
  - Logs EU → BQ dataset **regional EU**
  - Cloud DNS (zonas privadas para `*.internal` de App2)

### Spokes EU (App2)

> También como **service projects** de `hub-eu-network`.

- `app2-dev-eu`
  - Subnet: `172.20.200.0/24`
  - Cloud Run/Cloud SQL, etc. **solo en región EU**
- `app2-qa-eu`
  - Subnet: `172.20.230.0/24`
- `app2-prod-eu`
  - Subnet: `172.20.100.0/24`

---

## Conectividad entre regiones

- **VPC Network Peering** solo entre:
  - `vpc-hub-na` ↔ `vpc-hub-eu`
- Rutas:
  - Exportar/importar **solo rangos de mgmt** (por ej. subredes de bastions / herramientas),
  - No exportar subredes de datos de App2 hacia NA.
- Firewalls:
  - Permitir solo puertos de mgmt (SSH/IAP, monitoring, etc.).

---

## Cumplimiento / Security Center

Este diseño **ayuda** a que Security Command Center y compliance te aprueben porque:

- App2 (EU) tiene:
  - Proyectos y recursos **solo en regiones EU**.
  - Logs, KMS y bases de datos regionales a EU.
  - No hay dependencia de Shared VPC en NA, solo peering de mgmt muy acotado.
- Podés reforzarlo con:
  - Org Policy `constraints/gcp.resourceLocations` aplicada al folder/proyectos de EU
    → solo regiones `europe-*`.
  - Reglas de VPC Firewall / Cloud Armor / SCC para bloquear tráfico cruzado no deseado.
- La aprobación final siempre depende del auditor, pero este patrón está alineado con
  “**data residency EU** + separación NA/EU”.

---

### 2. ¿Jumpbox/Bastion o OpenVPN para devs + DNS interno?

- **Jumpbox/Bastion** (IAP / OS Login):
  - Ideal para **acceso admin** puntual (SSH, kubectl, psql, etc.).
  - No resuelve el caso de “mi notebook ve todos los servicios privados y DNS interno”
    salvo que montes túneles/port-forwarding a mano.
- **VPN (Cloud VPN o OpenVPN server en GCE)**:
  - Devs “se meten” a la VPC; ven:
    - IPs privadas,
    - Cloud DNS interno (zonas privadas),
    - servicios accesibles como si estuvieran dentro.
  - Para pruebas de apps con muchos endpoints, gRPC, etc., es mucho más cómodo.

**Recomendación práctica:**

- Mantener **bastion** para admins (con IAP, sin IP pública si quieres).
- Para devs que necesitan probar APIs internas + DNS privado:
  - Montar **OpenVPN / WireGuard** en una VM dentro del **HUB EU**  
    (y otra en HUB NA si algunos devs necesitan NA).
  - Configurar el VPN para usar como DNS la IP de Cloud DNS (`169.254.169.254`)
    o un resolver interno que reenvíe a Cloud DNS.
- Así la política de residencias se sigue cumpliendo:  
  devs que trabajan con App2 se conectan al **VPN del HUB EU**,  
  y todo el tráfico de datos EU se queda en EU.

---

### 3. Terraform de ejemplo

Un solo archivo `networks.tf` de base (simplificado, pero completo para arrancar):

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.na_hub_project_id
  region  = var.na_region
}

variable "na_hub_project_id" {}
variable "eu_hub_project_id" {}

variable "app1_dev_project_id" {}
variable "app1_qa_project_id" {}
variable "app1_prod_project_id" {}

variable "app2_dev_project_id" {}
variable "app2_qa_project_id" {}
variable "app2_prod_project_id" {}

variable "na_region" {
  default = "northamerica-northeast1"
}

variable "eu_region" {
  default = "europe-west1"
}

# -------------------------
# NA HUB
# -------------------------

resource "google_compute_network" "na_hub" {
  name                    = "vpc-hub-na"
  project                 = var.na_hub_project_id
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "na_hub_montreal" {
  name          = "subnet-hub-montreal"
  ip_cidr_range = "10.0.0.0/24"
  region        = var.na_region
  network       = google_compute_network.na_hub.id
}

resource "google_compute_subnetwork" "na_hub_toronto" {
  name          = "subnet-hub-toronto"
  ip_cidr_range = "10.10.0.0/24"
  region        = "northamerica-northeast2"
  network       = google_compute_network.na_hub.id
}

resource "google_compute_shared_vpc_host_project" "na_host" {
  project = var.na_hub_project_id
}

resource "google_compute_shared_vpc_service_project" "app1_dev_na" {
  host_project    = var.na_hub_project_id
  service_project = var.app1_dev_project_id
}

resource "google_compute_shared_vpc_service_project" "app1_qa_na" {
  host_project    = var.na_hub_project_id
  service_project = var.app1_qa_project_id
}

resource "google_compute_shared_vpc_service_project" "app1_prod_na" {
  host_project    = var.na_hub_project_id
  service_project = var.app1_prod_project_id
}

# Subnets para spokes NA (pueden estar en el host project)
resource "google_compute_subnetwork" "na_spoke_prod" {
  name          = "subnet-app1-prod-na"
  ip_cidr_range = "172.16.100.0/24"
  region        = var.na_region
  network       = google_compute_network.na_hub.id
}

resource "google_compute_subnetwork" "na_spoke_qa" {
  name          = "subnet-app1-qa-na"
  ip_cidr_range = "172.16.230.0/24"
  region        = var.na_region
  network       = google_compute_network.na_hub.id
}

resource "google_compute_subnetwork" "na_spoke_dev" {
  name          = "subnet-app1-dev-na"
  ip_cidr_range = "172.16.200.0/24"
  region        = var.na_region
  network       = google_compute_network.na_hub.id
}

# NAT para salidas de Internet
resource "google_compute_router" "na_router" {
  name    = "router-na-hub"
  region  = var.na_region
  network = google_compute_network.na_hub.id
}

resource "google_compute_router_nat" "na_nat" {
  name                               = "nat-na-hub"
  router                             = google_compute_router.na_router.name
  region                             = google_compute_router.na_router.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}

# Bastion / Jumpbox NA (puede alojar OpenVPN si querés)
resource "google_compute_instance" "na_bastion" {
  name         = "bastion-na-01"
  project      = var.na_hub_project_id
  zone         = "${var.na_region}-b"
  machine_type = "e2-medium"
  tags         = ["bastion", "ssh"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.na_hub_montreal.id
    access_config {} # IP pública
  }

  metadata = {
    enable-oslogin = "TRUE"
  }
}

# -------------------------
# EU HUB
# -------------------------

resource "google_compute_network" "eu_hub" {
  name                    = "vpc-hub-eu"
  project                 = var.eu_hub_project_id
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "eu_hub_primary" {
  name          = "subnet-hub-eu-primary"
  ip_cidr_range = "10.20.0.0/24"
  region        = var.eu_region
  network       = google_compute_network.eu_hub.id
}

resource "google_compute_shared_vpc_host_project" "eu_host" {
  project = var.eu_hub_project_id
}

resource "google_compute_shared_vpc_service_project" "app2_dev_eu" {
  host_project    = var.eu_hub_project_id
  service_project = var.app2_dev_project_id
}

resource "google_compute_shared_vpc_service_project" "app2_qa_eu" {
  host_project    = var.eu_hub_project_id
  service_project = var.app2_qa_project_id
}

resource "google_compute_shared_vpc_service_project" "app2_prod_eu" {
  host_project    = var.eu_hub_project_id
  service_project = var.app2_prod_project_id
}

resource "google_compute_subnetwork" "eu_spoke_prod" {
  name          = "subnet-app2-prod-eu"
  ip_cidr_range = "172.20.100.0/24"
  region        = var.eu_region
  network       = google_compute_network.eu_hub.id
}

resource "google_compute_subnetwork" "eu_spoke_qa" {
  name          = "subnet-app2-qa-eu"
  ip_cidr_range = "172.20.230.0/24"
  region        = var.eu_region
  network       = google_compute_network.eu_hub.id
}

resource "google_compute_subnetwork" "eu_spoke_dev" {
  name          = "subnet-app2-dev-eu"
  ip_cidr_range = "172.20.200.0/24"
  region        = var.eu_region
  network       = google_compute_network.eu_hub.id
}

resource "google_compute_router" "eu_router" {
  name    = "router-eu-hub"
  region  = var.eu_region
  network = google_compute_network.eu_hub.id
}

resource "google_compute_router_nat" "eu_nat" {
  name                               = "nat-eu-hub"
  router                             = google_compute_router.eu_router.name
  region                             = google_compute_router.eu_router.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}

resource "google_compute_instance" "eu_bastion_vpn" {
  name         = "bastion-eu-vpn-01"
  project      = var.eu_hub_project_id
  zone         = "${var.eu_region}-b"
  machine_type = "e2-medium"
  tags         = ["bastion", "vpn"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.eu_hub_primary.id
    access_config {} # IP pública para OpenVPN
  }

  metadata = {
    enable-oslogin = "TRUE"
  }
}

# -------------------------
# Peering NA ↔ EU (mgmt-only)
# -------------------------

resource "google_compute_network_peering" "na_to_eu" {
  name         = "na-to-eu-peering"
  network      = google_compute_network.na_hub.id
  peer_network = google_compute_network.eu_hub.self_link

  export_custom_routes = false
  import_custom_routes = false
}

resource "google_compute_network_peering" "eu_to_na" {
  name         = "eu-to-na-peering"
  network      = google_compute_network.eu_hub.id
  peer_network = google_compute_network.na_hub.self_link

  export_custom_routes = false
  import_custom_routes = false
}
