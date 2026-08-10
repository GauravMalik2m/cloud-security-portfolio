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
