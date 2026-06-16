> **This page is auto-published to Confluence on each tag push.** Do not edit the Confluence page directly — changes will be overwritten. Make changes in the GitHub repository.

# terraform-aws-es-cleanup

**GitHub:** [elocalorg/terraform-aws-es-cleanup](https://github.com/elocalorg/terraform-aws-es-cleanup)

## Status

> ⚠️ **Deprecated** — targets Amazon Elasticsearch Service (AWS ES), which reached end-of-life and has been replaced by Amazon OpenSearch Service. A scan of 305 Terraform workspaces found zero active callers of this module. No new usage should be added. Existing callers should migrate to OpenSearch. No new features will be added; critical fixes only.

## Overview

This module deploys an AWS Lambda function that automatically deletes old Elasticsearch indices on a configurable schedule. It authenticates to the AWS Elasticsearch endpoint using SigV4 and is authorized via the Lambda execution role rather than IP whitelisting. The module also provisions the necessary IAM role, CloudWatch Events schedule, and (optionally) VPC networking resources for the Lambda function.

## Use this module instead of

Do not define the following resources directly. Use this module instead:

* `aws_lambda_function` — the cleanup Lambda; defined directly, it bypasses the IAM and scheduling wiring this module provides
* `aws_iam_role` — the Lambda execution role
* `aws_iam_policy` — the ES and VPC permissions policy
* `aws_iam_role_policy_attachment` — policy attachments for the execution role
* `aws_cloudwatch_event_rule` — the cron schedule trigger
* `aws_cloudwatch_event_target` — connects the schedule to the Lambda
* `aws_lambda_permission` — grants CloudWatch Events permission to invoke the Lambda
* `aws_cloudwatch_log_group` — the Lambda log group

The following resources are **not** managed by this module and should be defined separately by callers when needed:

* `aws_security_group` — a security group is created automatically when `subnet_ids` is provided; if you need additional ingress rules or a custom security group, define it separately and pass its ID via `security_group_ids`

## What this module enforces

* **SigV4 authentication** — the Lambda always authenticates to the ES endpoint using the IAM role; IP whitelisting is not used
* **CloudWatch logging** — a `/aws/lambda/<prefix>es-cleanup<suffix>` log group is always created
* **Least-privilege IAM** — the execution role is scoped to the specific Lambda log group ARN and the account's ES domains

## Usage

```hcl
module "es_cleanup" {
  source  = "app.terraform.io/elocal/es-cleanup/aws"
  version = "~> 1.0"

  es_endpoint = "search-my-cluster-xxxxxxxxxx.us-east-1.es.amazonaws.com"
}
```

VPC-deployed Lambda with custom index retention:

```hcl
module "es_cleanup" {
  source  = "app.terraform.io/elocal/es-cleanup/aws"
  version = "~> 1.0"

  es_endpoint        = "vpc-my-cluster-xxxxxxxxxx.us-east-1.es.amazonaws.com"
  prefix             = "prod_"
  delete_after       = 30
  index              = "logstash-.*"
  subnet_ids         = ["subnet-xxxxxxxx"]
  security_group_ids = ["sg-xxxxxxxx"]
}
```

## Configuration guide

### Index selection (`index` and `skip_index`)

`index` is a regex matched against index names to select which indices are candidates for deletion. The default `.*` matches all indices. `skip_index` is a regex for indices that should always be excluded regardless of `index`; the default `.kibana*` protects Kibana system indices. Override both when you need to target a specific index pattern (e.g. `logstash-.*`) while skipping system indices.

### Index age format (`index_format`)

The module determines index age by parsing a date suffix from the index name using `index_format` (default `%Y.%m.%d`). If your indices use a different date format in their names, set this variable to match. Indices whose names do not contain a parseable date are skipped.

### Retention window (`delete_after`)

Sets how many days of indices to retain. Indices older than this threshold are deleted. The default is 15 days. Choose a value based on your data retention policy and storage constraints.

### Schedule (`schedule`)

A CloudWatch Events cron expression controlling when the Lambda runs. The default `cron(0 3 * * ? *)` runs daily at 03:00 UTC. Adjust if your ES cluster has a preferred maintenance window.

### VPC deployment (`subnet_ids`, `security_group_ids`)

By default the Lambda runs outside a VPC and accesses the ES endpoint over the public internet (authenticated via SigV4). To place the Lambda inside a VPC — required for VPC-only ES endpoints — provide `subnet_ids`. The module creates a security group with HTTPS egress automatically; pass additional security group IDs via `security_group_ids` if needed.

## What this module does not support

* **OpenSearch Service** — this module targets the legacy Amazon Elasticsearch Service API. It is not tested against Amazon OpenSearch Service endpoints and should not be used for new OpenSearch clusters.
* **Index lifecycle management (ILM)** — this module performs time-based deletion only. ILM policies and rollover aliases must be managed outside the module.
* **Custom Lambda layers or runtime versions beyond Python 3.x** — the Lambda runtime is set via `python_version`; only Python 3.x runtimes supported by AWS Lambda are valid values.

## Version history

### 1.0 — initial release

_2021_

* Initial module release: Lambda-based ES index cleanup with SigV4 auth, CloudWatch Events schedule, IAM role, and optional VPC support.

---

## Requirements

| Name | Version |
|------|---------|
| terraform | >= 0.12 |
| aws | n/a |
| archive | n/a |
| null | n/a |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| es\_endpoint | AWS ES FQDN e.g. search-es-demo-xxxxxxxxxx.us-east-1.es.amazonaws.com | `string` | n/a | yes |
| delete\_after | Number of days of indices to preserve | `number` | `15` | no |
| index | Index/indices to process using regex, except those matching `skip_index` | `string` | `".*"` | no |
| index\_format | Combined with `index` to evaluate the index age from its name | `string` | `"%Y.%m.%d"` | no |
| prefix | Prefix for resource names; allows multiple instances in one account | `string` | `""` | no |
| python\_version | Lambda Python runtime version | `string` | `"3.6"` | no |
| schedule | CloudWatch Events cron expression for the cleanup schedule | `string` | `"cron(0 3 * * ? *)"` | no |
| security\_group\_ids | Additional security group IDs to attach to the Lambda (VPC deployments) | `list(string)` | `[]` | no |
| skip\_index | Index/indices to always skip, matched by regex | `string` | `".kibana*"` | no |
| subnet\_ids | Subnet IDs for VPC deployment; leave empty to run outside a VPC | `list(string)` | `[]` | no |
| suffix | Suffix for resource names; allows multiple instances in one account | `string` | `""` | no |
| tags | Tags to apply to all resources | `map` | `{"Name": "es-cleanup"}` | no |
| timeout | Maximum Lambda execution time in seconds | `number` | `300` | no |

## Outputs

None.

## Resources

| Name | Type |
|------|------|
| aws\_lambda\_function.es\_cleanup | resource |
| aws\_iam\_role.role | resource |
| aws\_iam\_policy.policy | resource |
| aws\_iam\_role\_policy\_attachment.policy\_attachment | resource |
| aws\_iam\_role\_policy\_attachment.policy\_attachment\_vpc | resource |
| aws\_cloudwatch\_event\_rule.schedule | resource |
| aws\_cloudwatch\_event\_target.es\_cleanup | resource |
| aws\_lambda\_permission.allow\_cloudwatch | resource |
| aws\_cloudwatch\_log\_group.cwlog | resource |
| aws\_security\_group.lambda | resource |
