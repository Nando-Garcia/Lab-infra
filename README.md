## Instalación y verificación de Terraform

Debido a limitaciones de compatibilidad en el entorno ARM64 utilizado para este laboratorio, Terraform se instaló y ejecutó mediante WSL (Ubuntu).

Desde PowerShell, utiliza los siguientes comandos:

```shell
wsl --list --verbose
wsl -d Ubuntu

terraform version
```