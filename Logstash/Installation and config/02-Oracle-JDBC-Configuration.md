# Oracle JDBC Pipeline Configuration

## Overview

This document describes the configuration of Logstash to collect Oracle Unified Audit Trail logs using the JDBC input plugin. It covers the installation of the Oracle JDBC driver, secure credential storage using the Logstash Keystore, pipeline configuration, and the deployment of multiple Oracle database pipelines.

---

# Architecture

```text
                    Oracle Database
                           │
                           │ JDBC Query
                           ▼
                     Logstash Pipeline
                           │
                    JSON File Output
                           ▼
              /var/log/logstash/oraclelogs/
                           │
                           ▼
                Azure Monitor Agent (AMA)
                           │
                           ▼
               Data Collection Rule (DCR)
                           │
                           ▼
                Log Analytics Workspace
                           │
                           ▼
                  Microsoft Sentinel
```

---

# Prerequisites

Ensure the following components are available before proceeding:

- Logstash installed successfully
- Oracle Database reachable over TCP 1521
- Oracle JDBC Driver (`ojdbc8.jar`)
- Oracle audit user with required privileges
- Logstash service account

---

# Step 1 - Install Oracle JDBC Driver

The JDBC input plugin requires the Oracle JDBC driver to communicate with the Oracle database.

Create a directory to store the JDBC driver.

```bash
sudo mkdir -p /etc/logstash/drivers
```

Copy the Oracle JDBC driver.

```bash
sudo cp ojdbc8.jar /etc/logstash/drivers/
```

Verify the file.

```bash
ls -l /etc/logstash/drivers/
```

Example Output

```text
ojdbc8.jar
```

Copy the driver into the Logstash library.

```bash
sudo cp ojdbc8.jar /usr/share/logstash/logstash-core/lib/jars/
```

Verify.

```bash
ls -l /usr/share/logstash/logstash-core/lib/jars/ojdbc8.jar
```

Assign appropriate permissions.

```bash
sudo chmod 644 /usr/share/logstash/logstash-core/lib/jars/ojdbc8.jar
```

> **Screenshot**
>
> `images/01-jdbc-driver.png`

---

# Step 2 - Verify Oracle Connectivity

Before configuring the pipeline, verify that the Oracle listener is reachable.

```bash
nc -zv <oracle_server> 1521
```

Expected Output

```text
Connection to <oracle_server> 1521 port [tcp/*] succeeded!
```

> **Screenshot**
>
> `images/02-oracle-connectivity.png`

---

# Step 3 - Configure Logstash Keystore

To avoid storing credentials in plain text, Logstash provides a secure keystore.

Temporarily disable shell history.

```bash
set +o history
```

Export the keystore password.

```bash
export LOGSTASH_KEYSTORE_PASS=<password>
```

Re-enable shell history.

```bash
set -o history
```

Add the Oracle username.

```bash
sudo -E /usr/share/logstash/bin/logstash-keystore \
--path.settings /etc/logstash \
add oracle_audit_user
```

Add the Oracle password.

```bash
sudo -E /usr/share/logstash/bin/logstash-keystore \
--path.settings /etc/logstash \
add oracle_audit_password
```

List the configured secrets.

```bash
sudo -E /usr/share/logstash/bin/logstash-keystore \
--path.settings /etc/logstash \
list
```

Example Output

```text
oracle_audit_user
oracle_audit_password
```

> **Screenshot**
>
> `images/03-keystore.png`

---

# Step 4 - Create the Oracle Pipeline Configuration

Create a pipeline configuration file under:

```text
/etc/logstash/conf.d/
```

Example

```text
oracle-db01.conf
```

The pipeline contains three major sections:

- JDBC Input
- Oracle Audit Query
- JSON File Output

Refer to:

```
oracle-jdbc-template.conf
```

for the complete configuration.

---

# JDBC Configuration Explained

| Parameter | Description |
|------------|-------------|
| `jdbc_driver_library` | Oracle JDBC Driver location |
| `jdbc_connection_string` | Oracle connection string |
| `jdbc_user` | Username retrieved from Logstash Keystore |
| `jdbc_password` | Password retrieved from Logstash Keystore |
| `statement` | SQL query executed during each polling cycle |
| `tracking_column` | Stores the last processed timestamp |
| `tracking_column_type` | Timestamp tracking method |
| `schedule` | Polling interval (Cron format) |
| `last_run_metadata_path` | Stores last successful execution timestamp |
| `record_last_run` | Enables incremental polling |
| `clean_run` | Prevents rereading historical records |
| `add_field` | Adds custom metadata to each event |

---

# Oracle Audit Query

The JDBC plugin executes the configured SQL query every polling interval.

The query retrieves only newly generated audit records.

Example logic:

```sql
WHERE event_timestamp > :sql_last_value
AND event_timestamp <= SYSTIMESTAMP - INTERVAL '30' SECOND
```

The 30-second delay prevents partially committed audit records from being collected.

The `:sql_last_value` variable is automatically maintained by Logstash using the last successful execution timestamp.

---

# Step 5 - Configure Multiple Pipelines

When collecting logs from multiple Oracle databases, configure a dedicated pipeline for each database.

Edit:

```text
/etc/logstash/pipelines.yml
```

Example

```yaml
- pipeline.id: oracle-db01
  path.config: "/etc/logstash/conf.d/oracle-db01.conf"
  pipeline.workers: 1

- pipeline.id: oracle-db02
  path.config: "/etc/logstash/conf.d/oracle-db02.conf"
  pipeline.workers: 1
```

Using separate pipelines ensures that each Oracle database is processed independently without impacting the others.

---

# Step 6 - Configure Output Location

Each Oracle pipeline writes events to a dedicated JSON file.

Example Output Path

```text
/var/log/logstash/oraclelogs/
```

Example

```text
oracle-db01/
└── oracle-db01_YYYY.MM.DD_audit.json
```

Separating output files by database simplifies monitoring and downstream ingestion by Azure Monitor Agent.

---

# Step 7 - Configure Last Run Metadata

Each pipeline maintains an independent tracking file.

Example

```text
/var/lib/logstash/logstash_last_run/
```

Example

```text
oracle-db01_last_run
oracle-db02_last_run
```

These files store the timestamp of the last successfully processed Oracle audit record and prevent duplicate ingestion during subsequent polling cycles.

---

# Recommended Directory Structure

```text
/etc/logstash/
├── pipelines.yml
├── conf.d/
│   ├── oracle-db01.conf
│   ├── oracle-db02.conf
│
└── drivers/
    └── ojdbc8.jar

/var/log/logstash/
└── oraclelogs/
    ├── oracle-db01/
    └── oracle-db02/

/var/lib/logstash/
└── logstash_last_run/
    ├── oracle-db01_last_run
    └── oracle-db02_last_run
```

---

# Best Practices

- Use Logstash Keystore for all database credentials.
- Maintain a separate pipeline for each Oracle database.
- Configure a unique `last_run_metadata_path` for every pipeline.
- Store output files in separate directories for each Oracle instance.
- Use meaningful pipeline IDs that match the database being monitored.
- Test the configuration before starting the Logstash service.

---

# Next Steps

After completing the Oracle JDBC pipeline configuration, proceed to:

**03-Validation-and-Troubleshooting.md**

This document covers:

- Configuration validation
- Service management
- Pipeline verification
- Log generation validation
- Common troubleshooting scenarios
