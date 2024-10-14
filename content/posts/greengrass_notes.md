---
title: My Notes while learning about Greengrass
author: Vasu
date: 2024-06-09T17:25:08-05:00
draft: false
---

# Greengrass Concepts Basics

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

## Greengrass Core Software 
The set of al greengrass software that you install on a core device. 
- **Nucleus**: The required component that provides the minimum functionality of the aws iot greengrass core software. The nucleus manages deployment, orchestration, and lifecycle management of other components. 
- **Optional components**: Configurable components

## Features of AWS IoT Greengrass

- Software distributions
	- Nucleus
	- AWS provided components
	- Greengrass development tools
	- AWS IoT SDK for IPC library
	- Stream manager SDK 
	
-  Cloud services
	- API
	- Console

## GG Core Software Features:
- Process data streams on local device and automatic export to AWS Cloud
- MQTT messaging between AWS IoT and components
- Support local publish and subcribe messaging between components
- Deploy and invoke components and lambda functions.
- Manage component lifecycle.
- Perform secure OTA updates of core software and custom components
- Provide secure, encrypted storage of local secrets and controlled access by components.
- Secure connections between devices and aws cloud with device auth

## My thoughts on Greengrass Components vs AWS IoT jobs

_At work, we are working on a remote OS update mechanism for our custom downstream ubuntu distro that updates atomically through a image-based update mechanism. The actual OS updates i.e downloading an update bundle and writing it to disk is managed the [RAUC](https://rauc.io/). These are my notes taken while evaluating whether to use Greengrass Components or IoT jobs to remotely trigger this agent, in hindsight's it's clear IoT jobs is the appropriate mechanism, but these notes depict how I got to that decision._

AWS IoT jobs is the right tool to use for IFTTT type operations. 
AWS IoT Job is akin to ansible playbooks.

A Greengrass Components is right tool to use
	a. To package your software, in it's supported formats.
	b. To deploy said software.
A Greengrass component is akin to docker-compose.

IoT jobs is the right tool to use to run a task(OS upgrades are tasks), as it has in-built mechanisms of state management that we would have to re-invent with a  Greengrass component. For now we expect state management of an OS upgrade as a simple 3 step process, but this state complexity might grow in the long run.

If we define, a Greengrass component in such a manner to run a task, we are not using Greengrass components for what it was intended to do. Which is alright, if we really have to pick one or the other. But as we have the ability to choose right tool for the job, we should use IoT jobs to run tasks.

From CI/CD pipelines or orchestrator perspective, it may seem like it simplification if we use only one specific mechanism to run tasks on a device, but from device lifecycle perspective, IoT jobs and Greengrass components are completely different things with different purposes.

IoT jobs will be recommended interface for customers to define and deploy one-off or even continuous tasks on the device. Greengrass components, will be recommended interface to deploy applications, and maintain it's lifecycle. This is how I look at this, I understand both mechanisms can be altered to either of the operations, but if that mechanism increases in complexity, the author as to be very aware of what they are doing.






