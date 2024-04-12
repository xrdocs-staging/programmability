---
published: false
date: '2024-03-05 13:13 -0500'
title: AAA using gNSI on IOX-XR
position: hidden
author: Rahul Sharma
---

# Introduction

<p align="justify">This article focuses on gNSI, which is a gRPC based security framework that manages AAA on a network device. AAA refers to:
<br>
  <b>Authentication:</b> Getting access to the system.
  <b>Authorization:</b> Assigning authenticated user/group privileges.
  <b>Accounting:</b> Logging events,actions, or data.
</p>  
  
# Authorization
	
gNSI provides authorization at two levels:
	1. gRPC based <b>service level</b> using <b>authz</b>.
    2. <b>xpath level</b> using <b>pathz</b>. </p>
    
## authz

<p align="justify">The gRPC suite encompasses several services like gNMI, gNOI, gNSI, each containing numerous RPCs such as Get(), Set(), Rotate(), etc. With authz, network devices can be configured to regulate user access to specific RPCs within these services.</p>

For example- 

			Rahul, gRPC = gNMI, rpc = Get() --> Allow
			Mike, gRPC = gRIBI, rpc = Add_route() --> Deny
			Eric, gRPC = gNMI, rpc = Subscribe() --> Allow

<p align="justify">To implement user access control at the RPC level, a policy known as the service authorization policy must be defined. Following is a sample policy that implements authorization rules as per the example above.</p> 

<p align="justify">Here is an example for an authz policy that allow all users to use Capabilities RPC of gNMI service.</p>
    		
{
    "name": "Manage RPC level access for users Mike, Rahul and Eric",
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
                    "gnmi.gNMI.Get"
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
                    "gnmi.gNMI.Subscribe"
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
                    "/gnoi.os.OS.Verify"
                ]
            }
        }
    ]
}

 
<p align="justify">There are 3 RPCs defined in gNSI.authz that implement and test these policies.</p>
		
<b>1. Rotate()</b> RPC is used to change policy on the device. Each policy is identified by its 'Version' and 'Timestamp'.
		
<b>2.Probe()</b> RPC is used to probe the current or candidate policy. 

		
<b>3.Get()</b> RPC return the current instance of the authz policy.

 
<p align="justify">Authz is supported from XR 7.11.1 onwards. There are two ways to implement authz:</p>





