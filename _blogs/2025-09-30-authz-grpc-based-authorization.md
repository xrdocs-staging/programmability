---
published: false
date: '2025-09-30 12:24 -0400'
title: 'AuthZ: gRPC based Authorization'
position: hidden
author: Rahul Sharma
---
# Executive Summary

OpenConfig’s gNSI PathZ introduces path-level authorization for gNMI, giving network operators fine-grained control over who can access which parts of the system. Traditional approaches authorize entire sessions or interfaces, but PathZ evaluates access on a per-path basis, ensuring that even within a single gNMI session, some paths may be permitted while others are denied.  

PathZ policies are structured around users, groups, and rules. Rules specify whether a particular user or group can read or write to a specific gNMI path. Instead of relying on rule order, PathZ uses a “best match” algorithm that guarantees consistent and predictable decisions. This design allows broad access to be defined for groups, with specific overrides for individual users or narrower paths.  

The framework is built with security and operations in mind. Devices load an initial policy securely during bootstrap and then rely on a dedicated gRPC service to rotate, probe, and retrieve policies. PathZ also includes mechanisms for high availability and recovery, ensuring that valid policies remain in effect even during failures, restarts, or controller transitions.  

For operators, PathZ delivers both control and visibility. It integrates with CLI and YANG models to provide counters, statistics, and debugging support. By adopting PathZ, organizations can align gNMI access with their security policies, achieve stronger operational controls, and prepare for a more secure default posture where unrecognized access is denied by default.  

# Introduction to PathZ

PathZ is an authorization framework designed to control which gNMI paths of a network device users can access. Unlike authentication (which confirms a user’s identity), PathZ focuses on what authenticated users are allowed to do once connected.  

The framework allows operators to define:  
- **Policy rules** – each rule defines a single authorization policy.  
- **Groups of users** – users can be logically grouped into categories such as operators or administrators.  
- **Users** – individuals that can be referenced directly in rules or group definitions.  

PathZ rules are evaluated using a **best match** approach rather than first match. This means that the most specific policy takes precedence, allowing flexible configurations that permit broad access while restricting particular sub-paths, or the reverse.  

# Bootstrapping PathZ

Upon system startup, an initial PathZ authorization policy must be securely delivered to the device. This typically happens through **secure ZTP (sZTP)** or **Bootz** ([OpenConfig Bootz project](https://github.com/openconfig/bootz)). The bootstrap policy must be loaded before any gNMI requests are processed.  

If no PathZ policy is present, IOS-XR assumes an implicit *ACCEPT_ALL* policy to maintain backward compatibility with existing gNMI deployments that have not yet adopted PathZ. Once the bootstrap policy is active, the **gNSI PathZ gRPC service** ([PathZ proto](https://github.com/openconfig/gnsi/blob/main/pathz/pathz.proto)) can be used to rotate, probe, finalize (commit), or read the policy.  

# PathZ Scope and Separation

PathZ applies exclusively to **gNMI path-level authorization**. It does not apply to:  
- CLI authorization (handled by CLI task IDs and task groups),  
- NETCONF authorization (handled by NACM), or  
- Other management interfaces such as gNOI, gRIBI, or SNMP.  

It also does not handle **RPC-level authorization** (i.e., which gRPC services a user can call). That is defined separately in the **gNSI Authz specification**. Both policies can coexist and operate independently.  

# The Policy Model

A PathZ policy defines who can access what paths and in what mode (read or write). Based on the [PathZ authorization README](https://github.com/openconfig/gnsi/blob/main/pathz/authorization-README.md), the key elements are:  

- **Users** – individuals who can be matched directly in rules.  
- **Groups** – collections of users (e.g., “admins” or “operators”). Specific user matches always take precedence over group matches.  
- **Policy rules** – define access permissions by matching users or groups to gNMI paths and access modes.  

Rules are evaluated based on the following strict priority order:  
1. Best or most specific match (longest path).  
2. Defined key preferred over wildcard.  
3. User match preferred over group match.  
4. Deny preferred over permit.  
5. Rule that matches the request mode (READ/WRITE) preferred over others.  

If multiple best matches are found, evaluation fails and an error is logged. If no rule matches, an implicit **deny** is assumed. Explicit denies are logged in full fidelity.  

The result of an evaluation includes the **action (PERMIT or DENY)**, the **policy version**, and the **rule identifier**.  

# Example Policy

The table below illustrates a sample policy:  

| Rule # | User | Group  | Path                                               | Action | Mode  |
|--------|------|--------|----------------------------------------------------|--------|-------|
| 1      | Bob  | –      | /interfaces/interface[FourHundredGigE0/0/0/0]      | PERMIT | READ  |
| 2      | Bob  | –      | /interfaces/interface[FourHundredGigE0/0/0/0]      | PERMIT | WRITE |
| 3      | Bob  | –      | /interfaces/interface[FourHundredGigE1/1/1/1]      | DENY   | WRITE |
| 4      | –    | Admins | /interfaces/interface[*]                           | PERMIT | WRITE |
| 5      | Bob  | –      | /interfaces                                        | PERMIT | READ  |
| 6      | –    | Admins | /interfaces/interface[FourHundredGigE0/0/0/0]      | PERMIT | WRITE |
| 7      | Jim  | –      | /interfaces/interface[FourHundredGigE0/0/0/0]      | DENY   | WRITE |

Evaluations from this policy include:  
1. If Bob reads or writes `/interfaces/interface[FourHundredGigE0/0/0/0]`, access is permitted (rules 1 and 2).  
2. If Bob reads `/interfaces/interface[FourHundredGigE1/1/1/1]`, access is permitted (rule 5). If he writes, access is denied (rule 3).  
3. If Bob writes `/interfaces/interface[FourHundredGigE2/2/2/2]`, access is permitted only if he belongs to the Admins group (rule 4). Otherwise, it is denied by the implicit deny.  
4. If Bob reads `/interfaces/interface[FourHundredGigE2/2/2/2]`, access is permitted (rule 5).  
5. If Jim belongs to Admins, he is denied access on `/interfaces/interface[FourHundredGigE0/0/0/0]` (rule 7 overrides rule 6 because user-specific rules take precedence over groups).  

This behavior has been verified using the [Google open-source PathZ engine](https://github.com/openconfig/lemming/blob/main/gnsi/pathz/pathz.go).  

# Model Origin and Namespaces

- For **OpenConfig (OC) models**, module names are omitted from both gNMI paths and PathZ policies, since OC guarantees unique path names.  
- For **non-OC models**, module names must be included in both gNMI paths and PathZ policies to avoid ambiguity.  
- Mixing module naming conventions between policies and gNMI requests may result in incorrect authorization decisions (denials or erroneous permits).  

# Multi-Instance Support

The PathZ service supports multiple policy instances during rotation:  
- **Active instance** – the current policy used for all gNMI operations.  
- **Candidate instance** – uploaded via Rotate RPC, evaluated via Probe RPC, and applied only after Finalize RPC.  

The candidate persists until it is either committed, replaced, or the RPC session ends. Only one rotate RPC can be active at a time; additional attempts are rejected with `UNAVAILABLE`.  

# PathZ Service

The PathZ service centers on the policy engine. Policies can be loaded at bootstrap or updated at runtime.  

## Bootstrap
During bootstrap, the **Bootz** service fetches an initial PathZ policy. The policy is stored in `/misc/config/ems/gnsi/<file-name>`, validated, and then synchronized across RPs. Invalid or corrupted policies are rejected, and EMSd either reverts to a default (permit_all or deny_all) or requests a new policy.  

## RPC Service
The service defines three RPCs:  
- **Rotate()** – Uploads a candidate policy and finalizes it upon commit. Failed or disconnected sessions discard the candidate and revert to the previous policy.  
- **Probe()** – Tests authorization decisions for a given user and gNMI path without executing the operation.  
- **Get()** – Retrieves the active or candidate policy, along with version and timestamp information.  

Policies are updated carefully to avoid corruption. Old files are renamed with `.bak` before saving new versions, and files are mirrored across RPs using RDSFS.  

# High Availability and Recovery

PathZ includes extensive fault handling:  
- **Zero-policy or invalid policy** – Defaults to *permit_all* for backward compatibility but may move to *deny_all* in future releases.  
- **Policy file versioning** – A versioning scheme ensures compatibility across releases. Major version mismatches mark a file as corrupt.  
- **Policy file management** – Only privileged processes can write policy files, enforced through SE-Linux.  
- **RP synchronization** – Active and standby RPs synchronize files. If corruption occurs, the system attempts to recover from backups before defaulting to deny/permit all.  
- **Restarts and switchover** – Upon EMSd restart or RP failover, the system reloads the active policy before starting gNMI services. Split-brain scenarios (where RPs hold different policies) cannot be deterministically resolved in a 2-RP system; instead, the controller must reconcile using `pathz.Get()` and rotate a new policy.  

# CLI and YANG Support

PathZ extends the OpenConfig system model with counters and timestamps per rule, including:  
- **access-rejects** and **last-access-reject**  
- **access-accepts** and **last-access-accept**  

Key CLI commands include:  
- `show gnsi path authorization counters [path <xpaths>] [server-name <server-name>]`  
- `clear gnsi path authorization counters [path <xpaths>] [server-name <server-name>]`  
- `show gnsi path authorization policy`  
- `show gnsi path authorization statistics [standby]`  
- `show gnsi trace pathz`  
- `show tech-support gnsi`  

Configuration options (planned for future releases) will allow operators to change the default implicit policy (`allow` or `deny`). For example:  

```

gnsi path authorization <implicit-no-match> [allow | deny]

```

Syslog integration is not yet available in the current phase.  

# Hands-on with PathZ

# Conclusion

PathZ delivers granular gNMI path-based authorization, enabling operators to enforce precise access controls that align with organizational policies. By supporting secure bootstrapping, runtime updates, best-match rule evaluation, high availability, and operational visibility, PathZ significantly strengthens the operational security posture of network devices.  

For operators adopting gNMI at scale, PathZ ensures both **flexibility** in policy definition and **resilience** in enforcement, while preparing the system for a default deny posture that enhances security without sacrificing manageability.  
