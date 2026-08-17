
# AWS Account Hardening Baseline

## 1. Purpose and Scope

### Purpose

This document records the baseline security controls applied to a single AWS
account and assesses each against the risk it is intended to address. It records
what was configured and why. The same structure can be reused to review other
accounts.

### Scope

Scope is limited to account-level controls within a single AWS account. No AWS
Organizations structure is in place, so organisation-wide guardrails such as
Service Control Policies are not available and are not assessed here.

Four control areas are covered: identity, cost, logging, and monitoring.
Together these establish who can access the account, what they can spend, what
record exists of their actions, and what detects misconfiguration.

### Out of Scope

Workload security, network architecture, and data classification are excluded
as no workloads or data are currently deployed; these become relevant once
compute and storage resources exist. Multi-account governance is excluded for
the reason given above. Incident response is excluded as a process control
rather than a configuration control, though the logging measures described here
are a prerequisite for it.

### Audience

Written for security practitioners and reviewers assessing baseline AWS account
configuration.

### References

Control mappings reference the CIS AWS Foundations Benchmark and ISO/IEC 27001
Annex A. Assessment reflects the account configuration as at August 2026.

## 2. Control Summary

| Control | Risk addressed | Implementation | Verification |
|---|---|---|---|
| Root user hardening | Unrestricted account access via a credential that cannot be constrained by IAM policy or Service Control Policy | Virtual MFA enabled, no access keys present, backup MFA device registered, alternate contacts set | IAM credential report: `<root_account>` row shows `mfa_active` true and both access key fields false |
| Administrative IAM identity | Routine use of root for daily operations, removing separation between account ownership and account operation | Dedicated IAM user created with console access and virtual MFA; root reserved for tasks that require it | IAM credential report confirms MFA active; IAM console confirms attached policies |
| Cost guardrails | Uncontrolled or unnoticed spend from misconfigured or forgotten resources | Zero-spend budget plus monthly cost budget with alerts on actual and forecast thresholds | Billing console: budgets present, thresholds configured, email subscribers confirmed |
| Audit logging | Absence of a durable, tamper-evident record of API activity, preventing investigation and undermining evidential value | Multi-region CloudTrail trail delivering to S3, encrypted with a customer-managed KMS key, log file validation enabled | Trail settings show multi-region, KMS key ID, and validation enabled; digest files present in S3 |
| External access analysis | Resource policies granting access outside the account boundary without detection | IAM Access Analyzer enabled with zone of trust set to the account | Analyzer status Active; findings reviewed and archived with rationale |

## 3. Control Detail

### 3.1 Root user hardening

The root user is the identity created with the AWS account itself. Unlike an IAM user, it exists outside IAM — there is no identity to attach a policy to, and Service Control Policies do not apply to the root of a management account. It therefore cannot be constrained by permission controls, only by protecting the credential itself and detecting its use.

Virtual MFA was enabled on the root user using an authenticator application, with a second device registered as backup. AWS supports up to eight MFA devices per root user; registering more than one removes dependence on a single device, as recovery otherwise requires an identity verification process through AWS Support. No root access keys exist. Alternate contacts for billing, operations and security were set.

Verification is via the IAM credential report, which shows MFA active and both access key fields inactive for the root account.

### 3.2 Administrative IAM Identity

AWS distinguishes between IAM users and IAM roles. An IAM user is a permanent
identity holding long-lived credentials — a console password and, optionally,
access keys that remain valid until explicitly revoked. An IAM role holds no
credentials of its own; it is assumed temporarily by a principal, and AWS STS
issues short-lived credentials that expire automatically. Roles are the
preferred pattern precisely because there is no persistent credential to leak.

A dedicated IAM user <admin-user> was created for day-to-day
administration, with console access and virtual MFA enabled. This separates
account ownership, which remains with the root user, from account operation.
Even in a single-operator account the separation has value: it limits the
circumstances in which root credentials are used, and it means routine activity
is attributable to an identity that can be constrained, monitored, and revoked.

MFA on this identity is not optional. The user holds `AdministratorAccess` and
is therefore near-equivalent to root in practical capability; protecting root
while leaving an administrative IAM user on password-only authentication would
displace the risk rather than remove it.

**Documented exception.** `AdministratorAccess` is an AWS-managed policy
granting all actions on all resources. It is not least privilege and would
constitute a finding in any reviewed environment. It is accepted here as a
deliberate exception on the basis that the account has a single operator, holds
no production data, and exists for evaluation purposes. In a multi-user
environment the appropriate pattern is federated access through IAM Identity
Center, with permission sets scoped by function and elevated access granted
temporarily rather than held permanently.

**Verification.** IAM credential report confirms MFA is active and no access keys are 
present for the user. The IAM console confirms attached policies and last
activity.

### 3.3 Cost Guardrails

Cost controls are not conventionally treated as security controls, but they
serve two purposes in a baseline. The first is operational: unexpected charges
indicate resources running that were not intended to be running, which is a
configuration signal as much as a financial one. The second is detective.
Compromised AWS credentials are commonly used for cryptocurrency mining, and
the resulting compute spend is frequently the earliest visible indicator of
compromise — often surfacing before any dedicated detection control fires.

A zero-spend budget was configured to alert on any charge beyond free tier
usage, alongside a monthly cost budget with alerts at defined thresholds of
both actual and forecast spend. The forecast alert is the more useful of the
two, as it warns before the spend is incurred rather than after.

Budget alerts reduce time to detection; they do not prevent spend. Preventing
resource creation requires separate controls, such as Service Control Policies
restricting instance types or regions, which are unavailable in a single
account.

**Verification.** Billing console confirms budgets present with configured
thresholds and active email subscribers.

### 3.4 Audit Logging

CloudTrail records API activity across the account. Event history, enabled by
default, retains 90 days of management events and is queryable in the console
but cannot be configured or retained beyond that window. A trail delivers
events durably to S3 with retention governed by the bucket's lifecycle policy,
and is what an environment requires to claim CloudTrail is enabled.

A multi-region trail was configured, capturing management events for both read
and write operations. Multi-region coverage is material: most AWS services are
regionally scoped, and activity in an unmonitored region is a recognised gap.
Data events were deliberately excluded on cost grounds and are noted as a
limitation.

Logs are encrypted using SSE-KMS with a customer-managed key. This provides two
properties that SSE-S3 does not. First, decryption requires both S3 object
permission and permission under the KMS key policy — two independent
authorisation gates rather than one. Second, every use of the key is recorded
in CloudTrail, producing an auditable record of who decrypted what and when.
Customer-managed keys additionally allow the key policy and rotation schedule
to be controlled directly, and permit the key to be disabled, rendering the
data unreadable without modifying any object.

Log file validation is enabled. CloudTrail produces hourly digest files
hash-chained to their predecessors, allowing modification or deletion of log
files to be detected. This is a detective control on log integrity: it does not
prevent tampering, but ensures tampering cannot occur unnoticed. Without it,
the audit record is a set of files that any principal with write access to the
bucket could alter — which materially weakens its evidential value.

**Limitation.** CloudTrail collects; it does not analyse. Interrogating the
S3 bucket directly is impractical, and no query layer (Athena, CloudWatch Logs,
CloudTrail Lake, or an external SIEM) is currently configured. Log collection
without an accessible query path provides limited investigative capability.

**Verification.** Trail configuration shows multi-region enabled, a
customer-managed KMS key ID, and log file validation enabled. Digest files are
present in the S3 bucket.
