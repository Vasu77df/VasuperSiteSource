---
title: "Leveraging Container Registry APIs for your benefit."
date: 2024-11-27
tags: ["docker", "containers", "SBOM", "OS", "devops"]
author: "Vasuper"
draft: false
---

If you have ever run `docker pull` or pulled a container using the favorite container runtime tool of your choice, you've probably interacted with a container registry. 

A container registry does not only provide mechanisms to store and distribute container images, it also exposes APIs that clients can use to perform other actions like fetching the container's metadata without pulling the entire container.

At work, I had a problem to solve, which is when I thought why not leverage these APIs to solve my problem. In this post we'll see how I leveraged the Container Registry APIs for my benefit and possibly how you the reader can use it as well.

---

## 1. The problem.
We run build jobs of OS images meant for bare-metal edge devices, at work. We package the build job and the OS definitions as a container image and execute them in AWS CodeBuild.

Our eventual aim is ensure our builds are reproducible. We have made some progress through a versioned build environment i.e the container image, and through making snapshot-ted copies of the upstream package mirrors and locking each OS version to use only the same snapshot. 

This alone does not guarantee reproducible builds. There are some deeper implications and problems to solve to achieve deterministic reproducible builds. I recommend checking out the Reproducible Builds Orgs website to learn more about the problems and their commandments that every software package has to follow to truly achieve reproducibility.
- https://reproducible-builds.org/
	- https://reproducible-builds.org/docs/commandments/

In the meantime, we want to bridge this gap as best as we can, by collecting the SBOM(Software Bill of Materials) in how many vectors as possible for each OS build. If we can ensure reproducibility, we want to provide traceability for all the components involved in the build. For the OS in itself, we capture the apt package manifest(we build Ubuntu based images) and publish it along with the OS image to our internal artifact registry. 

A data we want to collect is the metadata of the container image that executed the build job, and publish those details as well.

I set some requirements for myself for mechanism that collects this data.
- I want to capture the details without affecting the OS build times.
- I want to capture the details without requiring to setup an out of band mechanism that peers into the Build job.
- Something that is simple and maintainable that can run just before executing the actual build.
- Avoid having to install the docker client or any other client to gather metadata if possible.

With these in-place, my first thought was, why not checkout the registry APIs and see what methods that exposes?

----
## 2. OCI Specifications.

Contrary to some folk's assumption Docker is not the only container platform out there.  But Docker was the founder of the Specifications body that defines the open source specifications for Containers. Most other container platforms implement this specification, like [podman](https://podman.io/) and, [kata containers](https://katacontainers.io/). The [Open Container Initiative](https://opencontainers.org/) (OCI) founder by Docker is a Linux Foundation project who's purpose is to define open industry standards around container formats and run-times.

OCI contains three specifications:
- [Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/main/spec.md)
	- Defines the configuration, execution environment, and life-cycle of a container.
- [Image Specification](https://github.com/opencontainers/image-spec/blob/main/spec.md)
	- Specifications for tools that build, transport and prepare container images to run.
- [Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)
	- Defines an API protocol to facilitate and standardize the distribution of content that follow the image specification.

For the problem I want to solve, I care about the Image and Distribution Specification.

In this post, we will not delve into the details of each field or method the specifications provide, you can dive deep following the links above. I will provide a high level overview on certain items that were pertinent to the problem I was solving.
### 2.1 Image specification.
Image specification defines 5 components that I care about, and are also fundamental to an OCI defined container image. They are the
- Image Manifest
	- It contains metadata about the contents and dependencies of the image.
- Image index
	- A higher-level manifest which points to a list of manifests.
- Image Configuration
	- This includes information like the application arguments, environment config, and defines the layer ordering for the final runtime bundle.
- File-System Layers: This is the container filesystems defined as a [changeset](https://github.com/changesets/changesets) for the image.
- Descriptor: a reference that describes the type, metadata and the content address of the referenced content.
Below are diagrams with a high level grouping of how they come together:

![oci_image_spec](/oci_image_spec.png)
![oci_image_downloader](/oci_downloader_example.png)
*Source*: https://specs.opencontainers.org/image-spec/?v=v1.0.1#overview

For my purposes, I would like to capture the Image manifest and the Image configuration for the Build Container running my OS build. 
### 2.2 Distribution specification.

The Distribution specification provides API specs for the API methods, request, response, and error objects, for defining an HTTP based registry used to distribute container content defined from the Image specification we saw above.

The distribution specification defines specification various use cases that you might have used while interacting with a container registry, like
- Content verification through hash digests
- Resumable Push
- Resumable pull
- Layer upload de-duplication.

The OCI distribution  specification is based on the already existing [Docker HTTP API V2 Protocol](https://distribution.github.io/distribution/spec/api/).

To solve my problem, of pulling the container metadata, all I require is methods from the V2 protocol that let me pull the image manifest and configuration. 

The `GET /v2/<name>/manifests/<references>` path provides the call to get the image manifest.
- https://distribution.github.io/distribution/spec/api/#pulling-an-image-manifest

This method will return the Image manifest that contains the Image configuration and file system layers information. 

The image configurations will contain a digest, this is a unique identifier of Descriptor that let's us  identify the location of the Image configuration that we can pull, to capture the full Image configuration.

The image configuration is just another layer from the registries perspective, so we can make this API call to get the Image configuration, from the digest we capture for the config from the image manifest.
```
GET /v2/<name>/blobs/<digest>
```

- https://distribution.github.io/distribution/spec/api/#pulling-a-layer

Now, I have a mechanism to gather the metadata of the container without actually pulling the container image file system layers.

We use AWS ECR as our Container Registry provider, at work, and AWS ECR supports the Docker Registry HTTP API.
- https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html#registry_auth_http

Now it's as simple as making these calls and capturing the details during OS build.

---
## 2. ecrimagemetadataextractor

To do so I setup a very simple python CLI tool that I can invoke adhoc or within my CodeBuild project running the OS build job. 
- https://github.com/Vasu77df/ecrimagemetadataextractor/tree/main

You can get the wheel of this package from my release 
- https://github.com/Vasu77df/ecrimagemetadataextractor/releases

Or you can use the uv package manager to install, to learn more about uv checkout my post [here](https://www.vasuper.net/posts/using_uv/)

With this simple CLI tool I can run this before my actually task , and not really affect the task's run time in a meaningful manner

**Usage**:
```terminal
usage: ecrimagemetadataextractor [-h] [-v] -u IMAGE_URI [-r REGION] {get_manifest,get_digest_metadata}

Simple CLI tool to extract the image manifest from private AWS ECR hosted container images

positional arguments:
  {get_manifest,get_digest_metadata}
                        Actions possible from the CLI, get_manifest: returns json manifest of container image,
                        get_digest_metadata: returns json of first digest's manifest

options:
  -h, --help            show this help message and exit
  -v, --verbose         Verbose logging (default: False)
  -u IMAGE_URI, --image-uri IMAGE_URI
                        uri of the container image in your registry example: 772738948692.dkr.ecr.us-
                        east-1.amazonaws.com/os_build_env:latest (default: None)
  -r REGION, --region REGION
                        aws region to use (default: None)
```

**Get Image Manifest**

*Command*:
```bash
ecrimagemetadataextractor get_manifest --image-uri 772738948692.dkr.ecr.us-east-1.amazonaws.com/os_build_env:latest --region us-east-1 | jq . 
```

**Output**:
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 4552,
    "digest": "sha256:8a4a4313bd7cff420330aaf52e49da247b4c6107f44211d92db576e49b3f9659"
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 30610919,
      "digest": "sha256:802008e7f7617aa11266de164e757a6c8d7bb57ed4c972cf7e9f519dd0a21708"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 238910491,
      "digest": "sha256:6feaffcff6b9af7f741dfbc129b29e460ec4a1030c3c7684bc8ba15434982fcf"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 237477613,
      "digest": "sha256:6dbf3486e2803e0e2d67068727038e05da509de9335ca7cdc7664b7833da48ad"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 67232420,
      "digest": "sha256:e725b91ef7f278dd32339da93308782ab1af918204cf4d27082b374a089b159c"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 780,
      "digest": "sha256:b3f5b9a15490c9c075140c809857f4724263b6637a9466ee8708879cee5331e9"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 28832838,
      "digest": "sha256:f0eda69f6707fb23e5631437de1204664aad856f581270ba45ab5dadd0719e01"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 6235879,
      "digest": "sha256:c0cf9d8d47897c5a800086b3d4d80e28cc1653046b6795bccab4a9351025530a"
    }
  ]
}
```

**Get image Configuration Metadata**

*Command*:
```shell
ecrimagemetadataextractor get_digest_metadata --image-uri 772738948692.dkr.ecr.us-east-1.amazonaws.com/os_build_env:latest --region us-east-1 | jq .
```

```json
{
  "architecture": "amd64",
  "config": {
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/bash"
    ],
    "Labels": {
      "maintainer": "@vasuper",
      "org.opencontainers.image.ref.name": "ubuntu",
      "org.opencontainers.image.version": "24.04"
    }
  },
  "created": "2024-10-13T10:35:38.524849246-05:00",
  "history": [
    {
      "created": "2024-10-11T03:48:01.529767151Z",
      "created_by": "/bin/sh -c #(nop)  ARG RELEASE",
      "empty_layer": true
    },
    {
      "created": "2024-10-11T03:48:01.571862048Z",
      "created_by": "/bin/sh -c #(nop)  ARG LAUNCHPAD_BUILD_ARCH",
      "empty_layer": true
    },
    {
      "created": "2024-10-11T03:48:01.607507065Z",
      "created_by": "/bin/sh -c #(nop)  LABEL org.opencontainers.image.ref.name=ubuntu",
      "empty_layer": true
    },
    {
      "created": "2024-10-11T03:48:01.642491381Z",
      "created_by": "/bin/sh -c #(nop)  LABEL org.opencontainers.image.version=24.04",
      "empty_layer": true
    },
    {
      "created": "2024-10-11T03:48:03.777394067Z",
      "created_by": "/bin/sh -c #(nop) ADD file:34dc4f3ab7a694ecde47ff7a610be18591834c45f1d7251813267798412604e5 in / "
    },
    {
      "created": "2024-10-11T03:48:04.086892655Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/bash\"]",
      "empty_layer": true
    },
    {
      "created": "2024-10-13T10:33:50.683844544-05:00",
      "created_by": "LABEL maintainer=@vasuper",
      "comment": "buildkit.dockerfile.v0",
      "empty_layer": true
    },
    {
      "created": "2024-10-13T10:33:50.683844544-05:00",
      "created_by": "ARG DEBIAN_FRONTEND=noninteractive",
      "comment": "buildkit.dockerfile.v0",
      "empty_layer": true
    },
    {
      "created": "2024-10-13T10:33:50.683844544-05:00",
      "created_by": "RUN |1 DEBIAN_FRONTEND=noninteractive /bin/sh -c apt update -y && apt upgrade -y && apt install -y     aptly     bubblewrap     ca-certificates     cpio     curl     debian-archive-keyring     dosfstools     e2fsprogs      git     kmod     procps     python3-cryptography     python3-pip     python3-setuptools     python3-venv     python3-wheel     rauc     sbsigntool     squashfs-tools     systemd     systemd-boot     systemd-repart     systemd-ukify     systemd-container     mtools     tpm2-tools     ubuntu-keyring     unzip     zstd # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2024-10-13T10:34:17.339702189-05:00",
      "created_by": "RUN |1 DEBIAN_FRONTEND=noninteractive /bin/sh -c apt install -y libc6 groff less     && curl \"https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip\" -o \"awscliv2.zip\"     && ls -alh     && unzip ./awscliv2.zip     && ./aws/install # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2024-10-13T10:34:25.791326946-05:00",
      "created_by": "RUN |1 DEBIAN_FRONTEND=noninteractive /bin/sh -c curl -o /tmp/amazon-ssm-agent.deb https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_amd64/amazon-ssm-agent.deb     && dpkg -i /tmp/amazon-ssm-agent.deb     && curl -o /etc/amazon/ssm/amazon-ssm-agent.json https://raw.githubusercontent.com/aws/aws-codebuild-docker-images/master/ubuntu/standard/5.0/amazon-ssm-agent.json # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2024-10-13T10:34:26.437300051-05:00",
      "created_by": "RUN |1 DEBIAN_FRONTEND=noninteractive /bin/sh -c rm --force --recursive /var/lib/apt/lists/*     && rm --force --recursive /usr/share/doc     && rm --force --recursive /usr/share/man     && apt clean # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2024-10-13T10:34:32.860798461-05:00",
      "created_by": "RUN |1 DEBIAN_FRONTEND=noninteractive /bin/sh -c rm --force /usr/lib/python3.12/EXTERNALLY-MANAGED &&     apt-get update &&     apt-get install -y python3-pip &&     pip install pefile # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2024-10-13T10:35:38.524849246-05:00",
      "created_by": "RUN |1 DEBIAN_FRONTEND=noninteractive /bin/sh -c python3 -m venv mkosivenv     && mkosivenv/bin/pip install git+https://github.com/systemd/mkosi.git@v24.3     && mkosivenv/bin/mkosi --version # buildkit",
      "comment": "buildkit.dockerfile.v0"
    }
  ],
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:a46a5fb872b554648d9d0262f302b2c1ded46eeb1ef4dc727ecc5274605937af",
      "sha256:99423e0f187f5cdb62e47051984634a6d11e157d3127ae506562857f80493928",
      "sha256:aea99ec45e93114c710f03e635dfafd7320f429740cd5087822bc98365882429", "sha256:fc1645534d5bec025b2a6fc13483396cd20184edcfe06580cc86579d0b2067cd",
      "sha256:048f3fe585e6ebf6a57286b71bed90acc19fe8fef1410e898ccde7a42735fba8",
      "sha256:47ea983fc16a48b103cf6faf2bc97ecca8463052e570651c5690c1dbfab0539c",
      "sha256:76a0f1399709e7d3f747cd0380359338142de988544f5dea6298976afca4333b"
    ]
  }
}
```

`ecrimagemetadataextractor` currently only supports private ECR registries, and uses the IAM credentials in the environment it's running and Basic Auth to gain access to the ECR registry provided. 

For AWS CodeBuild run tasks it picks the image from `CODEBUILD_BUILD_IMAGE` environment variable present in every Containerized Codebuild task.

Make sure you have the necessary ECR IAM credentials to pull layers and get images from your ECR.

In the future I might make this tool generic so that it can work with any public or private container registries, feel free to put in a pull request or fork this, if you are interested in extending this.

Hopefully you learnt about the first few layers of the onions behind  a `docker pull` command and also how we could leverage some of the specifications, to get the SBOM of a container without needing to pull the entire image down or requiring the docker daemon or any other client.