# NYP iCEMS v12 → v25.1.3 Production Upgrade Runbook
**PHP 8.4 + OCI8 / Elasticsearch 8.4 / WAF Cutover**

- **Date:** 8/21/2026
- **Window:** 19:15 – 23:00
- **Team (your portion):** Azrul / Indra / Prabhu (WAF step also with Yawar)
- **Status:** DRAFT v1 — built from NonProd validation; pending your cupcake1/cupcake2-specific findings and past successful run notes

---

## 1. Server Reference

| Role | Hostname | IP | Notes |
|---|---|---|---|
| FTS1 | Fruitcake | 100.121.200.110 | Elasticsearch host |
| CRM1 | Cupcake1 | 100.121.200.113 | App server |
| CRM2 | Cupcake2 | 100.121.200.158 | App server |

**Pre-upgrade baseline (NonProd, confirmed):**
- OS: RHEL 8.10 (Ootpa) on all three
- PHP: 8.0.30 (cli, built Aug 3 2023) on cupcake1/cupcake2
- Elasticsearch: 7.17.21-1 on fruitcake, running via systemd

> ⏳ *Add here: production baseline check output (`cat /etc/*release`, `php -v`, `rpm -qa | grep -i php`, `systemctl status elasticsearch`) taken on cupcake1/cupcake2/fruitcake before you start tonight.*

---

## 2. Pre-Checks (before 19:15)

- [ ] Confirm RIVA sync is OFF (owned by Yawar/Vincent, task ends 19:30) — no new email tickets should be created in the upgrade instance during the window
- [ ] Confirm UAT WAF rules (NYP-UAT-iCEMS-CCS/CRM/KMS/QMS-W...) are the finalized version to be promoted to PROD
- [ ] Confirm AWS access/credentials for snapshot step (EC2 console or CLI)
- [ ] Confirm Oracle Instant Client 11.2 is present at `/usr/lib/oracle/11.2/client64` on cupcake1 and cupcake2 (was already present in NonProd — verify same in PROD)
- [ ] Confirm `php-devel`, `php-pear`, `gcc`, `make` are NOT yet installed / current module stream status on PROD (`dnf module list php`)
- [ ] Have rollback/abort checkpoint time agreed with the team (see §7)

> ⏳ *Add here: any additional pre-checks from your past successful run.*

---

## 3. Step 6 — Snapshot (19:30 – 20:00)
**Owner:** Azrul / Indra / Prabhu

- [ ] Snapshot cupcake1 (app)
- [ ] Snapshot cupcake2 (app)
- [ ] Snapshot database
- [ ] Snapshot Elasticsearch (fruitcake)
- [ ] Snapshot CyDesk server(s)
- [ ] Verify each snapshot completes and shows "completed" status before proceeding to §4
- [ ] Record snapshot IDs below for rollback reference:

| Server | Snapshot ID | Completed at |
|---|---|---|
| Cupcake1 | | |
| Cupcake2 | | |
| Database | | |
| Elasticsearch | | |
| CyDesk | | |

---

## 4. Step 7 — Upgrade to SugarCRM iCEMS v25.1.3 (PHP 8.4 + OCI8) (20:00 – 21:00)
**Owner:** Azrul / Indra / Prabhu
**Scope:** cupcake1 and cupcake2

### 4.1 Fix the PHP module stream (validated on NonProd 23-24 Jul 2026)
The remi-8.4 stream was found **disabled** on the NonProd test host, which blocked `php-devel`/`php-pear` installation via modular filtering. Run this first:

```bash
sudo dnf module reset php
sudo dnf module enable php:remi-8.4 -y
php -v
rpm -qa | grep php-cli
```
Expect PHP to report **8.4.x** (NonProd test showed 8.4.15 initially).

### 4.2 Install build dependencies
```bash
sudo dnf install -y php-devel php-pear gcc make
```
On NonProd this also upgraded PHP from 8.4.15 → 8.4.23 as a side effect (remi repo pulled latest 8.4.x). **Confirm this is expected/acceptable for PROD**, or pin the version if not.

### 4.3 Confirm Oracle Instant Client is present
```bash
rpm -qa | grep -i oracle
ls -la /usr/lib/oracle/ 2>/dev/null
rpm -q libaio libnsl glibc
```
Expected (from NonProd): `oracle-instantclient11.2-basic`, `-devel`, `-sqlplus`, client libs under `/usr/lib/oracle/11.2/client64`.

### 4.4 Build and install OCI8 via PECL
```bash
pecl install oci8
```
When prompted for the ORACLE_HOME path, press **Enter** to auto-detect (successfully found `/usr/lib/oracle/11.2/client64` on NonProd).

### 4.5 Enable the OCI8 extension
```bash
echo "extension=oci8.so" | sudo tee /etc/php.d/40-oci8.ini
cat /etc/php.d/40-oci8.ini
```

### 4.6 Verify
```bash
php -m | grep -i oci8
php --ri oci8
```
Expected output includes:
```
OCI8 Support => enabled
OCI8 Version => 3.4.1
Oracle Run-time Client Library Version => 11.2.0.4.0
```

### 4.7 Restart services and confirm health
```bash
systemctl status php-fpm
systemctl status httpd
```

- [ ] Repeat 4.1 – 4.7 on **cupcake2**
- [ ] Confirm SugarCRM iCEMS application version reports v25.1.3 on both nodes
- [ ] Smoke test application login / core function on both nodes

> ⏳ *Add here: the actual SugarCRM v25.1.3 application upgrade steps (package/installer used, DB migration steps if any) — the NonProd session above only validated the PHP/OCI8 plumbing, not the SugarCRM version upgrade itself.*

---

## 5. Step 8 — Upgrade Elasticsearch to 8.4 (21:00 – 22:00)
**Owner:** Azrul / Indra / Prabhu
**Scope:** fruitcake

Baseline before upgrade: Elasticsearch 7.17.21-1, running via systemd, uptime observed ~2 weeks 6 days in NonProd check.

> ⚠️ **Known issue from NonProd testing:** running `/usr/share/elasticsearch/bin/elasticsearch --version` directly (outside systemd) failed with a JVM `RuntimeException: starting java failed` / `Native memory allocation (mmap) failed to map 8241807360 bytes... Not enough space`. This was a manual invocation issue (insufficient memory available to that ad-hoc JVM launch), not necessarily the running service — but worth checking available memory headroom on fruitcake before starting the 8.4 upgrade, since 7.x → 8.x is a major version jump.

Useful pre-checks:
```bash
rpm -qa | grep -i elasticsearch
systemctl status elasticsearch
curl -s -X GET "localhost:9200/_nodes?pretty" | grep -i version
```

> ⏳ *Add here: the actual 7.17 → 8.4 upgrade procedure (this is a major version jump — confirm whether it's a direct upgrade or requires an intermediate step, index compatibility checks, and reindex plan if needed). Not yet validated in the NonProd session provided.*

- [ ] Confirm cluster/indices health green before starting
- [ ] Perform upgrade
- [ ] Confirm `curl -s -X GET "localhost:9200/_nodes?pretty" | grep -i version` reports 8.4.x
- [ ] Confirm SugarCRM search functionality against the upgraded ES

---

## 6. Step 9 — Apply WAF Rule: UAT → Production (22:00 – 23:00)
**Owner:** Yawar / Azrul / Indra / Prabhu

Reference (confirmed from AWS console, NonProd doc):

| Environment | Web ACL | Rules |
|---|---|---|
| PROD | NYP-PRD-iCEMS-CCS-WA... | 7 rules |
| PROD | NYP-PRD-iCEMS-CRM-W... | 6 rules |
| PROD | NYP-PRD-iCEMS-KMS-W... | 12 rules |
| PROD | NYP-PRD-iCEMS-QMS-W... | 7 rules |
| UAT | NYP-UAT-iCEMS-CCS-WA... | 7 rules |
| UAT | NYP-UAT-iCEMS-CRM-W... | 7 rules |
| UAT | NYP-UAT-iCEMS-KMS-W... | 12 rules |
| UAT | NYP-UAT-iCEMS-QMS-W... | 5 rules |
| UAT | NYP-UAT-iCEMS-UATI2... | 5 rules |

- [ ] Confirm the exact rule diff to promote from UAT → PROD (rule counts above don't match 1:1 between CRM/QMS envs — verify which specific rule(s) are being ported vs. the whole ACL)
- [ ] Apply to PROD web ACL(s)
- [ ] Verify via AWS console that PROD rule count/content matches intended change
- [ ] Confirm no legitimate traffic is being blocked post-change (check WAF sampled requests / CloudWatch logs)

---

## 7. Rollback / Abort Criteria

> ⏳ *To fill in with the team:*
- [ ] Agreed "abort by" checkpoint time if §4 or §5 fails validation
- [ ] Rollback procedure using snapshots recorded in §3 (restore order: app → DB → ES → CyDesk?)
- [ ] Who makes the go/no-go call at each checkpoint
- [ ] Communication plan if rollback is triggered (who to notify, RIVA sync re-enable timing)

---

## 8. Post-Upgrade Verification

- [ ] SugarCRM iCEMS v25.1.3 confirmed on both cupcake1 and cupcake2
- [ ] PHP 8.4 + OCI8 confirmed via `php --ri oci8` on both nodes
- [ ] Elasticsearch 8.4 confirmed, search functioning in-app
- [ ] WAF rule live on PROD, no false positives observed
- [ ] RIVA sync re-enabled (coordinate with Yawar/Vincent)
- [ ] Application smoke test (login, ticket creation, search) passed on both nodes
- [ ] Stakeholders notified of completion

---

*This is v1, built from the NonProd OCI8 validation session (Prabhu Jeyakumar, Jul 23–24 2026) and the cutover schedule. Sections marked ⏳ need your cupcake-specific findings and past successful run details to complete.*
