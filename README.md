# practical-splunk

I've spent five years coordinating production changes on enterprise Splunk Cloud stacks; app & add-on changes (upgrades, installs, downgrades, custom app vetting), premium app (ES & ITSI) upgrades, access control changes (Splunk users, SAML users), firewall rules, core version trains. I work directly with SRE and engineering teams daily. I know the change lifecycle end to end.

What I haven't had is hands-on access to the infrastructure underneath. I know what the changes are and how to manage them. I haven't been the one building, configuring, or troubleshooting the Splunk instances.

This repo fixes that.

## What this is

A structured, hands-on Splunk Enterprise administration program built on a real lab environment — not a course, not a certification track. Every topic has a drill with real output and a lab that produces something real. Everything here is committed because it was built and tested, not because it was read.

**Lab environment:**

- Splunk Enterprise 10.4.0 on macOS ARM (M2)
- Ubuntu VM — Universal Forwarder deployment
- Rocky Linux VM — log source, scripted inputs
- Two VMs sending real log data into a real indexer



## What I'm building toward

A role on the technical side of Splunk — engineer, admin, or SRE-adjacent. The background is already operational. This is the infrastructure proof.



## Curriculum

| #   | Topic                              |
| --- | ---------------------------------- |
| 01  | Architecture and the Data Pipeline |
| 02  | Indexes and Data Organisation      |
| 03  | Universal Forwarder Setup          |
| 04  | Data Onboarding Methods            |
| 05  | Search Basics and the Pipeline     |
| 06  | Transforming Commands and Stats    |
| 07  | Field Extractions and Lookups      |
| 08  | Configuration File System          |
| 09  | Parsing and Data Manipulation      |
| 10  | User Management and RBAC           |
| 11  | Apps and Add-ons                   |
| 12  | Monitoring Splunk Itself           |
| 13  | Dashboards and Alerts              |
| 14  | Distributed Architecture           |
| 15  | Lab: Admin Capstone                |
| 16  | Lab: SPL Library                   |



## Repo structure

```
practical-splunk/
├── README.md
├── logs/        ← one session log per topic — concept, drills, lab, honest account of what broke
├── scripts/     ← committed scripts
├── configs/     ← annotated config files
└── notes/       ← runbooks, SPL library, reference material
```



## What you'll find in the logs

Each session log covers one topic: the concept explained in plain language, real drill output with explanations, and a lab section written like an engineer — including what broke, what the error was, and what fixed it. Clean logs with no errors aren't credible. The errors are the work.
