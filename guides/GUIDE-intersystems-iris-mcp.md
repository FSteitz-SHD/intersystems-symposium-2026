Table of Contents
---
- [InterSystems IRIS MCP Integration](#intersystems-iris-mcp-integration)
  - [Purpose and scope](#purpose-and-scope)
  - [Related documents](#related-documents)
  - [Architecture at a glance](#architecture-at-a-glance)
  - [Naming and conventions (official)](#naming-and-conventions-official)
  - [Credentials and security](#credentials-and-security)
  - [Part A: local setup via Docker (available now)](#part-a-local-setup-via-docker-available-now)
  - [Debugging a database locally (external IRIS.DAT)](#debugging-a-database-locally-external-irisdat)
  - [Part B: configuring an existing database (dev instance)](#part-b-configuring-an-existing-database-dev-instance)
  - [Part C: client wiring (Claude Code and Codex)](#part-c-client-wiring-claude-code-and-codex)
  - [Interim path: client-side bridge (works today on 2024.1.x)](#interim-path-client-side-bridge-works-today-on-20241x)
  - [ERP example ToolSet](#erp-example-toolset)
  - [Analysis: performance and developer support](#analysis-performance-and-developer-support)
  - [Reference: IRIS system data that is SQL-queryable](#reference-iris-system-data-that-is-sql-queryable)
  - [Reference: the ToolSet Query element](#reference-the-toolset-query-element)
  - [Pitfalls](#pitfalls)
  - [Status and roadmap](#status-and-roadmap)
  - [Appendix A: probe helper](#appendix-a-probe-helper)
  - [Appendix B: command cheat sheet](#appendix-b-command-cheat-sheet)
  - [Appendix C: local stub schema for the examples](#appendix-c-local-stub-schema-for-the-examples)

<!-- Project guide, neutralized: host names, accounts, internal module paths and entity names removed. The ObjectScript package and the IRIS namespace are renamed to `ERP`. Business table and column names are replaced by English stand-ins (`SalesContract`, `PurchaseOrder`, …); the example outputs, plan costs and statistics come from runs against the original names. Customer-specific filters are removed from the example query. -->

# InterSystems IRIS MCP Integration

## Purpose and scope

This document is the reliable, repeatable reference for setting up **IRIS MCP**: exposing IRIS SQL and system data as MCP tools. It is usable now for evaluation and is the standard we build on as InterSystems ships more MCP capability.

It covers:

- **Two IRIS targets.** A local Docker instance (AI Hub community EAP) that works today for evaluation, and the licensed dev database, which becomes the real target once it is upgraded to a 2026.x build.
- **Multiple namespaces.** One MCP endpoint per IRIS namespace. `ERP` is the namespace the ERP uses; instance-wide operational data lives in `%SYS`.

**What MCP needs from IRIS.** The AI Hub framework (`%AI.ToolSet`, `%AI.MCP.Service`, the `iris-mcp-server` bridge) ships in the **2026.x** EAP builds. It is **not** present in older releases such as the dev instance's current `2024.1.2`. Part B is therefore prepared now and applied after the upgrade. In the meantime, a client-side bridge over the Atelier REST API already gives agents live read-only SQL access to those 2024.1.2 instances (see [Interim path](#interim-path-client-side-bridge-works-today-on-20241x)).


## Architecture at a glance

```
Claude Code / Codex (agent)
   |
   |-- iris-mcp-server (stdio) ------> IRIS web port
            endpoints:                   /mcp/erp   (NameSpace ERP  : business data + schema)
                                         /mcp/sys   (NameSpace %SYS : processes, databases, locks)
            each endpoint = a %AI.MCP.Service web app exposing <Query> ToolSets
```

An IRIS MCP tool is a declarative SQL query: you write `<Query>` blocks in a `%AI.ToolSet` class, IRIS prepares them at compile time, and each becomes a callable MCP tool that returns a structured JSON envelope (`columns`, `rows`, `row_count`, `truncated`, `elapsed_ms`). Tools come in two kinds: **declarative `<Query>` tools** with fixed, compile-time SQL (best for curated, named, parameterized queries), and **method tools** that run ObjectScript and can therefore execute arbitrary dynamic SQL passed as an argument, with no per-query compilation. The ERP's large and varied body of SQL needs both (see [Fixed `<Query>` tools vs ad-hoc SQL](#fixed-query-tools-vs-ad-hoc-sql)).

**Read-only by design.** These tools expose read-only queries only, whether a declarative `<Query>` or a guarded ad-hoc SELECT runner ([Fixed `<Query>` tools vs ad-hoc SQL](#fixed-query-tools-vs-ad-hoc-sql)). Creating or changing schema and data is done through normal IRIS SQL/ObjectScript deployment (as in [Appendix C](#appendix-c-local-stub-schema-for-the-examples)), never through MCP: an agent gets query access, not DDL or writes. This is a deliberate boundary, not just a default. Enforce it operationally with a least-privilege service account and the `RESOURCE` parameter (see [Credentials and security](#credentials-and-security)); do not add tools that run DDL or mutate data.

## Naming and conventions (official)

| Item | Convention | Examples |
|------|------------|----------|
| CSP web-application path | `/mcp/<namespace>` (lowercase) | `/mcp/erp`, `/mcp/sys` |
| Web-application type | standard CSP app, `Type=2`, `NameSpace` set to the target namespace | one app per exposed namespace |
| MCP service class | `ERP.MCP.Service.<Scope>` extends `%AI.MCP.Service` | `ERP.MCP.Service.Erp`, `ERP.MCP.Service.Sys` |
| ToolSet class | `ERP.MCP.ToolSet.<Topic>` extends `%AI.ToolSet` | `ERP.MCP.ToolSet.SysMetrics`, `ERP.MCP.ToolSet.SqlIntrospection` |
| Helper | `ERP.MCP.Probe` | (optional, see Appendix A) |

The ObjectScript package `ERP.MCP` is the standard. It is a class-name prefix and is independent of the IRIS namespace: the same `ERP.MCP.*` classes can be compiled into any namespace, and there is no conflict with a namespace also named `ERP` (package names and namespace names are separate concept spaces in IRIS). Do **not** encode the namespace into the package (no `<namespace>.MCP`): it would be redundant with the deployment namespace, the web app's `NameSpace`, and the `/mcp/<namespace>` path, and it has no valid form for `%SYS` (the leading `%` is reserved for system classes). Per-endpoint scope is expressed in the class leaf instead (`ERP.MCP.Service.Erp`, `ERP.MCP.Service.Sys`). Adding another namespace later means adding one `/mcp/<namespace>` web app plus the ToolSets that namespace should expose.

## Credentials and security

- **Local Docker (throwaway).** A fresh IRIS install defaults to username `_SYSTEM`, password `SYS`. Acceptable for a local lab only.
- **Shared and licensed instances (dev instance, later production).** Do **not** use `_SYSTEM`/`SYS`. Create a dedicated, least-privilege **service account** for the `iris-mcp-server` connection and for endpoint authentication, and keep the ToolSets read-only. `%AI.MCP.Service` supports a `RESOURCE` parameter to gate every request behind one or more IRIS security resources; set it on the service subclass to lock a deployment down.

## Part A: local setup via Docker (available now)

Local instance of the AI Hub community EAP, for hands-on evaluation. Runs in WSL2 + Docker on Windows. Because the community image ships only the `%SYS` and `USER` namespaces, this part exposes the instance-wide operational tools at `/mcp/sys`. The business and schema endpoint (`/mcp/erp`) belongs to Part B, where a real `ERP` namespace exists.

### Prerequisites

- WSL2 with Docker Engine running inside it (`docker version` returns client and server).
- The EAP image tarball from the AI Hub Evaluation Portal: <https://evaluation.intersystems.com/Eval/early-access/AIHub>. The community image needs no license key. This part assumes it is at `~/intersystems/iris-community-<version>-docker.tar.gz` inside WSL.

> If the container appears to die on its own after about a minute, that is WSL2 idle-terminating the VM, not IRIS. See [Pitfalls](#pitfalls) for the one-line fix.

### Step 1: load and run the container

```bash
# Load the image (prints e.g. "Loaded image: .../iris-community:2026.3.0AI.108.0")
docker image load -i ~/intersystems/iris-community-2026.3.0AI.108.0-docker.tar.gz

# --restart unless-stopped lets it survive dockerd/WSL restarts.
docker run --name iris-ai-hub -d --restart unless-stopped \
  -p 1972:1972 -p 52773:52773 \
  <IMAGE_TAG_FROM_LOAD_OUTPUT>

docker inspect -f '{{.State.Health.Status}}' iris-ai-hub   # wait for "healthy"
```

Ports: `1972` superserver (SQL/native), `52773` web server / Management Portal (<http://localhost:52773/csp/sys/UtilHome.csp>).

> `iris-ai-hub` is a **custom container name** (set via `--name`), not a default. Pick any name; every command below assumes `iris-ai-hub`, so substitute if you change it.

### Step 2: unexpire the default password (local only)

```bash
printf 'do ##class(Security.Users).UnExpireUserPasswords("*")\nhalt\n' \
  | docker exec -i iris-ai-hub iris session IRIS -U %SYS
```

### Step 3: load classes reliably

Do not paste multi-line ObjectScript into an interactive `iris session`: the terminal runs each physical line on its own, so block constructs (`while {...}`) that span lines break. Write the `.cls` file, copy it in as clean LF, and compile it. Compile in `%SYS` so the system tables resolve.

```bash
tr -d '\r' < ERP.MCP.ToolSet.SysMetrics.cls | docker exec -i iris-ai-hub sh -c 'cat > /tmp/SysMetrics.cls'
printf 'do $system.OBJ.Load("/tmp/SysMetrics.cls","ck")\nhalt\n' \
  | docker exec -i iris-ai-hub iris session IRIS -U %SYS
```

`"ck"` = compile and keep source. Compiled classes persist on disk and survive container restarts.

### Step 4: the operational ToolSet

Save as `ERP.MCP.ToolSet.SysMetrics.cls` and load it per Step 3.

```xml
/// Instance-wide operational metrics and SQL-catalog introspection, exposed as MCP tools.
/// Compile in %SYS so the system tables resolve.
Class ERP.MCP.ToolSet.SysMetrics Extends %AI.ToolSet
{

XData Definition [ MimeType = application/xml ]
{
<ToolSet Name="SysMetrics">
  <Description>Read-only operational and SQL-layer introspection for InterSystems IRIS: live processes, configured databases, the table catalog and index metadata.</Description>

  <Query Name="TopProcesses"
         Description="Active IRIS processes ordered by CPU time, with memory usage and global-reference counts. Filter by minimum CPU time in milliseconds."
         Arguments="minCpu As %Integer = 0"
         MaxRows="25">
    SELECT Pid, State, NameSpace, Routine, UserName, CPUTime, MemoryUsed, GlobalReferences, CommandsExecuted, StartTimeUTC
    FROM %SYS.ProcessQuery
    WHERE CPUTime >= :minCpu
    ORDER BY CPUTime DESC
  </Query>

  <Query Name="Databases"
         Description="Configured IRIS databases: logical name and directory on disk."
         Arguments=""
         MaxRows="200">
    SELECT Name, Directory, MountRequired, MountAtStartup
    FROM Config.Databases
    WHERE SectionHeader = 'Databases'
    ORDER BY Name
  </Query>

  <Query Name="Tables"
         Description="SQL tables in a schema (empty schema = all schemas in the current namespace)."
         Arguments="schema As %String = ''"
         MaxRows="500">
    SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE
    FROM INFORMATION_SCHEMA.TABLES
    WHERE (:schema = '' OR TABLE_SCHEMA = :schema)
    ORDER BY TABLE_SCHEMA, TABLE_NAME
  </Query>

  <Query Name="Indexes"
         Description="Index metadata for one table: index name, indexed column, ordinal position, uniqueness and primary-key flag."
         Arguments="schema As %String, table As %String"
         MaxRows="200">
    SELECT INDEX_NAME, COLUMN_NAME, ORDINAL_POSITION, NON_UNIQUE, PRIMARY_KEY
    FROM INFORMATION_SCHEMA.INDEXES
    WHERE TABLE_SCHEMA = :schema AND TABLE_NAME = :table
    ORDER BY INDEX_NAME, ORDINAL_POSITION
  </Query>

</ToolSet>
}

}
```

A clean compile is the first validation: IRIS prepares every `<Query>` at compile time, so compiling means all four queries are valid.

### Step 5: the MCP service and endpoint

Save as `ERP.MCP.Service.Sys.cls` and load it per Step 3.

```objectscript
/// MCP service for instance-wide operational metrics (%SYS scope).
Class ERP.MCP.Service.Sys Extends %AI.MCP.Service
{

Parameter SPECIFICATION As STRING = "ERP.MCP.ToolSet.SysMetrics";

}
```

Register the endpoint as a standard CSP web application, in `%SYS`:

```bash
docker exec -i iris-ai-hub iris session IRIS -U %SYS <<'OBJSCRIPT'
kill p
set p("NameSpace")="%SYS"
set p("DispatchClass")="ERP.MCP.Service.Sys"
set p("Type")=2
set p("Enabled")=1
set p("AutheEnabled")=96
set p("Description")="IRIS MCP endpoint: instance-wide operational metrics"
if '##class(Security.Applications).Exists("/mcp/sys") { write "create: ",$system.Status.GetErrorText(##class(Security.Applications).Create("/mcp/sys",.p)),! }
halt
OBJSCRIPT
```

Key values: `Type=2` (the served CSP web-application type), `AutheEnabled=96` (password + unauthenticated, tighten on shared instances), `NameSpace=%SYS`. The service routes under `/v1/`: `GET /v1/health`, `GET /v1/services`, `POST /v1/tool_call`, `WS /v1/ws`. The application root returns 404 by design.

### Step 6: verify

Endpoint reachable:

```bash
docker exec iris-ai-hub wget -qO- --user=_SYSTEM --password=SYS \
  http://127.0.0.1:52773/mcp/sys/v1/health
# {"status":"healthy","service":"ERP.MCP.Service.Sys","specification":"ERP.MCP.ToolSet.SysMetrics",...}
```

Tools execute with live data (same path the service uses internally):

```bash
docker exec -i iris-ai-hub iris session IRIS -U %SYS <<'OBJSCRIPT'
set mgr=##class(%AI.ToolMgr).%New()
do ##class(%AI.MCP.Service).LoadToolSetsToManager(mgr,"ERP.MCP.ToolSet.SysMetrics")
set it=mgr.%Discover().%GetIterator()
while it.%GetNext(.k,.t) { write "tool: ",t.name,! }
set res=mgr.ExecuteTool("TopProcesses",{"minCpu":0})
write res.%ToJSON(),!
halt
OBJSCRIPT
```

Expected: the catalog lists `TopProcesses`, `Databases`, `Tables`, `Indexes`, and `ExecuteTool` returns `{"value":{"columns":[...],"rows":[...live processes...],"row_count":25,...}}`.

## Debugging a database locally (external IRIS.DAT)

The ERP can also run locally, so a powerful pattern is to mount an `IRIS.DAT` into the local container and point the MCP tools at real data. The file mechanics work across platform and version; there is one hard prerequisite, **collation** (see below).

### Steps

1. **Copy the `IRIS.DAT` into WSL's native filesystem.** IRIS cannot reliably mount a database over the Windows drive mount (`/mnt/c`, drvfs), so copy it into ext4 and make it accessible to the container's IRIS user:
   ```bash
   mkdir -p /home/<you>/erpdb
   cp /mnt/c/tmp/IRIS.DAT /home/<you>/erpdb/IRIS.DAT
   chmod 777 /home/<you>/erpdb && chmod 666 /home/<you>/erpdb/IRIS.DAT
   ```
2. **Run the container with the directory bind-mounted.** A running container cannot gain a new volume, so recreate it. Recreating from the base image resets in-container work, so redeploy any ToolSets afterward:
   ```bash
   docker rm -f iris-ai-hub
   docker run --name iris-ai-hub -d --restart unless-stopped \
     -p 1972:1972 -p 52773:52773 \
     -v /home/<you>/erpdb:/ext/erp \
     <IMAGE_TAG>
   ```
3. **Register the database and a namespace** (in `%SYS`). `Config.Databases.Create` on a directory that already holds an `IRIS.DAT` mounts the existing database (it does not overwrite it):
   ```bash
   docker exec -i iris-ai-hub iris session IRIS -U %SYS <<'OBJSCRIPT'
   set p("Directory")="/ext/erp/"
   write "db: ",$system.Status.GetErrorText(##class(Config.Databases).Create("ERPDATA",.p)),!
   set n("Globals")="ERPDATA", n("Routines")="ERPDATA"
   write "ns: ",$system.Status.GetErrorText(##class(Config.Namespaces).Create("ERP",.n)),!
   halt
   OBJSCRIPT
   ```
4. **Enter the namespace and query** (`iris session IRIS -U ERP`, or expose it as `/mcp/erp`) as elsewhere in this guide.

### Prerequisite: the local instance must have the DB's collation

This is the gate, and it is hard. A database carries a **default collation**, and IRIS refuses to mount it unless that collation is installed. Our databases use **collation 30** (a national, German-locale collation). The vanilla AI Hub **community** image is English-only NLS, does not have it, and cannot import it (`##class(Config.NLS.Tables).ImportAllFromNls()` returns 0). Symptoms:

```
# messages.log
Failed to mount /ext/erp/ because its default collation (30) is not available
# on entering the namespace
ERP access failed: <DIRECTORY> 237 /ext/erp/
```

### What is not a problem

- **Cross-platform.** The DB was created on IRIS for Windows (x86-64) and mounted on Linux (x86-64); same-endian databases are portable, so the platform difference is fine.
- **Location.** Copying even a large database into WSL ext4 is quick; mounting over `/mnt/c` is the thing to avoid.

## Part B: configuring an existing database (dev instance)

Target the licensed dev database instead of a throwaway container. This is the real path for the ERP.

**Instance:** `http://<dev-host>:57772/csp/sys/UtilHome.csp` (web port `57772`, note it differs from the container's `52773`).
**Current build:** `IRIS for Windows (x86-64) 2024.1.2`.
**Gate:** the AI Hub ships in **2026.x**. On `2024.1.2` the `%AI.*` framework and `iris-mcp-server` are not available. The steps below are prepared now and applied once the pending 2026.x upgrade lands. Until then this section is a configuration plan, not an executable runbook.

### Namespaces and endpoints

| Endpoint | NameSpace | Purpose | ToolSets |
|----------|-----------|---------|----------|
| `/mcp/erp` | `ERP` | ERP schema and business data | `ERP.MCP.ToolSet.SqlIntrospection` (tables/indexes/statement stats for the ERP's own tables), plus business ToolSets as they are defined |
| `/mcp/sys` | `%SYS` | instance-wide operational metrics | `ERP.MCP.ToolSet.SysMetrics` (as in Part A) |

`INFORMATION_SCHEMA` is per-namespace, so schema and index introspection for the ERP's tables must run in the `ERP` namespace (the `/mcp/erp` endpoint). System tables such as `%SYS.ProcessQuery` are percent-mapped and reachable from any namespace, while `Config.Databases` is `%SYS`-only, which is why operational tools live at `/mcp/sys`.

### Preparation checklist (apply after the 2026.x upgrade)

1. Confirm the AI Hub is present: `##class(%Dictionary.CompiledClass).%ExistsId("%AI.MCP.Service")` returns 1.
2. Create the least-privilege **service account** for MCP access (see [Credentials and security](#credentials-and-security)). Do not reuse `_SYSTEM`.
3. Deploy the `ERP.MCP.*` classes into the target namespaces via source control or IPM rather than ad hoc `$system.OBJ.Load`.
4. Register one web application per endpoint, `Type=2`, `NameSpace` set accordingly, authentication set to password (not unauthenticated), and set the `RESOURCE` parameter on the service class if you want resource-level gating.
5. Point `iris-mcp-server` at the instance (see Part C), the dev host, web port `57772`, service-account credentials, and list both endpoints.
6. Verify each endpoint with `GET /mcp/<ns>/v1/health` and an `ExecuteTool` smoke test as in Part A, Step 6.

### What differs from the local setup

- Web port is `57772`, not `52773`.
- A dedicated service account, not `_SYSTEM`/`SYS`; endpoints use password authentication.
- A real `ERP` namespace exists, so `/mcp/erp` is meaningful.
- Classes are deployed through source control, not compiled ad hoc.

## Part C: client wiring (Claude Code and Codex)

Add the IRIS bridge to the client's MCP config: `mcpServers` in `~/.claude.json` for Claude Code (user scope, JSON below), or `[mcp_servers.<name>]` in `~/.codex/config.toml` / project `.codex/config.toml` for Codex, with the same `command` and `args`.

> **Known EAP issues (AI Hub bridge, not yet verified end-to-end).** What is verified today is the endpoint health check (`GET /v1/health`) and *in-session* tool execution (`ToolMgr.ExecuteTool`, [Part A, Step 6](#step-6-verify)). The full agent → endpoint → tool chain through `iris-mcp-server` is **not** working on the current EAP build: the bridge hangs at "Connecting to IRIS…" and `POST /v1/tool_call` returns `ToolNotFound` (reported to InterSystems as EAP feedback). So treat the config below as prepared, not proven, and do not sink time into debugging the bridge hang; it is a known EAP bug. For an agent path that works **today** against a 2024.1.2 instance, use the [interim client-side bridge](#interim-path-client-side-bridge-works-today-on-20241x) instead.

**Local Docker target** (bridge runs inside the container over stdio):

```json
{
  "mcpServers": {
    "iris-sys": {
      "command": "wsl.exe",
      "args": ["docker","exec","-i","iris-ai-hub",
               "/usr/irissys/bin/iris-mcp-server","--transport","stdio","run",
               "--iris-host","127.0.0.1","--iris-port","52773",
               "--iris-user","_SYSTEM","--iris-password","SYS",
               "--iris-endpoint","/mcp/sys"]
    }
  }
}
```

**Existing database target** (dev instance, after the 2026.x upgrade; `iris-mcp-server` installed on the machine that runs the agent, one `--iris-endpoint` per namespace):

```json
{
  "mcpServers": {
    "iris-erp": {
      "command": "iris-mcp-server",
      "args": ["--transport","stdio","run",
               "--iris-host","<dev-host>","--iris-port","57772",
               "--iris-user","<service-account>","--iris-password","<secret>",
               "--iris-endpoint","/mcp/erp","--iris-endpoint","/mcp/sys"]
    }
  }
}
```

Keep secrets out of shared config where possible (environment variables or the client's secret store).

## Interim path: client-side bridge (works today on 2024.1.x)

The AI Hub path (Parts B/C) is the **target** model, but it is gated on the 2026.x upgrade and, on the current EAP build, not yet working end-to-end ([Known EAP issues](#part-c-client-wiring-claude-code-and-codex)). There is a second path that gives an agent live SQL access to an existing IRIS **today**, with no `%AI.*` and no IRIS-side deployment: a **client-side MCP bridge** that talks to IRIS over the **Atelier REST API** on the web port. This runs in daily use against our QA instance (namespace `ERP`) on **IRIS 2024.1.2**, the exact build the dev instance is on. It is an **addition, not a replacement**: shared and licensed instances still get the AI Hub posture once 2026.x lands.

### Why it works without the AI Hub

The AI Hub server-side framework is not involved. The bridge is an ordinary stdio MCP server running on the developer's machine; it authenticates to IRIS and forwards SQL to the Atelier REST **query action**, which has shipped in IRIS for many releases and is present on 2024.1.2. It exposes ad-hoc tools (`iris_query`, `iris_table_info`, …) rather than curated `<Query>` tools. The mechanism is a single POST, verified here against the local instance and identical on 2024.1.2 (only the web port differs, `57772`):

```bash
# The Atelier query action the bridge wraps. Returns {"result":{"content":[ ...rows... ]}}.
curl -s --user <user>:<secret> -H 'Content-Type: application/json' -X POST \
  http://<host>:<web-port>/api/atelier/v1/<namespace>/action/query \
  -d '{"query":"SELECT TOP 2 Pid, NameSpace, CPUTime FROM %SYS.ProcessQuery ORDER BY CPUTime DESC"}'
```

### Security posture (why this is dev-only)

The trade-off is where the read-only guard lives. The AI Hub model puts it in the **account** (a least-privilege read-only service account plus `RESOURCE`, see [Credentials and security](#credentials-and-security)). The interim bridge connects as an application account that *does* have write capability, and read-only-ness rests on the **tool layer** (only read tools wired, or a SELECT-only guard as in [`RunSelect`](#fixed-query-tools-vs-ad-hoc-sql)). That is acceptable for **dev/QA** instances; do **not** point it at a shared or licensed box with a write-capable account. There, the account-level posture remains the target.

### What already works (verified against QA 2024.1.2)

The whole [analysis workflow](#analysis-performance-and-developer-support) is available on 2024.1.2 through this path, not only after the 2026.x upgrade:

- **Ad-hoc SQL.** Arbitrary read-only SELECT/JOIN over the real schema (the `iris_query` tool), the same pattern [`RunSelect`](#fixed-query-tools-vs-ad-hoc-sql) rebuilds as a `%AI.ToolSet` method.
- **Execution plans.** `EXPLAIN SELECT …` returns the plan with cost and index access (e.g. a `SalesContract` filter on `CustomerID`: `Read index map SQLUser.SalesContract.IX_SalesContract_CustomerID_0`, Cost 633.84), covering [`ExplainQuery`](#an-introspection-toolset) / `RunSelect`-with-`EXPLAIN`.
- **Statement statistics.** `INFORMATION_SCHEMA.STATEMENTS` is queryable ad hoc, and on QA the runtime columns (`StatCount`/`StatTotal`/`StatAverage`) are **populated under load**, so the top statements by `StatTotal` are real triage candidates immediately (e.g. a migration-log self-join at ~648k executions / ~21,000 s total). This is the [caveat](#caveats) about runtime stats being blank on an idle instance, filled in on a real system.
- **Deeper introspection.** The bridge's `iris_table_info` also surfaces the backing globals (`^SQLUser.SalesContractD`/`I`/`C`) and whether a table is class-projected or DDL-created, a layer the AI Hub `<Query>` tools do not cover.

Net: the interim bridge is not just "ad-hoc SQL works already" but "the full triage/introspection loop (EXPLAIN, top statements, index and global introspection) works already." For the InterSystems evaluation it is also strong empirical support for the [ad-hoc SQL](#fixed-query-tools-vs-ad-hoc-sql) model: daily agent practice already runs almost entirely over read-only `iris_query`, exactly the shape `RunSelect` formalizes.

## ERP example ToolSet

Beyond the system queries, this ToolSet is built from real ERP queries, to show how business SQL becomes MCP tools. On the licensed instance these run in the `ERP` namespace behind `/mcp/erp`. The outputs below were produced locally against minimal stub tables in `USER` (see [Appendix C](#appendix-c-local-stub-schema-for-the-examples)); the sample rows are synthetic. The stub column names and types are taken from the real JPA entities; `getLastGoodsReceiptDate` mirrors an IRIS SQL user-defined function (UDF) shipped with the database migrations, which is `SELECT MAX(GoodsReceiptDate) FROM StockItem WHERE OrderPosID = :orderPosId`.

```xml
/// ERP business query examples as MCP tools.
Class ERP.MCP.ToolSet.ErpExamples Extends %AI.ToolSet
{

XData Definition [ MimeType = application/xml ]
{
<ToolSet Name="ErpExamples">
  <Description>ERP business query examples: sales contracts, interface protocol throughput, procurement payment block.</Description>

  <Query Name="SalesContracts"
         Description="Sales contracts that have at least one position, with contract month and year. Optionally filter by customer id; omit for all."
         Arguments="customerId As %Integer = 0"
         MaxRows="500">
    SELECT DISTINCT SalesContract.ContractID, SalesContract.CustomerID, DATEPART(MONTH,SalesContract.ContractDate) AS contractMonth, DATEPART(YEAR,SalesContract.ContractDate) AS contractYear
    FROM SalesContract INNER JOIN SalesContractPos ON SalesContractPos.ContractID = SalesContract.ContractID
    WHERE (:customerId IS NULL OR SalesContract.CustomerID = :customerId)
  </Query>

  <Query Name="InterfaceRowsPerHour"
         Description="Count of interface protocol rows per day and hour, filtered by TransferType pattern (omit for %DTO%). Interface throughput profiling."
         Arguments="pattern As %String = '%DTO%'"
         MaxRows="500">
    SELECT CAST(Stamp AS DATE) AS perDay, DATEPART(HOUR,Stamp) AS perHour, COUNT(*) AS noDatasets
    FROM InterfaceLog
    WHERE TransferType LIKE (CASE WHEN :pattern IS NULL THEN '%DTO%' ELSE :pattern END)
    GROUP BY CAST(Stamp AS DATE), DATEPART(HOUR,Stamp)
    ORDER BY perDay, perHour
  </Query>

  <Query Name="PaymentBlockedNoGoodsReceipt"
         Description="Order positions with a payment block on the incoming invoice and no goods-receipt date, optionally within a document-date range (omit for unbounded)."
         Arguments="fromDate As %String = '', toDate As %String = ''"
         MaxRows="500">
    SELECT DISTINCT PurchaseOrderPos.DeliveryBranchID, Supplier.SupplierNo, Supplier.ShortName, SalesContract.ContractNo, PurchaseOrder.OrderNo,
                    LIST(DISTINCT PurchaseOrderPos.PositionNo) AS PositionNo, IncomingInvoice.InvoiceNo, TO_CHAR(IncomingInvoice.DocumentDate,'YYYY-MM-DD') AS DocumentDate,
                    IncomingInvoice.PaymentBlock, IncomingInvoice.Type, PurchaseOrderPos.Status, Supplier.SupplierID, SalesContract.ContractID, PurchaseOrder.OrderID, IncomingInvoice.InvoiceID
    FROM PurchaseOrder
        INNER JOIN PurchaseOrderPos ON PurchaseOrder.OrderID = PurchaseOrderPos.OrderID
        LEFT JOIN PurchaseOrderPosItem ON PurchaseOrderPos.OrderPosID = PurchaseOrderPosItem.OrderPosID
        LEFT JOIN SalesContractPos ON PurchaseOrderPos.ContractPosID = SalesContractPos.ContractPosID
        LEFT JOIN SalesContract ON SalesContract.ContractID = SalesContractPos.ContractID
        LEFT JOIN IncomingInvoicePos ON PurchaseOrderPos.OrderPosID = IncomingInvoicePos.OrderPosID
        LEFT JOIN IncomingInvoice ON IncomingInvoice.InvoiceID = IncomingInvoicePos.InvoiceID
        LEFT JOIN Supplier ON PurchaseOrderPosItem.SupplierID = Supplier.SupplierID
    WHERE IncomingInvoice.PaymentBlock = 1
      AND CAST(getLastGoodsReceiptDate(PurchaseOrderPos.OrderPosID) AS DATE) IS NULL
      AND (:fromDate IS NULL OR IncomingInvoice.DocumentDate >= :fromDate)
      AND (:toDate IS NULL OR :toDate >= IncomingInvoice.DocumentDate)
      AND PurchaseOrderPos.Cancelled = 0
    GROUP BY PurchaseOrderPos.DeliveryBranchID, Supplier.SupplierNo, Supplier.ShortName, SalesContract.ContractNo, PurchaseOrder.OrderNo,
             IncomingInvoice.InvoiceNo, IncomingInvoice.DocumentDate, IncomingInvoice.PaymentBlock, IncomingInvoice.Type, PurchaseOrderPos.Status, Supplier.SupplierID, SalesContract.ContractID,
             PurchaseOrder.OrderID, IncomingInvoice.InvoiceID
  </Query>

</ToolSet>
}

}
```

Verified tool calls through `ToolMgr.ExecuteTool` (envelope trimmed to `rows`):

`SalesContracts` with no arguments returns every contract that has a position; with `{"customerId":1001}` only that customer's:

```json
// SalesContracts {}
[{"ContractID":1,"customerId":1001,"contractMonth":1,"contractYear":2026}, {"ContractID":2,"customerId":1002,"contractMonth":2,"contractYear":2026}]
// SalesContracts {"customerId":1001}
[{"ContractID":1,"customerId":1001,"contractMonth":1,"contractYear":2026}]
```

`InterfaceRowsPerHour` with no arguments (defaults to `%DTO%`) buckets protocol rows by day and hour:

```json
[{"perDay":"2026-07-01","perHour":9,"noDatasets":2}, {"perDay":"2026-07-01","perHour":10,"noDatasets":1}]
```

`PaymentBlockedNoGoodsReceipt` with no arguments returns the blocked positions:

```json
[{"deliveryBranchID":1,"SupplierNo":500,"ShortName":"SUPPLIER 500","ContractNo":"KV-1","OrderNo":"B-100","PositionNo":"1","InvoiceNo":"ER-200","DocumentDate":"2025-01-10","PaymentBlock":1,"Type":1,"Status":0}]
```

### Notable IRIS SQL behaviours (for tool authors)

Building these turned up IRIS behaviours worth knowing before writing `<Query>` tools:

- **Optional arguments bind as SQL `NULL`, not the declared default.** An omitted MCP argument arrives as `NULL`; the `Arguments="... = 0"` default is not applied by the framework. Guard optional filters with `(:p IS NULL OR col = :p)`. That is why `SalesContracts` with no `customerId` returns all rows. An earlier `(:p = '' OR ...)` guard silently returned zero rows.
- **A default marks an argument optional; its value is otherwise unused.** The *presence* of a default in `Arguments` (`p As %Integer = 0`) is what drops the parameter from the generated tool schema's `required` list; declaring `p As %Integer` with no default makes it `required`, so a well-behaved agent must supply it. The *value* of the default never binds (see the NULL point above), so it is purely a marker. Verified on the local instance via `%Discover()`: `= 0` and `= ''` both yield `"required":[]`, no default yields `"required":["p"]`. Keep a **type-appropriate** default on optional arguments (`= 0` for `%Integer`, `= ''` for `%String`) rather than dropping it: dropping it silently turns an "omit for all" filter into a mandatory one.
- **`<` breaks the XData XML.** `IncomingInvoice.DocumentDate <= :toDate` fails to compile with a SAX "element name expected" error. Reorder to `:toDate >= IncomingInvoice.DocumentDate`, or wrap the body in `<![CDATA[ ... ]]>`. `>=` is fine.
- **The MCP envelope renders `DATE`/`TIMESTAMP` readably** (ODBC form, e.g. `"2026-07-01"`), so you usually do not need `TO_CHAR` inside a tool. A raw Logical `%SQL.Statement` fetch returns the internal `$HOROLOG` integer instead (e.g. `67752`).
- **`TO_CHAR(x,'YYYY-MM-DD')` expects a `DATE`.** On a `TIMESTAMP` it raises `SQLCODE -400` "Illegal Date value". `CAST(... AS DATE)` first (as `InterfaceRowsPerHour` does), or just select the column and let the envelope format it.
- **String case-folding in `GROUP BY`/`DISTINCT` is a local-Docker artifact, not real-system behaviour.** On the out-of-box local instance strings fold to `SQLUPPER` (case-insensitive), so `ShortName` comes back `"SUPPLIER 500"` although the stored value is `Supplier 500`. IRIS controls this with the **system-wide** SQL configuration setting **"GRUPPIEREN NACH- und DISTINCT-Abfragen müssen Originalwerte erzeugen"** (GROUP BY and DISTINCT queries must produce the original values), in the Management Portal under System > Konfiguration > SQL (Konfigurationseinstellungen). Our installs enable it, so case is preserved there; a default local Docker instance does not. Enable it on the local instance to match production, or use `%EXACT(col)` per query as a local workaround (verified: returns `Supplier 500`).
- **The SQL ran on IRIS essentially unchanged.** `DATEPART(MONTH|YEAR|HOUR, ...)`, `CAST(... AS DATE)`, ODBC literals `{d '...'}` / `{ts '...'}`, `LIKE`, and `LIST(DISTINCT ...)` all work, so the T-SQL-flavoured field queries port with little friction.
- **A referenced SQL function must exist for the query to prepare.** `getLastGoodsReceiptDate(...)` is an ERP UDF; locally it is stubbed as a `SqlProc` returning `NULL` so the query compiles and runs (see Appendix C).

## Analysis: performance and developer support

This is the primary payoff of the integration. Wire both endpoints into the agent (`/mcp/erp` for schema, SQL statistics, and business data; `/mcp/sys` for processes, locks, databases) and it can reason across the application-to-IRIS seam: explain a slow query, see whether an index is used, list existing indexes, infer likely-missing ones, map a schema for a developer new to it, and correlate all of that with what the instance is doing right now. It stays [read-only](#architecture-at-a-glance): the agent analyses and proposes, a human applies any DDL. All SQL below was verified queryable on the eval instance. This workflow is **not** gated on the 2026.x upgrade: the same EXPLAIN/top-statements/introspection loop runs today against an existing 2024.1.2 instance through the [interim client-side bridge](#interim-path-client-side-bridge-works-today-on-20241x).

### Fixed `<Query>` tools vs. ad-hoc SQL

A `<Query>` tool is compiled: its SQL is fixed at compile time. That is right for a curated set of named, parameterized queries the agent picks by description, but the ERP runs a large and varied body of SQL, and recompiling a tool per query shape is not viable in production. You do **not** have to. A **method tool** runs ObjectScript, so a single tool can take a SQL string as an argument and execute it as dynamic SQL, with no per-query compilation. Verified: a `RunSelect` tool given an arbitrary join it had never seen returns rows, also accepts an `EXPLAIN` prefix (returning the plan), and rejects anything else:

```json
// RunSelect {"sql":"SELECT k.ContractID, k.ContractNo, COUNT(*) AS positions FROM SalesContract k JOIN SalesContractPos p ON p.ContractID = k.ContractID GROUP BY k.ContractID, k.ContractNo"}
{"row_count":2,"rows":[{"ContractID":1,"ContractNo":"KV-1","positions":2},{"ContractID":2,"ContractNo":"KV-2","positions":1}]}
// RunSelect {"sql":"EXPLAIN SELECT k.ContractID FROM SalesContract k JOIN SalesContractPos p ON p.ContractID = k.ContractID"}
{"row_count":1,"rows":[{"Plan":"<plans> ... Cost: 3066 ... Read master map SQLUser.SalesContractPos ... </plans>"}]}
// RunSelect {"sql":"DROP TABLE SalesContract"}
{"error":"only SELECT or EXPLAIN is allowed"}
```

The tool is a `%AI.ToolSet` method:

```objectscript
Class ERP.MCP.ToolSet.AdHoc Extends %AI.ToolSet
{
XData Definition [ MimeType = application/xml ]
{
<ToolSet Name="AdHoc">
  <Description>Ad-hoc read-only SQL: one tool runs any SELECT the agent composes, or EXPLAIN of one.</Description>
  <Tool Name="RunSelect" Method="RunSelect">
    <Description>Run a read-only SELECT, or EXPLAIN of one, and return rows as JSON. Other statements and multi-statement input are rejected. Input: the SQL text.</Description>
  </Tool>
</ToolSet>
}

Method RunSelect(sql As %String) As %Library.DynamicObject
{
    set verb = $zconvert($piece($zstrip(sql, "<W"), " ", 1), "U")
    if (verb '= "SELECT") && (verb '= "EXPLAIN") { return {"error": "only SELECT or EXPLAIN is allowed"} }
    if sql [ ";" { return {"error": "single statement only"} }
    set st = ##class(%SQL.Statement).%New()
    set sc = st.%Prepare(sql)
    if $system.Status.IsError(sc) { return {"error": ($system.Status.GetErrorText(sc))} }
    set rs = st.%Execute()
    if rs.%SQLCODE < 0 { return {"error": (rs.%Message), "sqlcode": (rs.%SQLCODE)} }
    set out = {}, rows = [], cc = st.%Metadata.columnCount, n = 0
    while rs.%Next() {
        set row = {}
        for i = 1:1:cc { do row.%Set(st.%Metadata.columns.GetAt(i).colName, rs.%GetData(i)) }
        do rows.%Push(row)
        set n = n + 1
        if n >= 100 { do out.%Set("truncated", 1) quit }
    }
    do out.%Set("row_count", n)
    do out.%Set("rows", rows)
    return out
}

}
```

**Keep it safe.** A tool that runs free-form SQL stays within [read-only by design](#architecture-at-a-glance) only if it is constrained: SELECT-only and single-statement (as above), backed by a **least-privilege read-only service account**, scoped with the `RESOURCE` parameter, and with row and time caps. The account is the real guard; the string checks are defence in depth. `EXPLAIN`-only (below) is the strictest form of this, since it never executes the statement at all.

Two properties of the sketch above are deliberate but worth calling out: the `[ ";"` check is **fail-closed** and also rejects a semicolon inside a string literal (a rare false positive, traded for a trivially safe single-statement guarantee), and the `100`-row cap is hardcoded rather than declared like a `<Query>` tool's `MaxRows`. For production, lift the cap to a bounded `maxRows As %Integer = 100` argument so the agent can page within a ceiling, and keep the account as the real guard.

**Use both.** Curated `<Query>` tools give the agent well-described, named queries (reports, the ERP examples above) it can discover and call by intent; the single `RunSelect` covers the long tail of varied, one-off analysis SQL. Neither recompiles at query time: `<Query>` tools compile once at deployment, `RunSelect` never needs recompiling.

### An introspection ToolSet

Deploy this alongside the business tools at `/mcp/erp`. It exposes the IRIS catalog and the SQL Statement Index as tools.

```xml
Class ERP.MCP.ToolSet.SqlIntrospection Extends %AI.ToolSet
{
XData Definition [ MimeType = application/xml ]
{
<ToolSet Name="SqlIntrospection">
  <Description>Schema and SQL-performance introspection for developer support and query tuning.</Description>

  <Query Name="TableColumns"
         Description="Columns of a table: name, type, nullability, ordinal position."
         Arguments="schema As %String, table As %String" MaxRows="500">
    SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, ORDINAL_POSITION
    FROM INFORMATION_SCHEMA.COLUMNS
    WHERE TABLE_SCHEMA = :schema AND TABLE_NAME = :table
    ORDER BY ORDINAL_POSITION
  </Query>

  <Query Name="TopStatements"
         Description="Heaviest cached SQL statements by total time, with execution count, average time, row count and text. Optionally require a minimum execution count."
         Arguments="minCount As %Integer = ''" MaxRows="50">
    SELECT StatCount, StatTotal, StatAverage, StatRowCount, Statement
    FROM INFORMATION_SCHEMA.STATEMENTS
    WHERE (:minCount IS NULL OR StatCount >= :minCount)
    ORDER BY StatTotal DESC
  </Query>

  <Query Name="ColumnHistogram"
         Description="Value-distribution buckets (percentile to value) the optimizer holds for a column; reveals data skew. Populated only for tuned tables."
         Arguments="schema As %String, table As %String, column As %String" MaxRows="200">
    SELECT BUCKET_POSITION, PERCENTILE, VALUE
    FROM INFORMATION_SCHEMA.COLUMN_HISTOGRAMS
    WHERE TABLE_SCHEMA = :schema AND TABLE_NAME = :table AND COLUMN_NAME = :column
    ORDER BY BUCKET_POSITION
  </Query>

  <Query Name="RunningStatements"
         Description="Statements executing right now, longest-running first: process, user, namespace, cached-query, duration and status."
         Arguments="" MaxRows="100">
    SELECT ProcessID, UserName, Namespace, CachedQuery, ExecutionDuration, Status
    FROM INFORMATION_SCHEMA.CURRENT_STATEMENTS
    ORDER BY ExecutionDuration DESC
  </Query>

  <Query Name="Connections"
         Description="Current SQL connections: process, namespace, user, client IP, transaction state, and the statement each is on."
         Arguments="" MaxRows="200">
    SELECT Pid, NameSpace, UserName, ClientIPAddress, InTransaction, Statement
    FROM INFORMATION_SCHEMA.CURRENT_CONNECTIONS
  </Query>

  <Tool Name="ExplainQuery" Method="ExplainQuery">
    <Description>Return the IRIS query plan for a SQL statement without executing it. Input: the full SQL text.</Description>
  </Tool>

</ToolSet>
}

/// EXPLAIN produces the plan only; it does not run the query.
Method ExplainQuery(sql As %String) As %String
{
    set rs = ##class(%SQL.Statement).%ExecDirect(, "EXPLAIN " _ sql)
    set plan = ""
    while rs.%Next() { set plan = plan _ rs.%GetData(1) _ $char(10) }
    quit plan
}

}
```

For existing indexes, use the `Indexes` tool from the [SysMetrics ToolSet](#part-a-local-setup-via-docker-available-now) (over `INFORMATION_SCHEMA.INDEXES`). Keys and foreign-key relationships come from `INFORMATION_SCHEMA.KEY_COLUMN_USAGE` and `INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS`, queryable the same way.

`EXPLAIN` is an SQL keyword that prefixes any statement and returns its query plan **without executing** it. Because a `<Query>` tool runs fixed SQL with bound value parameters (it cannot take free-form SQL), exposing EXPLAIN-over-any-query needs a method tool: `ExplainQuery` takes the statement text and prepends `EXPLAIN` itself, so an agent can plan any query it composes, or one it pulled from `TopStatements`, and iterate. This is the one place free-form SQL input is acceptable under the [read-only rule](#architecture-at-a-glance): the tool always prepends `EXPLAIN` and executes a single statement, so the agent can obtain plans but never run the underlying query. (Harden it for production: reject a trailing `;` or multiple statements so the prefix cannot be escaped.) If you already expose `RunSelect` with `EXPLAIN` allowed ([above](#fixed-query-tools-vs-ad-hoc-sql)), it covers this too and a separate `ExplainQuery` is optional; keep the dedicated tool only if you want a plan-only entry point the agent picks by name. A plan for a two-table join with an unindexed filter shows the tell-tale scan (real excerpt):

```
Cost: 3053
Read master map SQLUser.SalesContractPos.IDKEY, looping on P.ID.
  Read master map SQLUser.SalesContract.IDKEY, looping on K.ID.
    Test the = condition on K.customerId ...
```

"Read master map ... looping" with no index read is a full scan; a filter on `customerId` that scans is the missing-index signal.

### What the agent can do with both endpoints

- **Slow-query triage.** `TopStatements` finds the heaviest statements, `ExplainQuery` shows each plan, and `Indexes`/`TableColumns` show whether the predicate and join columns are indexed. The agent names the offender and the reason.
- **Missing-index inference.** IRIS has no missing-index catalog like some databases. The agent infers one: a statement with high `StatTotal`, a plan that scans (`looping`, no index), and a `WHERE`/`JOIN` column absent from `INFORMATION_SCHEMA.INDEXES` is an index candidate. The output is a suggested `CREATE INDEX`, applied by a human.
- **Schema onboarding for developers.** `TableColumns` plus `Indexes` plus keys and foreign keys map a domain (for example the sales-contract tables) so a developer new to it gets the shape without reading migration scripts.
- **Operational correlation.** With `/mcp/sys`, pair `TopStatements` with the live processes (which routine or query is hot), lock contention, and database growth. `RunningStatements` and `Connections` add the live SQL picture: what is executing right now, longest-first, and which session holds a transaction, complementing the `%SYS` process list. This is the seam the performance initiative targets; combined with Grafana MCP the agent also sees the OTel spans and GloRefs.
- **Stale-statistics and skew check.** IRIS plans depend on table statistics (extent size, field selectivity) gathered by `TuneTable`. A table that was never tuned can get poor plans; the agent can flag tables to tune (`$SYSTEM.SQL.TuneTable` or the Management Portal), which a human runs. `ColumnHistogram` shows the value distribution the optimizer holds for a column, so heavy skew can explain a bad plan even when an index exists.

### Caveats

- **Read-only.** The agent proposes indexes, tuning, and rewrites; a human applies DDL and `TuneTable`. Do not add write or DDL tools (see [Read-only by design](#architecture-at-a-glance)).
- **Runtime stats must be populated.** `INFORMATION_SCHEMA.STATEMENTS` always lists cached statements and their plans, but `StatCount` / `StatTotal` / `StatAverage` fill in only when SQL statistics gathering is active and the statements have run. On a fresh or idle instance they are blank (as on the eval box); on a system under load they are the signal.
- **`ExplainQuery` takes arbitrary SQL.** EXPLAIN does not execute the statement, but still scope this tool to trusted use.
- **Inference, not a catalog.** Missing-index and tuning suggestions are reasoned from plans, the index catalog, and statement stats; treat them as candidates to review, not automatic changes.

## Reference: IRIS system data that is SQL-queryable

Verified `FROM`-able system tables. Query the instance-wide ones from `%SYS`; query schema and index metadata from the namespace whose tables you want to inspect.

| Data | Table | Notes |
|------|-------|-------|
| Live processes | `%SYS.ProcessQuery` | Pid, State, NameSpace, Routine, CPUTime, MemoryUsed, GlobalReferences, CommandsExecuted, ... |
| Configured databases | `Config.Databases` | filter `SectionHeader = 'Databases'`; `%SYS`-only |
| SQL table catalog | `INFORMATION_SCHEMA.TABLES` | per-namespace; includes `TABLE_PARTITIONS*`, `CURRENT_STATEMENTS`, `STATEMENT_*_STATS` |
| Index metadata | `INFORMATION_SCHEMA.INDEXES` | per-namespace; INDEX_NAME, COLUMN_NAME, ORDINAL_POSITION, NON_UNIQUE, PRIMARY_KEY |

`%SYS.ProcessQuery`, `%Monitor.*` and `INFORMATION_SCHEMA.*` are percent-mapped (visible in every namespace). `SYS.Database`, `SYS.Lock` and `%SYS.LockQuery` are class queries, not tables, so they are not `FROM`-able. To discover columns before writing a `<Query>`, use the [probe helper](#appendix-a-probe-helper).

## Reference: the ToolSet Query element

```xml
<Query Name="ToolName"
       Description="Natural-language description the agent uses to pick the tool."
       Arguments="p1 As %Integer, p2 As %String = ''"
       MaxRows="50">
  SELECT ... WHERE Col = :p1 AND (:p2 = '' OR Other = :p2)
</Query>
```

- `Name`: the tool name, used verbatim when calling the tool.
- `Arguments`: ObjectScript-style parameters, bound in SQL as `:param`. A parameter with a default (`= ''`) is optional; handle "match all" in SQL, e.g. `(:p = '' OR Col = :p)`.
- `MaxRows`: result cap; the envelope's `truncated` flag signals when it is hit.
- Type mapping: `%Integer`/`%Boolean` become number/boolean, `%String` and date types become string.
- Keep `<` out of the SQL (it breaks the XML). Reorder predicates or wrap the body in `<![CDATA[ ... ]]>`.

## Pitfalls

- **Container appears to die after about a minute (exit 255, not OOM).** WSL2 idle-terminates the VM when nothing keeps it busy, taking dockerd and the container with it. Fix permanently by creating `C:\Users\<you>\.wslconfig` with the two lines below, then running `wsl --shutdown` once. Keep `--restart unless-stopped` on the container so it returns with the VM.
  ```ini
  [wsl2]
  vmIdleTimeout=-1
  ```
- **Multi-line ObjectScript pasted into `iris session` fails.** The terminal runs each line independently, so block constructs spanning lines throw `<SYNTAX>`. Put blocks on one line, or compile a class with `$system.OBJ.Load`. Always strip CR (`tr -d '\r'`) when piping host files in.
- **Use `127.0.0.1`, not `localhost`, for in-container tools reaching the web port.** `localhost` can resolve to IPv6 and stall.
- **Web app `Type` must be `2`.** Other values are not served by the gateway and return 404.
- **The application root returns 404 by design.** MCP routes live under `/v1/`. Hit `/mcp/<ns>/v1/health`, not `/mcp/<ns>`.
- **The web port differs per instance.** Container `52773`, dev instance `57772`. Check the instance before wiring a client.
- **The AI Hub requires a 2026.x build.** Older instances (for example the dev instance on `2024.1.2`) do not have `%AI.*` or `iris-mcp-server`.
- **`INFORMATION_SCHEMA` is per-namespace.** Run schema and index tools in the namespace whose tables you want to see; run operational tools that need `Config.Databases` in `%SYS`.
- **On shared instances, use a least-privilege service account.** Never wire `_SYSTEM`/`SYS` into a client config that reaches a licensed box.

## Status and roadmap

- **Works now (interim, no upgrade):** agent-driven read-only SQL, `EXPLAIN` plans, and statement-statistics/introspection against an existing 2024.1.2 instance via the [client-side bridge](#interim-path-client-side-bridge-works-today-on-20241x) over the Atelier REST API. This is the path in daily use against the QA system and covers the full [analysis workflow](#analysis-performance-and-developer-support) today.
- **Works now (AI Hub, local only):** local Docker evaluation (`/mcp/sys`), tool execution verified *in-session* through `%Invoke` and `ToolMgr.ExecuteTool`, endpoint health reachable. The agent → `iris-mcp-server` → tool chain is **not** yet working on the EAP build ([Known EAP issues](#part-c-client-wiring-claude-code-and-codex)).
- **Gated on the 2026.x upgrade of the dev instance:** the AI Hub server-side path, `/mcp/erp` and `/mcp/sys` as `%AI.MCP.Service` endpoints on the licensed instance (Part B).
- **Evaluation backlog:** execution-plan and statement-statistics tools (`INFORMATION_SCHEMA.STATEMENT_*`) and business ToolSets per domain, plus the pairing with Grafana MCP for the performance initiative.

## Appendix A: probe helper

Optional utility for exploring tables (for example discovering column names before writing a `<Query>`). Load per Part A, Step 3.

```objectscript
Class ERP.MCP.Probe Extends %RegisteredObject
{

/// Run an arbitrary SELECT and print a simple ASCII table.
ClassMethod RunSQL(sql As %String, max As %Integer = 20)
{
    set st = ##class(%SQL.Statement).%New()
    set sc = st.%Prepare(sql)
    if $system.Status.IsError(sc) { write "PREPARE ERROR: ", $system.Status.GetErrorText(sc), ! quit }
    set rs = st.%Execute()
    if rs.%SQLCODE < 0 { write "EXEC ERROR SQLCODE=", rs.%SQLCODE, " ", rs.%Message, ! quit }
    set cc = st.%Metadata.columnCount, hdr = ""
    for i=1:1:cc { set hdr = hdr _ $select(i>1:" | ", 1:"") _ st.%Metadata.columns.GetAt(i).colName }
    write hdr, !
    set n = 0
    while rs.%Next() {
        set line = ""
        for i=1:1:cc { set line = line _ $select(i>1:" | ", 1:"") _ rs.%GetData(i) }
        write line, !
        set n = n + 1
        if n >= max { write "... (truncated at ", max, ")", ! quit }
    }
    write "-- rows returned: ", n, !
}

}
```

## Appendix B: command cheat sheet

```bash
# container lifecycle (local)
docker start iris-ai-hub
docker inspect -f '{{.State.Health.Status}}' iris-ai-hub
docker logs --tail 40 iris-ai-hub

# interactive ObjectScript / SQL shell
docker exec -it iris-ai-hub iris session IRIS -U %SYS

# load a class from a host file (clean LF + compile)
tr -d '\r' < MyClass.cls | docker exec -i iris-ai-hub sh -c 'cat > /tmp/MyClass.cls'
printf 'do $system.OBJ.Load("/tmp/MyClass.cls","ck")\nhalt\n' | docker exec -i iris-ai-hub iris session IRIS -U %SYS

# endpoint health (local %SYS endpoint)
docker exec iris-ai-hub wget -qO- --user=_SYSTEM --password=SYS http://127.0.0.1:52773/mcp/sys/v1/health
```

## Appendix C: local stub schema for the examples

To reproduce the [ERP examples](#erp-example-toolset) on the local Docker instance, create minimal tables in `USER` (only the columns the queries touch) plus a stand-in for the `getLastGoodsReceiptDate` UDF, then compile `ERP.MCP.ToolSet.ErpExamples` there. Load classes as in Part A, Step 3, but target `-U USER`. These are stubs for validating SQL and tool wiring, not the real schema; on the licensed box the real tables live in the `ERP` namespace.

UDF stand-in, so `PaymentBlockedNoGoodsReceipt` prepares. It mirrors the real UDF against a local (empty) `StockItem`, so it returns NULL. The class package `User` maps to SQL schema `SQLUser`, so the function resolves unqualified:

```objectscript
Class User.McpStub Extends %RegisteredObject
{
ClassMethod getLastGoodsReceiptDate(orderPosID As %Integer = "") As %Library.TimeStamp [ SqlName = getLastGoodsReceiptDate, SqlProc ]
{
    set rs = ##class(%SQL.Statement).%ExecDirect(, "SELECT MAX(GoodsReceiptDate) FROM StockItem WHERE OrderPosID = ?", orderPosID)
    if rs.%Next() { quit rs.%GetData(1) }
    quit ""
}
}
```

Tables (execute as SQL, for example via `%SQL.Statement.%ExecDirect`):

```sql
CREATE TABLE SalesContract (ContractID INT, customerId INT, ContractDate DATE, ContractNo VARCHAR(20));
CREATE TABLE SalesContractPos (ContractPosID INT, ContractID INT);
CREATE TABLE InterfaceLog (LogID INT, Stamp TIMESTAMP, TransferType VARCHAR(100));
CREATE TABLE PurchaseOrder (OrderID INT, OrderNo VARCHAR(20));
CREATE TABLE PurchaseOrderPos (OrderPosID INT, OrderID INT, contractPosID INT, PositionNo VARCHAR(20), Status INT,
                               deliveryBranchID INT, Cancelled BIT);
CREATE TABLE PurchaseOrderPosItem (OrderPosItemID INT, OrderPosID INT, supplierID INT);
CREATE TABLE IncomingInvoicePos (InvoicePosID INT, OrderPosID INT, InvoiceID INT);
CREATE TABLE IncomingInvoice (InvoiceID INT, InvoiceNo VARCHAR(50), DocumentDate DATE, PaymentBlock BIT, Type INT);
CREATE TABLE Supplier (SupplierID INT, SupplierNo INT, ShortName VARCHAR(100));
CREATE TABLE StockItem (GoodsReceiptDate TIMESTAMP, OrderPosID INT);
```

Insert a few synthetic rows so the tools return data, for example, one `SalesContract` with a matching `SalesContractPos`, a handful of `InterfaceLog` rows with `TransferType` containing `DTO`, and one order chain (`PurchaseOrder` to `PurchaseOrderPos` to `IncomingInvoice` with `PaymentBlock = 1`) for the payment-block tool.
