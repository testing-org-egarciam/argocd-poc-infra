# GitOps Environment Config
This repository stores environment-specific values for all applications managed by the Hub.

## Role in GitOps (The "Where")
This repo is the source of truth for the state of specific environments (Prod, Staging, Dev).

## Structure
- `values/laura-app/prod.yaml`: Production overrides.
- `values/laura-app/stg.yaml`: Staging overrides.

## Isolation
Access is controlled via GitHub CODEOWNERS to ensure tenant laura-app cannot modify payment-app configs.
