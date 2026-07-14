# Oracle Audit Log Ingestion using Logstash

## Overview
This configuration demonstrates how to ingest Oracle Unified Audit Trail logs into Microsoft Sentinel using Logstash JDBC Input.

## Components
- Oracle Database
- Oracle JDBC Driver (ojdbc8.jar)
- Logstash JDBC Input Plugin
- File Output
- Azure Monitor Agent (AMA)
- Microsoft Sentinel

## Folder Structure
```
/etc/logstash/
├── pipelines.yml
└── conf.d/
    ├── oracle-db01.conf
    └── oracle-db02.conf

/var/log/logstash/
└── oraclelogs/
    └── <SERVER_NAME>/

/var/lib/logstash/logstash_last_run/
└── <SERVER_NAME>_last_run
```

## Features
- Incremental polling using `event_timestamp`
- Timestamp-based tracking
- Multiple pipelines
- Separate last_run file per database
- File output compatible with AMA Custom Logs

## Polling Schedule
Every 2 minutes

## Notes
- Replace placeholders before deployment.
- Store credentials securely using Logstash environment variables or Logstash Keystore.
