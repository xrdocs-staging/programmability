---
published: true
date: '2024-03-05 13:13 -0500'
title: AAA using gNSI on IOX-XR
position: hidden
author: Rahul Sharma
---

# Introduction

<p align="justify">This blog focuses on gNSI, which is a gRPC based security framework that manages AAA on a network device. AAA refers to:
<br>
  <b>Authentication:</b> Getting access to the system.
  <b>Authorization:</b> Assigning authenticated user/group privileges.
  <b>Accounting:</b> Logging events,actions, or data.
</p> 

It is important to note the order of AAA. 
  
# Authorization
	
gNSI offers authorization through two protocols, each operating at different levels:
	1. gRPC based <b>service level</b> using <b>authz</b>.
    2. <b>xpath level</b> using <b>pathz</b>. </p>
    
## authz

<p align="justify">The gRPC suite encompasses several services like gNMI, gNOI, gNSI, each containing numerous RPCs such as Get(), Set(), Rotate(), etc. With authz, network devices can be configured to regulate user access to specific RPCs within these services.</p>

For example- 

        Rahul, gRPC = gNMI, rpc = Get() --> Allow
		Eric, gRPC = gNMI, rpc = Subscribe() --> Allow
      	Cisco, gRPC = all, rpc = all --> Allow
     	Mike, gRPC = gNOI, rpc = Verify() --> Deny

<p align="justify">To implement user access control at the RPC level, a policy known as the service authorization policy must be defined. Following is a sample policy that implements authorization rules as per the example above.</p> 

<details>
  <summary><b>Output</b></summary>
  <script src="https://gist.github.com/rahusha7/25547d750f6de94a071624a914c33f02.js"></script>
</details>
 
<p align="justify"> The gNSI.authz protocol comprises of three RPCs designed to manage authorization: </p>
		
1. The <b>Rotate()</b> RPC changes the policy on the device,with each policy dentified by its 'Version' and 'Timestamp'.
		
2. The <b>Probe()</b> RPC verifies the current or candidate policy against user authorizations. 

		
3. The <b>Get()</b> RPC return the current instance of the authz policy.

 
<p align="justify">Authz is supported from XR 7.11.1 onwards. There are two ways to leverage authz:</p>

<b> 1. XR CLI </b>

Below are the steps to utilize authz using XR CLI.


Step1: Create a service authorization policy. It is a JSON file which can either be copied from a server, or directly produced on the router itself. In this case, a file named authz_policy.json has been created in the /misc/scratch directory. The file content mirrors the sample policy outlined above.

```
RP/0/RP0/CPU0:cannonball#bash
Mon Apr 15 18:40:06.555 UTC
[xr-vm_nodeios_CPU0:/misc/scratch]$ cat authz_policy.json
{
    "name": "Manage RPC level access for users Rahul, Eric, cisco and Mike",
    "allow_rules": [
        {
            "name": "Allow user Rahul to access Get() RPC from gNMI service",
            "source": {
                "principals": [
                    "Rahul"
                ]
            },
            "request": {
                "paths": [
                    "/gnmi.gNMI/Get"
                ]
            }
        },
        {
            "name": "Allow user Eric to access Subscribe() RPC from gNMI service",
            "source": {
                "principals": [
                    "Eric"
                ]
            },
            "request": {
                "paths": [
                    "/gnmi.gNMI/Subscribe"
                ]
            }
        },
	{
            "name": "Allow user cisco access to all RPCs",
            "source": {
                "principals": [
                    "cisco"
                ]
            },
            "request": {
                "paths": [
                    "*"
                ]
            }
        }
    ],
    "deny_rules": [
        {
            "name": "Deny user Mike to access Verify() RPC from gNOI service",
            "source": {
                "principals": [
                    "Mike"
                ]
            },
            "request": {
                "paths": [
                    "/gnoi.os.OS/Verify"
                ]
            }
        }
    ]
}
```

Step 2: Load the policy using the following command:

```
RP/0/RP0/CPU0:cannonball#gnsi load service authorization policy /misc/scratch/authz_policy.json
Mon Apr 15 18:48:27.288 UTC
Successfully loaded policy
```

Step 3: Check current service authorization policy:

```
		RP/0/RP0/CPU0:cannonball#show gnsi service authorization policy
Mon Apr 15 20:46:13.332 UTC
{
    "version": "1.0",
    "created_on": 1713213968,
    "policy": "{\"name\":\"Manage RPC level access for users Rahul, Eric, cisco and Mike\",\"allow_rules\":[{\"name\":\"Allow user Rahul to access Get() RPC from gNMI service\",\"request\":{\"paths\":[\"\/gnmi.gNMI\/Get\"]},\"source\":{\"principals\":[\"Rahul\"]}},{\"name\":\"Allow user Eric to access Subscribe() RPC from gNMI service\",\"request\":{\"paths\":[\"\/gnmi.gNMI\/Subscribe\"]},\"source\":{\"principals\":[\"Eric\"]}},{\"name\":\"Allow user cisco access to all RPCs\",\"request\":{\"paths\":[\"*\"]},\"source\":{\"principals\":[\"cisco\"]}}],\"deny_rules\":[{\"name\":\"Deny user Mike to access Verify() RPC from gNOI service\",\"request\":{\"paths\":[\"\/gnoi.os.OS\/Verify\"]},\"source\":{\"principals\":[\"Mike\"]}}]}"
}
```

Step 4: Test the policy using several clients such as gNMIc etc.

```
➜  authz git:(main) ✗ gnmic -a 172.20.163.79:57400 -u Rahul -p Rahul123! --insecure capabilities --encoding json_ietf
		target "172.20.163.79:57400", capabilities request failed: "172.20.163.79:57400" CapabilitiesRequest failed: rpc error: code = PermissionDenied desc = unauthorized RPC request rejected
		Error: one or more requests failed
		
		➜  authz git:(main) ✗ gnmic -a 172.20.163.79:57400 -u Rahul -p Rahul123! --insecure get --path 'Cisco-IOS-XR-shellutil-oper:system-time/clock' --encoding json_ietf

[
  {
    "source": "172.20.163.79:57400",
    "timestamp": 1713207628424981721,
    "time": "2024-04-15T15:00:28.424981721-04:00",
    "updates": [
      {
        "Path": "Cisco-IOS-XR-shellutil-oper:system-time/clock",
        "values": {
          "system-time/clock": {
            "day": 15,
            "hour": 19,
            "millisecond": 420,
            "minute": 0,
            "month": 4,
            "second": 28,
            "time-source": "calendar",
            "time-zone": "UTC",
            "wday": 1,
            "year": 2024
          }
        }
      }
    ]
  }
]
```
This is an expected behavior, as the user Rahul is only authorized for the RPC Get(). Similar tests can be conducted for other users.
