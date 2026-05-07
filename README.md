
# VirtualBox Ate My Homework?
## Debugging Wazuh + Filebeat Ingestion Failures on Linux Hosts

This repository documents a real-world troubleshooting session involving:

- Wazuh OVA
- Filebeat
- VirtualBox shared folders
- Linux host systems
- Elasticsearch/OpenSearch ingestion
- Broken Coursera lab assumptions

The original lab expected Windows-style shared folder behavior and did not account for Linux host permission handling, VirtualBox mount behavior, or Filebeat instability under these conditions.

Instead of switching immediately to a Windows partition, the goal was to determine:

> Was the configuration actually wrong, or was the environment itself incompatible with the assumptions made by the course?

---

# Core Problem

Filebeat appeared to:

- detect files,
- harvest logs,
- generate activity,
- and partially increase indexed document counts,

while simultaneously:

- crashing repeatedly,
- failing to ingest the expected training dataset,
- producing misleading telemetry,
- and behaving inconsistently depending on mount location and permissions.

The issue became significantly more complex because:

- VirtualBox shared folders (`/media/sf_*`) behaved differently than native Linux paths,
- Filebeat sometimes crashed silently,
- Logstash assumptions in the course materials did not match the Wazuh OVA architecture,
- and Linux-specific filesystem behaviors were never discussed in the lab instructions.

---

# What Was Investigated

## Shared Folder Mounts

Initial ingestion paths used:

```yaml
/media/sf_buttercup-shared/
```

Initial file discovery checks:

```bash
find /media/sf_buttercup-shared -type f | head -50
ls -lah /media/sf_buttercup-shared
ls /media/sf_buttercup-shared/mailsv/*.log
```

Observed issues included:

- permission inconsistencies,
- mount instability,
- incomplete harvesting behavior,
- and Filebeat runtime crashes.

Shared-folder related kernel and VirtualBox messages were also inspected using:

```bash
sudo dmesg | tail -50
sudo journalctl -xe | tail -50
```

---

## Local Filesystem Migration

Data was later copied into:

```bash
/opt/buttercup-data/
```

to eliminate VirtualBox shared-folder behavior as a variable.

Commands used:

```bash
sudo mkdir -p /opt/buttercup-data
sudo cp -r /media/sf_buttercup-shared/* /opt/buttercup-data/
sudo find /opt/buttercup-data -type f
```

This partially improved ingestion consistency.

File existence and readability were validated:

```bash
file /opt/buttercup-data/www1/access.log
head -5 /opt/buttercup-data/www1/access.log
```

---

## Filebeat Output Debugging

The original tutorial used:

```yaml
output.logstash:
  hosts: ["localhost:5044"]
```

However:

- Logstash was not installed,
- port 5044 was not listening,
- and the Wazuh OVA actually used Elasticsearch/OpenSearch directly.

Connectivity testing showed:

```bash
sudo /usr/share/filebeat/bin/filebeat test output -c /home/wazuh-user/ingest.yml
```

returned:

```text
dial tcp 127.0.0.1:5044: connect: connection refused
```

Further validation confirmed:

```bash
sudo systemctl status logstash
sudo ss -lntp | grep 5044
```

that Logstash was absent.

Output was later changed to:

```yaml
output.elasticsearch:
  hosts: ["https://127.0.0.1:9200"]
  protocol: https
  username: admin
  password: admin

  ssl.certificate_authorities:
    - /etc/filebeat/certs/root-ca.pem

  ssl.certificate: "/etc/filebeat/certs/wazuh-server.pem"
  ssl.key: "/etc/filebeat/certs/wazuh-server-key.pem"
```

The final ingest configuration was validated repeatedly with:

```bash
sudo cat /home/wazuh-user/ingest.yml
sudo grep mail /home/wazuh-user/ingest.yml
```

---

## Filebeat Runtime Testing

Configuration validation:

```bash
sudo /usr/share/filebeat/bin/filebeat test config -c /home/wazuh-user/ingest.yml
```

Direct execution testing:

```bash
sudo /usr/share/filebeat/bin/filebeat \
-c /home/wazuh-user/ingest.yml \
-e \
--path.home /usr/share/filebeat \
--path.data /var/lib/filebeat-buttercup \
--path.logs /var/log/filebeat
```

Runtime logs were inspected for:

- harvester behavior,
- registry state,
- monitoring counters,
- panic traces,
- and publisher failures.

Multiple Filebeat crashes produced Golang goroutine stack traces rather than conventional application errors.

---

## Elasticsearch Validation

Indices were verified using:

```bash
sudo curl -k -u admin:admin 'https://127.0.0.1:9200/_cat/indices?v'
```

This confirmed:

- Wazuh indices existed,
- document counts increased partially,
- but expected dataset ingestion totals were never fully achieved.

The primary indices observed:

```text
wazuh-alerts-4.x-2026.05.07
wazuh-statistics-2026.19w
```

Document counts increased incrementally during repeated ingestion attempts.

---

# Major Findings

## The Lab Instructions Are Environment-Fragile

The lab assumes:

- Windows host behavior,
- stable VirtualBox shared-folder semantics,
- and infrastructure components that may not exist in the provided OVA.

These assumptions break down on Linux hosts.

---

## Filebeat Can Appear Healthy While Failing

Filebeat frequently showed:

- active monitoring,
- harvester activity,
- and increasing counters,

while simultaneously:

- failing to ingest expected data,
- crashing internally,
- or stalling on filesystem interactions.

This produced misleading debugging signals.

---

## VirtualBox Shared Folders Are a Significant Variable

The strongest suspected root issue is Filebeat instability or incompatibility with VirtualBox shared mounts on Linux hosts.

Moving data into native VM storage improved behavior substantially.

---

# Repository Contents

| Directory | Purpose |
|-----------|---------|
| `assets/` | Screenshots from debugging session |
| `configs/` | Filebeat configuration examples |
| `logs/` | Captured debug output |
| `notes/` | Timeline and troubleshooting observations |

---

# Why This Repository Exists

This is not simply a "lab completion."

It is documentation of:

- infrastructure debugging,
- environment validation,
- Linux virtualization edge cases,
- and the difference between a workaround and an actual fix.

The goal is to help future Linux users avoid losing hours to undocumented assumptions inside the course environment.

---

# Current Status

- ✅ Partial ingestion achieved
- ✅ Elasticsearch connectivity confirmed
- ✅ Native VM storage improved stability
- ⚠️ Shared-folder behavior remains suspect
- ❌ Full ingestion parity with expected lab results remains unresolved

---

# Future Work

- Reproduce on VMware
- Reproduce on bare-metal Linux
- Compare against Windows host behavior
- Test newer Filebeat versions
- Determine whether VirtualBox guest additions contribute to instability
- Produce a Linux-first rewrite of the lab instructions

---

# Author

**Mar Carter**  
GitHub: https://github.com/Mousie-mouse
