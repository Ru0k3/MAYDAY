# MAYDAY

> A Kubernetes add-on that detects potential ephemeral-storage exhaustion early, warns applications, coordinates checkpointing, and enables safe cleanup before storage pressure causes application disruption.

**Project Status:** Planning and Learning Phase  
**Project Type:** Kubernetes Add-on / Operator  
**Primary Focus:** Ephemeral Storage Monitoring, Early Warning, Application Checkpointing and Safe Cleanup

---

## Table of Contents

- [1. Introduction](#1-introduction)
- [2. Problem Statement](#2-problem-statement)
- [3. Project Objectives](#3-project-objectives)
- [4. Proposed Architecture](#4-proposed-architecture)
- [5. Complete Project Workflow](#5-complete-project-workflow)
- [6. Mathematical Calculations](#6-mathematical-calculations)
- [7. Checkpointing Strategy](#7-checkpointing-strategy)
- [8. Checkpoint Verification](#8-checkpoint-verification)
- [9. Safe Cleanup Strategy](#9-safe-cleanup-strategy)
- [10. Technologies](#10-technologies)
- [11. Kubernetes Integration](#11-kubernetes-integration)
- [12. Important Technical Considerations](#12-important-technical-considerations)
- [13. Development Roadmap](#13-development-roadmap)
- [14. Initial Development Environment](#14-initial-development-environment)
- [15. Expected Outcome](#15-expected-outcome)
- [16. Current Project Status](#16-current-project-status)
- [17. Project Vision](#17-project-vision)
- [Contributing](#contributing)

---

## 1. Introduction

Applications running inside Kubernetes Pods sometimes generate large amounts of temporary data. For example, an application may continuously write files into `/tmp` or other temporary storage locations.

If storage consumption increases rapidly, the Pod may reach its ephemeral-storage limit or the Kubernetes node may experience disk pressure.

This can result in Pod eviction, application interruption, or loss of work that has not been saved.

This project aims to reduce that risk by introducing an early-warning and checkpointing mechanism.

Instead of waiting for Kubernetes to evict a Pod, our system monitors storage consumption, estimates how quickly storage may reach a critical threshold, and notifies the application early enough to save its important state.

The application can then create a checkpoint, store it in durable storage backed by Rook and Ceph, verify the checkpoint, and identify temporary data that is genuinely safe to remove.

The objective is to help applications recover or continue their work with less disruption.

---

## 2. Problem Statement

Consider an application running inside a Kubernetes Pod.

The application continuously writes temporary files:

```text
Application
    |
    v
Writes temporary files
    |
    v
/tmp storage increases
    |
    v
Storage approaches its limit
    |
    v
Pod may be evicted
    |
    v
Application gets interrupted
```

Kubernetes provides resource management and eviction mechanisms, but an application may not receive enough warning to create a useful checkpoint before storage exhaustion.

Our project introduces an additional monitoring and coordination layer to help address this problem.

---

## 3. Project Objectives

The main objectives are:

1. Monitor ephemeral-storage consumption of selected Kubernetes Pods.
2. Collect timestamped storage-usage measurements.
3. Calculate storage growth velocity.
4. Estimate the time remaining before a relevant storage threshold is reached.
5. Send early warnings to applications through a defined communication mechanism.
6. Allow applications to create consistent checkpoints of their important state.
7. Store checkpoints in durable storage using a Rook-managed Ceph backend.
8. Verify that checkpoint data has been uploaded successfully.
9. Identify obsolete temporary data through application-approved cleanup.
10. Measure storage usage again after cleanup.
11. Recalculate storage growth and estimated time to threshold.
12. Reduce avoidable application interruptions caused by ephemeral-storage pressure.

---

## 4. Proposed Architecture

The project will be developed as a Kubernetes Operator or custom controller.

It will run inside an existing Kubernetes cluster and monitor selected workloads.

### Main Components

**1. Storage Monitoring Component**

- Monitors selected Pods.
- Collects ephemeral-storage usage information.
- Maintains timestamped usage samples.
- Detects increasing storage consumption.

**2. Storage Calculation Engine**

Calculates storage growth velocity and estimates the time remaining before a relevant storage threshold is reached.

**3. Early Warning Engine**

- Evaluates storage conditions.
- Determines when an application should receive a warning.
- Sends an early-warning notification through an HTTP webhook or another defined protocol.

**4. Checkpoint Coordination Component**

- Coordinates the checkpointing workflow.
- Notifies the application.
- Tracks checkpoint progress and results.
- Handles failures and timeouts.

The application itself is responsible for creating a consistent checkpoint because it understands its internal state.

**5. Rook and Ceph Storage Backend**

Rook manages Ceph within Kubernetes.

Ceph provides durable storage for application checkpoints.

Rook/Ceph is the checkpoint storage backend, not the component responsible for detecting eviction or automatically creating application checkpoints.

**6. Checkpoint Verification Component**

Checks whether the checkpoint was uploaded successfully and whether it can be trusted for recovery.

**7. Safe Cleanup Component**

Coordinates cleanup of temporary files that the application has explicitly identified as no longer required.

The controller must not independently delete application data based only on file age, size, or the fact that it has been checkpointed.

**8. Post-Cleanup Monitoring**

Measures storage usage again after cleanup and recalculates storage growth velocity and time to threshold.

---

## 5. Complete Project Workflow

```text
             Kubernetes Cluster
                     |
                     v
           Storage Monitoring
                     |
                     v
        Collect Storage Measurements
                     |
                     v
          Calculate Growth Velocity
                     |
                     v
         Estimate Time to Threshold
                     |
                     v
          Is Warning Required?
                 /       \
               No         Yes
               |           |
               |           v
               |     Notify Application
               |           |
               |           v
               |    Application Creates
               |       Checkpoint
               |           |
               |           v
               |    Upload Checkpoint
               |           |
               |           v
               |     Rook + Ceph
               |           |
               |           v
               |    Verify Checkpoint
               |           |
               |           v
               |    Application Approves
               |      Safe Cleanup
               |           |
               |           v
               |      Remove Only
               |      Obsolete Data
               |           |
               |           v
               |    Measure Storage Again
               |           |
               |           v
               |    Recalculate TTE
               |           |
               |           v
               |    Continue Monitoring
               |
               v
          Continue Monitoring
```

The workflow is intended to reduce the likelihood of avoidable eviction. It cannot guarantee that a Pod will never be evicted, because Kubernetes node-pressure eviction and other failure conditions can occur independently.

---

## 6. Mathematical Calculations

The project will use mathematical calculations to estimate storage growth and predict when a relevant threshold may be reached.

### 6.1 Storage Growth Velocity

Formula:

```text
V = (U2 - U1) / (T2 - T1)
```

Where:

- V = Storage growth velocity.
- U1 = Earlier storage usage.
- U2 = Current storage usage.
- T1 = Earlier measurement time.
- T2 = Current measurement time.

Example:

If storage increases from 2 GB to 3 GB in 10 seconds:

```text
V = (3 - 2) / 10

V = 0.1 GB/second
```

### 6.2 Time to Threshold (TTE)

Formula:

```text
TTE = (L - U) / V
```

Where:

- L = Relevant storage threshold.
- U = Current storage usage.
- V = Storage growth velocity.

Example:

```text
Storage threshold = 10 GB
Current usage     = 8 GB
Growth velocity   = 0.1 GB/second

TTE = (10 - 8) / 0.1

TTE = 20 seconds
```

This means that, if the current growth rate continues, the threshold may be reached in approximately 20 seconds.

If velocity is zero or negative, the system should not calculate a finite positive time to threshold using this formula.

### 6.3 Checkpoint Time Estimation

The system must consider how much time the checkpointing workflow requires.

```text
Required Time =
    Pause Time
  + Upload Time
  + Verification Time
  + Cleanup Time
  + Safety Buffer
```

Upload time can be estimated as:

```text
Upload Time = Checkpoint Size / Effective Upload Throughput
```

The warning mechanism should consider the estimated time required to complete the workflow.

A fixed warning threshold such as 95% should not be treated as universally safe. The appropriate warning point should depend on storage growth velocity and the time needed to checkpoint and recover space.

---

## 7. Checkpointing Strategy

Checkpointing is an important part of this project.

When an application receives an early warning, it should:

1. Reach a safe execution point.
2. Pause or coordinate relevant operations.
3. Save the important application state.
4. Create a consistent checkpoint.
5. Upload the checkpoint to durable storage.
6. Verify the uploaded checkpoint.
7. Report the checkpoint result to the coordination component.

The application must own the checkpoint creation process because different applications have different internal states and recovery requirements.

### Checkpoint Storage

The selected storage backend is:

**Rook + Ceph**

Rook will manage Ceph within the Kubernetes environment.

Ceph will provide durable storage for checkpoint data.

The exact storage interface, such as a Ceph-backed PVC or an object-storage interface, will be finalized during implementation.

---

## 8. Checkpoint Verification

A checkpoint should not be considered successful merely because an upload request returned successfully.

The verification process may include:

- Upload success status.
- Checkpoint metadata validation.
- Uploaded size comparison.
- Checksum or hash verification.
- Read-back validation when appropriate.

The implementation should balance verification reliability and overhead.

A full read-back of every checkpoint may be expensive for large files, so the verification strategy will be designed according to checkpoint size, storage capabilities, and reliability requirements.

---

## 9. Safe Cleanup Strategy

Cleanup must be handled carefully.

**A checkpointed file is not automatically safe to delete.**

Some checkpoint data or intermediate files may still be needed for future calculations, recovery, or application execution.

Therefore, the application must identify which files are genuinely obsolete.

### Proposed Data Classification

| Data category | Cleanup decision |
|---|---|
| Active working data | Preserve |
| Checkpoint data | Preserve |
| Data needed for future calculations | Preserve |
| Reusable intermediate results | Preserve unless explicitly approved |
| Obsolete temporary files | Cleanup candidate |
| Completed output | Delete only when safely persisted and no longer needed |

The application should explicitly authorize cleanup candidates.

The controller should coordinate cleanup rather than independently inspect application files and decide what to delete.

The implementation must also prevent race conditions where a file becomes active after it has been identified as a cleanup candidate.

---

## 10. Technologies

The planned technology stack is:

| Technology | Purpose |
|---|---|
| Linux | Operating system and process fundamentals |
| Docker | Building and packaging the controller |
| Kubernetes | Container orchestration platform |
| Go | Controller and Operator development |
| Kubernetes API | Pod discovery and cluster interaction |
| CRDs | Project-specific Kubernetes configuration |
| Kubelet metrics | Storage usage measurements |
| Rook | Ceph lifecycle management in Kubernetes |
| Ceph | Durable checkpoint storage |
| HTTP Webhooks | Communication with applications |
| Helm | Installing the add-on into Kubernetes |

Prometheus and Alertmanager are not planned as mandatory components in the initial implementation. The project will implement its own monitoring calculations and warning mechanism.

---

## 11. Kubernetes Integration

The project will be developed as a Kubernetes add-on.

The intended approach is to build a custom controller or Operator.

The controller will:

- Run as a Pod inside the Kubernetes cluster.
- Communicate with the Kubernetes API.
- Discover and monitor selected Pods.
- Collect storage usage measurements.
- Calculate storage growth and time to threshold.
- Notify applications.
- Coordinate checkpointing and cleanup workflows.
- Track workflow status.

The add-on should be installable into an existing Kubernetes cluster without modifying Kubernetes core components.

A Custom Resource Definition (CRD) may be introduced to configure which workloads are monitored and how warning and checkpoint policies are applied.

---

## 12. Important Technical Considerations

The project must account for the following:

1. Ephemeral-storage usage and memory usage are different resources.
2. Pod ephemeral-storage limits and node-level disk pressure are different eviction mechanisms.
3. Storage measurements may be delayed or unavailable.
4. Growth velocity can change suddenly.
5. A checkpoint upload can fail or take longer than expected.
6. Applications may not respond to warnings.
7. A Pod may be evicted before checkpointing completes.
8. Node-pressure eviction may not provide enough termination grace time for checkpointing.
9. Checkpoint verification must be reliable.
10. Cleanup must never remove data required for recovery or future execution.
11. Pod restarts and replacements must be handled correctly.
12. The controller must handle duplicate warnings, retries, timeouts, and stale measurements.

The project should treat checkpointing as a proactive operation rather than relying on Kubernetes termination notifications as the primary trigger.

---

## 13. Development Roadmap

Development will happen step by step.

### Phase 1: Learning Fundamentals

- Linux process and filesystem concepts.
- Docker fundamentals.
- Kubernetes fundamentals.
- Kubernetes storage concepts.
- Go programming fundamentals.

### Phase 2: Kubernetes Controller Development

- Understand the Kubernetes API.
- Learn Custom Resource Definitions.
- Build a basic controller.
- Deploy the controller into a Kubernetes cluster.
- Discover and monitor selected Pods.

### Phase 3: Storage Monitoring

- Collect ephemeral-storage measurements.
- Store timestamped samples.
- Calculate storage growth velocity.
- Implement time-to-threshold calculations.
- Handle missing and stale measurements.

### Phase 4: Early Warning Mechanism

- Define warning levels.
- Implement warning decisions.
- Build application communication using HTTP webhooks.
- Handle application responses and timeouts.

### Phase 5: Checkpoint Integration

- Define an application checkpoint protocol.
- Coordinate safe checkpoint creation.
- Integrate durable storage.
- Upload and verify checkpoints.

### Phase 6: Rook and Ceph Integration

- Learn Rook architecture.
- Deploy Rook in a suitable test environment.
- Configure Ceph storage.
- Connect the checkpoint workflow to the selected Ceph storage interface.

### Phase 7: Safe Cleanup and Recovery

- Define application-approved cleanup candidates.
- Implement cleanup coordination.
- Verify storage recovery after cleanup.
- Recalculate TTE.
- Test checkpoint restoration.

### Phase 8: Testing and Reliability

- Simulate rapid `/tmp` growth.
- Test Pod storage-limit pressure.
- Test node disk pressure.
- Test checkpoint upload failures.
- Test application timeouts.
- Test cleanup failures.
- Test Pod restarts.
- Measure controller overhead.
- Improve reliability and documentation.

---

## 14. Initial Development Environment

The first development environment will be a local Linux-based lab.

Planned tools:

- Linux machine.
- Docker.
- Kubernetes local test environment.
- Go development environment.
- kubectl.
- Helm.
- Rook and Ceph test environment in a later phase.

All development and testing should initially happen in a controlled lab environment rather than a production Kubernetes cluster.

---

## 15. Expected Outcome

The expected outcome is a Kubernetes add-on capable of proactively monitoring ephemeral-storage consumption and coordinating application checkpointing before a storage-related disruption becomes likely.

The system should be able to:

- Detect rapid storage growth.
- Estimate time to a relevant storage threshold.
- Notify applications early.
- Coordinate application checkpointing.
- Store checkpoints in durable Ceph-backed storage.
- Verify checkpoint uploads.
- Coordinate safe cleanup.
- Reassess storage conditions after cleanup.

The project aims to improve application resilience and reduce avoidable work loss due to ephemeral-storage pressure.

---

## 16. Current Project Status

**Current stage: Learning and initial design.**

The project architecture and major technology choices have been identified, but implementation and experimental validation are still pending.

The immediate next step is to learn Docker fundamentals and then progressively build the Kubernetes components.

The project will be developed incrementally, with each component tested before integrating it into the complete system.

---

## 17. Project Vision

The long-term vision is to create a reusable Kubernetes add-on that helps storage-sensitive applications protect their important work before ephemeral-storage pressure causes disruption.

Instead of reacting only after a Pod is in danger, the system will attempt to predict storage pressure, coordinate checkpointing, and recover temporary storage safely.

**Monitor early. Predict intelligently. Checkpoint safely. Clean carefully. Recover confidently.**

---

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) for how to fork the repository, create a focused branch, and open a pull request. Work is currently focused on documentation, design, and the learning-phase components described in the [Development Roadmap](#13-development-roadmap).
