---
name: terraform-gcp
description: TerraformによるGCPリソース管理のベストプラクティス。モジュール構成、State管理、環境分離、Secret Manager連携、IAM設計を提供します。
---

# Terraform GCP スキル

## 概要

Terraform による GCP リソース管理のベストプラクティス。

---

## モジュール構成

```
/infra
  /modules
    /cloud-run
    /cloud-sql
    /cloud-scheduler
    /networking
    /iam
  /environments
    /dev
      main.tf
      variables.tf
      terraform.tfvars
    /prod
      main.tf
      variables.tf
      terraform.tfvars
  main.tf
  variables.tf
  outputs.tf
  backend.tf
```

---

## State 管理

- GCS バックエンドを使用
- 環境ごとに state ファイルを分離

```hcl
terraform {
  backend "gcs" {
    bucket = "sns-ai-operator-tfstate"
    prefix = "terraform/state"
  }
}
```

---

## 環境分離（dev/prod）

- `terraform.tfvars` で環境固有値を管理
- モジュールは共通、パラメータで環境差を表現
- workspace は使用せず、ディレクトリで分離

---

## Secret Manager 連携

```hcl
resource "google_secret_manager_secret" "openai_api_key" {
  secret_id = "openai-api-key"
  replication {
    auto {}
  }
}

# Cloud Run から Secret Manager を参照
resource "google_cloud_run_v2_service" "api" {
  template {
    containers {
      env {
        name = "OPENAI_API_KEY"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.openai_api_key.id
            version = "latest"
          }
        }
      }
    }
  }
}
```

---

## リソース命名規約

- プロジェクトプレフィックス: `sns-ai-operator`
- 環境サフィックス: `-dev`, `-prod`
- 例: `sns-ai-operator-api-dev`

---

## IAM 設計

- サービスアカウントは用途ごとに分離
- 最小権限原則
- Cloud Run → Cloud SQL: `cloudsql.client`
- Cloud Run → Secret Manager: `secretmanager.secretAccessor`
- Cloud Scheduler → Cloud Run: `run.invoker`
