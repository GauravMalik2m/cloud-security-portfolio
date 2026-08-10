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
