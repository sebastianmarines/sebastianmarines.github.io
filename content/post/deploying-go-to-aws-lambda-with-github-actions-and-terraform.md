---
title: "Deploying Go to AWS Lambda with GitHub Actions and Terraform"
date: 2023-07-12T18:39:05-06:00
summary: "In this article, we will explore how to deploy Go-based applications to AWS Lambda using the seamless integration of GitHub Actions and Terraform."
tags: ["Go", "AWS", "Lambda", "GitHub", "Terraform"]
author: "Sebastian Marines"
categories: [" "]
---

## Introduction

Deploying serverless applications has become increasingly popular due to its scalability, cost-efficiency, and ease of management. Amazon Web Services (AWS) Lambda provides a powerful serverless computing platform, while Go (Golang) offers a performant and statically typed programming language.

In this article, we will explore how to deploy Go-based applications to AWS Lambda using the seamless integration of GitHub Actions and Terraform. By combining these tools, developers can automate the deployment process, streamline version control, and ensure consistent infrastructure as code practices for their serverless projects.

## Prerequisites

Before we begin, you will need to have the following prerequisites:

- An AWS account
- A GitHub account
- AWS CLI installed and configured with your AWS credentials (see the [AWS documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) for instructions)
- Go installed (see the [Go documentation](https://golang.org/doc/install) for instructions)
- Terraform installed (see the [Terraform documentation](https://learn.hashicorp.com/tutorials/terraform/install-cli) for instructions)

## Step 1: Create a GitHub repository

First of all, create a GitHub repository for your Go application.

Go to [GitHub](https://github/com) and click on "New repository". Enter a name for your repository and click on "Create repository".

## Step 2: Create a Go application

In an empty directory, create a Go application:

```bash
go mod init go_lambda
```

This will create a `go.mod` file that will be used to manage dependencies for your Go application.

Next, initialize the Git repository:

```bash
git init
```

Next, create a `.gitignore` file:

```bash
# .gitignore
**/.terraform/*
*.tfstate
bootstrap
lambda-handler.zip
```

Lastly, set the remote origin:

```bash
git remote add origin <YOUR_REPOSITORY_URL>
```

## Step 3: Create a main.go file

Next, create the main.go file:

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
 // Make the handler available for Remote Procedure Call by AWS Lambda
 lambda.Start(hello)
}
```

This file contains the minimal code required to create a Go application that can be deployed to AWS Lambda. It imports the `events` and `lambda` packages from the `aws-lambda-go` library and defines a `hello` function that returns a response that our API Gateway will return to the client. The `main` function is the entry point for our application and calls the `hello` function. The `lambda.Start` function starts the AWS Lambda handler[^1].

[^1]: <https://docs.aws.amazon.com/lambda/latest/dg/golang-handler.html>

## Step 4: Install dependencies

Next, install the dependencies for your Go application:

```bash
go get github.com/aws/aws-lambda-go/events
go get github.com/aws/aws-lambda-go/lambda
```

This will install *aws-lambda-go/events*, which contains the `APIGatewayProxyResponse` struct, and *aws-lambda-go/lambda*, which contains the `lambda.Start` function.

## Step 5: Create the remote backend

We will use S3 as our remote backend for Terraform. This is necessary because we will be using GitHub Actions to deploy our code and we need to store the Terraform state in a remote location.

> **NOTE** We will not be creating a DynamoDB table for state locking to keep things simple, but you should consider doing this if you are deploying to production. You can read more about this in the [Terraform documentation](https://www.terraform.io/docs/language/settings/backends/s3.html).

To create the S3 bucket for our remote backend, we will use the following command:

```bash
aws s3api create-bucket --bucket <YOUR_BUCKET_NAME> --region us-east-1
```

Alternatively, you can create the bucket using the AWS Console following the instructions in the [AWS documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html).

## Step 6: Creating the Terraform configuration

Now that we have created our Go application and installed the dependencies, we can create the Terraform configuration. We will use the following Terraform configuration to deploy our Go application to AWS Lambda:

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
    # Replace this with your bucket name!
    bucket = "<YOUR_BUCKET_NAME>"
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
  description = "This is my API for demonstration purposes"
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

## Step 7: Configure AWS credentials for GitHub Actions

To configure AWS credentials for GitHub Actions, we will use OIDC federation. This is a secure way to authenticate with AWS without having to store your AWS credentials in your GitHub repository.

We will need to follow the following steps to configure AWS credentials for GitHub Actions:

1. Open the AWS Console and navigate to the [IAM service](https://console.aws.amazon.com/iam).
2. Click on "Identity providers" in the left navigation bar.
3. Click on "Add provider".
4. Select "OpenID Connect" as the provider type.
5. For "Provider URL", enter `https://token.actions.githubusercontent.com`[^2].
6. Click on "Get thumbprint".
7. For "Audience", enter `sts.amazonaws.com`[^2].
8. Click on "Add provider".

Next, we will need to create an IAM role for GitHub Actions. We will need to follow the following steps to create an IAM role for GitHub Actions:

1. Click on the provider you just created. It should be called "token.actions.githubusercontent.com".
2. Click on "Assign role".
3. Select "Create a new role" and click on "Next".
4. For "Identity provider", select "token.actions.githubusercontent.com".
5. For *Audience*, enter `sts.amazonaws.com`.
6. Click on "Next: Permissions".
7. Select "AmazonS3FullAccess", "AWSLambdaFullAccess", "IAMFullAccess", and "AmazonAPIGatewayAdministrator" and click on "Next: Tags".
8. Click on "Next: Review".
9. For "Role name", enter `github-actions` and click on "Create role".
10. Search for the role you just created and click on it.
11. Copy the "ARN" (It should look something like `arn:aws:iam::000000000000:role/github-actions`). You will need this in the next step.

> ***NOTE:*** You should consider using a more restrictive IAM policy for your GitHub Actions IAM role.

[^2]: <https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#adding-the-identity-provider-to-aws>

## Step 8: Create the GitHub Actions workflow

Next, we will create the GitHub Actions workflow. We will use the following workflow to deploy our Go application to AWS Lambda:

Do not forget to replace `<IAM_ROLE>` with the IAM role ARN you created in the previous step.

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
    environment: production

    defaults:
      run:
        shell: bash

    steps:
    # Checkout the repository to the GitHub Actions runner
    - name: Checkout
      uses: actions/checkout@v3
      
    # Configure AWS Credentials
    # You will need to replace <IAM_ROLE> with the IAM role ARN you created in the previous step
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: <IAM_ROLE>
        aws-region: us-east-1
      
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
        
    - name: Setup Go environment
      uses: actions/setup-go@v4.0.1

    # This step builds the Go application and creates a zip file containing the binary
    # It is important to note that the binary must be named "bootstrap"
    - name: Build Go application
      run: |
        GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o bootstrap main.go
        zip lambda-handler.zip bootstrap

    # Initialize a new or existing Terraform working directory by creating initial files, loading any remote state, downloading modules, etc.
    - name: Terraform Init
      run: terraform init
      
    - name: Terraform Format
      run: terraform fmt -check

    - name: Terraform Plan
      run: terraform plan -input=false
      
    - name: Output ref and event_name
      run: |
        echo ${{github.ref}}
        echo ${{github.event_name}}

      # On push to "main", build or change infrastructure according to Terraform configuration files
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve -input=false

    - name: Output API Gateway invocation URL
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: |
          terraform output api_endpoint
```

## Step 9: Deploy the Go application to AWS Lambda

Now that we have created the GitHub Actions workflow, we can push our code to GitHub and let GitHub Actions deploy our Go application to AWS.

```bash
git add .
git commit -m "Deploy Go application to AWS Lambda"
git push --set-upstream origin main
```

You can view the GitHub Actions workflow by going to the "Actions" tab in your GitHub repository.

## Step 10: Test the API

Now that we have deployed our Go application to AWS Lambda, we can get the API Gateway invocation URL by going to the "Actions" tab in your GitHub repository and clicking on the latest workflow run. Then seeing the output of the "Output API Gateway invocation URL" step.

If we make a request to the API Gateway invocation URL, we should see the following response:

![GitHub Actions output](/deploying-go-to-aws-lambda-with-github-actions-and-terraform/http-request.png)

## Conclusion

In this article, we explored how to deploy Go-based applications to AWS Lambda using GitHub Actions and Terraform. By combining these tools, developers can automate the deployment process, streamline version control, and ensure consistent infrastructure as code practices for their serverless projects.

> You can find the source code for this article in the [GitHub repository](https://github.com/sebastianmarines/go_lambda_tutorial).