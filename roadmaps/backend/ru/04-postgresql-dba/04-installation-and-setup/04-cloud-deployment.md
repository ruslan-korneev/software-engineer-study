# Развёртывание PostgreSQL в облаке

[prev: 03-connect-using-psql](./03-connect-using-psql.md) | [next: 01-using-systemd](../05-managing-postgres/01-using-systemd.md)
---

## Введение

Облачные провайдеры предлагают управляемые сервисы PostgreSQL (Managed PostgreSQL), которые берут на себя администрирование, резервное копирование, масштабирование и обеспечение высокой доступности. Это позволяет сосредоточиться на разработке приложений.

## Преимущества облачного PostgreSQL

- **Автоматическое резервное копирование** — настраиваемые расписания и retention
- **Высокая доступность** — автоматический failover и репликация
- **Масштабирование** — вертикальное и горизонтальное масштабирование
- **Безопасность** — шифрование данных, VPC, IAM
- **Мониторинг** — встроенные метрики и алерты
- **Автоматические обновления** — патчи безопасности

## AWS RDS for PostgreSQL

### Создание инстанса через AWS CLI

```bash
# Установка AWS CLI
pip install awscli
aws configure

# Создание RDS-инстанса PostgreSQL
aws rds create-db-instance \
    --db-instance-identifier mypostgres \
    --db-instance-class db.t3.micro \
    --engine postgres \
    --engine-version 16.1 \
    --master-username postgres \
    --master-user-password MySecurePassword123! \
    --allocated-storage 20 \
    --storage-type gp3 \
    --vpc-security-group-ids sg-xxxxxxxx \
    --db-subnet-group-name my-subnet-group \
    --backup-retention-period 7 \
    --multi-az \
    --storage-encrypted \
    --publicly-accessible false
```

### Terraform конфигурация для AWS RDS

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-central-1"
}

# Группа безопасности
resource "aws_security_group" "postgres" {
  name        = "postgres-sg"
  description = "Security group for PostgreSQL"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "postgres-sg"
  }
}

# Subnet group
resource "aws_db_subnet_group" "postgres" {
  name       = "postgres-subnet-group"
  subnet_ids = var.private_subnet_ids

  tags = {
    Name = "PostgreSQL Subnet Group"
  }
}

# Parameter group
resource "aws_db_parameter_group" "postgres" {
  family = "postgres16"
  name   = "postgres-params"

  parameter {
    name  = "log_statement"
    value = "ddl"
  }

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"
  }

  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements"
  }
}

# RDS Instance
resource "aws_db_instance" "postgres" {
  identifier = "mypostgres"

  # Engine
  engine               = "postgres"
  engine_version       = "16.1"
  instance_class       = "db.t3.medium"

  # Storage
  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true

  # Credentials
  db_name  = "myapp"
  username = "postgres"
  password = var.db_password

  # Network
  db_subnet_group_name   = aws_db_subnet_group.postgres.name
  vpc_security_group_ids = [aws_security_group.postgres.id]
  publicly_accessible    = false
  port                   = 5432

  # High Availability
  multi_az = true

  # Backup
  backup_retention_period = 14
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"

  # Monitoring
  monitoring_interval          = 60
  monitoring_role_arn          = var.monitoring_role_arn
  performance_insights_enabled = true
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  # Parameters
  parameter_group_name = aws_db_parameter_group.postgres.name

  # Deletion protection
  deletion_protection      = true
  skip_final_snapshot      = false
  final_snapshot_identifier = "mypostgres-final-snapshot"

  tags = {
    Environment = "production"
    Name        = "mypostgres"
  }
}

# Read Replica (опционально)
resource "aws_db_instance" "postgres_replica" {
  identifier = "mypostgres-replica"

  replicate_source_db = aws_db_instance.postgres.identifier
  instance_class      = "db.t3.medium"

  publicly_accessible = false
  skip_final_snapshot = true

  tags = {
    Environment = "production"
    Name        = "mypostgres-replica"
  }
}

# Outputs
output "rds_endpoint" {
  value = aws_db_instance.postgres.endpoint
}

output "rds_port" {
  value = aws_db_instance.postgres.port
}
```

### Подключение к AWS RDS

```bash
# Получение endpoint
aws rds describe-db-instances \
    --db-instance-identifier mypostgres \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text

# Подключение через psql
psql -h mypostgres.xxxxxxxxxxxx.eu-central-1.rds.amazonaws.com \
     -p 5432 \
     -U postgres \
     -d myapp

# Через connection string
psql "postgresql://postgres:password@mypostgres.xxxx.rds.amazonaws.com:5432/myapp?sslmode=require"
```

### Важные настройки AWS RDS

```bash
# Включение расширений
aws rds modify-db-parameter-group \
    --db-parameter-group-name postgres-params \
    --parameters "ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements,ApplyMethod=pending-reboot"

# Ручной snapshot
aws rds create-db-snapshot \
    --db-snapshot-identifier mypostgres-manual-snapshot \
    --db-instance-identifier mypostgres

# Восстановление из snapshot
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier mypostgres-restored \
    --db-snapshot-identifier mypostgres-manual-snapshot
```

## Google Cloud SQL for PostgreSQL

### Создание через gcloud CLI

```bash
# Установка и настройка gcloud
gcloud auth login
gcloud config set project my-project-id

# Создание инстанса
gcloud sql instances create mypostgres \
    --database-version=POSTGRES_16 \
    --tier=db-custom-2-4096 \
    --region=europe-west1 \
    --root-password=MySecurePassword123! \
    --storage-type=SSD \
    --storage-size=100GB \
    --storage-auto-increase \
    --availability-type=REGIONAL \
    --backup-start-time=03:00 \
    --enable-bin-log \
    --maintenance-window-day=MON \
    --maintenance-window-hour=4

# Создание базы данных
gcloud sql databases create myapp --instance=mypostgres

# Создание пользователя
gcloud sql users create appuser \
    --instance=mypostgres \
    --password=UserPassword123!
```

### Terraform конфигурация для GCP Cloud SQL

```hcl
# main.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = "europe-west1"
}

# VPC и Private IP
resource "google_compute_global_address" "private_ip" {
  name          = "postgres-private-ip"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = var.vpc_network
}

resource "google_service_networking_connection" "private_vpc_connection" {
  network                 = var.vpc_network
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip.name]
}

# Cloud SQL Instance
resource "google_sql_database_instance" "postgres" {
  name             = "mypostgres"
  database_version = "POSTGRES_16"
  region           = "europe-west1"

  depends_on = [google_service_networking_connection.private_vpc_connection]

  settings {
    tier              = "db-custom-2-4096"  # 2 vCPU, 4GB RAM
    availability_type = "REGIONAL"          # High Availability
    disk_type         = "PD_SSD"
    disk_size         = 100
    disk_autoresize   = true

    # Network
    ip_configuration {
      ipv4_enabled    = false
      private_network = var.vpc_network
      require_ssl     = true
    }

    # Backup
    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      point_in_time_recovery_enabled = true
      backup_retention_settings {
        retained_backups = 14
      }
    }

    # Maintenance
    maintenance_window {
      day          = 1  # Monday
      hour         = 4
      update_track = "stable"
    }

    # Flags (параметры PostgreSQL)
    database_flags {
      name  = "log_statement"
      value = "ddl"
    }

    database_flags {
      name  = "log_min_duration_statement"
      value = "1000"
    }

    # Insights
    insights_config {
      query_insights_enabled  = true
      query_string_length     = 1024
      record_application_tags = true
      record_client_address   = true
    }
  }

  deletion_protection = true
}

# Database
resource "google_sql_database" "myapp" {
  name     = "myapp"
  instance = google_sql_database_instance.postgres.name
}

# User
resource "google_sql_user" "appuser" {
  name     = "appuser"
  instance = google_sql_database_instance.postgres.name
  password = var.db_password
}

# Read Replica
resource "google_sql_database_instance" "postgres_replica" {
  name                 = "mypostgres-replica"
  master_instance_name = google_sql_database_instance.postgres.name
  region               = "europe-west1"
  database_version     = "POSTGRES_16"

  replica_configuration {
    failover_target = false
  }

  settings {
    tier            = "db-custom-2-4096"
    disk_type       = "PD_SSD"
    disk_autoresize = true

    ip_configuration {
      ipv4_enabled    = false
      private_network = var.vpc_network
    }
  }
}

# Output
output "connection_name" {
  value = google_sql_database_instance.postgres.connection_name
}

output "private_ip" {
  value = google_sql_database_instance.postgres.private_ip_address
}
```

### Подключение к Cloud SQL

```bash
# Через Cloud SQL Proxy (рекомендуется)
# Скачать proxy
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

# Запуск proxy
./cloud-sql-proxy my-project:europe-west1:mypostgres &

# Подключение через proxy
psql -h localhost -p 5432 -U postgres -d myapp

# Прямое подключение к Private IP (из VPC)
psql -h 10.x.x.x -p 5432 -U postgres -d myapp

# С SSL
psql "sslmode=verify-ca sslrootcert=server-ca.pem sslcert=client-cert.pem sslkey=client-key.pem host=10.x.x.x user=postgres dbname=myapp"
```

## Azure Database for PostgreSQL

### Создание через Azure CLI

```bash
# Установка и вход
az login
az account set --subscription "My Subscription"

# Создание resource group
az group create --name myresourcegroup --location westeurope

# Создание сервера PostgreSQL (Flexible Server)
az postgres flexible-server create \
    --resource-group myresourcegroup \
    --name mypostgres \
    --location westeurope \
    --admin-user postgres \
    --admin-password MySecurePassword123! \
    --sku-name Standard_D2s_v3 \
    --tier GeneralPurpose \
    --storage-size 128 \
    --version 16 \
    --high-availability ZoneRedundant \
    --zone 1 \
    --standby-zone 2

# Создание базы данных
az postgres flexible-server db create \
    --resource-group myresourcegroup \
    --server-name mypostgres \
    --database-name myapp

# Настройка firewall (для разработки)
az postgres flexible-server firewall-rule create \
    --resource-group myresourcegroup \
    --name mypostgres \
    --rule-name AllowMyIP \
    --start-ip-address 1.2.3.4 \
    --end-ip-address 1.2.3.4
```

### Terraform конфигурация для Azure

```hcl
# main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "rg-postgres"
  location = "West Europe"
}

# Virtual Network
resource "azurerm_virtual_network" "main" {
  name                = "vnet-postgres"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "postgres" {
  name                 = "subnet-postgres"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]

  delegation {
    name = "postgres-delegation"
    service_delegation {
      name = "Microsoft.DBforPostgreSQL/flexibleServers"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/join/action",
      ]
    }
  }
}

# Private DNS Zone
resource "azurerm_private_dns_zone" "postgres" {
  name                = "privatelink.postgres.database.azure.com"
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "postgres" {
  name                  = "postgres-vnet-link"
  private_dns_zone_name = azurerm_private_dns_zone.postgres.name
  virtual_network_id    = azurerm_virtual_network.main.id
  resource_group_name   = azurerm_resource_group.main.name
}

# PostgreSQL Flexible Server
resource "azurerm_postgresql_flexible_server" "main" {
  name                   = "mypostgres"
  resource_group_name    = azurerm_resource_group.main.name
  location               = azurerm_resource_group.main.location
  version                = "16"
  administrator_login    = "postgres"
  administrator_password = var.db_password

  # SKU
  sku_name = "GP_Standard_D2s_v3"

  # Storage
  storage_mb = 131072  # 128 GB

  # Network
  delegated_subnet_id = azurerm_subnet.postgres.id
  private_dns_zone_id = azurerm_private_dns_zone.postgres.id

  # High Availability
  high_availability {
    mode                      = "ZoneRedundant"
    standby_availability_zone = "2"
  }

  # Backup
  backup_retention_days        = 14
  geo_redundant_backup_enabled = true

  # Maintenance
  maintenance_window {
    day_of_week  = 1  # Monday
    start_hour   = 4
    start_minute = 0
  }

  zone = "1"

  depends_on = [azurerm_private_dns_zone_virtual_network_link.postgres]

  tags = {
    Environment = "production"
  }
}

# Database
resource "azurerm_postgresql_flexible_server_database" "myapp" {
  name      = "myapp"
  server_id = azurerm_postgresql_flexible_server.main.id
  charset   = "UTF8"
  collation = "en_US.utf8"
}

# Server Parameters
resource "azurerm_postgresql_flexible_server_configuration" "log_statement" {
  name      = "log_statement"
  server_id = azurerm_postgresql_flexible_server.main.id
  value     = "ddl"
}

resource "azurerm_postgresql_flexible_server_configuration" "log_min_duration" {
  name      = "log_min_duration_statement"
  server_id = azurerm_postgresql_flexible_server.main.id
  value     = "1000"
}

# Output
output "fqdn" {
  value = azurerm_postgresql_flexible_server.main.fqdn
}
```

### Подключение к Azure PostgreSQL

```bash
# Получение FQDN
az postgres flexible-server show \
    --resource-group myresourcegroup \
    --name mypostgres \
    --query fullyQualifiedDomainName \
    --output tsv

# Подключение
psql "host=mypostgres.postgres.database.azure.com \
      port=5432 \
      dbname=myapp \
      user=postgres \
      sslmode=require"
```

## Сравнение облачных провайдеров

| Функция | AWS RDS | GCP Cloud SQL | Azure PostgreSQL |
|---------|---------|---------------|------------------|
| **Версии PostgreSQL** | 11-16 | 11-16 | 11-16 |
| **High Availability** | Multi-AZ | Regional | Zone Redundant |
| **Read Replicas** | До 15 | До 10 | До 5 |
| **Автоматический failover** | Да | Да | Да |
| **Point-in-time Recovery** | До 35 дней | До 35 дней | До 35 дней |
| **Шифрование at rest** | AES-256 | AES-256 | AES-256 |
| **Private connectivity** | VPC, PrivateLink | Private IP | VNet, Private Link |
| **Мониторинг** | CloudWatch | Cloud Monitoring | Azure Monitor |

## Best Practices для Production

### 1. Безопасность

```hcl
# Никогда не используйте публичный доступ
publicly_accessible = false

# Используйте шифрование
storage_encrypted = true

# Настройте SSL
require_ssl = true

# Ограничьте сетевой доступ
vpc_security_group_ids = [aws_security_group.postgres.id]
```

### 2. High Availability

```hcl
# AWS RDS
multi_az = true

# GCP Cloud SQL
availability_type = "REGIONAL"

# Azure
high_availability {
  mode = "ZoneRedundant"
}
```

### 3. Backup и Recovery

```hcl
# Backup retention
backup_retention_period = 14  # Минимум 7 дней для production

# Point-in-time recovery
point_in_time_recovery_enabled = true

# Geo-redundant backup (Azure)
geo_redundant_backup_enabled = true
```

### 4. Мониторинг

```bash
# AWS - включить Performance Insights
aws rds modify-db-instance \
    --db-instance-identifier mypostgres \
    --enable-performance-insights \
    --performance-insights-retention-period 7

# GCP - Query Insights включён в Terraform выше

# Azure - настроить Azure Monitor и alerts
```

### 5. Масштабирование

```bash
# Вертикальное масштабирование (изменение instance class)
# AWS
aws rds modify-db-instance \
    --db-instance-identifier mypostgres \
    --db-instance-class db.r6g.large \
    --apply-immediately

# GCP
gcloud sql instances patch mypostgres --tier=db-custom-4-8192

# Горизонтальное масштабирование (read replicas)
# Используйте connection pooler (PgBouncer) для распределения нагрузки
```

### 6. Connection Pooling

Для production рекомендуется использовать connection pooler:

```bash
# PgBouncer на AWS RDS Proxy
aws rds create-db-proxy \
    --db-proxy-name myproxy \
    --engine-family POSTGRESQL \
    --auth Description="RDS Proxy auth",AuthScheme=SECRETS,SecretArn=arn:aws:secretsmanager:... \
    --role-arn arn:aws:iam::...:role/rds-proxy-role \
    --vpc-subnet-ids subnet-xxx subnet-yyy
```

## Полезные ссылки

- [AWS RDS for PostgreSQL Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html)
- [GCP Cloud SQL for PostgreSQL](https://cloud.google.com/sql/docs/postgres)
- [Azure Database for PostgreSQL](https://docs.microsoft.com/en-us/azure/postgresql/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform Google Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [Terraform Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)

---
[prev: 03-connect-using-psql](./03-connect-using-psql.md) | [next: 01-using-systemd](../05-managing-postgres/01-using-systemd.md)