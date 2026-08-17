
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

### 3.5 External Access Analysis

IAM Access Analyzer evaluates resource-based policies — including S3 bucket
policies, KMS key policies, and IAM role trust policies — to identify resources
accessible from outside a defined zone of trust. An analyzer was enabled with
the zone of trust set to the account boundary, meaning any grant to an external
principal is reported as a finding.

The service applies automated reasoning rather than signature or pattern
matching: policies are translated into logical statements and evaluated to
determine whether external access is provable. This is materially different
from keyword-based scanning, as it accounts for conditions that restrict access
and therefore avoids reporting grants that are correctly constrained. The
CloudTrail S3 bucket and KMS key both grant access to an AWS service principal
but are scoped by source-account conditions, and are correctly not reported.

No active findings are present, consistent with an account containing no
externally shared resources.

Findings assessed as intended are archived with a documented rationale rather
than left open. Archive rules can be configured to suppress recurring
known-good patterns automatically. This matters operationally: a findings queue
that accumulates accepted exceptions ceases to be monitored, and alert fatigue
is itself a control failure.

**Limitation.** The free tier of Access Analyzer covers external access only.
Unused access analysis, which identifies IAM identities holding permissions
they have not exercised, is a separate paid feature and is not enabled. Least
privilege is therefore not currently evidenced.

**Verification.** Analyzer status Active with zone of trust set to the account;
findings list reviewed.


### 3.6 Public Access Prevention

S3 Block Public Access prevents bucket policies and access control lists from
granting public access to buckets or objects. Applied at the account level, it
overrides bucket-level configuration: a bucket cannot be made public even by a
principal with permission to modify its policy.

This was confirmed empirically. An attempt to create a publicly readable bucket
during testing was refused at the account level, despite the operating identity
holding `AdministratorAccess`. The bucket policy could not be applied while the
account-level setting remained enabled.

The distinction is worth stating precisely. S3 is not public by default;
Block Public Access does not remediate existing exposure so much as remove the
ability to create it. It is a preventive control operating above the resource,
structurally comparable to a Service Control Policy in that the principal
subject to it cannot override it locally.

This is stronger than detection of the same condition. IAM Access Analyzer
would report a publicly accessible bucket, but only after it exists, and only
if the finding is read and acted upon. Block Public Access makes the state
unreachable. Detection remains necessary — preventive controls can be disabled,
and account-level Block Public Access can itself be turned off by a
sufficiently privileged principal — but the two operate at different points and
are complementary rather than alternative.

**Verification.** S3 console confirms all four account-level Block Public
Access settings enabled. Modification of this setting is recorded in CloudTrail
and would be a candidate for alerting.

## 4. Preventive and Detective Controls

The controls in this baseline divide into two categories, and the distinction
determines what each is capable of.

Preventive controls make an undesired state unreachable. Account-level S3 Block
Public Access is the clearest example: a bucket cannot be made public while it
is enabled, regardless of the permissions held by the principal attempting it.
Its effectiveness derives from where it sits — above the resource, outside the
reach of the identity being constrained. Service Control Policies operate on
the same principle at organisation scale, and permission boundaries do so for
individual identities.

Detective controls report an undesired state after it exists. CloudTrail, IAM
Access Analyzer, budget alerts, and log file validation are all detective. They
do not prevent anything; they shorten the time between an event occurring and
it being known.

Preventive controls are stronger where they are available, but the set of
available preventive controls is small. Most security-relevant states in a
cloud environment cannot be prevented without also preventing legitimate work,
which is why the majority of AWS security tooling is detective. This has a
practical consequence: the useful question when assessing an environment is not
which detective controls are enabled, but whether their output is monitored and
acted upon. A finding that is generated and never read provides no assurance.
Alert fatigue is a control failure, not an operational inconvenience.

Root user hardening does not fit either category cleanly, and the reason is
instructive. The root user cannot be constrained by permission controls at all
— there is no IAM identity to attach a policy to, and Service Control Policies
do not apply to the root of a management account. The control is therefore
applied to the credential rather than the permission: multi-factor
authentication raises the difficulty of unauthorised use, and monitoring of
root activity provides detection. Where a permission cannot be restricted, the
available controls shift to authentication strength and detection.

The two categories are complementary rather than alternative. Preventive
controls can themselves be disabled by a sufficiently privileged principal;
account-level Block Public Access can be turned off. Detection of changes to
preventive controls is therefore part of a complete baseline, and is a gap in
the current configuration.

## 5. Documented Exceptions

| Exception | Rationale | Compensating controls | Review trigger |
|---|---|---|---|
| `AdministratorAccess` attached directly to the administrative IAM user | Single-operator account holding no production data or workloads; scoping permissions by function provides no meaningful separation where only one principal exists | MFA enforced on the identity; all API activity recorded in CloudTrail with log file validation; no access keys issued, limiting use to authenticated console sessions | Addition of a second user; deployment of any workload or data of value |

`AdministratorAccess` is an AWS-managed policy granting all actions on all
resources. It is not least privilege and would be recorded as a finding in any
reviewed environment. It is accepted here as a deliberate and time-limited
exception rather than an oversight.

The appropriate pattern in a multi-user environment is federated authentication
through IAM Identity Center, with permission sets scoped by job function and
administrative access granted on an elevated, time-bound basis rather than held
permanently. Recording the exception with an explicit review trigger is what
distinguishes an accepted risk from an unmanaged one.

## 7. Limitations and Next Steps

### Limitations

This baseline addresses account-level configuration only. It provides no
assurance regarding workload security, network architecture, or data handling,
as no workloads, network resources, or data are currently deployed. The
controls described establish a starting position rather than a complete
security posture.

Several specific gaps are noted:

No detection exists for changes to the controls themselves. Disabling
account-level Block Public Access, stopping the CloudTrail trail, or removing
MFA from the administrative identity would each be recorded in CloudTrail but
would not generate an alert. Preventive controls that can be silently disabled
provide weaker assurance than their configuration suggests.

No query layer is configured over CloudTrail. Logs are collected and retained
but cannot be interrogated efficiently, which limits investigative capability
to the 90-day console event history.

Least privilege is not evidenced. The single administrative identity holds
unrestricted permissions, and unused access analysis is not enabled.

Configuration state is not continuously monitored. Verification described in
this document is point-in-time and manual; drift between assessments would not
be detected.

Data events are not captured in CloudTrail, so object-level access to S3 and
Lambda invocations are not recorded.

### Next Steps

The following are sequenced by dependency rather than priority:

1. Configure alerting on root account usage and on modification of the controls
   described in this document, via CloudWatch Logs metric filters or
   EventBridge rules.
2. Enable AWS Config with a baseline rule set to provide continuous
   configuration monitoring in place of point-in-time verification.
3. Establish a query path over CloudTrail, using Athena over the existing S3
   bucket as the lowest-cost option.
4. Replace the directly-attached administrative policy with IAM Identity Center
   permission sets once more than one principal requires access.
5. Extend the baseline to network and workload controls as resources are
   deployed.

### Scope for Reuse

The control structure in this document is intended to be reusable. Applied to a
multi-account environment, the material additions would be organisation-level
guardrails through Service Control Policies, centralised logging into a
dedicated account with restricted write access, and delegated administration of
security services. The account-level controls described here remain necessary
in that context but are no longer sufficient.
