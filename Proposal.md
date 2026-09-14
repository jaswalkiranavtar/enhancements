## Product Requirement Document (PRD)## Project Overview## Executive Summary
The goal of this project is to build a Multi-Tenant Distributed ResourceQuota Controller on top of Open Cluster Management (OCM). Kubernetes natively enforces ResourceQuota limits strictly within the boundaries of an individual namespace on a single cluster. This solution creates a custom control loop on the OCM Hub cluster that aggregates, balances, and dynamically distributes a single global resource budget (CPU and Memory) across multiple namespaces spanning multiple managed spoke clusters.
This solution is designed for multi-tenant environments where a tenant (e.g., tenant1) owns matching functional namespaces across a fleet of clusters. It ensures optimal resource utilization by dynamically shifting quota allocations to clusters experiencing high deployment or scaling demands (such as HPA scale-ups), borrowing safely from clusters with idle capacity.
## Objectives & Scope

* Dynamic Resource Allocation: Manage aggregate CPU and Memory pools globally based on declared allocation (total requests and limits) rather than real-time consumption.
* Multi-Tenant Isolation: Scale seamlessly across dozens of independent tenants using dedicated Custom Resources (CRs) per tenant.
* Zero Direct Cluster Access: Ensure the Hub cluster acts entirely via asynchronous, pull/push-based OCM primitives (ManifestWork, Policy), removing any requirement for the Hub to directly connect to spoke API endpoints.
* Low Maintenance & Native Design: Implement as a lightweight, custom Go operator built via Kubebuilder, preventing standard manual overhead.

------------------------------
## Architecture Design & Primitives

                           +------------------------------------------+

                           |                 OCM HUB                  |
                           |                                          |
                           |  [DRQ: tenant1]       [DRQ: tenant2]     |
                           |         │                   │            |
                           |         ▼                   ▼            |
                           |       [ Custom Multi-Tenant Controller ] │

                           |         │                   │            |
                           |         ▼ (Updates Spec)    ▼            |
                           |   [ManifestWork]      [OCM Policy]       |
                           +---------┬───────────────────┬------------+
                                     │                   │
             ┌───────────────────────┴──────┐            │ (Asynchronous Push/Pull)
             ▼                              ▼            ▼
     +───────────────────────+   +───────────────────────+   +───────────────────────+

     |    SPOKE CLUSTER 1    |   |    SPOKE CLUSTER 2    |   |    SPOKE CLUSTER 3    |
     |                       |   |                       |   |                       |
     |  ns: tenant1 (Quota)  |   |  ns: tenant1 (Quota)  |   |  ns: tenant1 (Quota)  |
     |  ns: tenant2 (Quota)  |   |  ns: tenant2 (Quota)  |   |  ns: tenant2 (Quota)  |
     +───────────────────────+   +───────────────────────+   +───────────────────────+

## Component Breakdown

   1. DistributedResourceQuota (DRQ) CRD: The Hub-side API where administrators define global limits, tenant mapping, and cluster targeting per tenant.
   2. Custom Hub Controller: The core reconciliation engine watching DRQs, OCM ManifestWorks, and OCM Policy violations. It calculates capacity shifts using an isolated distribution loop.
   3. OCM Governance Framework (Policy Engine): Deployed from the Hub to spokes to capture local FailedCreate events in tenant namespaces caused by quota constraints. It reports violations back to the Hub asynchronously via a NonCompliant state.
   4. OCM Workload Delivery (ManifestWork): Houses and mutates the concrete ResourceQuota objects applied inside the target spoke cluster namespaces.

------------------------------
## Functional Requirements## 1. Multi-Tenant Global Pool Management

* FR-1.1: The solution must support defining separate global resource budgets per tenant using a unique DistributedResourceQuota instance.
* FR-1.2: The controller must safely target custom namespaces across clusters mapped to that specific tenant identity.
* FR-1.3: The sum of all distributed sub-quotas (spec.hard inside individual spoke ManifestWorks) must never exceed the global spec.hard defined in the tenant's DRQ.

## 2. Zero-Direct-Inbound Architecture (Pull/Push Status Mechanism)

* FR-2.1: The Hub cluster must not initiate direct network connections or raw API calls to managed spoke clusters.
* FR-2.2: The controller must read spoke quota usage metrics solely through OCM ManifestWork status feedback data.
* FR-2.3: The controller must intercept out-of-quota signals (such as an HPA failing to scale or a direct deployment block) by tracking OCM ConfigurationPolicy compliance states.

## 3. Dynamic Quota Redistribution Loop

* FR-3.1: When an OCM Policy reports a NonCompliant event (quota exceeded) for a tenant on Cluster-A, the Hub controller must execute a rebalancing routine.
* FR-3.2: The controller must locate clusters within the same tenant pool that have unutilized allocation headroom (based on the status.used value returned by ManifestWork feedback).
* FR-3.3: The controller must decrement the quota slice of idle clusters and increment the quota slice of the choked cluster by rewriting their respective ManifestWork specs on the Hub.
* FR-3.4: The controller must maintain a customizable "Minimum Floor" constraint per cluster (e.g., 2 CPUs, 4GiB) to ensure clusters are never completely depleted of resources during a rebalance.

------------------------------
## Technical Specifications & Schema Definitions## 1. DistributedResourceQuota CRD Spec

apiVersion: apiextensions.k8s.io/v1kind: CustomResourceDefinitionmetadata:
  name: distributedresourcequotas.quota.custom.iospec:
  group: quota.custom.io
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [tenantRef, targetNamespace, hard, clusterSelector]
              properties:
                tenantRef:
                  type: string
                targetNamespace:
                  type: string
                hard:
                  type: object
                  properties:
                    requests.cpu: { type: string }
                    requests.memory: { type: string }
                    limits.cpu: { type: string }
                    limits.memory: { type: string }
                clusterSelector:
                  type: object
                  properties:
                    matchLabels:
                      type: object
                      additionalProperties: { type: string }

## 2. Automated OCM ConfigurationPolicy Template
The controller will automatically deploy and monitor this policy for every managed cluster matching the tenant's selector:

apiVersion: policy.open-cluster-management.io/v1kind: Policymetadata:
  name: watch-quota-failures-tenant1
  namespace: hub-governance-nsspec:
  remediativeAction: inform
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: check-tenant1-quota-events
        spec:
          severity: high
          object-templates:
            - complianceType: mustnothave
              objectDefinition:
                apiVersion: v1
                kind: Event
                metadata:
                  namespace: tenant1  # Dynamic based on DRQ spec
                reason: FailedCreate
                message: "*exceeded quota*"

------------------------------
## Non-Functional Requirements

* Scalability: The custom Go controller must handle up to 100 clusters and 50 unique tenants without running into Kubernetes API throttling or controller memory leaks.
* Performance / Convergence Speed: Quota redistribution—from the initial local failure event to the updated ResourceQuota landing on the spoke cluster—must complete within under 30 seconds.
* Resiliency: If the custom Hub controller crashes, the local ResourceQuota configurations on the spoke clusters must remain active at their last assigned state (fail-safe operational stability).

------------------------------
## Verification & Validation Plan

* Test Case 1: Multi-Tenant Isolation Verification
* Action: Trigger a FailedCreate event due to quota starvation in tenant1 on Cluster-1.
   * Expected: Only tenant1's quota configurations are updated. tenant2 configurations on Cluster-1 and all other clusters must remain completely unaffected.
* Test Case 2: Out of Quota Redistribution via HPA
* Action: Trigger an HPA scale-up on Cluster-2 that exceeds its local ResourceQuota allocation.
   * Expected: The OCM Policy flips to NonCompliant. The Hub controller shaves quota allocation off an idle cluster and provisions it to Cluster-2. The OCM Policy returns to a Compliant status as the HPA scaling finishes successfully.
* Test Case 3: Global Cap Enforcement
* Action: Force multiple clusters to request maximum capacity simultaneously, exceeding the combined global DRQ budget.
   * Expected: The controller distributes resources up to the exact limit of the global pool definition, safeguarding the system from over-provisioning.

------------------------------
Now that the PRD constraints and definitions are fully established, would you like to review a detailed Kubebuilder boilerplate structure or see the complete Go code implementation for the main controller reconciliation loop?

