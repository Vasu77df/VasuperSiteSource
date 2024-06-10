---
title: "Notes on AWS Greengrass"
date: 2024-06-09
description: "These are my notes on AWS IoT Greengrass"
type: "post"
tags: ["aws"]
---

## Listing basic Greengrass Concepts

GG is a edge runtime with a deployment and management mechanism for the egde runtime

![gg_arc](/gg_post.png)

**AWS IoT thing**: A device or logic entity

**Greengrass core device**: Device running greengrass core software

**Greengrass client device**: A device that connects to and communicates with a greengrass core device over MQTT.

*client device* and *core device* could the same device or could be separated, with a sort of master slave relationship. client devices are meant for small footprint device that run embedded linux or RTOS maybe?

**Greengrass component**: A software module that is deployed on a Greengrass core device. All software developed and deployed through Greengrass is a modeled as a component. 
- **Recipe**: A JSON  and YAML that describes the software module.
- **Artifacts**: The source code, binaries, or scripts
- **Dependency**: The relationship between components that enable you to enfore automatic updates or restarts of dependent components. 

**Greengrass Core Software**: 
The set of al greengrass software that you install on a core device. 
- **Nucleus**: The required component that provides the minimum functionality of the aws iot greengrass core software. The nucleus manages deployment, orchestration, and lifecycle management of other components. 
- **Optional components**: Configurable components

**Features of AWS IoT Greengrass**:

- Software distributions
	- Nucleus
	- AWS provided components
	- Greengrass development tools
	- AWS IoT SDK for IPC library
	- Stream manager SDK 
	
-  Cloud services
	- API
	- Console

**GG Core Software Features**:
- Process data streams on local device and automatic export to AWS Cloud
- MQTT messaging between AWS IoT and components
- Support local publish and subcribe messaging between components
- Deploy and invoke components and lambda functions.
- Manage component lifecycle.
- Perform secure OTA updates of core software and custom components
- Provide secure, encrypted storage of local secrets and controlled access by components.
- Secure connections between devices and aws cloud with device auth

