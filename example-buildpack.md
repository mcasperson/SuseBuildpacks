---
title: "Creating your own buildpack"
author: Matthew Casperson
date: 05/31/2021
description: "Learn how to create a simple buildpack for compiling Java Maven applications"
type: "blog"
tags: ["Conceptual"]
categories: [blog]
image: ""
draft: true
---

## Introduction

In the previous blog post we introduced buildpacks, discussed their benefits, and demonstrated how with nothing more than the `pack` CLI and Docker installed we could compile a traditional Java web application into an executable Docker image with a single command.

The publicly available buildpacks are high quality and have a proven track record in large hosting services. Many engineering teams will find existing buildpacks serve their needs. But for those that need to control their build experience, it is possible to create your own custom buildpacks.

In this blog post we'll create a simple buildpack to build a Java Maven application.

## A sample buildpack

The buildpack specification has been in development for at least a decade. Buildpacks were first conceived by Heroku in 2011, and since then Heroku and Pivotal have worked to formalize a specification as part of the Cloud Native Buildpacks project. 

Buildpacks have been used by these platforms to consume application source code written in a huge variety of languages, compile it into a Docker image, and then host that image on a variety of Platform as a Service (PaaS) offerings. To accommodate the variety of code that developers host on these platforms, the [buildpack interface specification](https://github.com/buildpacks/spec/blob/main/buildpack.md) is detailed and flexible.

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

The `buildpack` section defines the details of our buildpack. The `id` property is a globally unique identifier, the `version` property defines the buildpack version, and the `name` defines a human readable name:

```
[buildpack]
id = "mcasperson/java"
version = "0.0.1"
name = "Java Buildpack"
```

The `stacks` array (double brackets defines an array item in TOML) defines a stack that this buildpack is compatible with.

A *[stack](https://buildpacks.io/docs/concepts/components/stack/)* is a pair of Docker images: one image that is used to build the software (the build image), and a second image used to host the application (the run image). 

The run image is usually quite lean to reduce the size of the final executable Docker image. The build image will typically contain the kind of packages required when compiling software, such as compilers like GCC, and header files like those for the Linux kernel. This allows the build image to support languages that compile native libraries on the fly.

It is possible to create these images yourself, but for this example we'll reuse an existing stack. We can find a list of stacks with the command `pack stack suggest`, which returns the following list:

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

The next file is a bash script called `detect`, in the `bin` subdirectory. These executables are called as part of the buildpack *[lifecycle](https://buildpacks.io/docs/concepts/components/lifecycle/)*. The `detect` executable determines if the source code the buildpack has been run against is a Java Maven application.

Note buildpacks do not mandate what kind of executable can be used here. We have created a Bash script for convenience, but these executables could just as easily be written in Go, Python, Java, or any other language.

One or more buildpacks can be combined into a *[builder](https://buildpacks.io/docs/concepts/components/builder/)*. Each buildpack is responsible for determining if it is compatible with the supplied source code, and the first compatible buildpack will be used to compile the code. This is how we can run a command like `pack build` against an arbitrary code base without having to define what language our code is written in, as builders like the ones supplied by Heroku come with many buildpacks that detect many different languages.

Our detection script is simple; if a `pom.xml` file does not exist, we return a non-zero exit code to indicate that our buildpack is not compatible. Otherwise the script returns an exit code of zero to indicate that it is compatible:

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

The work of building the source code is performed in the `build` executable, also in the `bin` subdirectory:

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

We create a new layer which will hold a Maven distribution. This layer is also cached and used for the build process.

Note that we create a layer per application to take advantage of the fact that binary files under the layer `/bin` directory [are added to the path automatically](https://github.com/buildpacks/spec/blob/main/buildpack.md#layer-paths). Most Linux applications distributions place binary files inside a `bin` directory for us, so by extracting application archives into their own layer, we conveniently expose those applications on the path:

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

The `type` property is set to `web`. If the `type` was set to any other value, we'd have to define the `PACK_PROCESS_TYPE` to the same value when running the executable image. But the `type` of `web` means we can run the executable image with no special configuration.

The `command` property defines how the `jar` file is executed:

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

With the `buildpack.toml`, `bin/detect`, and `bin/build` files written, we now have a buildpack we can use to compile our application. Assuming the petclinic code is in the `spring-petclinic` directory, and the buildpack files are in the `JavaBuildPack` directory, we build our Docker image with the command:

```
pack build petclinic --path ./spring-petclinic --buildpack ./JavaBuildPack
```

As before, our code is compiled into an executable image, this time called `petclinic`. Our custom build pack will:

1. Detect the presence of a `pom.xml` file, and indicate that this buildpack is compatible with the supplied source code.
1. Create four layers to hold the Maven dependencies, the Maven distribution, the JDK, and the JRE.
2. If the layers have not yet been populated, Maven, a JDK, and a JRE are downloaded and extracted into their associated layer.
3. A Maven build is performed, placing any downloaded dependencies into the associated layer.
4. The executable image is created containing the generated `jar` file and the JRE layer, and configured to execute the `jar` file.
5. All layers are cached, so subsequent builds can skip most, if not all, of the file downloads performed during the first build.

And with that we have created our very own buildpack to compile Java Maven applications.

## Conclusion

There are many high quality and battle tested buildpacks created by PaaS platforms who are heavily invested in building and deploying whatever code their customers throw at them. Anyone getting started with buildpacks would be well served by these freely available options.

For those looking to provide a customized build experience for their team though, a custom buildpack may be the answer. In this post we built a simple buildpack to compile a Java Maven application, and with three relatively simple files we were able to construct and run a custom build pack against our sample application.

<!--

A brief wrap up describing the topic covered and linking to additional
conceptual or practical articles that the reader can use to further their
understanding or begin implementation.

-->