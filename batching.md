# [Mega-Issue] Modernize Optics Batch Inference Pipeline (Airflow -> Cloud Batch)

## Executive Summary & Context (The "Why")

The current `optics-pipeline` Airflow DAG was originally designed for a 1:1 workflow: a user submits a single video, Airflow mutates a configuration file, and a single Cloud Batch job is spun up to process it. As our users have scaled to processing thousands of videos (submitting thousands of single-batch jobs), this architecture has hit critical bottlenecks. 

We are refactoring this pipeline to solve the following systemic issues:

1. **Resource & Quota Exhaustion:** The 1:1 batching strategy makes it extremely difficult to contain or monitor resource usage by a single group, and submitting thousands of jobs actively triggers GCP API rate limits.
2. **Airflow Worker Starvation:** Blocking polling loops (`while True: sleep(60)`) hold Airflow worker slots hostage while waiting hours for GCP GPU allocation, leading to scheduler crashes and pipeline deadlocks.
3. **The "Idle GPU Tax" & Startup Costs:** Spinning up 1,000 independent Cloud Batch jobs means paying for 1,000 separate VM boots, container pulls, and weight downloads on expensive GPU instances. 
4. **Impossible Resumes:** Handling partial failures is currently a nightmare. If 20 batches out of 1,000 fail, tracking and resuming just those 20 requires highly manual, error-prone intervention.
5. **Log Fragmentation:** Debugging pipeline failures currently requires untangling 1,000 scattered, independent Cloud Batch job logs.
6. **Fragile Provenance:** The current YAML mutation pattern creates orphaned files and breaks relative links, destroying historical data tracking.

**The Goal:** Refactor this DAG into a massively scalable, asynchronous orchestrator that uses a "Run Context" pattern to isolate inputs, leverages Cloud Batch Array Jobs to farm out tasks efficiently, and provides immutable receipts to make resuming failed batches trivial.

*(Note: Data indexing and `kwcoco` generation are explicitly out of scope for this orchestration pipeline and will be handled entirely by downstream bespoke tools reading the pipeline outputs.)*

## Execution Plan & Dependency Graph

To avoid breaking active users and to ensure subsequent epics have the architecture they need, the work MUST be executed in this order:

* **Phase 1: Foundation (Epic 1)**
  * *Why:* We must standardize the API signature (manifests, outputs, automated ownership) and establish the "Run Context" directory first. All downstream scaling relies on having this clean, isolated folder in GCS.
* **Phase 2: Scale Compute (Epic 2)**
  * *Why:* Relies on the `input_manifest` created in Epic 1 to calculate the Array Job size (`task_count`).
* **Phase 3: Storage Optimization (Epic 4)**
  * *Why:* Relies on mounting the exact `run_context` folder generated in Epic 1.
* **Phase 4: Scheduler Optimization (Epic 3)**
  * *Why:* Can technically be done in parallel, but best tackled last to ensure the asynchronous polling plays nicely with the new Array Job states from Epic 2.

---

# Epic 1: Standardize API Signature & Implement "Run Context" Provenance (Breaking Change)

**Context & Problem:**
Currently, the DAG is built for a 1:1 mapping of Airflow runs to single-file inference jobs. The parameter signature forces users to manage their own output buckets and folders. Furthermore, handling partial failures is currently impossible without messy state tracking, and mutating files within Airflow breaks relative file paths and provenance. 

**Architectural Solution:**
We are moving to a "Zero-Config" output paradigm, a single massive "Envelope" manifest, and an immutable resume strategy. Airflow will no longer move individual payload files (which breaks relative links) but will instead rely on one master JSON manifest. We will also automate user identity extraction to keep the API perfectly lean.

*   **Task 1.1: Zero-Config Outputs & Immediate Run ID**
    *   **Description:** Deprecate `output_bucket` and `output_folder`. Hardcode the output destination to `gs://model-outputs/<run_id>/<batch_id>/`. 
    *   **Acceptance Criteria:** The DAG no longer accepts output destination parameters. Instead of generating a random UUID mid-run, the DAG uses Airflow's native `run_id` (e.g., `manual__2026-11-18...`) so the user immediately receives their output path in the REST API response upon submission.

*   **Task 1.2: Implement Envelope Manifest Support (Massive Manifest)**
    *   **Description:** Replace `input_file` with `input_manifest`. To prevent breaking relative file paths and to simplify GCS I/O, force users to embed their entire batch payload into a single massive JSON file using the Envelope pattern.
    *   **Acceptance Criteria:** DAG accepts an `input_manifest` parameter pointing to a single JSON file structured as: `[{"batch_id": "string", "inputs": { ...arbitrary payload... }}]`.

*   **Task 1.3: Create Run Context (Provenance)**
    *   **Description:** Instead of Airflow downloading, mutating, and re-uploading the pipeline YAML to a temp folder, Airflow will copy the user's config YAML and the *single massive manifest* to a unified `gs://airflow_logs/.../[run_id]/` folder.
    *   **Acceptance Criteria:** Remove `get_pipeline_config` mutation logic. DAG copies the manifest and config to the Run Context and passes that URI to the container. No individual payload files are moved, preserving all user links.

*   **Task 1.4: Automated Identity & Immutable Resume Receipt**
    *   **Description:** Extract the user's identity natively from the Airflow authentication context (no manual parameter). Drop a `receipt.json` into the root of the output path (`gs://model-outputs/<run_id>/receipt.json`).
    *   **Acceptance Criteria:** DAG automatically pulls the user email and fails the run if it does not end in `@noaa.gov`. The DAG writes a receipt that explicitly lists failed batches to empower users to manually submit a fresh, filtered "resume" run.
        ```json
        {
          "timestamp": "2026-11-18T10:10:00Z",
          "submitted_by": "alice.smith@noaa.gov",
          "airflow_run_id": "manual__2026-11-18T10:08:00+00:00",
          "batch_job_name": "ultralytics-8f3b2a1c",
          "run_context_path": "gs://airflow_logs/optics-pipeline/runs/manual__2026-11-18T10:08:00+00:00/",
          "failed_batches": ["survey_04", "survey_89"] 
        }
        ```

# Epic 2: Scale with Cloud Batch Array Jobs

**Context & Problem:**
The current pipeline requires one Airflow DAG run for every single video processed. If a user needs to process 1,000 videos, triggering 1,000 DAGs overwhelms the Airflow scheduler, exhausts Cloud Batch API quotas, and creates massive resource fragmentation.

**Architectural Solution:**
We will utilize Cloud Batch "Array Jobs." Airflow will read the length of the newly implemented `input_manifest` (from Epic 1) and submit a *single* Cloud Batch job configured with `task_count > 1`. Cloud Batch will dynamically provision resources and inject a `BATCH_TASK_INDEX` environment variable into each container replica.

*   **Task 2.1: Refactor TaskGroup for Array Jobs**
    *   **Description:** Update the `batch_v1.TaskGroup()` configuration to support multiple tasks (an Array Job) based on the size of the input manifest.
    *   **Acceptance Criteria:** The DAG correctly calculates the task count from the manifest and submits a single Batch Job with `task_count > 1`.

*   **Task 2.2: Implement Task Index Routing**
    *   **Description:** Ensure the container commands utilize the `BATCH_TASK_INDEX` environment variable (injected automatically by Cloud Batch) to select the correct item from the massive input manifest.
    *   **Acceptance Criteria:** The Cloud Batch payload is updated to instruct the container how to use `BATCH_TASK_INDEX` to parse the JSON array.

# Epic 3: Asynchronous Polling & Worker Optimization

**Context & Problem:**
Currently, the DAG uses a blocking `while True: time.sleep(60)` loop to monitor the Cloud Batch job. If a job is queued for hours waiting for GPU allocation, an Airflow worker slot is held hostage doing nothing. Eventually, the Airflow worker will hit its maximum execution timeout and fail the DAG.

**Architectural Solution:**
We must replace the blocking loop with an asynchronous pattern, allowing the Airflow worker to release its compute slot back to the pool while the job is in the GCP queue.

*   **Task 3.1: Replace Polling with Deferrable Operator/Sensor**
    *   **Description:** Replace the `wait_for_cloud_batch_job` custom polling loop with an asynchronous approach (e.g., a custom Sensor in `reschedule` mode or utilizing Airflow deferrable operators).
    *   **Acceptance Criteria:** Airflow workers are released while the Batch job is in `QUEUED` or `RUNNING` states.

*   **Task 3.2: Refactor State Cleanup**
    *   **Description:** Move any remaining cleanup/archiving logic into a proper Airflow callback or a downstream task that executes regardless of the asynchronous wait state.
    *   **Acceptance Criteria:** DAG properly closes out its state upon both success and failure of the Batch job without orphaned processes.

# Epic 4: Storage & Cost Optimization (GCS FUSE & Shared Disks)

**Context & Problem:**
Currently, the model wrapper script inside the Docker container uses the GCP Python SDK to download configuration files or videos locally before running. This heavily couples the logic to GCP (preventing local laptop testing) and wastes valuable GPU compute time on basic network I/O.

**Architectural Solution:**
We will utilize Google Cloud Storage FUSE to mount the Run Context (from Epic 1) and the `model-outputs` path directly into the container's file system (e.g., `/mnt/run_context/` and `/mnt/outputs/`). To the model wrapper, these look like local hard drives.

*   **Task 4.1: Mount Run Context via GCS FUSE**
    *   **Description:** Configure the Cloud Batch `AllocationPolicy` to natively mount the GCS output and input buckets into the container file system (e.g., `/mnt/run_context/` and `/mnt/outputs/`).
    *   **Acceptance Criteria:** Airflow DAG passes standard file paths instead of `gs://` URIs to the container, removing the need for GCS libraries in the wrapper scripts.

*   **Task 4.2: (Optional) Investigate Multi-Reader PD for Weights**
    *   **Description:** If model startup time is hindered by downloading weights, provision a Read-Only Multi-Attach Persistent Disk containing model weights and mount it via the Airflow Batch payload.
    *   **Acceptance Criteria:** Container accesses model weights from a local disk mount instantly, zeroing out download times.

