---
published: true
date: '2024-03-05 13:13 -0500'
title: AAA using gNSI on IOX-XR
position: hidden
author: Rahul Sharma
---

# Introduction

<p align="justify">This blog focuses on gNSI, which is a gRPC based security framework that manages AAA on a network device. AAA refers to:</p>

 1. <b>Authentication:</b> Getting access to the system.<br>
 2. <b>Authorization:</b> Assigning authenticated user/group privileges.<br>
 3. <b>Accounting:</b> Logging events,actions, or data.

It is important to note the order of AAA. 
  
# Authorization
	
gNSI offers authorization through two protocols, each operating at different levels:<br>
	1. gRPC based <b>service level</b> using <b>authz</b>.<br>
    2. <b>xpath level</b> using <b>pathz</b>.
    
## authz

<p align="justify">The gRPC suite encompasses several services like gNMI, gNOI, gNSI, each containing numerous RPCs such as Get(), Set(), Rotate(), etc. With authz, network devices can be configured to regulate user access to specific RPCs within these services.</p>

For example- 
```
User Rahul, gRPC = gNMI, rpc = Get() --> Allow
User Eric, gRPC = gNMI, rpc = Subscribe() --> Allow
User Cisco, gRPC = all, rpc = all --> Allow
User Mike, gRPC = gNOI, rpc = Verify() --> Deny
```
<p align="justify">To implement user access control at the RPC level, a policy known as the <b>'Service Authorization Policy'</b> must be defined. Following is a sample policy that implements authorization rules as per the example above:</p> 
```
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
 
<p align="justify"> The gNSI.authz protocol comprises of three RPCs designed to manage authorization: </p>
		
1. The <b>Rotate()</b> RPC changes the policy on the device, with each policy dentified by its 'version' and 'timestamp'.<br>
		
2. The <b>Probe()</b> RPC verifies the current or candidate policy against user authorizations. <br>

3. The <b>Get()</b> RPC return the current instance of the authz policy.<br>
 
<p align="justify">Authz is supported from XR 7.11.1 onwards. There are two ways to leverage authz:</p>

<b> 1. XR CLI </b>

Below are the steps to utilize authz using XR CLI:<br>

<p align="justify"><u>Step1:</u> Create a service authorization policy. It is a JSON file which can either be copied from a server, or directly produced on the router itself. In this case, a file named authz_policy.json has been created in the /misc/scratch directory. The file content mirrors the sample policy outlined above:</p>

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

<u>Step 2:</u> Load the policy using the following command:

```
RP/0/RP0/CPU0:cannonball#gnsi load service authorization policy /misc/scratch/authz_policy.json
Mon Apr 15 18:48:27.288 UTC
Successfully loaded policy
```

<u>Step 3:</u> Check current service authorization policy:

```
RP/0/RP0/CPU0:cannonball#show gnsi service authorization policy
Mon Apr 15 20:46:13.332 UTC
{
    "version": "1.0",
    "created_on": 1713213968,
    "policy": "{\"name\":\"Manage RPC level access for users Rahul, Eric,
    cisco and Mike\",\"allow_rules\":[{\"name\":\"Allow user Rahul to 
    access Get() RPC from gNMI service\",\"request\":{\"paths\":
    [\"\/gnmi.gNMI\/Get\"]},\"source\":{\"principals\":[\"Rahul\"]}},
    {\"name\":\"Allow user Eric to access Subscribe() RPC from gNMI 
    service\",\"request\":{\"paths[\"\/gnmi.gNMI\/Subscribe\"]},\"source\":
    {\"principals\":[\"Eric\"]}},{\"name\":\"Allow user cisco access to all
    RPCs\",\"request\":{\"paths\":[\"*\"]},\"source\":{\"principals\":
    [\"cisco\"]}}],\"deny_rules\":[{\"name\":\"Deny user Mike to access
    Verify() RPC from gNOI service\",\"request\":{\"paths\":
    [\"\/gnoi.os.OS\/Verify\"]},\"source\":{\"principals\":[\"Mike\"]}}]}"
}
```

<u>Step 4:</u> Test the policy using several clients such as gNMIc etc.

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
<p align="justify">This is an expected behavior, as the user Rahul is only authorized for the RPC Get(). Similar tests can be conducted for other users.</p>

<b> 2. gNSI client - gRPCurl </b>

grpcurl is a command-line tool for interacting with gRPC servers, essentially a curl for gRPC. To learn more, click [here](https://github.com/fullstorydev/grpcurl).
  
Following are the steps to use this to interact with gRPC server on the router.
<br>
<br>
<u>Step 1:</u> Clone the OpenConfig gNSI repository: 
```
git clone https://github.com/openconfig/gnsi.git
```
<u>Step 2.</u>  Install grpcurl using one of the method mentioned 
[here](https://github.com/fullstorydev/grpcurl). Here, we have used brew to install grpcurl using the following command:

```
brew install grpcurl
```

<u>Step 3.</u>  Change the working directory to gnsi/authz.
```			
cd gnsi/authz
```
	
<u>Step 4.</u> Use gNSI.authz RPCs to manage authorization on the router.

A.<u> Rotate() RPC</u> - This RPC creates a new policy, or changes the existing policy. Following is the command to create a policy same as mentioned above.

```
➜  authz git:(main) ✗ grpcurl  -vv -plaintext -d '{
"uploadRequest":
    {
        "version": "Creating policy through grpcurl",
        "created_on": 171321154967868,
        "policy": "{\"name\":\"Manage RPC level access for users Rahul, Eric, cisco and 
        Mike\",\"allow_rules\":[{\"name\":\"Allow user Rahul to access Get() RPC from gNMI 
        service\",\"request\":{\"paths\":[\"\/gnmi.gNMI\/Get\”]},\”source\":{\"principals\":
        [\"Rahul\"]}},{\"name\":\"Allow user Eric to access Subscribe() RPC from gNMI 
        service\",\"request\":{\"paths\":[\"\/gnmi.gNMI\/Subscribe\"]},\"source\":{\"principals\":
        [\"Eric\"]}},{\"name\":\"Allow user cisco access to all RPCs\",\"request\":{\"paths\":
        [\"*\"]},\"source\":{\"principals\":[\"cisco\"]}}],\"deny_rules\":[{\"name\":\"Deny user 
        Mike to access Verify() RPC from gNOI service\",\"request\":{\"paths\":
        [\"\/gnoi.os.OS\/Verify\"]},\"source\":{\"principals\":[\"Mike\"]}}]}"
    },
    "force_overwrite": true
    }{"finalize_rotation":{}}' -import-path ../authz -proto authz.proto  -H username:cisco -H
password:cisco123! 172.20.163.79:57400 gnsi.authz.v1.Authz.Rotate

Resolved method descriptor:
// Rotate will replace an existing gRPC-level Authorization Policy on the
// target.
//
// If the stream is broken or any of the steps fail the
// target must rollback to the original state, i.e. revert any changes to
// the gRPC-level Authorization Policy made during this RPC.
//
// Note that only one such RPC can be in progress. An attempt to call this
// RPC while another is already in progress will be rejected with the
// `UNAVAILABLE` gRPC error.
//
// The following describes the sequence of messages that must be exchanged
// in the Rotate() RPC.
//
// Sequence of expected messages:
//   Step 1: Start the stream
//     Client ----> Rotate() RPC stream begin ------> Target
//
//   Step 2: Send gRPC-level Authorization Policy to Target.
//     Client --> UploadRequest(authz_policy) ----> Target
//     Client <-- UploadResponse <--- Target
//
//   Step 3 (optional): Test/Validation by the client.
//     During this step client attempts to call a RPC that is allowed
//     in the new policy and validates that the new policy "works".
//     Additionally the client should call a RPC that is not allowed and
//     the attempt must fail proving that the gRPC-level Authorization Policy
//     "works".
//     Once verified, the client then proceeds to finalize the rotation.
//     If the new verification did not succeed the client will cancel the
//     RPC thereby forcing the target to rollback of the new gRPC-level
//     Authorization Policy.
//
//   Step 4: Final commit.
//     Client ---> FinalizeRequest ----> Target
//
rpc Rotate ( stream .gnsi.authz.v1.RotateAuthzRequest ) returns ( stream .gnsi.authz.v1.RotateAuthzResponse );

Request metadata to send:
password: cisco123!
username: cisco

Response headers received:
content-type: application/grpc

Estimated response size: 0 bytes

Response contents:
{}

Estimated response size: 2 bytes

Response contents:
{
  "uploadResponse": {}
}

Response trailers received:
(empty)
Sent 2 requests and received 2 responses
```

B. <u>Get() RPC</u> - To get current policy.
```
➜  authz git:(main) ✗ grpcurl  -vv -plaintext -import-path ../authz -proto authz.proto  -H 
username:cisco -H password:cisco123! 172.20.163.79:57400  gnsi.authz.v1.Authz.Get

Resolved method descriptor:
// Get returns current instance of the gRPC-level Authorization Policy
// together with its version and created-on information.
// If no policy has been set, Get() returns FAILED_PRECONDITION.
rpc Get ( .gnsi.authz.v1.GetRequest ) returns ( .gnsi.authz.v1.GetResponse );

Request metadata to send:
password: cisco123!
username: cisco

Response headers received:
content-type: application/grpc

Estimated response size: 1471 bytes

Response contents:
{
  "version": "1.0",
  "createdOn": "1713207488",
  "policy": "{\n    \"name\": \"Manage RPC level access for users Rahul, Eric, cisco and Mike\",\n
  \"allow_rules\": [\n        {\n            \"name\": \"Allow user Rahul to access Get() RPC from
  gNMI service\",\n            \"source\": {\n                \"principals\": [\n     
  \"Rahul\"\n                ]\n            },\n            \"request\": {\n 
  \"paths\": [\n                    \"/gnmi.gNMI/Get\"\n                ]\n            }\n  
  },\n        {\n            \"name\": \"Allow user Eric to access Subscribe() RPC from gNMI 
  service\",\n            \"source\": {\n                \"principals\": [\n             
  \"Eric\"\n                ]\n            },\n            \"request\": {\n              
  \"paths\": [\n                    \"/gnmi.gNMI/Subscribe\"\n                ]\n            }\n
  },\n\t{\n            \"name\": \"Allow user cisco access to all RPCs\",\n            \"source\":
  {\n                \"principals\": [\n                    \"cisco\"\n                ]\n   
  },\n            \"request\": {\n                \"paths\": [\n                    \"*\"\n 
  ]\n            }\n        }\n    ],\n    \"deny_rules\": [\n        {\n            \"name\": 
  \"Deny user Mike to access Verify() RPC from gNOI service\",\n            \"source\": {\n 
  \"principals\": [\n                    \"Mike\"\n                ]\n            },\n   
  \"request\": {\n                \"paths\": [\n                    \"/gnoi.os.OS/Verify\"\n  
  ]\n            }\n        }\n    ]\n}\n\n"
}

Response trailers received:
(empty)
Sent 0 requests and received 1 response
```

<p align="justify">C. <u>Probe() RPC</u> - To check if a specific user is authorized to use a particular RPC according to  the current/candidate policy. In this case, user Rahul is being probed for both, Get() and Capabilities() RPCs.</p>

```
➜  authz git:(main) ✗ grpcurl  -vv -plaintext -d '{"user":"Rahul","rpc":"/gnmi.gNMI/Get"}' -
import-path ../authz -proto authz.proto  -H username:cisco -H password:cisco123! 
172.20.163.79:57400  gnsi.authz.v1.Authz.Probe

Resolved method descriptor:
// Probe allows for evaluation of the gRPC-level Authorization Policy engine
// response to a gRPC call performed by a user.
// The response is based on the instance of policy specified in the request
// and is evaluated without actually performing the gRPC call.
rpc Probe ( .gnsi.authz.v1.ProbeRequest ) returns ( .gnsi.authz.v1.ProbeResponse );

Request metadata to send:
password: cisco123!
username: cisco

Response headers received:
content-type: application/grpc

Estimated response size: 7 bytes

Response contents:
{
  "action": "ACTION_PERMIT",
  "version": "1.0"
}

Response trailers received:
(empty)
Sent 1 request and received 1 response 

➜  authz git:(main) ✗ grpcurl  -vv -plaintext -d '{"user":"Rahul","rpc":"/gnmi.gNMI/Capabilities"}' -import-path ../authz -proto authz.proto  -H 
username:cisco -H password:cisco123! 172.20.163.79:57400  gnsi.authz.v1.Authz.Probe

Resolved method descriptor:
// Probe allows for evaluation of the gRPC-level Authorization Policy engine
// response to a gRPC call performed by a user.
// The response is based on the instance of policy specified in the request
// and is evaluated without actually performing the gRPC call.
rpc Probe ( .gnsi.authz.v1.ProbeRequest ) returns ( .gnsi.authz.v1.ProbeResponse );

Request metadata to send:
password: cisco123!
username: cisco

Response headers received:
content-type: application/grpc

Estimated response size: 7 bytes

Response contents:
{
  "action": "ACTION_DENY",
  "version": "1.0"
}

Response trailers received:
(empty)
Sent 1 request and received 1 response
```
