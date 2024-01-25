---
title: "Desplegando Go en AWS Lambda con GitHub Actions y Terraform"
date: 2023-07-12T18:39:05-06:00
summary: "En este artículo, exploraremos cómo desplegar aplicaciones basadas en Go en AWS Lambda integrando GitHub Actions y Terraform."
---

## Introducción

El despliegue de aplicaciones serverless se ha vuelto cada vez más popular debido a su escalabilidad, eficiencia en costos y facilidad de gestión. Amazon Web Services (AWS) Lambda proporciona una poderosa plataforma de computación sin servidor, mientras que Go (Golang) ofrece un lenguaje de programación de alto rendimiento y tipado estático.

En este artículo, exploraremos cómo desplegar aplicaciones basadas en Go en AWS Lambda integrando GitHub Actions y Terraform. Al combinar estas herramientas, los desarrolladores pueden automatizar el proceso de despliegue, simplificar el control de versiones y asegurar prácticas consistentes de infraestructura como código para sus proyectos sin servidor.

## Prerrequisitos

Antes de comenzar, necesitarás lo siguiente:

- Una cuenta de AWS
- Una cuenta de GitHub
- AWS CLI instalado y configurado con tus credenciales de AWS (consulta la [documentación de AWS](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) para obtener instrucciones)
- Go instalado (consulta la [documentación de Go](https://golang.org/doc/install) para obtener instrucciones)
- Terraform instalado (consulta la [documentación de Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli) para obtener instrucciones)

## Paso 1: Crear un repositorio GitHub

En primer lugar, crea un repositorio GitHub para tu aplicación en Go.

Ve a [GitHub](https://github/com) y haz clic en "New repository". Ingresa un nombre para tu repositorio y haz clic en "Create repository".

## Paso 2: Crear una aplicación Go

En un directorio vacío, crea una aplicación Go:

```bash
go mod init go_lambda
```

Esto creará un archivo `go.mod` que se usará para gestionar las dependencias de tu aplicación en Go.

Luego, inicializa el repositorio Git:

```bash
git init
```

Después, crea un archivo `.gitignore`:

```bash
# .gitignore
**/.terraform/*
*.tfstate
bootstrap
lambda-handler.zip
```

Por último, establece el origen remoto:

```bash
git remote add origin <TU_URL_DEL_REPOSITORIO>
```

## Paso 3: Crear un archivo main.go

Crea el archivo main.go:

```go
// main.go
package main

import (
 "github.com/aws/aws-lambda-go/events"
 "github.com/aws/aws-lambda-go/lambda"
)

func hello() (events.APIGatewayProxyResponse, error) {
 return events.APIGatewayProxyResponse{
  Body:       "Hello World!",
  StatusCode: 200,
 }, nil
}

func main() {
 // Hacer que el controlador esté disponible para llamadas a procedimientos remotos por AWS Lambda
 lambda.Start(hello)
}
```

Este archivo contiene el código mínimo necesario para crear una aplicación Go que se pueda desplegar en AWS Lambda. Importa los paquetes `events` y `lambda` de la biblioteca `aws-lambda-go` y define una función `hello` que devuelve una respuesta que nuestra API Gateway retornará al cliente. La función `main` es el punto de entrada para nuestra aplicación y llama a la función `hello`. La función `lambda.Start` inicia el controlador de AWS Lambda[^1].

[^1]: <https://docs.aws.amazon.com/lambda/latest/dg/golang-handler.html>

## Paso 4: Instalar dependencias

Luego, instala las dependencias para tu aplicación Go:

```bash
go get github.com/aws/aws-lambda-go/events
go get github.com/aws/aws-lambda-go/lambda
```

Esto instalará *aws-lambda-go/events*, que contiene la estructura `APIGatewayProxyResponse`, y *aws-lambda-go/lambda*, que contiene la función `lambda.Start`.

## Paso 5: Crear el backend remoto

Usaremos S3 como nuestro backend remoto para Terraform. Esto es necesario porque estaremos usando GitHub Actions para desplegar nuestro código y necesitamos almacenar el estado de Terraform en una ubicación remota.

> **NOTA** No estaremos creando una tabla DynamoDB para el bloqueo de estado para mantener las cosas simples, pero deberías considerar hacer esto si estás desplegando a producción. Puedes leer más sobre esto en la [documentación de Terraform](https://www.terraform.io/docs/language/settings/backends/s3.html).

Para crear el bucket S3 para nuestro backend remoto, usaremos el siguiente comando:

```bash
aws s3api create-bucket --bucket <TU_NOMBRE_DE_BUCKET> --region us-east-1
```

Alternativamente, puedes crear el bucket utilizando la Consola de AWS siguiendo las instrucciones en la [documentación de AWS](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html).

## Paso 6: Crear la configuración de Terraform

Ahora que hemos creado nuestra aplicación Go y hemos instalado las dependencias, podemos crear la configuración de Terraform. Usaremos la siguiente configuración de Terraform para desplegar nuestra aplicación Go en AWS Lambda:

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.7.0"
    }
  }
}
```

```hcl
# backend.tf
terraform {
  backend "s3" {
    # ¡Reemplaza esto con el nombre de tu bucket!
    bucket = "<TU_NOMBRE_DE_BUCKET>"
    key    = "go-lambda-test.tfstate"
    region = "us-east-1"
  }
}
```

```hcl
# main.tf
resource "aws_lambda_function" "go_function" {
  filename      = "lambda-handler.zip"
  function_name = "go-lambda-test"
  handler       = "bootstrap"
  role = aws_iam_role.iam_for_lambda.arn

  source_code_hash = filebase64sha256("lambda-handler.zip")

  runtime = "go1.x"
}

data "aws_iam_policy_document" "assume_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "iam_for_lambda" {
  name               = "iam_for_lambda"
  assume_role_policy = data.aws_iam_policy_document.assume_role.json
}

# API Gateway

resource "aws_api_gateway_rest_api" "go_api" {
  name        = "go_api"
  description = "Esta es mi API para fines de demostración"
}

resource "aws_api_gateway_resource" "go_api_resource" {
  rest_api_id = aws_api_gateway_rest_api.go_api.id
  parent_id   = aws_api_gateway_rest_api.go_api.root_resource_id
  path_part   = "test"
}

resource "aws_api_gateway_method" "go_api_method" {
  rest_api_id   = aws_api_gateway_rest_api.go_api.id
  resource_id   = aws_api_gateway_resource.go_api_resource.id
  http_method   = "GET"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "go_api_integration" {
  rest_api_id = aws_api_gateway_rest_api.go_api.id
  resource_id = aws_api_gateway_resource.go_api_resource.id
  http_method = aws_api_gateway_method.go_api_method.http_method

  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.go_function.invoke_arn
}

resource "aws_api_gateway_method" "proxy_root" {
  rest_api_id   = aws_api_gateway_rest_api.go_api.id
  resource_id   = aws_api_gateway_rest_api.go_api.root_resource_id
  http_method   = "ANY"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "proxy_root_integration" {
  rest_api_id = aws_api_gateway_rest_api.go_api.id
  resource_id = aws_api_gateway_rest_api.go_api.root_resource_id
  http_method = aws_api_gateway_method.proxy_root.http_method

  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.go_function.invoke_arn
}

resource "aws_api_gateway_deployment" "go_api_deployment" {
  depends_on = [
    aws_api_gateway_integration.go_api_integration,
    aws_api_gateway_integration.proxy_root_integration,
  ]

  rest_api_id = aws_api_gateway_rest_api.go_api.id
  stage_name  = "test"
}

resource "aws_lambda_permission" "apigw" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.go_function.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.go_api.execution_arn}/*/*"
}

```

## Paso 7: Configurar credenciales de AWS para GitHub Actions

Para configurar credenciales de AWS para GitHub Actions, usaremos federación OIDC. Esta es una forma segura de autenticarse con AWS sin tener que almacenar tus credenciales de AWS en tu repositorio de GitHub.

Necesitaremos seguir los siguientes pasos para configurar credenciales de AWS para GitHub Actions:

1. Abre la Consola de AWS y navega al [servicio IAM](https://console.aws.amazon.com/iam).
2. Haz clic en "Identity providers" en la barra de navegación izquierda.
3. Haz clic en "Add provider".
4. Selecciona "OpenID Connect" como el tipo de proveedor.
5. Para "Provider URL", ingresa `https://token.actions.githubusercontent.com`[^2].
6. Haz clic en "Get thumbprint".
7. Para "Audience", ingresa `sts.amazonaws.com`[^2].
8. Haz clic en "Add provider".

Luego, necesitaremos crear un rol IAM para GitHub Actions. Necesitaremos seguir los siguientes pasos para crear un rol IAM para GitHub Actions:

1. Haz clic en el proveedor que acabas de crear. Debería llamarse "token.actions.githubusercontent.com".
2. Haz clic en "Assign role".
3. Selecciona "Create a new role" y haz clic en "Next".
4. Para "Identity provider", selecciona "token.actions.githubusercontent.com".
5. Para *Audience*, ingresa `sts.amazonaws.com`.
6. Haz clic en "Next: Permissions".
7. Selecciona "AmazonS3FullAccess", "AWSLambdaFullAccess", "IAMFullAccess", y "AmazonAPIGatewayAdministrator" y haz clic en "Next: Tags".
8. Haz clic en "Next: Review".
9. Para "Role name", ingresa `github-actions` y haz clic en "Create role".
10. Busca el rol que acabas de crear y haz clic en él.
11. Copia el "ARN" (Debería parecer algo como `arn:aws:iam::000000000000:role/github-actions`). Lo necesitarás en el siguiente paso.

> ***NOTA:*** Considera usar una política IAM más restrictiva para tu rol IAM de GitHub Actions.

[^2]: <https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#adding-the-identity-provider-to-aws>

## Paso 8: Crear el flujo de trabajo de GitHub Actions

Luego, crearemos el flujo de trabajo de GitHub Actions. Usaremos el siguiente flujo de trabajo para desplegar nuestra aplicación Go en AWS Lambda:

No olvides reemplazar `<ROL_IAM>` con el ARN del rol IAM que has creado en el paso anterior.

```yaml
# .github/workflows/deploy.yml
name: 'Deploy'

on:
  push:
    branches: [ "main" ]
  pull_request:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest
    environment: producción

    defaults:
      run:
        shell: bash

    steps:
    # Checkout de repositorio al runner de GitHub Actions
    - name: Checkout
      uses: actions/checkout@v3
      
    # Configurar credenciales de AWS
    # Necesitarás reemplazar <ROL_IAM> con el ARN del rol IAM que creaste en el paso anterior
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: <ROL_IAM>
        aws-region: us-east-1
      
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
        
    - name: Configurar entorno Go
      uses: actions/setup-go@v4.0.1

    # Este paso construye la aplicación Go y crea un archivo zip que contiene el binario
    # Es importante notar que el binario debe llamarse "bootstrap"
    - name: Construir aplicación Go
      run: |
        GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o bootstrap main.go
        zip lambda-handler.zip bootstrap

    # Inicializar un directorio de trabajo de Terraform nuevo o existente creando archivos iniciales, cargando cualquier estado remoto, descargando módulos, etc.
    - name: Terraform Init
      run: terraform init
      
    - name: Terraform Format
      run: terraform fmt -check

    - name: Terraform Plan
      run: terraform plan -input=false
      
    - name: Output ref y event_name
      run: |
        echo ${{github.ref}}
        echo ${{github.event_name}}

      # Al hacer push a "main", construir o cambiar la infraestructura según los archivos de configuración de Terraform
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve -input=false

    - name: Output URL de invocación de API Gateway
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: |
          terraform output api_endpoint
```

## Paso 9: Desplegar la aplicación Go en AWS Lambda

Ahora que hemos creado el flujo de trabajo de GitHub Actions, podemos subir nuestro código a GitHub y dejar que GitHub Actions despliegue nuestra aplicación Go en AWS.

```bash
git add .
git commit -m "Desplegar aplicación Go en AWS Lambda"
git push --set-upstream origin main
```

Puedes ver el flujo de trabajo de GitHub Actions yendo a la pestaña "Actions" en tu repositorio de GitHub.

## Paso 10: Probar la API

Ahora que hemos desplegado nuestra aplicación Go en AWS Lambda, podemos obtener la URL de invocación de API Gateway yendo a la pestaña "Actions" en tu repositorio de GitHub y haciendo clic en la última ejecución del flujo de trabajo. Luego, viendo la salida del paso "Output API Gateway invocation URL".

Si hacemos una solicitud a la URL de invocación de API Gateway, deberíamos ver la siguiente respuesta:

![Salida de GitHub Actions](/deploying-go-to-aws-lambda-with-github-actions-and-terraform/http-request.png)

## Conclusión

En este artículo, exploramos cómo desplegar aplicaciones basadas en Go en AWS Lambda usando GitHub Actions y Terraform. Al combinar estas herramientas, los desarrolladores pueden automatizar el proceso de despliegue, simplificar el control de versiones y asegurar prácticas consistentes de infraestructura como código para sus proyectos sin servidor.

> Puedes encontrar el código fuente para este artículo en el [repositorio de GitHub](https://github.com/sebastianmarines/go_lambda_tutorial).