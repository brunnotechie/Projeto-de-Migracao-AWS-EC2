# Documentação de Migração On-Premises para AWS
## Visão Geral

Este documento contém os scripts e instruções necessárias para migrar uma infraestrutura on-premises para AWS, incluindo a configuração de VPC, subnets, EC2 e RDS.

## Pré-requisitos

- AWS CLI instalado e configurado
- Terraform instalado (versão >= 1.0.0)
- Acesso à conta AWS com permissões adequadas
- Chaves SSH para acesso ao EC2

## 1. Estrutura do Projeto

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── scripts/
    ├── user-data.sh
    └── db-migration.sh
```

## 2. Configuração da Infraestrutura

### 2.1 Provider e Configurações Básicas (main.tf)

```hcl
provider "aws" {
  region = var.aws_region
}

terraform {
  required_version = ">= 1.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}
```

### 2.2 Definição de Variáveis (variables.tf)

```hcl
variable "aws_region" {
  description = "Região AWS para deploy"
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "CIDR da VPC"
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidr" {
  description = "CIDR da Subnet Pública"
  default     = "10.0.1.0/24"
}

variable "private_subnet_cidr" {
  description = "CIDR da Subnet Privada"
  default     = "10.0.2.0/24"
}

variable "app_instance_type" {
  description = "Tipo da instância EC2"
  default     = "t3.micro"
}

variable "db_instance_class" {
  description = "Classe da instância RDS"
  default     = "db.t3.micro"
}
```

### 2.3 Configuração da VPC

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "vpc-migration"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "igw-migration"
  }
}

# Subnet Pública
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-migration"
  }
}

# Subnet Privada
resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidr
  availability_zone = "${var.aws_region}a"

  tags = {
    Name = "private-subnet-migration"
  }
}
```

### 2.4 Configuração de Segurança

```hcl
# Security Group para EC2
resource "aws_security_group" "ec2" {
  name        = "sg-ec2-application"
  description = "Security group for EC2 instance"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security Group para RDS
resource "aws_security_group" "rds" {
  name        = "sg-rds-database"
  description = "Security group for RDS instance"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.ec2.id]
  }
}
```

### 2.5 Configuração do EC2

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI ID
  instance_type = var.app_instance_type

  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.ec2.id]
  associate_public_ip_address = true

  user_data = file("scripts/user-data.sh")

  tags = {
    Name = "ec2-application"
  }
}
```

### 2.6 Configuração do RDS

```hcl
resource "aws_db_instance" "database" {
  identifier        = "rds-migration"
  engine            = "mysql"
  engine_version    = "8.0.28"
  instance_class    = var.db_instance_class
  allocated_storage = 20

  db_name  = "migrationdb"
  username = "admin"
  password = "your-secure-password"

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name  = aws_db_subnet_group.default.name

  skip_final_snapshot = true

  tags = {
    Name = "rds-database"
  }
}

resource "aws_db_subnet_group" "default" {
  name       = "main"
  subnet_ids = [aws_subnet.private.id]

  tags = {
    Name = "DB subnet group"
  }
}
```

## 3. Scripts de Suporte

### 3.1 User Data Script (scripts/user-data.sh)

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Aplicação Migrada com Sucesso!</h1>" > /var/www/html/index.html
```

### 3.2 Script de Migração do Banco de Dados (scripts/db-migration.sh)

```bash
#!/bin/bash

# Variáveis de ambiente
SOURCE_HOST="origem-on-premises"
SOURCE_USER="user"
SOURCE_DB="database"
TARGET_HOST="${aws_db_instance.database.endpoint}"
TARGET_USER="admin"
TARGET_DB="migrationdb"

# Dump do banco de dados origem
mysqldump -h $SOURCE_HOST -u $SOURCE_USER -p $SOURCE_DB > db_backup.sql

# Restauração no RDS
mysql -h $TARGET_HOST -u $TARGET_USER -p $TARGET_DB < db_backup.sql
```

## 4. Outputs (outputs.tf)

```hcl
output "ec2_public_ip" {
  value = aws_instance.app.public_ip
}

output "rds_endpoint" {
  value = aws_db_instance.database.endpoint
}
```

## 5. Instruções de Deployment

1. Clone o repositório
2. Configure as credenciais AWS
3. Inicialize o Terraform:
```bash
terraform init
```

4. Revise o plano de execução:
```bash
terraform plan
```

5. Aplique as mudanças:
```bash
terraform apply
```

## 6. Pós-Deployment

1. Verifique a conexão com a instância EC2
2. Execute o script de migração do banco de dados
3. Atualize as configurações de conexão da aplicação
4. Teste a aplicação migrada

## 7. Backup e Recuperação

- Configure snapshots automáticos do RDS
- Implemente backup da aplicação no EC2
- Documente procedimentos de recuperação

## 8. Monitoramento

- Configure CloudWatch Alerts
- Monitore métricas de performance
- Configure logs de aplicação

## 9. Segurança

- Revise as políticas de segurança regularmente
- Mantenha as patches atualizadas
- Implemente AWS WAF se necessário

## 10. Custos

- Monitore custos através do AWS Cost Explorer
- Configure budgets e alertas
- Otimize recursos conforme necessidade
