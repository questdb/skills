# QuestDB Enterprise

Skip this for QuestDB Open Source. Enterprise adds authentication and RBAC,
replication with in-place failover, multi-host client failover, storage
policies and cold storage, resource groups, audited views, OIDC, and exchange
calendars in TICK.

Docs: `https://questdb.com/docs/security/rbac.md`,
`.../security/oidc.md`, `.../high-availability/overview.md`,
`.../high-availability/failover.md`, `.../concepts/storage-policy.md`,
`.../concepts/resource-groups.md`, `.../query/sql/acl/grant.md`.

## Principals and Grants

```sql
CREATE USER analyst WITH PASSWORD secret;    -- humans, Web Console, BI tools
CREATE SERVICE ACCOUNT ingestor;             -- applications and pipelines
CREATE GROUP traders;
ADD USER analyst TO traders;

-- Endpoint permissions (required to connect)
GRANT HTTP, PGWIRE TO analyst;   -- HTTP covers REST, Web Console, QWP clients and ILP/HTTP
GRANT HTTP TO ingestor;          -- QWP ingestion and queries
GRANT ILP TO legacy_feed;        -- only for ILP over TCP (9009)

-- Database permissions
GRANT CREATE TABLE TO ingestor;              -- creator gets all permissions on its tables
GRANT INSERT ON trades TO other_writer;
GRANT SELECT ON trades TO analyst;
GRANT SELECT ON trades(* EXCLUDE (counterparty)) TO analyst;   -- column-level
GRANT CREATE MATERIALIZED VIEW TO analyst;
```

- Principals only see objects they hold some permission on (4.1+).
  `active_permissions()` and `active_grants()` list effective permissions.
- Use a service account per application; never hand out the built-in admin.
- Per-principal memory limits: `ALTER USER u SET MEMORY LIMIT 512M`. Workload
  isolation: `CREATE RESOURCE GROUP`, `ALTER USER u SET RESOURCE GROUP g`.

## Tokens

```sql
-- Bearer token for QWP clients and the REST API
ALTER SERVICE ACCOUNT ingestor CREATE TOKEN TYPE REST WITH TTL '3650d';
-- add REFRESH to extend expiry on each login (primary node only)

-- JWK key, only for legacy ILP over TCP
ALTER SERVICE ACCOUNT legacy_feed CREATE TOKEN TYPE JWK;
-- returns public_key_x, public_key_y and private_key (not stored: capture it)
```

TTL units: `s`, `m`, `h`, `d` (1 minute to 10 years). OIDC (Entra ID, Okta,
...) can issue tokens instead; the Java client renews them with
`token_provider=azure`.

## Connecting

```python
import os, questdb

# QWP: token auth, TLS, failover across nodes, disk-backed store-and-forward
conf = ("wss::addr=db-a.example.com:9000,db-b.example.com:9000;"
        f"token={os.environ['QDB_TOKEN']};"
        "sf_dir=/var/spool/questdb;sender_id=ingest-01;")
with questdb.connect(conf) as db:
    ...
```

- Read tokens from the environment or a secret store, never from source.
- `tls_verify=unsafe_off` only for self-signed certificates in development,
  and only after asking the user. Prefer `tls_roots=/path/ca.pem` or
  `tls_ca=os_roots`.
- REST: `curl -H "Authorization: Bearer $QDB_TOKEN" -G --data-urlencode "query=..." https://host:9000/api/v1/sql/execute`.
- PGWire: `user=... password=... host=... port=8812 dbname=qdb`.
- Legacy ILP: `https::addr=host:9000;token=<REST token>;` or
  `tcps::addr=host:9009;username=<name>;token=<private_key>;token_x=<x>;token_y=<y>;protocol_version=2;`.
  ILP over TCP does not report errors back to the client.
- Open-source `http.user` / `http.password` settings make Enterprise fail to
  start; remove them.

## High Availability

- Replication: a primary uploads WAL to an object store; replicas replay it.
  `SELECT node_role()` reports the role.
- In-place failover (4.0+): `SWITCH ROLE TO PRIMARY` / `TO REPLICA` with the
  `SWITCH ROLE` permission, or `POST /lifecycle/switch` on port 9003.
- Clients fail over when given several `addr` hosts. Tables behind failover
  **must** have DEDUP keys, because replay after a switch can resend rows.
- `request_durable_ack=local|replicated` makes acks wait for local or
  replicated durability (Java, C, C++, Rust clients).
- Point-in-time recovery and backups: `.../operations/backup.md`,
  `.../operations/point-in-time-recovery.md`.

## Storage Tiers

Use `STORAGE POLICY` on Enterprise tables (TTL becomes a policy and
`ALTER TABLE SET TTL` is rejected). Materialized views still use TTL.
Details in `views-and-storage.md`.

```sql
ALTER TABLE trades SET STORAGE POLICY(
  TO PARQUET 7 DAYS, TO REMOTE 14 DAYS, DROP LOCAL 30 DAYS, DROP REMOTE 7 YEARS);
```

`TO REMOTE` / `DROP REMOTE` need cold storage (`cold.storage.enabled=true`)
on every instance and a WAL table.

## Other Enterprise Features

- **Exchange calendars in TICK:** `WHERE ts IN '[2025-01]#XNYS'`.
- **Audited views:** `CREATE VIEW v AS (...) WITH AUDIT`; reads land in `sys.view_audit`.
- **Kubernetes Operator:** `.../enterprise-kubernetes-operator/` docs.
- **Databento connector:** streams market data straight into QuestDB
  (`.../connect/connectors/databento.md`).
