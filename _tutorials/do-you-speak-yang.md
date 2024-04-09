---
published: true
date: '2021-09-19 10:39 +0200'
title: YANG Suite - Do you speak YANG?
position: top
author: Antoine Orsoni
excerpt: >-
  Presentation of YANG Suite tool, to retrieve and observe YANG models from your
  network devices.
tags:
  - NETCONF
  - YANG
  - YANG Suite
  - Automation
  - Python
---

{% include toc icon="table" title="Table of Contents" %}

You have been talking SNMP for years with your network devices and you've faced many limitations as discussed in [RFC3535](https://datatracker.ietf.org/doc/html/rfc3535#page-4)? You understand YANG could definitely help you tackle these challenges but you don't yet speak YANG? Then YANG Suite is the right tool for you!

YANG Suite was first developped as an internal Cisco project. In February 2021, the wait was over and it has been made available for everyone.

YANG Suite provides network operators with a common tool to interact with Cisco IOS XE, IOS XR, and the NX-OS Network Operating Systems as they look to modernize their network management and migrate from traditional network management tools. Its core features include YANG model browsing and exploring, as well as device management. On top, you can add many plugins such as NETCONF, gRPC and Diff to enrich its capabilities.

![English YANG translate_5.png]({{site.baseurl}}/images/English YANG translate_5.png){: .align-center}{:height="60%" width="60%"}

# What is a model?

A data model is simply a method to describe something using **keys**, a **type** and optionally a **description**. It could be a person, defined by:
- Height (cm)
- Weight (kg)
- Eye color (Blue, Green, Brown)
- Hair color (Brown, Blond, Black, Other)
- Nationality (French, German, Italian, ...)

In the above example, `Height` is a key and `cm` a type.

But it could define many more things! It could define a networking feature, such as BGP:
- Autonomous System (integer)
- Neighbors (list of IP addresses)
- Filters (list of prefix lists)

In the above example, `Autonomous System` is a key and `integer` is a type.

And it could also define a networking service configured on many nodes, such as a L3VPN:
- Name (string)
- Nodes (list of nodes)
- Address Families (list of address families)
- Filters (list of prefix lists)

To describe network features and services, we use YANG. It's a modeling langage to represent **data structures** in an **XML** tree format. Optionally, YANG can use **XPath** (XML Path Langage) expression to filter the elements of a YANG data model.

# Where do YANG models come from?

YANG models can come from two sources:
- **Standard definition** (ex: IETF, ITU, Openconfig),
- **Vendor definition** (ex: Cisco).

In the first case, the models are compliant with industry standard. The goal is to agree on models that can be **vastly** supported by **networking devices** (ex: Routers) and **services** (ex: [L2VPN](https://datatracker.ietf.org/doc/html/draft-ietf-bess-l2vpn-yang)).
In the second case, the goal is to support vendor specific features and operational data. It's often the case for **hardware specific features** such as QoS and ASIC operational data.

![01_17_38.jpg]({{site.baseurl}}/images/01_17_38.jpg){: .align-center}{:height="50%" width="50%"}


# Installing YANG Suite

First, let's install YANG Suite. Let's start by cloning the github repo where the code is stored.

<code>
git clone https://github.com/CiscoDevNet/yangsuite
</code>

To setup YANG Suite (generate a certificate, choose a login/password), you will need to run the below commands. It's only required for the **first** utilisation. 

<code>
cd yangsuite/docker/ 
./start_yang_suite.sh
</code>

Then, start YANG Suite by entering the below commands:

<code>
cd yangsuite/docker/ 
docker compose up
<code>

The **nginx** container (web server) redirects port 80 to port 8433 which is used to interface with the YANG Suite core. You can now connect to http://localhost or https://localhost:8443 to access YANG Suite.
  
You **cannot** use the username credential `admin` in the setup. It will trigger the error: `django.db.utils.IntegrityError: UNIQUE contraint failed: auth user.username`, when trying to start the Docker container.
{: .notice--warning}
  
You will need to install Docker in order to use YANG Suite. You can find more information on how to get Docker and how to install it [here](https://docs.docker.com/get-docker/).
{: .notice--info}

The complete documentation on how to install YANG Suite is available [here](https://github.com/CiscoDevNet/yangsuite).
{: .notice--info}
  
# Learning YANG models
  
Let's say your entire backbone is running **IOS XR 7.3.1**. You're trying to find a way to collect the serial number of all devices on your backbone. In this second section, we are goign to see how we can download all YANG models from a remote repository, find the ritht model to use in order to collect the serial number.
  
## Adding a new YANG repository
  
To add a new YANG repository, from which we can sync our YANG models, here are the steps to follow:
  1. Go to **Setup > YANG Files and repositories**.
  2. Click **New repository** in order to create a new local folder where we will add our YANG models.
  3. Give it a name.
  4. Click **Create repository**.
  
  ![Add remote repository.jpg]({{site.baseurl}}/images/Add remote repository.jpg){: .align-center}

## Cloning YANG models from a remote repository
  
In this first scenario, we are going to clone a remote repository. Today, we are going to use this remote repository: https://github.com/YangModels/yang. Feel free to use another one.
  
Did you know that all YANG models for all Cisco IOS for all versions are stored on [https://github.com/YangModels/yang](https://github.com/YangModels/yang) ?
{: .notice--info}
  
To clone a remote repository, follow the below steps:
  1. Click on **Git** to use it as our way to collect YANG models.
  2. Add the **Repository URL** (https://github.com/YangModels/yang), the **Git branch** (master) and the **directory** where the YANG models are stored (vendor/cisco/xr/731). You can nagivate in the repository and use the directory you like; for example if you need to collect YANG models for another IOS XR version.
  3. Click **Import YANG files** to start cloning the repository.
  4. Once done, you should see the downloaded YANG models on the left box.
  
![Add remote repository 2.jpg]({{site.baseurl}}/images/Add remote repository 2.jpg){: .align-center}

This could take a few minutes, depending on how many models are in the repository. For me, it took around 5 minutes.
  
## Cloning YANG models from a device
 
Alternatively, you can also clone YANG models directly from a device. When a client (your device) and a server (YANG Suite) initiate a NETCONF session, they exchange **Hello messages** listing the set of capabilities they support.
  
For example, below is an example of capabilities of a device running IOS XR 6.5.3.
  
<script src="https://gist.github.com/AntoineOrsoni/f401cb979a81b62798c8f022b8f064d6.js"></script>
  
You can get a similar output using CLI with the command: `ssh username@host -p 830 -s netconf` where 830 is the NETCONF port on the device.
{: .notice--info}

In the part, we are going to learn the supported YANG models from a device. Then we're going to download them in YANG Suite.
  
It could be interesting to download the YANG models from a device rather than from a repository. A folder in a repository would contain all the models supported for a given IOS version by all devices eligible to this IOS version. These devices would probably have different sensors and API, thus having different YANG capabilities. To be certain to only use models supported by a given device, it's often better to sync YANG models directly from the device itself.
{: .notice--info}

### Adding a device
  
First, we need to add a new device to YANG Suite. In order for everyone to be able to collect YANG models from a device, we will use the [IOS XR always-on sandbox on Cisco Devnet](https://devnetsandbox.cisco.com/RM/Diagram/Index/e83cfd31-ade3-4e15-91d6-3118b867a0dd?diagramType=Topology). Below the sandbox information. Feel free to use another device.

| Key               	| Value                    	|
|-------------------	|--------------------------	|
| IOS XRv 9000 host 	| sandbox-iosxr-1.cisco.com |
|     SSH Port      	|     22                 	|
|     Username      	|     admin                	|
|     Password      	|     C1sco12345           	|
  
To add a new device, follow the below steps:
  1. Go to **Setup > Device profiles**.
  2. Create a new device.
  3. Fill up the device general information (use the one above if you don't have one).
  4. Check **Skip SSH key validation for this device** if you want to skip this validation.
  5. NETCONF information will be filled automatically using the default port (830). Check the connectivity to make sure everything works as expected.
  6. You should get a similar output. The Devnet sandbox does not answer to ICMP messages. This test should fail. Make sure the NETCONF test is pass.
  7. You can now **Create Profile**.

  ![Add device profile.jpg]({{site.baseurl}}/images/Add device profile.jpg){: .align-center}

  You've successfully added a new device. 
  
### Adding a new YANG repository
  
To add a new YANG repository, from which we can sync our YANG models, here are the steps to follow:
  1. Go to **Setup > YANG Files and repositories**.
  2. Click **New repository** in order to create a new local folder where we will add our YANG models.
  3. Give it a name.
  4. Click **Create repository**.
  
  ![Add remote repository.jpg]({{site.baseurl}}/images/Add remote repository.jpg){: .align-center}

### Downloading YANG models from a device
  
To download YANG models from a device, follow the below steps:
  1. Make sure you have selected the right YANG module repository, where the YANG models will be stored.
  2. Select **NETCONF**.
  3. Select the device from which you would like to download YANG models.
  4. Optionally, you can check again **device connectivity**.
  5. **Get schema list** from the device. YANG Suite will send a NETCONF hello to the device. It will answer back with all its YANG capabilities.
  6. This box will be filled with all YANG models supported by the device.
  7. Click **select all** or click on the YANG models you would like to download from the device.
  8. Click **Download selected schema**.
  9. This box will be filled with all YANG models you've downloaded from the device.
  
![Download from device.jpg]({{site.baseurl}}/images/Download from device.jpg){: .align-center}
  
# Exploring YANG models

In this section, we are going to explore YANG models. Find the one we need and see what's inside.

## How to find the model you need?
  
Githut changed their search engine and the method I suggested before doesn't work anymore. Below is another way to do it.
{: .notice--warning}

First, you need to find the right model. Might sound easy when you would like to find the IP address of a given interface on a node running IOS XR 7.3.1. You could use the **ietf-interfaces** model. But what about something less straightforward like a serial number?
  
Cisco models are usually divided in two categories. **oper** (operational data) models and **cfg** (configuration data). On the first case, it will contain **oper** in the name. This indicates you will find operational data in this model like its status (shut/admin shut/no shut), type, name, speed and statistics. On the other case, it will contain **cfg** in the name. This model will store configuration information like its description, speed, ip address... You will be able to use this model to modify the configuration of a device.
{: .notice--info}
  
A good way to find the model you need is to look at the naming. That's might not always work. Optionnally, I use the `grep` feature in my shell. Here is how I use it:

```
grep -Rin --include="*.yang" "your_search_string" ./path/to/model/folder
```
  
After the `--include="` you can use a regex (for example `*oper*.yang`) to filter the search only on specific models (**oper** or **config**).
  
From the search results, the model that best fits our needs appears to be `Cisco-IOS-XR-sysadmin-sm.yang`. Let’s verify it’s the one we need.
  
## YANG module sets
  
Now that you know all your device capabilities and in which model you need, it's time to create your first **YANG model set**. By opposition to a repository that contains all the models supported by your device, a YANG module set is a smaller view of your device capabilities. For example, the environment information of your device; or the OSPF information.
  
To create a new YANG module set, follow the below steps:
  1. Go to **Setup > YANG module sets**.
  2. Create a **New YANG Set**.
  3. Give it a name. For example `XR 7.3.1 - Serial Number`.
  4. Select the assciated YANG file repository. It's the repository where you collected the available models (from the device or from Github). If you followed my naming, it should be `XR Docs 7.3.1`.
  5. Click **Create YANG set**
  
![Create module set.jpg]({{site.baseurl}}/images/Create module set.jpg){: .align-center}

Now that we have created our YANG module set, let's populate it. Follow the below steps:
  1. Make sure you have selected the right YANG set (the one we have just created) and YANG repository.
  2. Find your model in the list. You can type the model in the search bar. The list will be filtered automatically.
  3. Click **Add Selected**.
  4. Click **Locate and add missing dependencies**. It will look for dependencies for this model and add them to your YANG module sets.
  5. Click **Validate YANG modules in greater depth**. It will look down if the models that have been added also have dependencies.
  6. You're all set. The list on the left should be populated with your model and its dependencies.
  
![Populate YANG set.jpg]({{site.baseurl}}/images/Populate YANG set.jpg){: .align-center}

  
Your YANG model can be an augmentation of another model. Meaning that it could extend the capabilities of another model.
In this case, your YANG model will have the keyword `augment` such as:
  `augment "/a1:dynamic-template/a1:ip-subscribers/a1:ip-subscriber"` in the **Cisco-IOS-XR-ip-pfilter-subscriber-cfg.yang** model.
{: .notice--info}
  
It could also use references from another model. Very similar to when your import a module in Python so you don't have to write all the classes and methods.
In this case, your YANG model will have the keyword `import` such as:
  `import Cisco-IOS-XR-types` in the **Cisco-IOS-XR-ip-pfilter-subscriber-cfg.yang** model.
{: .notice--info}
  
# Checking node coverage: converting node's configuration into equivalent NETCONF filter
  
Another cool feature of YANG suite is **YANG coverage**. First, it allows you to understand which part of your device's configuration can be mapped to a NETCONF filter. It means that you can see which part of the CLI configuration can be configured using NETCONF.
  
That's not all! It also gives you the equivalent NETCONF filter to generate the configuration. Let's take an example. In order for eveyrone to be able to give it a try, we're going to use the [IOS XE always-on sandbox on Cisco Devnet](https://devnetsandbox.cisco.com/RM/Diagram/Index/7b4d4209-a17c-4bc3-9b38-f15184e53a94?diagramType=Topology). Below the sandbox information. Feel free to use another device.

| Key               	| Value                    	|
|-------------------	|--------------------------	|
| IOS XE host 	| sandbox-iosxe-latest-1.cisco.com |
|     SSH Port      	|     22                 	|
|     NETCONF Port      	|     830                 	|
|     Username      	|     developer                	|
|     Password      	|     C1sco12345           	|
  
  
There are two ways to check YANG coverage in YANG Suite:
  - You can download the entire running configuration from a device.
  - You can only use a partial configuration.
  
As of October 2021, the YANG coverage feature do **not** support IOS XR and NXOS.
{: .notice--warning}

On IOS XR, you can get the equivalent NETCONF filter for a given configuration by running the command `show run | xml`. It will print the equivalent configuration in XML format.
{: .notice--info}

In this example, we are going to download the full running configuration from a device. The device information should already be configured in YANG suite as we saw earlier in this tutorial. You can skip `step 1` if you don't want to sync-up from a device and paste a configuration. 
  
To check a configuration YANG coverage, follow the below steps:
  1. Select a device on which you would like to download the running configuration. It will take a few seconds for YANG suite to download the device's configuration.
  2. Once downloaded, the device configuration will appear here. Otherwise, you can paste your configuration here. The configuration can be partial: you can only paste a configuration for a specific feature (ex: BGP).
  3. Select the device OS (for our sandbox, we are using IOS XE).
  4. Select the device release (the sandbox is running IOS XE 17.3.1).
  5. Click on **Get model coverage**.
  6. The equivalent NETCONF filter for your device configuration will appear here.
  7. You can see which parts of the configuration have an equivalent NETCONF filter for a specific OS and release. Any missing coverage is highlighted in red.
  
![check YANG coverage.jpg]({{site.baseurl}}/images/check YANG coverage.jpg){: .align-center}

# Conclusion
  
In this article, we saw a portion of the power of YANG Suite capabilities:
  - Learning YANG models (from a device and from a remote repository),
  - Exploring YANG models,
  - How to find the model you need,
  - Checking node coverage.
  
In a coming article, we are going to see how we can leverage YANG Suite to build and send a NETCONF request to get operational data.

# Useful resources
  
- [YANG suite official repo](https://github.com/CiscoDevNet/yangsuite)
- [YANG suite homepage](https://github.com/CiscoDevNet/yangsuite)
- [YANG suite official documentation](https://pubhub.devnetcloud.com/media/yang-suite/docs/index.html)
