# IAM Architecture & Role Delegation Map

## 1. Core Principles

* **GitOps for Admins:** Highly privileged "Data Manager" roles are managed via a plain-text YAML file. This provides an audit trail and prevents accidental "super user" sprawl.

* **Delegated Access for Users:** Data Managers manage end-user (Scientist) access directly in the GCP console (ad hoc), but are physically restricted by GCP IAM Conditions from granting destructive roles.

* **Least Privilege:** Write access is strictly limited. "Official" data is immutable to users, and ML outputs are written exclusively by service accounts.

## 2. The Actors

### 1. Platform Administrators (You / Infrastructure Team)

* **Role:** Manages the Terraform infrastructure and the `data_managers.yaml` file.

* **Scope:** Owns the baseline security, bucket creation, and lifecycle rules. Assigns Data Managers.

### 2. Data Managers (Localized Admins)

* **Role:** A designated steward for a specific Science Center (e.g., SEFSC, AFSC).

* **Scope:** Reviews incoming data, executes data moves (Landing -> Official), and manages ad hoc access for their center's Scientists.

* **Provisioning:** Granted via the `data_managers.yaml` file parsed by Terraform.

### 3. Scientists (End Users)

* **Role:** Researchers uploading raw data, querying official data, and submitting ML jobs.

* **Scope:** Restricted, intent-based access. Upload-only in Landing Zones, Read-only in Official buckets, Full-control in personal temporary sandboxes.

* **Provisioning:** Granted ad hoc in the GCP Console by their localized Data Manager.

### 4. Airflow Service Account (ML Compute)

* **Role:** The automated identity running the ML models.

* **Scope:** Reads configs/data, writes snapshots and outputs.

* **Provisioning:** Hardcoded via Terraform.

## 3. Delegation Strategy & The YAML Roster

To assign a new Data Manager, their email is added to a `data_managers.yaml` file stored with the Terraform code.

```yaml
# data_managers.yaml
science_centers:
  afsc:
    data_managers:
      - jane.doe@noaa.gov
  sefsc:
    data_managers:
      - john.smith@noaa.gov
```

* **The Workflow:** A PR is submitted to update this file. A Platform Admin reviews it. Once approved, `terraform apply` is run (either manually by an admin or automatically via CI/CD pipelines in the future). Terraform reads this file and applies the complex IAM conditions to these specific users.

## 4. Bucket-by-Bucket Role Matrix

### A. The Landing Zones (`gs://landing-zone-{center}`)

* **Lifecycle:** Temporary (e.g., 90-day auto-delete).

* **Data Managers:**

  * `Storage Object Admin`: Allows them to read, download, and delete files (after a successful move).

  * `Storage Admin` *(Constrained)*: Allows them to grant roles to users, strictly limited via IAM Conditions to **only** assigning `roles/storage.objectCreator` (upload-only).

* **Scientists:**

  * `Storage Object Creator`: Can upload files, but cannot overwrite, read, or delete existing files.

### B. The Official Storage (`gs://official-{center}`)

* **Lifecycle:** Permanent (Governed). Versioning enabled.

* **Data Managers:**

  * `Storage Object Admin`: Allows them to write data into this bucket (the merge/promotion process).

  * `Storage Admin` *(Constrained)*: Allows them to grant roles, strictly limited via IAM Conditions to **only** assigning `roles/storage.objectViewer` (read-only).

* **Scientists:**

  * `Storage Object Viewer`: Can read and download data for analysis. Cannot upload or modify.

### C. The ML Sandboxes / R&D Track (`gs://ml-sandbox`)

* **Lifecycle:** Ephemeral (e.g., 30-day auto-delete).

* **Scientists:**

  * `Storage Object Admin` *(Constrained)*: Full control, but restricted via **GCP Managed Folders** (or name prefix conditions) to only their specific `/{email}/` directory within the shared bucket.

* **Airflow Service Account:**

  * `Storage Object Viewer`: Can read the configs and manifests when a user submits an ML job.

### D. The ML Outputs (`gs://ml-outputs`)

* **Lifecycle:** Permanent (Immutable run provenance).

* **Airflow Service Account:**

  * `Storage Object Admin`: The *only* entity capable of writing the `inputs_snapshot`, the `results`, and the `receipt.json` into the `{UUID}` directory.

* **Scientists:**

  * `Storage Object Viewer`: Read-only access to view their results. This access is granted ad hoc by the Data Manager, similar to Official storage access.

* **Data Managers:**

  * `Storage Admin` *(Constrained)*: Allowed to grant `Storage Object Viewer` to users who need to retrieve ML results.

