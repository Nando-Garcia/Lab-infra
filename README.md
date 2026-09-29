## LocalStack Lab: local + CI/CD

Este repositorio concentra la infraestructura (Terraform + LocalStack + PostgreSQL) y consume el artifact ZIP de Lambda generado en el repositorio backend.

## Requisitos

- Docker Desktop activo
- Terraform (en ARM64 puedes usar WSL/Ubuntu)
- Acceso a GitHub Actions del repo de infra

Verificacion rapida desde PowerShell:

```shell
wsl --list --verbose
wsl -d Ubuntu
terraform version
docker --version
```

## Configuracion local segura

Para desarrollo local usa estos archivos (ignorados por Git):

- `.env` en la raiz
- `infra/terraform.tfvars`

Toma como referencia:

- `.env.example`
- `infra/terraform.tfvars.example`

### Nota de seguridad (importante)

Los valores sensibles mostrados en este README (por ejemplo `admin`, `admin123`, `notesdb`) son solo de ejemplo para fines de laboratorio y aprendizaje para portafolio.

En un proyecto real no se deben publicar credenciales en documentacion, codigo fuente ni archivos versionados. En entornos reales se deben usar secretos gestionados (por ejemplo GitHub Secrets, AWS Secrets Manager o un vault corporativo), rotacion de credenciales y valores fuertes/no predecibles.

### .env para docker-compose

`docker-compose.yml` lee valores desde `.env`:

```env
POSTGRES_USER=admin
POSTGRES_PASSWORD=admin123
POSTGRES_DB=notesdb
```

### terraform.tfvars para Terraform local

`infra/variables.tf` define variables y `infra/terraform.tfvars` inyecta los valores locales:

```hcl
db_host              = "localhost"
db_port              = 5432
db_name              = "notesdb"
db_username          = "admin"
db_password          = "admin123"
lambda_architecture  = "arm64"
```

## Flujo local

1. Levanta servicios:

```shell
docker compose down
docker compose up -d
```

2. Aplica Terraform:

```shell
cd infra
terraform init
terraform plan
terraform apply
```

Nota: si cambias valores en `.env`, recrea contenedores para aplicar los cambios.

## Flujo CI/CD en GitHub Actions

Hay dos workflows separados:

- `.github/workflows/terraform-localstack.yml` para el flujo final y limpio que se usará en PR a master.
- `.github/workflows/terraform-localstack-debug.yml` para pruebas paso a paso en la rama de trabajo.

### Workflow principal

- `.github/workflows/terraform-localstack.yml`

Este workflow:

- descarga el artifact `sqs-consumer-zip` desde backend
- valida secrets/vars requeridos
- ejecuta `terraform plan` y opcionalmente `terraform apply`

### Workflow de depuracion

- `.github/workflows/terraform-localstack-debug.yml`

Este workflow sirve para aprender y depurar por etapas con `debug_until` sin tocar el YAML principal.

### Secrets requeridos

- `DB_NAME`
- `DB_USER`
- `DB_PASSWORD`
- `BACKEND_ARTIFACT_TOKEN` (recomendado si backend e infra son repos privados separados)

### Variable de repositorio requerida

- `BACKEND_REPOSITORY` con formato `owner/repo`

## Debug paso a paso con debug_until

El workflow tiene input `debug_until` para ejecutar por etapas sin editar YAML.

- `1`: checkout
- `2`: validacion de secrets
- `3`: validacion de `BACKEND_REPOSITORY`
- `4`: descarga artifact Lambda
- `5`: validacion de ZIP descargado
- `6`: setup Terraform
- `7`: healthcheck LocalStack
- `8`: resolucion arquitectura Lambda
- `9`: `terraform init` + `terraform plan` (+ `apply` si `apply=true`)

Uso recomendado para depurar:

1. Ejecuta con `debug_until=3`
2. Ejecuta con `debug_until=5`
3. Ejecuta con `debug_until=7`
4. Ejecuta con `debug_until=9` y `apply=false`
5. Ejecuta con `debug_until=9` y `apply=true`

## Por que hay dos .gitignore

Se deben mantener ambos:

- `.gitignore` (raiz): reglas globales del repo (por ejemplo `.env`, `localstack/`).
- `infra/.gitignore`: reglas de Terraform (por ejemplo `terraform.tfvars`, `*.auto.tfvars`, state y cache).

Esta separacion reduce riesgo de fuga de secretos y evita ruido en commits.