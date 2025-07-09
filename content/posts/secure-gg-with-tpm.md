---
title: My Tutorial to Securing AWS Greengrass with TPM2 is live!
date: 2025-07-09
description: x509 certs secured with through the TPM2 chip so that no-one can steal it.
tags: ["TPM", "Trusted Compute", "systems", "AWS IoT", "AWS Greengrass"]
draft: false
---

A tutorial I authored is finally available in the official AWS documentation!!

- https://docs.aws.amazon.com/greengrass/v2/developerguide/gg-with-tpm-tutorial.html

---

**What is it about?**

The tutorial shows process of stemming your AWS IoT Thing/Device's Certificates for your AWS Greengrass Nucleus against the Trusted Platform module chip available on a device that supports one.

---

**Why would one do this?**

Let's take this scenario, your business operates a fleet of [Cloud native Endpoints](https://www.truesec.com/service/cloud-native-endpoints) (basically computer/devices that talk to the cloud).  This could be devices like point of sale systems, embedded devices, industrial PCs, any computer you probably see out in the wild that isn't a personal computer.

A common mechanism such devices use to authenticate to a cloud control plane is a public key infrastructure standard called [x509](https://datatracker.ietf.org/wg/pkix/about/). 

This entails the process of creating a [public key](https://www.cloudflare.com/learning/ssl/how-does-public-key-encryption-work/), a [private key](https://www.coinbase.com/learn/crypto-basics/what-is-a-private-key), a [certificate signing request](https://www.globalsign.com/en/blog/what-is-a-certificate-signing-request-csr), and finally a x509 [certificate](https://www.fortinet.com/resources/cyberglossary/digital-certificates) signed by the cloud control plane that is authorized to be used.

This certificate is now credentials for the device to talk to the cloud control plane.

If a malicious actor(hacker!!!) get's access to our device, remotely, locally, any which way. They can now steal the certificate off the device. With this our hacker now has access to our cloud control plane. While we might have role based access control for our cloud services, that might not be enough to prevent attacks or start a DDOS, and still a person we really don't want has access to something they shouldn't have.

Now a days you can get devices with a module/chip called the Trusted Platform Module(TPM)(yes, this is the chip that was made mandatory to run Windows 11). This chip is dedicated to run cryptographic operations and can create keys, which is embedded to the motherboard, and acts as a root of trust. 

In the tutorial above, I show how we can root a private and public key from the TPM chip, and then store a certificate in the TPM for safekeeping. Now our hacker even if he has access to our device can't steal credentials off the device, and even if he can, he can't use it unless he has that exact TPM chip. TPM does not allow keys to be extracted off the chip.

If he steals the computer and has time to de-solder the chip and put it on a new computer, then you have physical security problem, I can't help you with that.

---

**What's the tech behind this?**

While this tutorial is geared towards AWS IoT Greengrass (as I work there), steps 1-7 can be use for any public private key auth infrastructure for example WiFi that is authenticated with [EAP-TLS](https://github.com/tpm2-software/tpm2-pkcs11/blob/master/docs/EAP-TLS.md). The same instructions can be found in this link as well
- https://github.com/tpm2-software/tpm2-pkcs11/blob/master/docs/EAP-TLS.md

The tutorial uses the tools provided by the [tpm2-software](https://github.com/tpm2-software) project. We use the the PKCS11 interface, and the `tpm2_ptool` from the [tpm2-pkcs11](https://github.com/tpm2-software/tpm2-pkcs11). You can compile these from source, or it's available in most mainstream Linux distributions. In the tutorial I use Ubuntu 24.04.

**Should I know something before implementation this for my edge compute fleet?**

The TPM chip is not a storage device, it cannot store infinite keys. I discovered, that the TPM can at-most contain 7 key material.

This means if you re-provision your device and re-bootstrap your credentials you can't do this more than 7 times. 

Fear not, I submitted a fix to the `tpm2_ptool`, that fixes the destroy command that will clear any keys on the TPM2 chip.
- https://github.com/tpm2-software/tpm2-pkcs11/pull/883

---

Feed the tutorial to the nearest LLM, with these fixes, to create scripts that can automate the bootstrap and teardown process.

Now you should have a secure endpoint that can also be re-provisioned with no limitations.

Happy hacking!!
