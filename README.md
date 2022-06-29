# docker-registry <br> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *query a registry without downloading images* #

* [Description](#description)
* [Quick tutorial](#quick-tutorial)
* [Installation](#installation)
* [Usage](#usage)
* [Background documentation](#background-documentation)
* [Alternatives for this script](#alternatives-for-this-script)


## Description ##
The `docker-registry` is a command line utility than fetches image info from a remote docker registry without needing to download the image. The command `docker inspect` works only on local images which first must be downloaded using `docker pull`. For inspecting a large image on the registry this causes unnecessary network traffic and waiting time.  The  `docker-registry` command line utility can do the inspection without doing any image download by using the docker registry REST-API to query information about the image.

The `docker-registry` lets you query the following info from a remote registry server: 

* **tags** : the list of tags in a image repository on the server
* **manifestlist** : for a multi-arch image for a specific tag a manifestlist lists the manifests for the different `os:arch` instances of an image
* **manifest**: for each instance of an image a manifest is defined which describes from which parts the images is constructed, and how to download them
* **digest**: SHA256 hash of the manifest content which acts as the identifier for a specific image on the server (Content Addressable IDs). Identifying the distribution recipe of the image.
* **config**: each image has a configuration object
* **id**:  SHA256 hash of the  configuration object which acts as the identifier for a specific image locally (Content Addressable IDs).
* **labels**: the labels within the configuration object
* **downloadlayer**: for a given layer digest, from the manifest, download the layer 

The tool has also a command **verifytool** which purely is used to verify that the tool is still working correctly by checking that the digest/id communicated are the same as the  digest/id calculated from the downloaded manifest/configuration.

The `docker-registry` is a command line utility is just a single `bash script`. <br/> This has the following advantages:

* Doesn't need `docker` to be installed to work!
* Easy to deploy<br> 
  e.g. in github actions you can easily install by downloading it with the `curl` command. 
* Reference usage<br>The script can act as a reference implementation of (a part) of the registry API.<br> Because the script is easily readable makes it usefull as documentation for how to query the registry server.


## Quick tutorial ##

Quickly query the available **tags** of the `ubunty` image: 


    $ docker-registry  tags library/ubuntu
    ["10.04","12.04","12.04.5","12.10","13.04","13.10", ... ]

Combined with the `jq` tool we can filter the previous result to only show tags with '.04' in there name:

	$ docker-registry  tags library/ubuntu | jq -c 'map( select( contains("14.04") ))'
	["14.04","14.04.1","14.04.2","14.04.3","14.04.4","14.04.5"]

Quickly query the **labels** of an image: 

	$  docker-registry --arch amd64   labels  sphinxdoc/sphinx:4.4.0
	{
	  "maintainer": "Sphinx Team <https://www.sphinx-doc.org/>",
	  "org.opencontainers.image.created": "2022-01-16T15:19:24.686Z",
	  "org.opencontainers.image.description": "",
	  "org.opencontainers.image.licenses": "",
	  "org.opencontainers.image.revision": "03e2a663ba5eff8f460336c59e732283d37ea0f6",
	  "org.opencontainers.image.source": "https://github.com/sphinx-doc/docker",
	  "org.opencontainers.image.title": "docker",
	  "org.opencontainers.image.url": "https://github.com/sphinx-doc/docker",
	  "org.opencontainers.image.version": "4.4.0"
	}

The OCI team published a [standardized set of keys](https://github.com/opencontainers/image-spec/blob/main/annotations.md)  all
prefixed with `org.opencontainers.image.`. Above we can see some used in the Labels config section of the image. 
	
Note: with the `--arch amd64` you can select the architecture of the image. In the above case it is needed because the 	`sphinxdoc/sphinx` image is only available for `amd64`. If you are using a macbook with a `M1` `arm64` processor then the default architecture will be `arm64`. 

A **manifestlist** gives an overview of all instances of an image for the different `os/arch` types. In the above case where only a single image exists, then there exists no manifestlist.  If you then request a manifestlist the utility gives an error, but still informs you about the existing single image:

	$  docker-registry manifestlist  sphinxdoc/sphinx:4.4.0
	ERROR: no manifestlist found, but there does exist a manifest for 'amd64/linux'
	       with (manifest) digest 
	       'sha256:d4ed92aa4c21c861adc325539b0049ab3e7acc4f7b73fa82e82702ebdfa2b8f4'

The return **digest** uniquely identifies the image on the server without needing to specify `os/arch` or `tag`. So only using the `digest` we can query this specific image:

	$  docker-registry id sphinxdoc/sphinx@sha256:d4ed92aa4c21c861adc325539b0049ab3e7acc4f7b73fa82e82702ebdfa2b8f4
	sha256:4e019c638c3eb72844de0ef87479e4de467dfed199cb5d6d070607a20af6e9f1

We can also query the image with using  `os/arch` and `tag`:


	$ docker-registry --os linux --arch amd64 id  sphinxdoc/sphinx:4.4.0
	sha256:4e019c638c3eb72844de0ef87479e4de467dfed199cb5d6d070607a20af6e9f1

However note that in same cases the specific image behind a `tag` may be updated with security patches. This particulary holds for the `os` images as `ubuntu:18.04`.
So over time the latter query may return a different image.

The result of above queries is the **id** of the image, which indentifies the image locally on your machine after pulling it. Whereas the **digest** uniquely defines the image on the server. 
 
In above case there was no **manifestlist**. No lets inspect the manifestlist for an image for which **many instances** for **different `os/arch` platforms** are defined:	   

	$ docker-registry  manifestlist library/ubuntu:18.04
	{
		"manifests": [
		{
		  "digest": "sha256:a3a04a670edda3847f73eafacec1399210483f28b446e70ea87571f01df47d43",
		  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
		  "platform": {
		    "architecture": "amd64",
		    "os": "linux"
		  },
		  "size": 529
		},
		{
		  "digest": "sha256:83b18c04210c9d28a0480b97c98f8fb9e13fd3564da24b8de3504cd2b4394b8d",
		  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
		  "platform": {
		    "architecture": "arm",
		    "os": "linux",
		    "variant": "v7"
		  },
		  "size": 529
		},
		{
		  "digest": "sha256:d62e3c99dcd0880f3d4f4c68db5462f883a5ce965b5149ca0e1490a267add8b0",
		  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
		  "platform": {
		    "architecture": "arm64",
		    "os": "linux",
		    "variant": "v8"
		  },
		  "size": 529
		},
	   ...


Using the digest in the above output we can query an image on the remote registry server. To verify this `digest` is correct we can also query the `digest` for a specific  `os/arch` and `tag`:

	$ docker-registry --os linux --arch arm64  digest library/ubuntu:18.04
	sha256:d62e3c99dcd0880f3d4f4c68db5462f883a5ce965b5149ca0e1490a267add8b0	    
The digest found matches the digest found for linux/arm64 in the manifestlist above. 

The digest is unique per image instance, eg. the `amd64` instance has a different digest:

	$ docker-registry --os linux --arch amd64  digest library/ubuntu:18.04
	sha256:a3a04a670edda3847f73eafacec1399210483f28b446e70ea87571f01df47d43

An image `id` uniquely defines an image locally. Note that with `qemu` support in docker we can run image instances for different platforms the current platform docker is running on. Therefore also the image `id`'s of different `os/arch` instances are different:
	
	$ docker-registry --os linux --arch amd64  id library/ubuntu:18.04
	sha256:eb2556e0f6e4dcf6a2b4820190d0013c83a97851c1444bc86bb09af085254850
	$ docker-registry --os linux --arch arm64  id library/ubuntu:18.04
	sha256:29e70752d7b2d3a4e9ab013cc75f4652bd41360a22ec3d0c64657e05c8b25ec8

We can verify this the tool gives the right image id:  (needs pulling image)

	$ # run on m1 macbook (arm64)
	$ docker pull ubuntu:18.04     
	$ docker image ls
	REPOSITORY                              TAG       IMAGE ID       CREATED       SIZE
	ubuntu                                  18.04     29e70752d7b2   12 days ago   56.7MB

Finally we can fetch all info about the image by fetching its configuration object:

	$ docker-registry config library/ubuntu:18.04
	{
	  "architecture": "arm64",
	  "config": {
	    "Hostname": "",
	    "Domainname": "",
	    "User": "",
	    ...    

The `config` command gives similar information as the `docker inspect` command, but then without needing to download the image.

We can fetch the config also for another platform:

	$ docker-registry --os linux --arch amd64 config library/ubuntu:18.04
	{
	  "architecture": "amd64",
	  "config": {
	    "Hostname": "",
	    "Domainname": "",
	    "User": "",
	    ...   
	    
Using the `jq` tool we can easily fetch specific config values. For instance we can fetch the image's environment variables with:

	$ docker-registry config library/ubuntu:18.04 | jq '.config.Env'
	[
	  "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
	]
	    
The images `labels` are also part of the `config`, so with `jq` we can filter them also from the config: 

	$ docker-registry --arch amd64   config  sphinxdoc/sphinx | jq '.config.Labels'
	{
	  "maintainer": "Sphinx Team <https://www.sphinx-doc.org/>",
	  "org.opencontainers.image.created": "2022-01-16T15:19:24.686Z",
	  "org.opencontainers.image.description": "",
	  "org.opencontainers.image.licenses": "",
	  "org.opencontainers.image.revision": "03e2a663ba5eff8f460336c59e732283d37ea0f6",
	  "org.opencontainers.image.source": "https://github.com/sphinx-doc/docker",
	  "org.opencontainers.image.title": "docker",
	  "org.opencontainers.image.url": "https://github.com/sphinx-doc/docker",
	  "org.opencontainers.image.version": "4.4.0"
	}  

The `labels` command is extra defined for the convenience of easily fetching the labels and does exactly the above.	

You can easily fetch a specific label with:

        $ docker-registry --arch amd64   labels  sphinxdoc/sphinx | jq -r '."org.opencontainers.image.revision"'
        03e2a663ba5eff8f460336c59e732283d37ea0f6

 
## Installation ##



The `docker-registry` utility is a simple script in `bash`, so you can easily fetch it for a specific version from github:


    VERSION="v1.0.1" 
    INSTALL_DIR=/usr/local/bin.  # make sure INSTALL_DIR is in your PATH environment variable
    DOWNLOAD_URL="https://raw.githubusercontent.com/harcokuppens/docker-registry/${VERSION}/docker-registry"
    curl -Lo ${INSTALL_DIR}/docker-registry  "$DOWNLOAD_URL"
    chmod a+x ${INSTALL_DIR}/docker-registry
    
      
Requirements:  

* `bash` shell
* `sed` filter tool
* `cat` tool
* `jq`  json query tool 
* `curl` http request tool
* `sha256sum` checksum tool
* `cut` tool

Most tools, except `curl` and `jq` are normally installed in a `bash` shell environment.

      
For Windows you could use WSL to run the `docker-registry` utility. You can also install the `docker-registry` utility natively with [Git for Windows](https://gitforwindows.org) to get a bash shell to run the script. In the latter case you have to write a wrapper `.cmd` script to call `bash.exe docker-registry` with the right paths. The wrapper script is needed because Windows doesn't support the shebang execution used in linux/macos.      
	    
	    
## Usage ##
  
    Description:
      Fetch from a remote docker registry image info without needing to download the image.

      A manifestlist implements a multi-arch image by listing the same docker image but build
      for different os/arch platforms.

      A manifest for an image for a specific os/arch platform specifies how to download the image config
      and layers for that os/arch platfrom. A manifestlist has thus a manifest per os/arch platform.

      The content addressable setup of docker means that we have
       * digest: id for a manifest; the sha256 digest of the image manifest.
       * id : id for the image; the sha256 digest of the image config.

    Usage:
      docker-registry [options] tags [REGISTRY/]IMAGE

      docker-registry [options] manifestlist [REGISTRY/]IMAGE[:TAG]

      docker-registry [options] digest [REGISTRY/]IMAGE[:TAG]
      docker-registry [options] manifest [REGISTRY/]IMAGE[:TAG|@DIGEST]


      docker-registry [options] id [REGISTRY/]IMAGE[:TAG|@DIGEST]
      docker-registry [options] config [REGISTRY/]IMAGE[:TAG|@DIGEST]
      docker-registry [options] labels [REGISTRY/]IMAGE[:TAG|@DIGEST]
      
      docker-registry [options] downloadlayer [REGISTRY/]IMAGE@LAYER_DIGEST

      docker-registry           verifytool [REGISTRY/]IMAGE[:TAG]

    Options:
      -h  or  --help                print help (this message) and exit
      -r                            enables raw output. Without this option the json output will be human
                                    readable formatted.
      -u PERSON_OR_ORG:PASSWORD     credentials needed only for accessing none-public images
      --os OS                       specify the operating system of the image (default is linux)
      --arch ARCH                   specify the cpu architecture of the image (default is arch of current machine)

    Notes:
      The REGISTRY argument is by default 'docker.io', for github's registry you should use 'ghcr.io'.
      The LAYER_DIGEST argument can be found in the manifest of an image where for each layer its digest and its size
           is specified.  Note: don't use the digests in the image config, because they are digests from the
           unpacked layers. Whereas the manifest contains digests for the compressed layers ready for fast downloads.



## Background documentation ##


### Inception of this script ###

Artilce Explaining Docker Image IDs
 
 * [article Explaining Docker Image IDs](https://windsock.io/explaining-docker-image-ids/)

Articles about using registry REST-API on:

* docker's registry(docker.io): [article "Inspecting Docker images without pulling them"](https://medium.com/hackernoon/inspecting-docker-images-without-pulling-them-4de53d34a604)
* github's registry(ghcr.io): [article "Checking in on GitHub Container Registry"](https://blog.atomist.com/github-container-registry/)

### Reference documentation


Registry API
 
 * [Reference documentation Registry API](https://docs.docker.com/registry/spec/api/)


Manifest
  
 * [docker manifest](https://docs.docker.com/engine/reference/commandline/manifest/)
 * [Image Manifest V 2, Schema 2](https://docs.docker.com/registry/spec/manifest-v2-2/)

Authorization
  
 * [JWT](https://docs.docker.com/registry/spec/auth/jwt/)
 * [Authentication scope](https://docs.docker.com/registry/spec/auth/scope/)

OAuth

 * [What is OAuth really all about (using JWT toke) - OAuth tutorial - Java Brains](https://www.youtube.com/watch?v=t4-416mg6iU)
 * [OAuth terminologies and flows explained  - OAuth tutorial - Java Brains](https://www.youtube.com/watch?v=3pZ3Nh8tgTE) <br/>
   note: flow 3 is used where in the first step you also authenticate with user/password credentials to get authorization token (in JWT format)
   
JWT

 * [What is JWT authorization really about - Java Brains](https://www.youtube.com/watch?v=soGRyl9ztjI)
 * [What is the structure of a JWT - Java Brains](https://www.youtube.com/watch?v=_XbXkVdoG_0)


## Alternatives for this script ##


**UPDATE:** I recently discovered another tool called [regclient](https://github.com/regclient/regclient) which can do the same, and can also copy multi-arch images efficiently between different registries. For real usage I would recomment that tool instead. However this script can still be a good guidance for anyone wanting to learn how the registry API works, and for simply querying labels this script is also fine. You can even copy just the lines of code from the script doing the label query for an efficient solution.

There are some alternatives for the `docker-registry` utility, but for querying a remote registry the `docker-registry` works better then all of them.

* `skopeo inspect`: alternative for this script's `config` command
   *  [article about skopeo](https://polyverse.com/blog/skopeo-the-best-container-tool-you-need-to-know-about/)
   *   [https://github.com/containers/skopeo](https://github.com/containers/skopeo)

* [docker manifest](https://docs.docker.com/engine/reference/commandline/manifest/): alternative for this script's `manifestlist` and `manifest` commands
 

* [manifest-tool](https://github.com/estesp/manifest-tool): alternative for this script's  `manifestlist` and `manifest` commands. This tool does not allow to see the manifest of an image without a manifestlist. 
    
    

    
