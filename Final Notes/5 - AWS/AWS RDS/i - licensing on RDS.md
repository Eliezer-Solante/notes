![[Pasted image 20260807110337.png]]

Licensing on RDS splits cleanly by engine — open-source engines are free (you just pay AWS infrastructure costs), while the commercial engines (Oracle, SQL Server) carry actual software license costs on top.

## Open-source engines (no license fee)

**PostgreSQL, MySQL, MariaDB** — all use their respective open-source licenses (PostgreSQL License, GPL for MySQL/MariaDB). There's no software license cost at all. You only pay AWS for compute, storage, I/O, and data transfer. This is a big part of why they're the default choice unless something specifically requires a commercial engine.

## Oracle — two models

|Model|How it works|
|---|---|
|**License Included (LI)**|Amazon RDS for Oracle offers License Included and Bring Your Own License. In the License Included model, you don't need to purchase Oracle Database licenses separately — AWS holds the license for the Oracle database software. This model is only supported on Oracle Database Standard Edition 2 (SE2) — not available for Enterprise Edition.|
|**Bring Your Own License (BYOL)**|You use your existing Oracle Database licenses to deploy databases on Amazon RDS — you pay AWS only for infrastructure. This results in lower RDS cost since the license cost isn't included, and BYOL is the only supported option if you want Oracle Enterprise Edition on RDS. If you use BYOL with Multi-AZ, you need a license for both the primary and standby instance.|

A useful nuance: the RDS instance-hour cost is driven by the licensing model, edition, and instance class together, and RDS BYOL avoids the 2:1 vCPU licensing penalty that Oracle applies to BYOL on plain EC2 — so BYOL on RDS is generally cheaper than BYOL on self-managed EC2 if you're counting vCPUs against your license.

## SQL Server — two models (recently changed)

|Model|How it works|
|---|---|
|**License Included (LI)**|You don't need to purchase a separate SQL Server license — Amazon RDS pricing includes the SQL Server software license, hardware resources, and RDS management capabilities.|
|**Bring Your Own Media (BYOM)**|If you prefer to use your own SQL Server licenses, RDS supports BYOM — RDS doesn't charge SQL Server license fees, and instead you use your existing licenses with License Mobility through Software Assurance. You pay for AWS infrastructure, Windows OS license fees, and any additional RDS features you enable.|

This BYOM option is fairly new — it lets customers use existing SQL Server Enterprise or Standard Edition licenses to cover both installation media and licensing on the managed service with no additional fees, whereas previously you had to pay for licensing a second time through the License Included model if you already owned SQL Server licenses.

Also worth knowing for dev/test: Express Edition is a free edition suitable for many development, testing, and non-production workloads, available with License Included; Developer Edition includes all SQL Server Enterprise Edition features but is licensed for non-production use only, available with BYOM.

## Quick reference

|Engine|License cost model|Notes|
|---|---|---|
|PostgreSQL|Free|Open source, no license fee|
|MySQL|Free|Open source, no license fee|
|MariaDB|Free|Open source, no license fee|
|Oracle|LI (SE2 only) or BYOL (SE2/EE)|EE requires BYOL|
|SQL Server|LI or BYOM|BYOM avoids double-paying if you already own licenses|

**Practical takeaway:** if you already own Oracle or SQL Server licenses (with active support), BYOL/BYOM is almost always cheaper on steady-state workloads — organizations that already own Oracle Database licenses have cut RDS Oracle spend by 40–60% with BYOL versus License Included on steady-state workloads, per one licensing advisory firm's benchmarks. License Included makes more sense for short-lived, spiky, or dev/test workloads where you don't want to manage license compliance yourself.

Want me to walk through the AWS CLI flags for setting `--license-model` when creating an Oracle or SQL Server instance?