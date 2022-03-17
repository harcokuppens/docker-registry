# docker-registry #

The `docker-registry` is a command line utility than fetches image info from a remote docker registry without needing to download the image. The command `docker inspect` works only on local images which first must be downloaded using `docker pull`. For inspecting a large image on the registry this causes unnecessary network traffic and waiting time.  The  `docker-registry` command line utility can do the inspection without doing any image download by using the docker registry REST-API to query information about the image.

## Examples

Quickly query the available tags of the `ubunty` image: 


    $ docker-registry  tags library/ubuntu
    ["10.04","12.04","12.04.5","12.10","13.04","13.10", ... ]

Combined with the `jq` tool we can filter the previous result to only show tags with '.04' in there name:

	$ docker-registry  tags library/ubuntu | jq 'map( select( contains(".04") ))'
	[
	  "10.04",
	  "12.04",
	  "12.04.5",
	  "13.04",
	  "14.04",
	  "14.04.1",
	  "14.04.2",
	  "14.04.3",
	  "14.04.4",
	  "14.04.5",
	  "15.04",
	  "16.04",
	  "17.04",
	  "18.04",
	  "19.04",
	  "20.04",
	  "21.04",
	  "22.04"
	]   
   
We can also quickly inspect the metadata on a image using Labels:

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

The OCI team published a [standardized set of keys](https://github.com/opencontainers/image-spec/blob/main/annotations.md)  all
prefixed with `org.opencontainers.image.`. Above we can see some used in the Labels config section of the image.   
   
Inspect for which os/arch platforms an image is available by quering its manifestlist
	   
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

Query an image digest (id) for a specific  os/arch:

	$ docker-registry --os linux --arch arm64  digest library/ubuntu:18.04
	sha256:d62e3c99dcd0880f3d4f4c68db5462f883a5ce965b5149ca0e1490a267add8b0	    
The digest founds matches the digest found for linux/arm64 in the manifestlist above.
  
Fetch the image id of an image 

	$ docker-registry id library/ubuntu:18.04
	sha256:29e70752d7b2d3a4e9ab013cc75f4652bd41360a22ec3d0c64657e05c8b25ec8

We can verify this is the right image id:  (needs pulling image)

	# docker pull ubuntu:18.04
	$ docker image ls
	REPOSITORY                              TAG       IMAGE ID       CREATED       SIZE
	ubuntu                                  18.04     29e70752d7b2   12 days ago   56.7MB

We can fetch all info about the image by fetching its configuration object:

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

	$ docker-registry --os linux --arch amd64 config library/ubuntu:18.04 |head -10
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

Articles about using registry REST-API on:

* docker's registry(docker.io): [article "Inspecting Docker images without pulling them"](https://medium.com/hackernoon/inspecting-docker-images-without-pulling-them-4de53d34a604)
* github's registry(ghcr.io): [article "Checking in on GitHub Container Registry"](https://blog.atomist.com/github-container-registry/)

### Reference ocumentation


Registry API
 
 * [Reference documentation Registry API](https://docs.docker.com/registry/spec/api/)

Explaining Docker Image IDs
 
 * [article Explaining Docker Image IDs](https://windsock.io/explaining-docker-image-ids/)

Manifest
  
 * [docker manifest](https://docs.docker.com/engine/reference/commandline/manifest/)
 * [Image Manifest V 2, Schema 2](https://docs.docker.com/registry/spec/manifest-v2-2/)

Authorization
  
 * [JWT](https://docs.docker.com/registry/spec/auth/jwt/)
 * [Authentication scope](https://docs.docker.com/registry/spec/auth/scope/)

OAuth

 * [What is OAuth really all about (using JWT toke) - OAuth tutorial - Java Brains](https://www.youtube.com/watch?v=t4-416mg6iU)
 * [OAuth terminologies and flows explained  - OAuth tutorial - Java Brains](https://www.youtube.com/watch?v=3pZ3Nh8tgTE) <br/>
   => flow 3 is used where in the first step you also authenticate with user/password credentials to get authorization token (in JWT format)
   
JWT

 * [What is JWT authorization really about - Java Brains](https://www.youtube.com/watch?v=soGRyl9ztjI)
 * [What is the structure of a JWT - Java Brains](https://www.youtube.com/watch?v=_XbXkVdoG_0)


# Alternatives for this script #

* `skopeo inspect`: alternative for this script's `config` command
    https://polyverse.com/blog/skopeo-the-best-container-tool-you-need-to-know-about/
    https://github.com/containers/skopeo

* [docker manifest](https://docs.docker.com/engine/reference/commandline/manifest/): alternative for this script's `manifest` `manifestlist` commands
 

* `manifest-tool`: alternative for this script's `manifest` `manifestlist` commands
    https://github.com/estesp/manifest-tool