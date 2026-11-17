# Data Management Plan & Issue Hierarchy

**Project Context:**
The Optics Strategic Initiative with NOAA NMFS is transitioning dozens of independent survey teams from localized, ad hoc optical data storage (hard drives, Blu-ray, local NAS) into a centralized Google Cloud Platform (GCP) environment. 
*   **Scale:** ~1PB per year of active survey data, transferred in batches up to 100TB.
*   **Incentive:** Access to Airflow-based ML workflows and GPU resources.
*   **Core Governance Principle:** Every object in the cloud must either be explicitly owned (verifiable PoC/InPort ID) or slated for imminent deletion (ephemeral workspaces).
*   **Constraint:** Limited engineering resources mean we must adopt pragmatic, lean solutions (like using standard ISO 19115 InPort metadata "as-is" and deferring cold-storage automation).

---

## Epic 1: Establish Landing Zones & Ingestion Procedures
**Context & Strategic Logic:** 
Survey teams need a place to drop massive amounts of data before it is officially reviewed. We are creating "Landing Zones"—temporary buckets assigned at the science-center level. 
*   *Why separate Landing Zones?* It acts as a quarantine. Unverified, messy data cannot be allowed to pollute the "Official" governed storage. 
*   *Why Terraform with a YAML roster?* Data Managers (admins) hold "keys to the kingdom." If we grant their access ad hoc, a typo in the GCP UI could accidentally grant someone the ability to delete the entire storage array. By driving Data Manager assignments through a `data_managers.yaml` file parsed by Terraform, we enforce a Pull Request audit trail and ensure complex IAM Conditions are applied flawlessly every time.
*   *Why ad hoc for end-users?* We want to delegate the day-to-day access requests (adding a new scientist to a bucket) to the non-technical Data Managers using the native GCP console, bypassing the need for CI/CD for routine tasks. 

### Story 1.1: Provision Landing Zone Infrastructure & Delegated Identity via Terraform
*   **Context/Background:** We need to provision the temporary buckets and set up the specific IAM permissions that allow Data Managers to manage their own users securely.
*   **Dependencies:** None.
*   **Description:** 
    1. Write Terraform modules to provision GCP Cloud Storage buckets for each Science Center (e.g., `gs://landing-zone-afsc`).
    2. Implement non-authoritative IAM bindings (`google_storage_bucket_iam_member`) so Terraform ignores humans manually added in the GCP UI.
    3. **Crucial Guardrail:** Grant Data Managers `Storage Admin` but apply a strict **IAM Condition** restricting them to *only* assigning the `roles/storage.objectCreator` role to end-users. This prevents them from accidentally giving a scientist permission to delete the bucket.
*   **Acceptance Criteria:**
    *   [ ] Landing Zone buckets provisioned via Terraform.
    *   [ ] IAM Conditions successfully limit Data Managers to assigning upload-only access.
    *   [ ] Delegation process documented in the Quarto Data Management Portal.

#### Task 1.1.1: Implement Imminent Deletion Lifecycle Policy
*   **Context/Background:** Landing Zones cannot become permanent dumping grounds. This violates our "owned or imminent deletion" rule.
*   **Description:** Update the Terraform module to include a GCP Object Lifecycle Management rule. Any object residing in a Landing Zone bucket longer than 90 days must be automatically deleted.
*   **Acceptance Criteria:**
    *   [ ] 90-day TTL (Time-To-Live) rule applied to all Landing Zone buckets.
    *   [ ] TTL policy clearly communicated in the Quarto Portal.

### Story 1.2: Develop User Data Ingestion Standard Operating Procedures (SOPs)
*   **Context/Background:** Users are moving 100TB+ datasets. Standard local uploads will fail. We need bulletproof documentation.
*   **Description:** Author documentation for moving data to the Landing Zones. Recommend that users package millions of tiny images into `.tar` or `.tfrecord` to save on GCP API operation costs.

#### Task 1.2.1: Write Storage Transfer Service (STS) Guide
*   **Description:** Create a user guide for initiating transfers using STS (the preferred, native, managed method), including prerequisites and step-by-step console instructions.

#### Task 1.2.2: Write `rclone` Fallback Guide
*   **Context/Logic:** For users without STS access, `rclone` is the fallback. However, running a 100TB transfer from a local workstation will inevitably fail when the local ISP drops.
*   **Description:** Write an SOP explicitly instructing users to provision a cheap cloud VM (e.g., Compute Engine) and run the `rclone` command within a detached `tmux` session.

---

## Epic 2: Develop Review & Promotion Workflow
**Context & Strategic Logic:** 
Once data is in the Landing Zone, a Data Manager reviews it and "promotes" it to the Official bucket. 
*   *Why strict naming rules?* InPort hierarchies often have spaces or special characters. If we allow these in GCP, it creates endless URL-encoding (`%20`) headaches for downstream automated ML pipelines. We enforce lowercase and underscores (`_`) on ingestion.
*   *Why not map folder names back to InPort?* We don't need to reverse-engineer the InPort hierarchy from the folder name because the XML metadata file travels *with* the data.
*   *Setting up Epic 4 (Archival):* We are enforcing that the XML file must live at the absolute root of the dataset folder. This predictability is mandatory for our future Archival Initiative (Epic 4) to scrape metadata programmatically.

### Story 2.1: Provision "Official" Storage Infrastructure & Security Guardrails
*   **Context/Background:** Official buckets are the permanent home for governed data. They require strict write-protection and read-only delegation.
*   **Description:** 
    1. Provision "Official" storage buckets via Terraform (e.g., `gs://official-afsc`).
    2. Implement IAM Conditions allowing Data Managers to grant users *only* the `roles/storage.objectViewer` (read-only) role. 
    3. Enable native **GCP Object Versioning**. *Logic: This is a cheap safety net. If an admin accidentally overwrites the main XML file during an update, we can simply click "restore".*
*   **Acceptance Criteria:**
    *   [ ] Official buckets provisioned with Object Versioning enabled.
    *   [ ] IAM Conditions limit Data Managers to assigning read-only access.

### Story 2.2: Establish Dataset Promotion and "Delta Update" SOP
*   **Context/Background:** Datasets are not static; users will need to add new ML configs or images to a 2025 survey in 2026. *Logic: We absolutely cannot force users to copy a 100TB dataset back to the Landing Zone just to add a 5KB config file. This wastes time and cloud costs.*
*   **Description:** Draft the SOP for the "Delta Update Process":
    1. Users upload *only* the new files to the Landing Zone.
    2. User submits a Google Form flagged as an "Update".
    3. The Data Manager executes a merge using STS or `rclone copy` (which natively supports moving only new files without overwriting the destination).
*   **Acceptance Criteria:**
    *   [ ] "Promotion & Delta Merge" SOP published in the Quarto Portal.

### Story 2.3: Define Metadata & Naming Convention Rules
*   **Description:** Create the official checklist Data Managers must verify before promoting data. 
*   **Acceptance Criteria:**
    *   [ ] Rule: Folder names lowercased, spaces/special characters replaced with `_` or `-`.
    *   [ ] Rule (Mandatory for Epic 4): XML metadata file must reside at the absolute root of the dataset.
    *   [ ] Rule: Valid PoC and InPort ID confirmed.

---

## Epic 3: Implement ML Workspace Architecture
**Context & Strategic Logic:** 
We must manage ML pipeline data while strictly adhering to the "owned or deleted" governance rule and the DRY (Don't Repeat Yourself) principle. We are utilizing a **Two-Track Storage Strategy** to balance strict survey governance with the messy reality of R&D.
*   *Track 1 (Official):* Immutable ML configs used for official survey counts live in the Official buckets right next to the survey data, inheriting the survey's PoC and XML metadata.
*   *Track 2 (Sandbox):* A temporary workspace for ablation studies and ad-hoc runs. We use a single bucket to avoid sprawl, but use GCP Managed Folders to restrict users to their own email namespace.
*   *The Output Snapshot:* To freeze provenance, the Airflow pipeline *always* copies the input config/manifest into the output UUID directory. *Logic: If a user deletes their messy sandbox config 30 days later, we still know exactly what ran because the snapshot lives forever alongside the results.*

### Story 3.1: Provision the ML Sandbox Environment (Track 2)
*   **Context/Background:** Scientists need a temporary scratchpad to test configurations.
*   **Description:** 
    1. Provision a single `gs://ml-sandbox` bucket.
    2. Configure **GCP Managed Folders** so users can only read/write to `gs://ml-sandbox/{user_email}/`.
    3. Implement a 30-day auto-delete Object Lifecycle rule to satisfy the imminent deletion governance.
*   **Acceptance Criteria:**
    *   [ ] Single sandbox bucket provisioned with a 30-day TTL.
    *   [ ] Managed Folders ensure users are isolated to their namespaces.

### Story 3.2: Provision ML Outputs & Run Provenance Infrastructure
*   **Context/Background:** ML pipeline outputs must be completely immutable to guarantee run reproducibility.
*   **Description:** 
    1. Provision `gs://ml-outputs`.
    2. Configure IAM so *only* the Airflow Service Account has write access.
    3. Configure IAM Conditions so Data Managers can grant users read-only access to view their results.
*   **Acceptance Criteria:**
    *   [ ] Output bucket provisioned; write access limited to Airflow SA.

#### Task 3.2.1: Define Output Directory Structure & Snapshot Logic
*   **Description:** Document the strict UUID-based folder structure for the Airflow engineering team. The pipeline must:
    1. Pull inputs (manifest/config).
    2. Save a copy to `gs://ml-outputs/{UUID}/inputs_snapshot/`.
    3. Write results to `gs://ml-outputs/{UUID}/results/`.
    4. Generate a structured `receipt.json` (timestamp, submitted_by, run_id).

### Story 3.3: Document the Two-Track ML Workflow for Scientists
*   **Description:** Draft user-facing documentation detailing when to use Track 1 (Official storage, updated via the Delta merge process) versus Track 2 (the 30-day Sandbox). Ensure users understand how to graduate a successful Sandbox config to Track 1.

---

## Epic 4: The Archival Initiative (Future / Backlog)
**Context & Strategic Logic:**
Currently, our primary focus is managing "Hot" storage and getting users onto the cloud. We are explicitly kicking the can down the road regarding Cold Storage (Coldline/Archive tiers) because we lack the engineering bandwidth to build a custom lifecycle management tool right now.

However, we are setting ourselves up for future success by enforcing the rules in **Epic 2**. 

*   *Why are we not tagging data right now?* Adding native GCP object labels (tags) manually or writing an ingestion script to do it is too heavy of a lift right now. Because the ratio of metadata to data is so low (roughly one XML file per TB of data/survey), we don't need to do this incrementally. 
*   *The Future Strategy:* When bandwidth allows, we will write a one-off script that sweeps across the 1PB+ of Official storage. Because we enforced (in Story 2.3) that the ISO 19115 XML file is *always* at the absolute root of the dataset folder, this script will easily locate the ~1,000 XML files. It will parse them, extract the `InPort ID` and the `PoC Email`, and apply those as native GCP Object/Bucket Labels. 
*   *The Result:* Once those native labels are applied, we can utilize out-of-the-box GCP Object Lifecycle Management to say: *"If this dataset hasn't been accessed in 2 years, move it to Coldline storage to save money."* Furthermore, if the original PoC departs NOAA, the InPort ID allows us to query the parent organization to find the new data steward before archiving or deleting the data.

