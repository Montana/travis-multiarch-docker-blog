---
title: "Using Multi-Arch Docker Images to a Single Repository with Docker Manifest"
created_at: Fri Sep 17 2021 15:00:00 EDT
author: Montana Mendy
layout: post
permalink: 2021-09-17-docker-multiarch
category: news
excerpt_separator: <!-- more --> 
tags:
  - news
  - feature
  - infrastructure
  - community
---
![TCI-Graphics for AdsBlogs (4)](https://user-images.githubusercontent.com/20936398/133836264-4fcc56cd-7929-4157-a0d0-b0803b8646b2.png)

We all love Docker and is on the toolbelt of many builders out there. Docker allows you to have isolated containers with speicifc dependencies, so the line "I don't know it works on my machine though" is now unspoken, but what happens if you pull down an image that was intended for an `x64` based architecture while on an `Arm32` device like a Raspberry Pi or Arduino? (it won't work.) Now wouldn't it be glorious if the repository spotted this and routed you to the correct Docker image for your architecture based on your host OS? Well let's explore Docker Manifests. 

<!-- more --> 

## We sometimes need multi-arch images

You may have at some point while using Docker intentionally or unintentionally attempted to start a container from a Docker image intended for a foreign architecture, that you did not want to pull. 

With multi-arch images, we can quickly see what works and what doesn't. To test this, it's my personal opinion, when pushing for manifests you should always commit with `-asm` so you're signing off on it as well, instead of the regular `git commit -m "whatever"` you'd run `git commit -asm "whatever"`. In this particular example I put together, I grabbed from the DockerHub the following packages:

```bash
lucashalbert/curl
ppc64le/node
s390x/python  
ibmjava:jre 
```

These are the perfect packages (cURL), (ppc64le), (s390x), (ibmjava:jre), to show a multia-rch Docker image using ppc64le, s390x, and it's manifests.

## Why do I need to use manifest

Any registry or runtime that claims to have a certain Docker distribution image specification (this can easily be checked) support will be interacting with the various manifest types to find out the following things inside a image:

What are the actual filesystem content, (layers) will be needed to build the root filesystem for the container.

Any specific image config that is necessary to know how to run a container, some are more niche than others for using certain images. For example, information like what command(s) to run when starting the container (as probably represented in the Dockerfile that was used to build the image).

In short, The Docker container manifest is a file that contains data about a container image. Specifically `digest`, `sha256`, and most importantly arch. We can create a manifest which points to images for different architectures so that when using the image on a particular architecture Docker automatically pulls the desired image.

A good visual of this, would be: 

![image](https://user-images.githubusercontent.com/20936398/133838031-a6e9a037-8c05-4518-83bf-924022ac4b8e.png)

## Using Travis whilst using Docker Manifest

The following is the `.travis.yml` file in the repo which also uses travis.sh bash script I wrote to complete some of the env var tasks, while pushing to the custom `.travis.yml` file I made for this use case.

```yaml
---
language: shell # Montana Mendy also recommends generic, as an option here
sudo: required
dist: xenial
os: linux

services:
  - docker

addons:
  apt:
    packages:
      - docker-ce

env:
  - DEPLOY=false repo=ibmjava:jre docker_archs="amd64 ppc64le s390x"

install:
  - docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

before_script:
  - export ver=$(curl -s "https://pkgs.alpinelinux.org/package/edge/main/x86_64/curl" | grep -A3 Version | grep href | sed 's/<[^>]*>//g' | tr -d " ")
  - export build_date=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  - export vcs_ref=$(git rev-parse --short HEAD)

  # Montana's crucial workaround
  
script:
  - chmod u+x ./travis.sh
  - chmod u+x /build.sh
  - export DOCKER_CLI_EXPERIMENTAL=enabled # crucial to use manifest

after_success:
  - docker images
  - docker manifest inspect --verbose lucashalbert/curl # multiarch build
  - docker manifest inspect --insecure lucashalbert/curl # multiarch build 
  - docker manifest inspect --verbose ppc64le/node # IBM power build 
  - docker manifest inspect --insecure ppc64le/node # IBM power build 
  - docker manifest inspect --verbose s390x/python # IBM Z build 
  - docker manifest inspect --insecure s390x/python # IBM z build
  - docker manifest inspect --verbose ibmjava:jre # official Docker IBM Java (Multiarch) build
  - docker manifest inspect --insecure ibmjava:jre # official Docker IBM Java (Multiarch) build

branches:
  only:
    - master
  except:
    - /^*-v[0-9]/
    - /^v\d.*$/
```

## Viewing the Manifest 

Once you've added manifest, it's crucial to add to your `.travis.yml`:

```yaml
script: export DOCKER_CLI_EXPERIMENTAL=enabled
```

Alternatively you can run this in your project directory tree via:

```bash
export DOCKER_CLI_EXPERIMENTAL=enabled
```
As you can see using a combination of architecture-specific tags and an associated Docker manifest, we can achieve a one-size-fits-all architecture agnostic image pulls approach from our repository. While Travis is building your project, you'll start seeing your manifest, it will look like this: 

![image](https://user-images.githubusercontent.com/20936398/133839128-ffff6c56-5945-4a1c-860a-70545fd52468.png)

## Running `docker manifest` 

![image](https://user-images.githubusercontent.com/20936398/133839728-14add45a-2a02-4ee7-a398-eead53d367ce.png)

Each layer of the manifest is comprised of a JSON file (which looks like the .config file we talked about earlier), a VERSION file with the string 1.0, and a layer.tar file containing the images files. In this particular case above, we are going to inspect ppc64le/node from DockerHub in my Ubuntu VM, using VMWare.

You might need to resolve some dependencies if doing manifest on Ubuntu, this is fairly easiy:

```bash
sudo apt install ruby-dev libffi-dev make gcc
sudo gem install travis
```

Then make sure Travis is installed:

```bash
which travis
```

Then you're good to go, now lets move on to pushing a manifest:

```json
"schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 945,
         "digest": "sha256:2ab48cb5665bebc392e27628bb49397853ecb1472ecd5ee8151d5ff7ab86e68d",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1363,
         "digest": "sha256:956f5cf1146bb6bb33d047e1111c8e887d707dde373c9a650b308a8ea7b40fa7",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v6"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1363,
         "digest": "sha256:c6cc369f9824b7f6a19cca9d7f1789836528dd7096cdb4d1fc0922fd43af9d79",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v7"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1363,
         "digest": "sha256:b9ae5a5f88f9e4f35c5ad8f83fbb0705cf4a38208a4e40c932d7abd2e7b7c40b",
         "platform": {
            "architecture": "arm64",
            "os": "linux",
            "variant": "v8"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1363,
         "digest": "sha256:4eca7b4f398526c8bf84be21f6c2c218119ed90a0ffa980dd4ba31ab50ca8cc5",
         "platform": {
            "architecture": "386",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1363,
         "digest": "sha256:2239e5d3ee0e032514fe9c227c90cc5a1980a4c12602f683f4d0a647fb092797",
         "platform": {
            "architecture": "ppc64le",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1363,
         "digest": "sha256:57523d3964bc9ee43ea5f644ad821838abd4ad1617eed34152ee361d538bfa3a",
         "platform": {
            "architecture": "s390x",
            "os": "linux"
         }
      }
   ]
}
Done. Your build exited with 0.
```

## Conclusion

While this mechanism is powerful, it is still considered "Experimental" and is currently only able to be manipulated from the CLI, but with Travis you can integrate this within your build, and make things a little less complicated. 

If you have any questions please email me at [mailto:montana@travis-ci.org](montana@travis-ci.org). 

Happy building!


