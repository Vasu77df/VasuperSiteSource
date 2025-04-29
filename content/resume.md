---
title: Resume
date: 2025-04-28
draft: false
---

# **Vasudevan Perumal** 
[vasu3797@gmail.com](mailto:vasu3797@gmail.com) | [www.linkedin.com/in/vasuper](http://www.linkedin.com/in/vasuper) | 773-707-1927 | [vasuper.net](http://www.vasuper.net)  

---
## **Experience**  
**System Development Engineer II | Amazon OpsTech & Robotics | Austin, TX | *Dec 2023 \- Present (1 yr 5 mos)***

* Led the development of the OS Bakery, a CI mechanism for OS builds, and a toolchain of dev-tools for local builds. Fully automating git merge to build, and publish of OS artifacts to device deployment.  
* Designed a framework of abstractions for OS customization in the Amazon context, making OS customization accessible to cloud-native engineers, reducing time to prototype for teams, from days to a couple of hours.  
* Spearheaded development of AmazonEdgeOS, first ever custom Linux distribution at Amazon Robotics. Security and reliability as core tenets, we implemented A/B Updates, TPM2 full disk encryption, etc.  
* Designed a virtual OS testing service enabling developers and integration testing frameworks, to spin up test instances, run tests, and teardown instances on-demand, built with libvirt, vagrant, and pytest.  
* Developed abstractions, for teams to leverage TPM2 root of trust for Cloud service and  Wifi EAP registration.

**System Development Engineer I | Amazon OpsTech & Robotics | Austin, TX |  *Jul 2021 \- Dec 2023 (2 yrs 6 mos)***

* Introduced a mechanism to build custom Ubuntu images at Amazon Robotics, enabling teams to “shift-left” their dev and testing process. Leading to a 200%  increase in iteration velocity in developing solutions.  
* Introduced Image-based Provisioning at Amazon FCs, reducing device provisioning time from 45 mins to 6 mins. Done through a custom boot initrd, pre-built OS images, with a in-house developed installer.  
* Developed dev-tools, and CI/CD mechanism for internal developer teams to package their code into debian packages, and automating the process from git merge to debian package build and publishing.  
* Helped build a mechanism to vend internal Debian Repos to 200K edge devices. Using AWS Cloudfront CDN.

**Web Systems Operations Intern | Americaneagle.com | Des Plaines, IL | *June 2020 \- Dec 2020 (7 mos)***

* Built a python app translating Cisco LoadBalancer, to A10 LoadBalancer configurations, for a migration initiative. The app automated a 2 month initiative to 2 weeks across our Colo DataCenters.  
* Developed a bash/powershell script to automate clearing RAID config of disks on CentOS 6 servers, reducing SysAdmin’s time to recycle drives  from 20 mins to 1 min.

---

## **Skills**
* **Programming Languages**: Python, Bash, Rust, Typescript  
* **Cloud Tech**: AWS Lambda, DynamoDB, API Gateway, Codebuild, Cloudformation, IoT, SSM, Greengrass, SNS, Kinesis, Cloudwatch, CodePipelines, ECS, CloudFront, KMS.  
* **Tools, and Platforms**:  systemd, mkosi, yocto, Containerization(Docker,Podman,systemd-nspawn), Flatpak, Debian/APT package management, btrfs, micro-repo management, Smithy API Framework, HashiCorp Packer and Vagrant, libvirt, qemu, MQTT SDKs for AWS IoT, and Greengrass.  
* **Technologies and Concepts:** Trusted Platform Module(TPM2), Remote Attestation,  Immutable Linux(A/B updates, OsTree), Linux Distribution Management(Ubuntu, Fedora, NixOS),  API Design and Data Modelling. 

---

## **Education**
* **Masters in Electrical and Computer Engineering | Illinois Tech | Chicago, IL | *Aug 2019 \- May 2021***  
    - **Coursework**: Advanced Computer Networking, Cybersecurity, Network Security, IoT, Wireless and 5G Networks
* **Bachelor in Electronics and Communication Engineering | SRM IST | Chennai, India | *Aug 2015 \- May 2019***
    - **Coursework**: Computer Networking, Communication Design, Mobile Networks, MicroProcessors, and MicroControllers

---
## **Activities**
* Made upstream [contributions](https://github.com/tpm2-software/tpm2-pkcs11/pull/883) to TPM2 TCG Linux project, fixing their PKCS11 teardown process, available now in all major linux distros.
* Open sourced internal projects, like [aws-ready-edge-os](https://github.com/Vasu77df/aws-ready-edge-os), a public fork of AmazonEdgeOS, [aws-mkosi-builder](https://github.com/Vasu77df/aws-mkosi-builder), an AWS native OS build Infra library. Ecrmedataextractor, an OCI compliant tool to gather SBOM data.  
* Made minor contributions to monitord, a Dbus based systemd monitoring tool, and mkosi(OS build tool).  
* Attended [All Systems Go 2024](https://all-systems-go.io/), foundational linux userspace conference. 
* Member of the pyTexas community.

---

## Publications
* Smartphone APP for Continuous Observation of Pollution Levels Due to Particulate Matter Measured by Laser Mie Scattering
    - Collaborated with 4 researchers in developing a mobile IoT pollution monitoring system that uses Mie Scattering Technique to measure particulate matter-based pollution to gauge living conditions of an area.
    - Publication link [here](https://link.springer.com/article/10.1007/s40030-020-00446-4#citeas)
* Effects of ambient air pollution on respiratory and eye illness in population living in Kodungaiyur, Chennai
    - Publication link [here](https://www.sciencedirect.com/science/article/abs/pii/S1352231019301050)