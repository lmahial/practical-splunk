# Splunk Architecture and the Data Pipeline

**Skill area:** Splunk
**Topic:** Architecture and the Data Pipeline
**Session:** 01

---

## The Big Picture

**In plain language:** Splunk takes raw log data — text files, system events, network traffic — and makes it searchable at scale. Before any of that is possible, four things have to happen to every piece of data: it gets collected, broken into individual events, written to disk, and indexed for fast retrieval. Every Splunk component exists to handle one or more of those four steps.

**Why it matters:** In a production environment, knowing where each component lives and what it does is the difference between diagnosing a data gap in 10 minutes and spending two hours guessing. When a customer's forwarder goes silent, when events stop arriving in an index, when a search returns no results — the first question is always "which stage of the pipeline broke?" You cannot answer that without knowing the architecture.

## The Concept

### Mental Model

Think of Splunk as a mail sorting facility. Raw mail arrives from remote locations (**Forwarder**). A worker opens each envelope, reads the postmark, stamps a date and category on it (**Parsing**). It gets filed into the right cabinet drawer by date and type (**Indexing**). When you need a specific letter, you don't open every drawer — you check the index card and go straight to it (**Searching**). Four distinct jobs, even when one person is doing all of them — which is exactly what a single-instance lab does.

### How It Works

**The five components and what each one actually does:**

1. The **Universal Forwarder (UF)** is a lightweight agent installed on a remote host. Its only job is to read log files and ship raw data to an indexer. It does no parsing, no searching — just collection and forwarding. This is what you'll install on Ubuntu and Rocky Linux VMs in Topic 3.

```bash
# On a remote host — UF reads this file and ships it
# Configured via inputs.conf monitor stanza
[monitor:///var/log/auth.log]
index = linux_logs
sourcetype = linux_secure
```

2. The **Heavy Forwarder** does the same collection job but can also parse, filter, and route events before they leave the source. Used when you need logic at the edge — masking sensitive data, routing specific events to different indexes, dropping noise before it hits the network. More resource-intensive than a UF.

3. The **Indexer** receives data, breaks it into events, writes it to disk in indexes, and makes it searchable. This is the engine. In our lab, the indexer lives on the Mac inside `/Applications/Splunk`.

4. The **Search Head** is where SPL queries run. In our lab it shares the same instance as the indexer. In production it's a dedicated node — or a cluster of nodes — so search load doesn't compete with indexing.

5. The **Deployment Server** pushes config and app updates to fleets of forwarders centrally. Irrelevant with two VMs, essential at 500 forwarders. Configs staged under `etc/deployment-apps/` get pushed to forwarders automatically when they check in.

**The data pipeline — four stages, every event passes through all of them:**

```
Raw log file → [Input] → [Parsing] → [Indexing] → [Searching]
```

1. **Input:** data enters Splunk via a monitor input watching a file path, HEC receiving JSON over HTTP, a scripted input running on a schedule, or a network port.

2. **Parsing:** Splunk breaks the raw stream into individual events, extracts a timestamp, assigns a sourcetype. Line breaking rules and timestamp formats are configured in `props.conf`.

3. **Indexing:** the parsed event is written to disk, compressed, and filed into a bucket inside the named index. The index directory structure on disk reflects this directly.

4. **Searching:** SPL queries hit the indexed data. Splunk uses the bucket time ranges to skip buckets that fall outside your search window — it never scans everything.

**Index time vs search time:**

Index time decisions are permanent. Timestamp extraction, line breaking, sensitive data masking — get these wrong and you must re-index. You cannot patch a misconfigured timestamp after the fact.

Search time decisions — field extractions, lookups, `eval` calculations — are applied at query time and can be changed anytime without touching the raw data. This is why production admins are careful about `props.conf` before data ever flows.

**Single-instance vs distributed:**

Our lab runs everything on one machine — indexer and search head together on the Mac, forwarders on the VMs. In production those roles split onto separate machines for redundancy and scale. The pipeline is identical. The configs are identical. The only difference is replication, failover, and horizontal scale — which comes in Topic 14.

**Splunk Enterprise vs Splunk Cloud:**

Splunk Cloud is Enterprise managed by Splunk on their infrastructure. As a Cloud customer you don't touch the OS, indexer cluster nodes, or most `etc/system` files. Change requests for app installs, version upgrades, and config changes go through Splunk — who will make the changes to the same components you explored on disk in this session.

## Drills

### Drill 1 — Confirm instance is running and identify version

**What I did:**

```bash
/Applications/Splunk/bin/splunk status
/Applications/Splunk/bin/splunk version
```

**Output:**

```
splunkd is running (PID: 66616).
splunk helpers are running (PIDs: 66617 66688 66692 66695 66713).

Splunk 10.4.0 (build f798d4d49089)
```

**What this taught me:** `splunk status` confirms the daemon is up and lists the helper processes Splunk spawns alongside the main process. If splunkd isn't running, nothing else in a session is valid — no drills, no lab, no data. Version matters because config syntax and features differ between releases; knowing you're on 10.4.0 means you can trust the current docs without hunting for version-specific caveats.


### Drill 2 — Locate where indexes live on disk

**What I did:**

```bash
ls /Applications/Splunk/var/lib/splunk/
```

**Output:**

```
_configtracker   _introspection    botsv3       modinputs
_dm_summary      _metrics          defaultdb    persistentstorage
_dsappevent      _metrics_rollup   fishbucket   summarydb
_dsclient        _telemetry        hashDb       _audit.dat
_dsphonehome     audit             historydb    _configtracker.dat
_internaldb      authDb            kvstore      _internal.dat
                                                _introspection.dat
                                                _metrics.dat
                                                _telemetry.dat
                                                main.dat
```

**What this taught me:** Every directory here is an index — each one is a named bucket store on disk. Underscore-prefixed directories (`_internaldb`, `_introspection`, `_metrics`, `_audit`) are Splunk's own internal indexes, hidden from normal search dropdowns but fully searchable. `defaultdb` backs the `main` index — the catch-all when no index is specified at ingest. `fishbucket` is not a data index; it's Splunk's checkpoint tracker for monitor inputs, recording how far into each file has been read so restarts don't cause re-ingestion.


### Drill 3 — Explore `etc/` and identify each directory's purpose

**What I did:**

```bash
ls /Applications/Splunk/etc/
```

**Output:**

```
anonymizer        myinstall         log-btool.cfg          packagetype
apps              openldap          log-cmdline-debug.cfg  passwd
auth              packages          log-cmdline.cfg        prettyprint.xsl
deployment-apps   shcluster         log-debug.cfg          splunk-enttrial.lic
disabled-apps     system            log-node-platform.cfg  splunk-launch.conf
init.d            users             log-searchprocess.cfg  splunk-launch.conf.default
licenses          copyright.txt     log-tlsproxy.cfg       splunk.version
manager-apps      datetime.xml      log-utility.cfg
master-apps       instance.cfg      log.cfg
modules           log-btool-debug.cfg  login-info.cfg
```

**What this taught me:** 
- `etc/` is the config heart of the entire install. 
- `system/` is the most important directory — it holds `default/`(Splunk's shipped configs, never edited) and `local/` (where all admin changes go). 
- `apps/` holds every installed app, each with its own `default/` and `local/`. 
- `users/` holds user-specific knowledge objects — saved searches and private dashboards. 
- `deployment-apps/` is where configs are staged for the Deployment Server to push to forwarders. 
- `auth/` holds SSL certs and LDAP config. 
- The `default/` vs `local/` split that appears in `system/` and inside every app is the single most important config pattern in Splunk — local always wins over default when there's a conflict.

## Lab

**Scenario:** A new Splunk admin inherits an instance with no documentation. Before touching anything, they need to confirm the instance is healthy, understand the version they're running, and map the filesystem so they know where data lives and where configs go. Skipping this step means making changes blind — editing the wrong file, not knowing where indexes are, and having no baseline if something breaks.

**Task:** Confirm splunkd is running. Identify the Splunk version. Map `$SPLUNK_HOME` top-level directories and explain the three that matter for admin work. Locate where indexes live on disk and explain what the directory structure represents. Explore `etc/` and document what each key subdirectory is for.

**What I built:**

```bash
# Confirm the instance is live before doing anything else
/Applications/Splunk/bin/splunk status

# Identify the version — needed to match against correct docs
/Applications/Splunk/bin/splunk version

# Map the top-level install structure
ls /Applications/Splunk/

# Find where indexed data lives on disk
# Every directory here is a named index
ls /Applications/Splunk/var/lib/splunk/

# Map the config directory — most important for admin work
ls /Applications/Splunk/etc/
```

**What actually happened:** First attempt at the session started without confirming splunkd was running — went straight into filesystem exploration with the daemon down. That's backwards. No daemon means no live data, no accurate index state, nothing to validate against. Restarted splunkd before continuing. Lesson: `splunk status` is always step one.

**The result:**

```
splunkd is running (PID: 66616).
splunk helpers are running (PIDs: 66617 66688 66692 66695 66713).

Splunk 10.4.0 (build f798d4d49089)

$SPLUNK_HOME top-level:
bin/    — Splunk executables and CLI tools
etc/    — all configuration files, apps, users, licenses
var/    — indexed data on disk, internal logs, runtime state

var/lib/splunk/ — one directory per index:
_internaldb, _introspection, _metrics, _audit  — internal Splunk indexes
defaultdb    — backs the main index (catch-all)
fishbucket   — monitor input checkpoint tracker, not a data index
historydb    — search history
summarydb    — data model acceleration

etc/ key directories:
system/          — default/ and local/ at system level. Never edit default/.
apps/            — one directory per installed app, each with own default/ and local/
users/           — user-specific knowledge objects
auth/            — SSL certs, LDAP config, MFA settings
licenses/        — license files
deployment-apps/ — configs staged here get pushed to forwarders by Deployment Server
master-apps/     — configs pushed to indexer cluster peers (distributed only)
shcluster/       — Search Head Cluster configs (distributed only)
disabled-apps/   — inactive apps, moved here instead of deleted
```

**Why this approach and not another:** Filesystem exploration before touching any config is standard practice when inheriting or onboarding to an instance. The alternative — jumping straight into the UI — gives you no mental model of where anything actually lives, which means config changes are made blind and debugging is harder than it needs to be.

**What I'd do differently in production:** On a production instance, I'd also check `splunk list licenser-localslave` to confirm license master connectivity, and run `splunk btool check --no-logfile` to surface any config errors before starting any admin work. I'd document the version, license type, and index list as a baseline before making any changes.

## Key Takeaways

- Always confirm `splunk status` before any session work. A down daemon means no live data and no valid baseline.
- Every directory under `var/lib/splunk/` is a named index on disk. This is where your data physically lives — not in a database, not in a blob store, in directories with buckets inside them.
- `etc/system/local/` is where admin config changes go. `etc/system/default/` ships with Splunk and is never edited. Local always wins over default when there's a conflict.
- Index time decisions (timestamp extraction, line breaking, masking) are permanent once data is indexed. Search time decisions (field extractions, lookups, eval) can be changed anytime. Know which bucket your change falls in before you make it.

## Where People Go Wrong

- **Editing `default/` instead of `local/`:** Changes made to files under `default/` get overwritten on upgrade. The correct pattern is always to create or edit the equivalent file under `local/`. The symptom is a config change that mysteriously disappears after a Splunk upgrade.
- **Assuming the instance is healthy without checking:** Starting admin work without running `splunk status` first means you might be exploring a stale filesystem state with the daemon down. Index directories exist on disk even when Splunk isn't running — the output looks the same, but nothing is live.
- **Treating `main` as a permanent index:** `defaultdb` (the `main` index) has no custom retention policy by default. Sending all data to `main` means no per-sourcetype retention control and no index-level RBAC. In production, everything gets its own index with explicit retention settings.


## Senior Engineer Notes

- When you inherit an unknown instance, run `splunk btool check --no-logfile` before touching anything. It surfaces config errors that may already exist — you don't want to own someone else's broken config.
- The `_internal` index is your diagnostic starting point for almost every Splunk problem. Forwarder silent? `index=_internal sourcetype=splunkd component=TcpOutputProc`. Data gap? `index=_internal sourcetype=metrics`. Get comfortable searching `_internal` before you need it under pressure.
- In production, `etc/` is managed by version control or a config management tool (Ansible, Puppet). Admins don't edit configs manually on indexer nodes — they push from a controlled source. Start thinking about `etc/` as something you commit, not something you hand-edit.


## Retain & Reinforce

Read the Splunk docs overview of the data pipeline: [How data moves through Splunk deployments: The data pipeline | Splunk Enterprise](https://help.splunk.com/en/splunk-enterprise/administer/distributed-deployment-manual/10.4/overview-of-splunk-enterprise-distributed-deployments/how-data-moves-through-splunk-deployments-the-data-pipeline)

Next session, before any drills start — explain the five Splunk components and the four pipeline stages out loud without referencing these notes. If you can't, the log wasn't read.