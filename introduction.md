<!--

This file is a template to help you get started writing a "conceptual" article.
Read our writing guidelines for more information on how to write articles for
the Rancher community:

    https://rancher.com/writing-program/writing-guidelines

-->
---
title: "A developers introduction to buildpacks"
author: Matthew Casperson
date: 05/31/2021
description: "Learn how buildpacks evolved and the benefits they can bring to developers workflows"
type: "blog"
tags: ["Conceptual"]
categories: [blog]
image: ""
draft: true
---

<!-- In the front matter above, fill out the title, author, and description
fields. -->

## Introduction

<!-- Include paragraphs describing article scope, why it's helpful, who should
read it, and what the reader will learn. -->

Compiling software is not a glamourous job, but it is a critical part of every developer's workflow. The process of compiling software has evolved over the decades moving from developers building artifacts locally, to centralized build servers, to multistage Docker images, and now to a relatively new process called buildpacks.

In this post we'll look at the history of compiling software and see how buildpacks have evolved to provide developers with an opinionated and convenient process for building their source code.

## The history of compiling software

Not so long ago it was common for a developer on a team to perform a release by compiling code on their local workstation, generating deployable artifacts, and performing a deployment using tools like FTP or RDP onto a server. This process had some obvious limitations: it was not reproducible, it was entirely manual, and it depended on an individual's workstation to be properly configured with all the required platforms and scripts required to complete a build.

Build servers, often called Continuous Integration (CI) servers, provided the next logical step. A CI server provided a source of truth, where the process of testing and building code was centrally defined and managed. Traditionally, the build process was defined as a series of steps configured through a web console, although more recently those steps are defined as code and checked in alongside the application source code. By automating the process of testing and building code, a CI server allows code changes to be continuously built as a shared Source Control Management (SCM) system receives new code commits.

CI servers still require a great deal of specialized configuration though. The Software Development Kits (SDKs) for the various languages used to write applications must be installed, as well as project and package management tools. Nowadays languages release new versions multiple times a year, which can place a burden on CI servers to maintain multiple SDKs side by side to accommodate software projects that upgrade to new language versions at differing rates. In addition, the CI build steps require a particular CI server to execute them. So even if the step configuration has been expressed as code and checked in alongside the application code, only a particular CI server can execute those steps.

These problems are neatly solved by using Docker. Each Docker image provides an isolated file system, allowing entire build chains to exist within their own independent Docker image. This means even software that has no native support for installing and running multiple versions can exist side by side in their own Docker images. And by using multistage Docker builds, it is possible to define the steps and environment required to build application code with instructions that can be executed by any server or workstation with Docker installed.

The isolated nature of Docker images can present a challenge when building software though. Most applications these days rely on dozens, if not hundreds, of external dependencies that must be downloaded from the internet. A naive implementation of a Docker build script will result in these dependencies being downloaded with each and every build. It is not uncommon for an application to download hundreds of megabytes worth of dependencies, and so without care, compiling applications using Docker can be extremely inefficient.

Still, Docker's ability to precisely define the environment in which an application is built, to allow multiple such definitions to painlessly coexist on a single machine, and to allow builds to be performed anywhere Docker is installed, makes it a very compelling platform to build software in. The only thing missing is the ability to abstract away the highly specialized build processes required to efficiently build software using Docker.

This is where buildpacks come in. A buildpack implements an opinionated build process within a Docker container that takes care of all the fiddly aspects of managing SDKs, caching dependencies, and reducing build times to create an executable Docker image from the supplied source code. This means developers can write their code as they always have, with no Docker build scripts, compile that code with a buildpack, and have their compiled application embedded into an executable Docker image.

Buildpacks are:
* Convenient, as their opinionated workflows require no special knowledge to use, and are trivial to execute.
* Efficient, as they are specially designed to leverage Docker's functionality to ensure builds are as fast as possible.
* Repeatable, requiring only one application to be installed alongside Docker to build any number of languages.
* Flexible, with an open specification allowing anyone to define their own build process.

For all these benefits though, buildpacks are not a complete replacement for your CI system. For a start, buildpacks only generate Docker images. If you deploy applications to a web or application server, buildpacks won't generate the kind of traditional artifacts you need. It is also quite likely that you will execute buildpacks on a CI server to retain the benefits of a centralized source of truth.

To demonstrate just how powerful buildpacks are, let's take a typical Java application that has no Docker build configurations and create an executable Docker image with a buidlpack.

## Building an example application

[Petclinic](https://github.com/spring-projects/spring-petclinic) is a sample Java Spring web application that has been lovingly maintained over the years as a demonstration of the Spring platform. It represents the kind of code base you would find in many engineering departments. Although the git repository contains a `docker-compose.yml` file, this is only to run a MySQL database instance - neither the application source code nor the build scripts provide any facility to build Docker images.

Clone the git repository with the command:

```
git clone https://github.com/spring-projects/spring-petclinic.git
```

In order to use buildpacks, ensure you have [Docker](https://www.docker.com/) installed.

To build the sample application with a buildpack, we'll need to install what is known as a *platform*, which in our case is a CLI tool called `pack`. Instructions for installing `pack` can be found [here](https://buildpacks.io/docs/tools/pack/), with packages available for most major operating systems.

When you first run `pack` you will be prompted to configure a default builder. We'll cover terminology like *builder* in more detail later in the post, but for now all we need to understand is that a builder contains the buildpacks that compile our code, and that companies like Heroku and Google, and groups like Paketo, provide a number of builders we can use. 

Here is the output of `pack` asking us to define a default builder:

```
Please select a default builder with:

        pack config default-builder <builder-image>

Suggested builders:
        Google:                gcr.io/buildpacks/builder:v1      Ubuntu 18 base image with buildpacks for .NET, Go, Java, Node.js, and Python
        Heroku:                heroku/buildpacks:18              Base builder for Heroku-18 stack, based on ubuntu:18.04 base image
        Heroku:                heroku/buildpacks:20              Base builder for Heroku-20 stack, based on ubuntu:20.04 base image
        Paketo Buildpacks:     paketobuildpacks/builder:base     Ubuntu bionic base image with buildpacks for Java, .NET Core, NodeJS, Go, Ruby, NGINX and Procfile
        Paketo Buildpacks:     paketobuildpacks/builder:full     Ubuntu bionic base image with buildpacks for Java, .NET Core, NodeJS, Go, PHP, Ruby, Apache HTTPD, NGINX and Procfile
        Paketo Buildpacks:     paketobuildpacks/builder:tiny     Tiny base image (bionic build image, distroless-like run image) with buildpacks for Java Native Image and Go

Tip: Learn more about a specific builder with:
        pack builder inspect <builder-image>
```

We'll make use of the Heroku Ubuntu 20.04 builder, which we configure with the command:

```
pack config default-builder heroku/buildpacks:20
```

Then, in the `spring-petclinic` directory, run the command:

```
pack build myimage
```

It is important to note that we do not need to have the Java Development Kit (JDK) or Maven installed for `pack` to build our source code. We also don't need to tell `pack` that we are trying to build Java code. The Heroku builder (or any other builder you may be using) conveniently takes care of all of this for us.

The first time you run a build, all of the application dependencies are downloaded. And there are a lot! On my home internet connection, it took around 30 minutes to complete the downloads. Once these downloads complete, the application is compiled, and a Docker image called `myimage` is created. We can verify this by running the command:

```
docker image ls --filter reference=myimage
```

To run the Docker image, run the command:

```
docker run -p 8080:8080 myimage
```

The resulting web application has been exposed on port 8080, so we can access it via the URL http://localhost:8080. You will see a page like this:

![](petclinic.png)

It is worth taking a moment to appreciate what we just achieved here. With a single command we compiled source code that had no Docker configuration into an executable Docker image. We never needed to install any Java tooling, and never needed to configure any Java settings.

To verify that the application dependencies were cached, run `pack build myimage2`. Notice this time the build process completes much faster as all the downloads from the previous build are reused. This demonstrates how buildpacks provide an efficient build process.

The process we just ran through here is also easily repeated on any machine with Docker and the `pack` CLI installed. It would take very little to recreate this process in a CI server, meaning builds on a centralized build server and local developers PCs work the same way.

To demonstrate the flexibility of buildpacks, we create a simple buildpack of our own to compile a Java application with Maven.

## A sample buildpack

The buildpack specification has been in development for at least a decade. Buildpacks were first conceived by Heroku in 2011, and since then Heroku and Pivotal have worked to formalize a specification as part of the Cloud Native Buildpacks project. 

Buildpacks have been used by these platforms to consume application source code written in a huge variety of languages, compile it into a Docker image, and then host that image on a variety of Platform as a Service (PaaS) offerings. To accommodate the variety of code that developers host on these platforms, the [buildpack interface specification](https://github.com/buildpacks/spec/blob/main/buildpack.md) is quite detailed and flexible.

Our sample buildpack will be quite simple and use only a subset of the functionality available to us. But even with this simple example, we can demonstrate many of the benefits that buildpacks bring to a build pipeline.

Our custom buildpack starts with the `buildpack.toml` file:

```toml
# Buildpack API version
api = "0.5"

# Buildpack ID and metadata
[buildpack]
id = "mcasperson/java"
version = "0.0.1"
name = "Java Buildpack"

# Stacks that the buildpack will work with
[[stacks]]
id = "heroku-20"
```

The schema for the `buildpack.toml` file is documented [here](https://buildpacks.io/docs/reference/spec/buildpack-api/#schema), but we'll break down the example file below.

The `api` property defines the buildpack API version the buildpack adheres to:

```
api = "0.5"
```

The `buildpack` section defines the details of our buildpack. The `id` property is a globally unique identifier, the `version` property defines the buildpack version, and the `name` defines a human readable name.

```
[buildpack]
id = "mcasperson/java"
version = "0.0.1"
name = "Java Buildpack"
```

The `stacks` array (double brackets defines an array item in TOML) defines a stack that this buildpack is compatible with.

A *stack* is a pair of Docker images: one image that is used to build the software, and a second image used to host the application. It is possible to create these images yourself, but we'll reuse an existing stack. We can find a list of stacks with the command `pack stack suggest`, which returns the following list:

```
Stacks maintained by the community:

    Stack ID: heroku-18
    Description: The official Heroku stack based on Ubuntu 18.04
    Maintainer: Heroku
    Build Image: heroku/pack:18-build
    Run Image: heroku/pack:18

    Stack ID: heroku-20
    Description: The official Heroku stack based on Ubuntu 20.04
    Maintainer: Heroku
    Build Image: heroku/pack:20-build
    Run Image: heroku/pack:20

    Stack ID: io.buildpacks.stacks.bionic
    Description: A minimal Paketo stack based on Ubuntu 18.04
    Maintainer: Paketo Project
    Build Image: paketobuildpacks/build:base-cnb
    Run Image: paketobuildpacks/run:base-cnb

    Stack ID: io.buildpacks.stacks.bionic
    Description: A large Paketo stack based on Ubuntu 18.04
    Maintainer: Paketo Project
    Build Image: paketobuildpacks/build:full-cnb
    Run Image: paketobuildpacks/run:full-cnb

    Stack ID: io.paketo.stacks.tiny
    Description: A tiny Paketo stack based on Ubuntu 18.04, similar to distroless
    Maintainer: Paketo Project
    Build Image: paketobuildpacks/build:tiny-cnb
    Run Image: paketobuildpacks/run:tiny-cnb
```

Our buildpack will be compatible with the `heroku-20` stack:

```
[[stacks]]
id = "heroku-20"
```

The next file is a bash script called `detect`. These executables are called as part of the buildpack lifecycle. The `detect` executable determines if the source code the buildpack has been run against is a Java Maven application.

Note buildpacks do not mandate what kind of executable can be used here. We have created a Bash script for convenience, but these executables could just as easily be written in Go, Python, Java, or any other language.

One or more buildpacks can be combined into a builder. Each buildpack is responsible for determining if it is compatible with the supplied source code, and the first compatible buildpack will be used to compile the code. This is how we can run a command like `pack build` against an arbitrary code base without having to define what language our code is written in; builders like the ones supplied by Heroku come with many buildpacks that detect many different languages.

Our detection script is simple: if a `pom.xml` file does not exist, we return a non-zero return code to indicate that our buildpack is not compatible. Otherwise the script has a zero return code to indicate that it is compatible:

```bash
#!/usr/bin/env bash

# The -e option will cause a bash script to exit immediately when a command fails.
# The -o pipefail option means if command in a pipeline fails, that return code will be used as the return code of the whole pipeline.
set -eo pipefail

# Check for the presense of a pom.xml file.
if [[ ! -f pom.xml ]]; then
	# If pom.xml does not exist, return an error code.
	exit 100
fi
```

The work of building the source code is done in the `build` executable:

```bash
#!/usr/bin/env bash

# The -e option will cause a bash script to exit immediately when a command fails.
# The -o pipefail option means if command in a pipeline fails, that return code will be used as the return code of the whole pipeline.
set -eo pipefail

layersdir=$1

dependencieslayer="$layersdir"/dependencies
mkdir -p "$dependencieslayer/.m2/repository"
echo -e 'cache = true\nbuild = true' > "$layersdir/dependencies.toml"

mavenLayer="$layersdir"/maven
mkdir -p "$mavenLayer"
echo -e 'cache = true\nbuild = true' > "$layersdir/maven.toml"

jdkLayer="$layersdir"/jdk
mkdir -p "$jdkLayer"
echo -e 'cache = true\nbuild = true' > "$layersdir/jdk.toml"

jreLayer="$layersdir"/java
mkdir -p "$jreLayer"
echo -e 'launch = true\ncache = true' > "$layersdir/java.toml"

# Download Maven if it doesn't exist already.
if [[ ! -f $mavenLayer/bin/mvn ]]; then
	echo "Downloading Maven"
	maven_url=https://apache.mirror.digitalpacific.com.au/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
	# Ensure the executables are placed under the layer's bin directory by stripping the first
	# directory from the tar file.
	wget -q -O - "$maven_url" | tar --strip-components=1 -xzf - -C "$mavenLayer"
else
	echo "Skipped Maven Download"
fi

# Download JDK if it doesn't exist already.
if [[ ! -f $jdkLayer/bin/java ]]; then
	echo "Downloading JDK"
	jdk_url=https://cdn.azul.com/zulu/bin/zulu11.48.21-ca-jdk11.0.11-linux_x64.tar.gz
	# Ensure the executables are placed under the layer's bin directory by stripping the first
	# directory from the tar file.
	wget -q -O - "$jdk_url" | tar --strip-components=1 -xzf - -C "$jdkLayer"
else
	echo "Skipped JDK Download"
fi

# Download JRE if it doesn't exist already.
if [[ ! -f $jreLayer/bin/java ]]; then
	echo "Downloading JRE"
	jre_url=https://cdn.azul.com/zulu/bin/zulu11.48.21-ca-jre11.0.11-linux_x64.tar.gz
	# Ensure the executables are placed under the layer's bin directory by stripping the first
	# directory from the tar file.
	wget -q -O - "$jre_url" | tar --strip-components=1 -xzf - -C "$jreLayer"
else
	echo "Skipped JRE Download"
fi

JAVA_HOME=$jdkLayer $mavenLayer/bin/mvn -Dmaven.repo.local=$dependencieslayer/.m2/repository clean package

for jarFile in $(find target -maxdepth 1 -name "*.jar" -type f); do
	cat >> "$layersdir/launch.toml" <<EOF
[[processes]]
type = "web"
command = "java -jar $jarFile"
EOF
	break;
done
```

The `build` script is doing a lot of heavy lifting, so let's break down the code.

We configure Bash to fail if any command in the script fails. This means we don't mask errors returned by any commands called in this script:

```bash
set -eo pipefail
```

The first argument passed to the `build` script is the directory holding our layers:

```bash
layersdir=$1
```

Buildpacks use layers to hold files that are of importance when building the code and running the compiled application. Layers can optionally be cached to ensure that any files saved by a previous build are reused.

A layer is simply a directory, paired with a TOML file to configure the layer metadata.

We start by creating a directory to hold our application's dependencies:

```bash
dependencieslayer="$layersdir"/dependencies
```

Under this directory we create a nested directory structure to hold our Maven dependencies:

```bash
mkdir -p "$dependencieslayer/.m2/repository"
```

To configure this directory as a layer, we create a TOML file with the same name as the directory, meaning in this case the file is called `dependencies.toml`. We set two properties to `true`: `cache`, which indicates that the files placed in this directory will be made available to subsequent builds, and `build`, which indicates this layer is used as part of the build process:

```bash
echo -e 'cache = true\nbuild = true' > "$layersdir/dependencies.toml"
```

We create a new layer which will hold a Maven distribution. This layer is also cached and used for the build process:

```bash
mavenLayer="$layersdir"/maven
mkdir -p "$mavenLayer"
echo -e 'cache = true\nbuild = true' > "$layersdir/maven.toml"
```

We have another layer to hold the JDK, and again this layer is cached and used for the build process:

```bash
jdkLayer="$layersdir"/jdk
mkdir -p "$jdkLayer"
echo -e 'cache = true\nbuild = true' > "$layersdir/jdk.toml"
```

The final layer will hold the JRE. This layer is used for the executable Docker image, so the `launch` property is set to `true`. This layer is also cached:

```bash
jreLayer="$layersdir"/java
mkdir -p "$jreLayer"
echo -e 'launch = true\ncache = true' > "$layersdir/java.toml"
```

We now populate the layers. The following commands check to see if the `mvn` executable already exists, and if not, will download and extract Maven.

The first time we run this buildpack, the Maven layer will be empty, and the Maven archive will be downloaded. Because this layer is cached, the second time the buildpack is run, the Maven files will already be available, and the download will be skipped:

```bash
# Download Maven if it doesn't exist already.
if [[ ! -f $mavenLayer/bin/mvn ]]; then
	echo "Downloading Maven"
	maven_url=https://apache.mirror.digitalpacific.com.au/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
	# Ensure the executables are placed under the layer's bin directory by stripping the first
	# directory from the tar file.
	wget -q -O - "$maven_url" | tar --strip-components=1 -xzf - -C "$mavenLayer"
else
	echo "Skipped Maven Download"
fi
```

We follow the same pattern with the JDK and JRE:

```bash
# Download JDK if it doesn't exist already.
if [[ ! -f $jdkLayer/bin/java ]]; then
	echo "Downloading JDK"
	jdk_url=https://cdn.azul.com/zulu/bin/zulu11.48.21-ca-jdk11.0.11-linux_x64.tar.gz
	# Ensure the executables are placed under the layer's bin directory by stripping the first
	# directory from the tar file.
	wget -q -O - "$jdk_url" | tar --strip-components=1 -xzf - -C "$jdkLayer"
else
	echo "Skipped JDK Download"
fi

# Download JRE if it doesn't exist already.
if [[ ! -f $jreLayer/bin/java ]]; then
	echo "Downloading JRE"
	jre_url=https://cdn.azul.com/zulu/bin/zulu11.48.21-ca-jre11.0.11-linux_x64.tar.gz
	# Ensure the executables are placed under the layer's bin directory by stripping the first
	# directory from the tar file.
	wget -q -O - "$jre_url" | tar --strip-components=1 -xzf - -C "$jreLayer"
else
	echo "Skipped JRE Download"
fi
```

We now have all the files we need to build our application. We set the `JAVA_HOME` environment variable to the directory holding the JDK layer, and then call `mvn` from the Maven layer. We set the `maven.repo.local` property to instruct Maven to download any dependencies to our dependency layer. Finally we run the `clean` and `package` Maven goals:

```bash
JAVA_HOME=$jdkLayer $mavenLayer/bin/mvn -Dmaven.repo.local=$dependencieslayer/.m2/repository clean package
```

Once the build completes, we'll have a `jar` file in the `target` directory. We don't know the exact name of this file, but we know there will be one `jar` file, so we use the `find` command to return the matching file.

The `jar` file needs to be configured to be executed in the executable image. This is done in the `launch.toml` file.

The `type` is set to `web`. If the `type` was set to any other value, we'd have to define the `PACK_PROCESS_TYPE` to the same value when running the executable image. But the `type` of `web` means we can run the executable image with no special configuration.

The `command` defines how the `jar` file is executed:

```bash
for jarFile in $(find target -maxdepth 1 -name "*.jar" -type f); do
	cat >> "$layersdir/launch.toml" <<EOF
[[processes]]
type = "web"
command = "java -jar $jarFile"
EOF
	break;
done
```

With the `buildpack.toml`, `bin/detect`, and `bin/build` files written, we now have a buildpack we can use to build our application. Assuming the petclinic code is in the `spring-petclinic` directory, and the buildpack files are in the `JavaBuildPack` directory, we build our Docker image with the command:

```
pack build petclinic --path ./spring-petclinic --buildpack ./JavaBuildPack
```

As before, our code is compiled into an executable image, but this time called `petclinic`. Our custom build pack will:

1. Detect the presence of a `pom.xml` file, and indicate that this buildpack is compatible with the supplied source code.
1. Create four layers to hold the Maven dependencies, the Maven distribution, the JDK, and the JRE.
2. If the layers have not yet been populated, Maven, a JDK, and a JRE are downloaded and extracted into their associated layer.
3. A Maven build is performed, placing any downloaded dependencies into the associated layer.
4. The executable image is created containing the generated `jar` file and the JRE layer, and configured to execute the `jar` file.
5. All layers are cached, so subsequent builds can skip most, of not all, of the file downloads performed during the first build.

And with that we have created our very own buildpack to compile Java Maven applications.

## Conclusion

There are many high quality and battle tested buildpacks created by PaaS platforms who are heavily invested in building and deploying whatever code their customers throw at them. Anyone getting started with buildpacks would be well served by these freely available options.

For those looking to provide a customized build experience for their team though, a custom buildpack may be the answer. In this post we built a simple buildpack to compile a Java Maven application, and with three relatively simple files we were able to construct and run a custom build pack against our sample application.

<!--

A brief wrap up describing the topic covered and linking to additional
conceptual or practical articles that the reader can use to further their
understanding or begin implementation.

-->