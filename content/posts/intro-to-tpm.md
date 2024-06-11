---
title: "Intro to Tpm"
date: 2024-06-10T03:25:52Z
draft: false
type: post
showTableOfContents: true
---

# Intro to Trusted Platform Modules(TPM) Notes

---
These are my notes taken from learning about TPM from this source material: https://courses.cs.vt.edu/cs5204/fall10-kafura-BB/Papers/TPM/Intro-TPM-2.pdf 

These are just excerpts and notes, that I have captured for my recollection and understanding, from that this chapter.
---

A common design goal across this diversity however, is the provision of some degree of secure processing, implemented in secure hardware.

The Trusted Platform Module, or TPM, is similar to a smart card device in that it is a small footprint low cost security module typically implemented as a tamper resistant integrated circuit (IC). The TPM however, has been specifically designed to be a building block for trusted computing

One major difference is that the TPM is considered to be a fixed token bound to a specific platform, whereas a smart card is a portable token traditionally associated with a specific user across multiple systems

This is not to say that the two technologies are mutually exclusive, but rather that they may be considered as complementary.
## Trusted Platforms

In order to understand the TPM design requirements, it is first necessary to understand what the desirable features of a trusted platform are. To do this, a definition is required as to exactly what is meant by the term “trusted platform”.

TCG defines trust to be “the expectation that a device will behave in a particular manner for a specific pur-pose

Providing assurance of expected behaviour does not in itself provide any security. To achieve that, the entity relying on this assurance still has to ascertain that the “expected behavior” is indeed secure.

__Definitions of a trusted platform__:

- _A Trusted Platform, is a computing platform that has a trusted component, probably in the form of built-in hardware, which it uses to create a foundation of trust for software processes._

- _A Trusted Platform(TP) is defined as a computing platform that has a trusted component which is used to create a foundation of trust for software processes._

There is subtle distinction between trusted component, such as the TPM, and a trustworthy component.
- _The proper definition is that a trusted system or component is one whose failure can break the security policy, while a trustworthy system or component is one that won't fail._
# Fundamental Features of a Trusted Platform.

The minimum set of features that a trusted platform should have are: protected capabilities; integrity measurement; and integrity reporting. 

Providing support for these features leads to the definition of the security requirement of the TPM.
## Protected capabilities

The concept of protected capabilities is used to "distinguish platform capabilities that must be trustworthy". 

__These trustworthy protected capabilities are abilities to execute a command or set of commands on the TPM which access shielded locations where it is safe to operate on sensitive data.__ Examples of protected capabilities in the TPM include
protection and reporting of integrity measurements (described below); and storage and management of cryptographic keys.
## Integrity measurement and storage.

__Any trusted platform will need some means of measuring how it's configured and what processes are running on that platform.__

As a lot of software is executed on a system implicitly and not explicitly by the user it is hard to determine the legitimacy of process. For a platform to be trusted, that it has some means of measuring integrity of the process it is running This measurement should result in some form of _integrity metric_.  Which _Pearsons_ defines as "_a condensed value of integrity measurements_". 

The integrity metric can then be compared with acceptable values for a trusted store and value somewhere for later use. Of course such data would be held in secure storage locations such as would be provided by the protected capabilities described above.

Measuring platform integrity in this manner has to be a starting point. It is acceptable for a app to perform integrity measurements provided that the application itself is trustworthy. Even if the operating system can verify the integrity of the application, the integrity of the operating systemc has to be verified.

Ultimately there will be a some integrity measurement process that exists that cannot itself be verified. This process is known as the Root of Trust for measurements, or __RTM__. This is the starting point in the chain of integrity measurements, and ideally this process should run on tamper proof hardware, with execution code being stored in secure storage.
## Integrity reporting
__The third requirement for a trusted platform is that it should be able to report its configuration to a challenger who requires this information in order to decide how much trust to place in the platform.__

Platform should have means to report it's _integrity metrics_,  and to vouch for the accuracy of this information. 

The challenger requires the a evidence to reply on to make trust decisions.

The implication here is that the integrity metrics can be signed by the trusted platform and that the challenger has a certificate that can be used to verify the signature. 
## Additional Features
- Confidentiality and integrity protection
- Secure storage
- Process isolation

### Confidentiality and integrity protection

A trusted platform should provide both confidentiality and integrity protection to any data as required by the user. This should include but user data, and application code. These services should be available to protect data while it is being stored and during the execution of any process.
### Secure Storage
The provision of confidentiality requires that a trusted platform is able to encrypt
any user data for secure storage. Access to the data is controlled by a securely stored
cryptographic key. Similar to encryption is the concept of sealing. In this case access to the data is controlled by platform state, the use of a cryptographic key is
optional. This means that data can be sealed to a set of integrity metrics that reflect
the platform state. The data will then only be accessible when the platform is in the
desired configuration. The sealing mechanism can be used, for example, to ensure
that no rogue applications are running before access is granted to sensitive data.
### Process isolation
The secure storage and integrity checking will protect data during storage. To protect data during execution the provision of process isolation is necessary.

The concept here is to provide an isolation kernel between the hardware and any operating system.  __This isolation kernel is used to create and manage multiple secure compartments that can exist in parallel, on the same machine Each can run its own OS and application in isolation from any other processes that are executing in parallel. In this way, app and data can be protected during execution.__

Isolation kernel greatly simplifies, the validation of acceptable integrity metrics: by isolating processes, the set of acceptable application platform configuration can be reduced to one operating system and one application only.

## TPM Features

### TPM Components

TCG defines standards of TPM functionality, leaving it open for developers to impolement this functionality as they wish, either in hardware or software. 

TPM Standards do not specify the communications, interfaces or bus architecture, leaving these decisions to be made by the developer.
### I/O Block.

The TCG do, however, specify an interface serialization transformation that allows data to be transported over virtually any bus or interconnect. This is one of the functionality of the I/O Block. __This block manages information flow between the components, and between the TPM and the external bus. The control access rights to access TPM components are determined by flags maintained by the opt-in block.__

![io_block](/intro_to_tpm/io_block.png)

### Non-volatile storage

TPM has some NV memory to long term keys. two long term keys are stored in the NV memory. The first of these is the __Endorsement Key(EK)__. the second is the __Storage root Key(SRK)__, which forms the basis of a key hierarchy that manages secure storage. 

The TPM also uses NV mem to store _owner_ authorization data. This authorization data, in effect, is the owner's password, not set by manufacturer, butt the user during the process of taking ownership of the TPM. 
#### Endorsement Keys

It is a fundamental component of the TPM. For the TPM to operate, it must have an endorsement key pair, of which the private key is embedded in the TPM and __never leaves it__. The public EK is contained in a cert and is only used in a limited number of procedures.

The reason for limiting the use of the EK Cert is because the EK is unique to each TPM and consequently may be used to identify the device, and by extension the platform. 

Therefore, to protect user privacy when interacting with other entities, the use of the EK is restricted and internally generated aliases, the Attestation Identity Keys, or AIKs, are used for routine transactions.

TPM manufacturers will provide the endorsement key pair and store this in tam-
per resistant non-volatile memory before shipping the TPM. A certificate, or endorsement credential, can then be created which contains the public EK and information about the security properties of the TPM

This endorsement credential should be signed by a certification authority, known as the TPME or Trusted Platform Module Entity, who can attest to the fact that the key contained in the certificate is a public EK whose corresponding private EK is stored in a TPM that conforms to the TCG standards. This TPME may be a third party or, if authorised to do so, it may be the manufacturer themselves.

Some organisations who wish to use the TPM may, for security reasons, pre-
fer to use their own endorsement keys. To accommodate this, the standards allow the EK to be deleted and re-installed by the user. Of course if the user generated
endorsement credential is not signed by a TPME, its use may be limited.

The purpose of the endorsement credential is to prove that the corresponding private EK is stored in a genuine TPM. So, in keeping with policy to control exposure of the public EK, the private EK is never used to generate signatures. Thus, the public EK is never required to verify signatures so it does not have to be widely distributed. The public EK is only used for encrypting data sent to the TPM during the process of taking ownership and the process of creating AIK certificates. These processes are described in sections 7.3.12 and 7.4.4. Encrypting data with the public EK ensures that the plaintext can only be recovered by the particular TPM identified in the endorsement credential.
#### Attestation Identity Keys

Both the EK and Storage Root key(SRK), never leave the device. The third type of key, the __Attestation Identity Key(AIK), may also be stored within the TPM. This key may be regarded as an alias for the endorsement key.__

Each TPM can support many AIKs, thus the user can have may unlinkable keys that can be used to maintain anonymity between different service providers who require proof of identity. These AIKS must, therefore, be persistent. They could stored in the TPM NV mem, but for practical reasons the standards recommend keeping the AIK keys in secure external storage. 

The TPM however must provide a volatile storage area where one or more AIK keys can be loaded when in use.

#### Platform Configuration Register.

PCRs are unique feature of TPMs used to store integrity metrics. The metrics stored in these registers measure the integrity of any code, from BIOS to application, __typically before the code is executed__. 

PCR may be implemented n volatile or NV storage, however the registers must be reset, whenever the system loses power or re-starts. If not old metrics may remain for new configurations after platform re-boot. 

TPM must have at least 16 Platform Configuration Registers and that each register stores 20 bytes. Registers 0 to 7 are reserved for exclusive use by the TPM, the remaining registers are free for use by the operating system and any application.

#### Programme Code 

In common with smart cards, the TPM requires storage for the firmware that is used to initialise the device. If the programme code is stored permanently on the tamper proof TPM then it would be reasonable to assume that it is trustworthy. Thus there would be no need to check its integrity making this the obvious location to store the code that carries out the integrity checks on all other platform devices and code. That is to say, the programme code on the TPM is the obvious “root of trust” for integrity measurements described in section 7.2. The TCG refer to such a root of trust as the CRTM, or Core Root of Trust for Measurement. Although the TPM programme code is the obvious choice for the CRTM, implementation decisions often require the CRTM be located in other firmware such as the BIOS boot block. Regardless of where the CRTM resides it should be considered as a trusted component of the system since if it fails all security policy based on integrity measurements will be broken.
#### Execution Engine.

The TPM has an execution engine which runs the programme code described above. The execution engine responds to external commands by se lecting the required programme code and executing it on the TPM.

#### Random Number Generator

TPM includes a _true_ random bit stream generator. Implementation of this is left to the developers so long the random source is used, rather than a deterministic method. Random bit streams are used to see a random number generator. The random numbers produced can be used to construct keys for symmetric cryptographic applications. The random numbers may also be used to provide nonces and, by mixing with user input, to increase the entropy in pass phrases.

#### SHA-1 Engine

The SHA-1 message digest engine is an implementation of the Secure Hash Algorithm [6] SHA-1. This algorithm hashes the input data and produces a 20-byte digest. It also forms the basis of an HMAC [7, 8] (Hash Based Message Authentication Code) engine, and is used in a number of cryptographic procedures carried out by the TPM, __for example: in the computation of digital signatures and for creating key objects where a hash of the key may be required as part of an integrity protection mechanism.__
#### RSA Key Generation

Generating keys suitable for use with the RSA algorithm [11] can be a computa-tionally intensive task and since such keys are widely used in the TPM, for signing and providing secure storage

All implementations of the TPM are required to support up to 2048 bit RSA, the Standard specifies that the TPM must be able to support upto 2048 bit modulus.
#### RSA Engine

Just as the generation of RSA keys is computationally complex, so is the execution of the algorithm itself. Therefore, the standards also require the TPM to have a ded- icated RSA engine used to execute the RSA algorithm. The RSA algorithm is used for signing, encryption, and decryption
#### Opt-In

The opt-in component and the concept of ownership, represents on of the biggest differences between smart cards and the TPM. Smart cards are usually owner byt the Issuer before the consumer recieves the device. TCG, is conscious of the perception that the TPM enabled platforms will somehow be controlled  by large remote organisations, have been very careful to provide mechanisms to ensure that it is the _user_ who takes ownership and configures the TPM.

During the process of taking ownership, the TPM will make transitions through a number of states depending on the initial state in which the device was shipped. The state in which the TPM exists is determined by a number of persistent and volatile flags. Changing the state of these flags requires authorization by the TPM owner, if he exists, or demonstration of physical presence
##### Tpm operational States

There are several mutually exclusive states in which the TPM can exist from _disabled_ and _deactivated_, through to _fully enabled_ and _ready for an owner_ to _take possession_. 

For a TPM to be owned by a user, it must have an endorsement key pair, and a secret owner authorisation data known by the owner. Once in the owned state, the owner of the TPM may perform all operations including operational state change. The TPM needs to have an owner and be enabled for all functions to be available
##### Taking ownership

When a user takes ownership of a TPM they establish a shared secret, referred to as _owner authorization data_, and insert this data into the secure storage on the TPM. This is in effect, the owner's password. and being to demonstrate knowledge of this secret provides proof of ownership. 

The process of taking ownership requires that the owner authorization data is protected from eavesdropping or theft by a malicious third party. This is one case where the endorsement cred is used.

It is the EK that establishes the secure channel to transmit the authorization data to a genuine TPM. So, during the process of taking ownership. So, during the process of taking ownership, the owner requests the endorsement credential and after verifying this credential, retrieves the public EK and uses this to encrypt the shared secret. Only the TPM identified in the endorsement credential has access to the provide EK, therefor the shared secret is only made available to the intended TPM. 
# TPM Services

The foundation for the provision of these services is the concept of a "root of trust" from which other services such as authenticated boot, secure storage, and attestation can be constructed.

## Roots of Trust

In the trusted platform architecture by the TCG, there are three distinct roots of trust:
- a Root of Trust for Measurement(RTM)
- a Root of Trust for Storage(RTS)
- a Root of Trust for Reporting(RTR)

RTM must be trusted to generate integrity measurement of the processes that are running on the platform. This should ideally be a tamper proof component that boots very early in the boot process and therefor is able to measure all other components that are loaded after it. The TPM is an ideal candidate for a __CRTM__ _Core Root of trust measurement_

> CRTM: The first piece of BIOS code that executes on the main processor during the boot process. On a system with a Trusted Platform Module the CRTM is implicitly trusted to bootstrap the process of building a measurement chain for subsequent attestation of other firmware and software that is executed on the computer system.

RTS is the component that provides confidentiality and integrity protection. The RTS can then trusted to store either data, e.g PCR, or keys such as the SRK that allow data to be securely stored in external locations.

Finally, the Root of Trust for Reporting is a trusted component that provides reports of any integrity measurements that may have been made - attesting to the platform configuration. 

## Boot process

![boot_process](/intro_to_tpm/tpm_boot_process.png)

TBB: Trusted Boot block

Not illustrated in the diagram, the CRTM should measure the rest of the BIOS before loading it. Once the BIOS is loaded it takes control and measure the integrity of the OS loader as shown in step 1. Control then passes to OS loader in step 2, and the OS loader then measure the integrity of the operating system. This process continues until the applications are loaded and executed in steps 6.

The integrity measurement at each stage are made by creating a SHA-1 digest of the code to be loaded. This digest is stored in one of the PCR registers, which are initialized to zero. The new integrity metric however does not simply overwrite the old PCR value. The process of updating(or extending) the PCR value concatenates the 20 bytes of data already held in the PCR with the 20 bytes of the new data calculated These 40 bytes of data are then hashed again using the SHA-1 algo and the result written to the original PCR. In pseudo code: PCR <- has(PCR||hash(new code)). This way the PCR  can store an unlimited number of measurements.

In order to interpret the value contained in the PCR, it is necessary to know the individual digests that have been added to it. These data are stored externally in what the TCG refer to as the Stored Measurement Log. Thus, if the data in the stored measurement log is known, and the PCR values are known, and trusted, then a challenger can verify the state of the platform.

## Secure Storage

In _Taking ownership_ section above the process of taking ownership resulted in the creation of a Storage Root Key, or SRK. This key is generated by the TPM and never leaves the device. It can only be accessed by demonstrating knowledge of a shared secret, in this case the SRK authorization data. This shared secret is similar to the _owner authorization data_ and loaded into the TPM at the same time,  during the process of taking ownership

![secure_storage](/intro_to_tpm/rts.png)

The SRK forms the root of a key hierarchy as illustrated in figure 7.3 which has also been taken from the TCG Architecture Overview [16]. This key hierarchy allows data, or keys, to be encrypted such that they can only be decrypted by accessing the TPM

The TPM provides two mechanisms for secure storage: binding and sealing. The binding operation encrypts the data using a key that is managed by a particular TPM as described above. The sealing process adds to this by only allowing the deciphering process to proceed if the platform is in a specific configuration. This configuration is determined by data held in the PCR registers. Thus, when data is sealed, not only must the same platform be used to unseal the data, but that platform must be in a predetermined configuration before the data can be recovered.
## Attestation

The AIK keys shown in the figure above are the __Attestation Identity Keys__. The purpose of the AIK is to provide users privacy when communicating with different sources. Although EK could be used to secure communication with different sources, since the EK is unique to the TPM,  this could potentially allow the platform's identity to be linked between each source it chose to communicate with.

The idea of the AIK is to provide a unique unlinkable identity for the TPM, for use with each different source. In each case the AIK acts as an alias for the EK. The AIK credential is a certificate containing the AIK public key which proves that the corresponding private key is bound to a genuine TPM. This proof is guaranteed by a signature on the credential created by a trusted third party known as a “privacy CA”.

To obtain an AIK a request is sent to the privacy CA together with the endorse- ment credential. This the second case where the public EK is exposed. The endorse-  ment credential proves to the privacy CA that the request came from a genuine TPM. In response to the request, the privacy CA creates and signs the AIK credential, and encrypts this under the public EK contained in the endorsement credential. Thus the AIK is cryptographically bound to the TPM that contains the private EK. The users can create as many AIK keys as they wish and provided the privacy CA is trust- worthy, these keys will remain unlinkable and provide the privacy required when communicating with different sources. The private AIK key is managed by the TPM as illustrated in 7.3 and may be freely used to generate signatures. In particular, the private AIK can sign the con- tents of the PCR registers. This is how the TPM can be used to attest to the platform configuration. A relying party in possession of a public AIK can now challenge the trusted platform, and the attestation mechanism can be used to provide an integrity report describing the platform state.

