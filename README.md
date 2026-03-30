## Barman Training and Runbooks (2026 Edition)

Welcome to the central repository for **Barman (Backup and Recovery Manager)** training and operational procedures. This resource is designed for Database Administrators, Systems and Support Engineers to master the deployment, management, and disaster recovery workflows associated with PostgreSQL environments.

---

### 📖 Overview

This repository serves as a "Source of Truth" for Barman best practices as of 2026. Whether you are onboarding as a new DBA or looking for specific recovery procedures during an incident, the organized branches below contain the necessary documentation.



### 📂 Repository Structure

To keep the workspace clean and focused, the content is split into two primary branches:

* **`Runbooks`**: Contains technical, step-by-step guides for live environments.
    * *Highlights:* Installation scripts, retention policy configurations, and point-in-time recovery (PITR) workflows.
* **`Slide-Shows`**: Contains visual training materials and presentations.
    * *Highlights:* High-level architecture diagrams, business continuity theory, and "New for 2026" feature updates.

---

### 🚀 Getting Started

1.  **Clone the Repository:**
    ```bash
    git clone https://github.com/[your-organization]/Barman-Training-and-Runbooks-2026.git
    ```
2.  **Switch to your desired branch:**
    * For technical execution: `git checkout Runbooks`
    * For educational material: `git checkout Slide-Shows`
3.  **Review the `README.md`** within each specific branch for localized instructions.

---

### 🛠 Key Topics Covered

* **Backups** Configuring Barman to take backups using multiple `WAL` streaming methods.
* **Cloud Integration:** Backing up to S3, Azure Blob, and Google Cloud Storage.
* **Disaster Recovery:** Configuring and performing `PITR`
* **TPAexec Integration:** Configuring Barman with TPAexec

---

### 📬 Support & Contributions

If you encounter bugs in the runbooks or have suggestions for the training slides, please open a **Pull Request** or an **Issue**.

For direct inquiries or specific organizational access, please contact:
**Prince Nwando** 📧 [prince.nwando@enterprisedb.com](mailto:prince.nwando@enterprisedb.com)

---
