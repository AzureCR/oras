# OCI Registry As Storage
[![Codefresh build status]( https://g.codefresh.io/api/badges/pipeline/orasbot/deislabs%2Foras%2Fmaster?type=cf-1)]( https://g.codefresh.io/public/accounts/orasbot/pipelines/deislabs/oras/master)
[![Go Report Card](https://goreportcard.com/badge/github.com/deislabs/oras)](https://goreportcard.com/report/github.com/deislabs/oras)
[![GoDoc](https://godoc.org/github.com/deislabs/oras?status.svg)](https://godoc.org/github.com/deislabs/oras)

[Registries are evolving as Cloud Native Artifact Stores](https://stevelasker.blog/2019/01/25/cloud-native-artifact-stores-evolve-from-container-registries/). To enable this goal, Microsoft has donated ORAS as means to enable various client libraries with a way to submit artifacts to [OCI Spec Compliant](https://github.com/opencontainers/image-spec) registires. This repo is a staging ground for some yet to be determined upstream home. 

As of Jan 24th, 2019, we're still evolving the library to incorporate annotation support. While we're initially testing ORAS with [Helm 3 Registries](https://github.com/helm/community/blob/3689b3202e35361274241dc4ec188e1e6f1a2e53/proposals/helm-repo-container-registry-convergence/readme.md) and [CNAB](https://cnab.io), we're very interested in feedback and contributions for other artifacts. 

## More Background
For more background, please see:

- [OCI Image Support Comes to Open Source Docker Registry](https://www.opencontainers.org/blog/2018/10/11/oci-image-support-comes-to-open-source-docker-registry)
- [Registries are evolving as Cloud Native Artifact Stores](https://stevelasker.blog/2019/01/25/cloud-native-artifact-stores-evolve-from-container-registries/)

## Registries with known support

`oras` can push/pull any files to/from any registry with OCI image support of various mime types.

- [Distribution](https://github.com/docker/distribution) (open source, version 2.7+)
- [Azure Container Registry](https://aka.ms/acr/docs)
- Quay.io is coming soon

## Getting Started

First, you must have access to a registry with OCI image support (see list above).

The simplest way to get started is to run the official
[Docker registry image](https://hub.docker.com/_/registry) locally:

```
docker run -it --rm -p 5000:5000 registry:2.7.0
```

This will start a Distribution server at `localhost:5000`
(with wide-open access and no persistence).

Next, install the `oras` CLI (see platform-specific installation instructions below).

Push a sample file to the registry:

```
cd /tmp && echo "hello world" > hi.txt
oras push localhost:5000/hello:latest hi.txt
```

Pull the file from the registry:
```
cd /tmp && rm -f hi.txt
oras pull localhost:5000/hello:latest
cat hi.txt  # should print "hello world"
```

Please see the **Go Module** section below for how this can be imported and used
inside a Go project.

## CLI

`oras` is a CLI that allows you to push and pull files from
any registry with OCI image support.


### Installation

Install `oras` using [GoFish](https://gofi.sh/):
```
gofish install oras
==> Installing oras...
🐠  oras 0.3.3: installed in 65.131245ms
```

or install manually from the latest [release artifacts](https://github.com/deislabs/oras/releases):
```
# Linux
curl -LO https://github.com/deislabs/oras/releases/download/v0.3.3/oras_0.3.3_linux_amd64.tar.gz

# macOS
curl -LO https://github.com/deislabs/oras/releases/download/v0.3.3/oras_0.3.3_darwin_amd64.tar.gz

# unpack, install, dispose
mkdir -p oras-install/
tar -zxf oras_0.3.3_*.tar.gz -C oras-install/
mv oras-install/oras /usr/local/bin/
rm -rf oras_0.3.3_*.tar.gz oras-install/
```

Then, to run:

```
oras help
```
### Docker Image 	

A public Docker image containing the CLI is available on [Docker Hub](https://hub.docker.com/r/orasbot/oras):	

```	
docker run -it --rm -v $(pwd):/workspace orasbot/oras:v0.3.3 help
```	

Note: the default WORKDIR  in the image is `/workspace`.
 
### Login Credentials
`oras` uses the local Docker credentials by default. Please run `docker login` in advance for any private registries.

`oras` also accepts explicit credentials via options, for example,
```
oras pull -u username -p password myregistry.io/myimage:latest
```

### Usage

#### Pushing single files to remote registry
```
oras push localhost:5000/hello:latest hi.txt
```

The default media type for all files is `application/vnd.oci.image.layer.v1.tar`.

The push a custom media type, use the format `filename[:type]`:
```
oras push localhost:5000/hello:latest hi.txt:application/vnd.me.hi
```

#### Pushing multiple files to remote registry
Just as docker images support multiple "layers", ORAS supports pushing multiple files. The file type is up to the implementer. You can push tar, yaml, text or whatever your artifact should be represented as.

To push multiple files with different media types:
```
oras push localhost:5000/hello:latest hi.txt:application/vnd.me.hi bye.txt:application/vnd.me.bye
```

#### Pulling files from remote registry
```
oras pull localhost:5000/hello:latest
```

By default, only blobs with media type `application/vnd.oci.image.layer.v1.tar` will be downloaded.

To specify which media types to download, use the `--media-type`/`-t` flag:
```
oras pull localhost:5000/hello:latest -t application/vnd.me.hi
```

Or to allow all media types, use the `--allow-all`/`-a` flag:
```
oras pull localhost:5000/hello:latest -a
```

## Go Module

The package `github.com/deislabs/oras/pkg/oras` can quickly be imported in other Go-based tools that
wish to benefit from the ability to store arbitrary content in container registries.

Example:

[Source](examples/simple_push_pull.go)

```go
package main

import (
	"context"
	"fmt"

	"github.com/containerd/containerd/remotes/docker"
	ocispec "github.com/opencontainers/image-spec/specs-go/v1"
	"github.com/deislabs/oras/pkg/content"
	"github.com/deislabs/oras/pkg/oras"
)

func check(e error) {
	if e != nil {
		panic(e)
	}
}

func main() {
	ref := "localhost:5000/oras:test"
	fileName := "hello.txt"
	fileContent := []byte("Hello World!\n")
	customMediaType := "my.custom.media.type"

	ctx := context.Background()
	resolver := docker.NewResolver(docker.ResolverOptions{})

	// Push file(s) w custom mediatype to registry
	memoryStore := content.NewMemoryStore()
	desc := memoryStore.Add(fileName, customMediaType, fileContent)
	pushContents := []ocispec.Descriptor{desc}
	fmt.Printf("Pushing %s to %s... ", fileName, ref)
	err := oras.Push(ctx, resolver, ref, memoryStore, pushContents)
	check(err)
	fmt.Println("success!")

	// Pull file(s) from registry and save to disk
	fmt.Printf("Pulling from %s and saving to %s... ", ref, fileName)
	fileStore := content.NewFileStore("")
	allowedMediaTypes := []string{customMediaType}
	_, err = oras.Pull(ctx, resolver, ref, fileStore, allowedMediaTypes...)
	check(err)
	fmt.Println("success!")
	fmt.Printf("Try running 'cat %s'\n", fileName)
}
```
