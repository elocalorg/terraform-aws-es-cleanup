> **This page is auto-published to Confluence on every push to `master` and on each tag push.** Do not edit the Confluence page directly — changes will be overwritten. Make changes in the GitHub repository.

# terraform-aws-es-cleanup

**GitHub:** [elocalorg/terraform-aws-es-cleanup](https://github.com/elocalorg/terraform-aws-es-cleanup)

## Status

<ac:structured-macro ac:name="status"><ac:parameter ac:name="colour">Grey</ac:parameter><ac:parameter ac:name="title">Retired</ac:parameter></ac:structured-macro>

**Retired (November 2026)** — targets Amazon Elasticsearch Service, which is end-of-life, and no workspace or module uses this module. It has been removed from the Terraform module registry and the repository is archived. Do not use. See [DEVOPS-3841](https://elocal.atlassian.net/browse/DEVOPS-3841).

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

<!-- BEGIN_TF_DOCS -->
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 0.12 |
## Providers

| Name | Version |
|------|---------|
| <a name="provider_archive"></a> [archive](#provider\_archive) | n/a |
| <a name="provider_aws"></a> [aws](#provider\_aws) | n/a |
| <a name="provider_null"></a> [null](#provider\_null) | n/a |
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_delete_after"></a> [delete\_after](#input\_delete\_after) | Numbers of days to preserve | `number` | `15` | no |
| <a name="input_es_endpoint"></a> [es\_endpoint](#input\_es\_endpoint) | AWS ES FQDN e.g. search-es-demo-xxxxxxxxxx.eu-west-1.es.amazonaws.com | `string` | n/a | yes |
| <a name="input_index"></a> [index](#input\_index) | Index/indices to process using regex, except the one matching `skip_index` regex | `string` | `".*"` | no |
| <a name="input_index_format"></a> [index\_format](#input\_index\_format) | Combined with 'index' varible is used to evaluate the index age | `string` | `"%Y.%m.%d"` | no |
| <a name="input_prefix"></a> [prefix](#input\_prefix) | A prefix for the resource names, this helps create multiple instances of this stack for different environments | `string` | `""` | no |
| <a name="input_python_version"></a> [python\_version](#input\_python\_version) | Lambda Python version to be used | `string` | `"3.6"` | no |
| <a name="input_schedule"></a> [schedule](#input\_schedule) | Cloudwatch Cron Schedule expression for running the cleanup function | `string` | `"cron(0 3 * * ? *)"` | no |
| <a name="input_security_group_ids"></a> [security\_group\_ids](#input\_security\_group\_ids) | Addiational Security Ids To add. | `list(string)` | `[]` | no |
| <a name="input_skip_index"></a> [skip\_index](#input\_skip\_index) | Index/indices to skip | `string` | `".kibana*"` | no |
| <a name="input_subnet_ids"></a> [subnet\_ids](#input\_subnet\_ids) | Subnet IDs you want to deploy the lambda in. Only fill this in if you want to deploy your Lambda function inside a VPC. | `list(string)` | `[]` | no |
| <a name="input_suffix"></a> [suffix](#input\_suffix) | A suffix for the resource names, this helps create multiple instances of this stack for different environments | `string` | `""` | no |
| <a name="input_tags"></a> [tags](#input\_tags) | Tags to apply | `map` | <pre>{<br/>  "Name": "es-cleanup"<br/>}</pre> | no |
| <a name="input_timeout"></a> [timeout](#input\_timeout) | Maximum lambda execution time | `number` | `300` | no |


## Resources

| Name | Type |
|------|------|
| [aws_cloudwatch_event_rule.schedule](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_event_rule) | resource |
| [aws_cloudwatch_event_target.es_cleanup](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_event_target) | resource |
| [aws_cloudwatch_log_group.cwlog](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group) | resource |
| [aws_iam_policy.policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy) | resource |
| [aws_iam_role.role](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) | resource |
| [aws_iam_role_policy_attachment.policy_attachment](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment) | resource |
| [aws_iam_role_policy_attachment.policy_attachment_vpc](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment) | resource |
| [aws_lambda_function.es_cleanup](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_function) | resource |
| [aws_lambda_permission.allow_cloudwatch](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_permission) | resource |
| [aws_security_group.lambda](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) | resource |
| [archive_file.es_cleanup_lambda](https://registry.terraform.io/providers/hashicorp/archive/latest/docs/data-sources/file) | data source |
| [aws_caller_identity.current](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/caller_identity) | data source |
| [aws_iam_policy_document.policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_region.current](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/region) | data source |
| [aws_subnet.selected](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/subnet) | data source |
| [null_data_source.lambda_file](https://registry.terraform.io/providers/hashicorp/null/latest/docs/data-sources/data_source) | data source |
<!-- END_TF_DOCS -->
