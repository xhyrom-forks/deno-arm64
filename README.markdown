# Deno for ARM64

[![Docker Image Version (tag latest semver)](https://img.shields.io/docker/v/lukechannings/deno/latest?label=Docker%20Image)](https://hub.docker.com/repository/docker/lukechannings/deno)
[![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/lukechannings/deno-arm64?label=ARM64%20Binary)](https://github.com/LukeChannings/deno-arm64/releases)

## What is this

I put this together because there are no Linux ARM64 binaries for Deno [yet](https://github.com/denoland/deno/issues/1846#issuecomment-725165778).
This project compiles ARM binaries and simultaneously releases a combined arm64 & amd64 container image based on Ubuntu.

### Compiling locally

To build a binary locally, using Docker, the following should compile and export a binary into the local directory.

**Change the DENO_VERSION as appropriate!**

```bash
DENO_VERSION=1.24.0

docker build -t deno-build --build-arg DENO_VERSION="${DENO_VERSION}" --progress=plain --file ./Dockerfile.compile .
DENO_BUILD_CONTAINER="$(docker create deno-build)"
docker cp "${DENO_BUILD_CONTAINER}:/deno/target/release/deno" .
docker rm "${CONTAINER_ID}"
```

### How does it work?

Originally this project compiled Deno on [GitHub Actions](https://github.com/features/actions) using QEMU to emulate ARM on x86, this would use 8GB+ of RAM and take hours (usually around 3 hours).

In recent Deno releases the memory requirements under QEMU compilation have exceeded the limits of free GitHub Actions and I have resorted to compiling on my ARM-based M1 MacBook Air, which was added as a [self-hosted GitHub Actions Runner](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners). This brings compile time for an ARM64 release down to 10-20 minutes.

### But I just want a Deno binary for my Raspberry Pi!

You can download the latest Deno binaries from the [Releases](https://github.com/LukeChannings/deno-arm64/releases) page.
There should be a `deno-linux-arm64.zip` asset attached to each release.

Once installed, the `deno upgrade` command will download the latest build from this repository. 

## FAQ

### Why isn't there official support?

The Deno team are waiting for ARM64 GitHub Actions runners:

> We use GitHub Actions exclusively. We are working with GitHub on the potential of their ARM64 support.
> 
> &mdash; [@kitsonk](https://github.com/denoland/deno/issues/1846#issuecomment-725209062)

At the moment [GitHub Actions Virtual Environments](https://github.com/actions/virtual-environments) are x86 only &mdash; although you can provide your own runners.

The QEMU-based builds take a long time, up to 2 hours just for compiling, and since Deno's CI runs on each push it needs to be fast.

You can follow the issue [here](https://github.com/denoland/deno/issues/1846).

### What about 32-bit ARM?!

Docker's buildx uses [QEMU](https://en.wikipedia.org/wiki/QEMU) to emulate ARM on x86.

Unfortunately there are bugs related to QEMU and 32-bit ARM that prevent compilation. 
You can read more [here](https://bugs.launchpad.net/qemu/+bug/1805913).

As such, compiling for 32-bit ARM needs to be done on a 32-bit ARM computer,
and because these systems are typically underpowered,
compiling may take a prohibitively long time 😬.

### Why can't you just cross-compile?

In order to speed up startup time Deno builds a V8 bytecode snapshot for its JavaScript runtime.
These snapshots are architecture-specific, and will cause a crash at runtime if the architecture isn't the same.

For example, when Deno is cross-compiled to ARM64 from an x86_64 architecture, the resulting binary will be ARM64.
However, because the [`build.rs`](https://github.com/denoland/deno/blob/master/cli/build.rs#L52) snapshot was generated on an x86 host, the snapshot will be x86 bytecode, and that causes a crash at runtime.

I tried to work around this by patching Deno to disable the compiler snapshot, but there was still a runtime crash with a difficult-to-debug stacktrace, so I think the rabbit hole goes deeper.

I don't think patching Deno is a viable option, since a future change to Deno could cause another kind of incompatibility, and I'd have to maintain a patch. For the time being emulation is a reasonable (if a bit slow) solution to this problem.

Further, Deno depends on [`rusty_v8`](https://github.com/denoland/rusty_v8), which takes a long time to compile in normal conditions, let alone under emulation.

At time of writing `rusty_v8` has problems publishing a pre-compiled ARM64 binary, and so Deno requires being compiled with `V8_FROM_SOURCE=1`, as well as a number of additional dependencies.

I have forked `rusty_v8` for this project and have been able to successfully cross-compile `rusty_v8` for ARM64. The standalone binaries can be found [here](https://github.com/lukechannings/rusty_v8/releases).


> The cross compile situation is very complicated. It all revolves around snapshots. When v8 loads Javascript code into memory it essentially compiles it to bytecode(bit of a simplification here) hence v8 being a JIT compiler. On a x86 platform running a native executable this means that the bytecode is x86 bytecode. When we create a snapshot of v8 state it mostly includes this bytecode. If we are cross compiling this snapshot is generated by running a native executable ('build.rs') thus the snapshot generated is native to the compiler host and is not compatible with the platform we are compiling for. If we can either find a way to execute a arm native version of the snapshot generator at compile time or disable snapshots for cross compiles(my previous solution), we should be able to cross compile for arm64.
> 
> &mdash; [@afinch7](https://github.com/denoland/deno/issues/4862#issuecomment-711110480)

### Why do you care about building an ARM64 binary?

I'm using Deno for one of my projects &mdash; [MovieMatch](https://github.com/LukeChannings/moviematch) &mdash; that has a requirement to provide standalone binaries for popular architectures, as well as a multi-architecture Docker image.

If you're interested in how I build standalone binaries for these architectures, check out the build file [here](https://github.com/LukeChannings/moviematch/blob/main/Justfile#L69).


### How do you run the GitHub actions runner on macOS?

```
docker run --name github-runner --rm \
  -e REPO_URL="https://github.com/LukeChannings/deno-arm64" \
  -e RUNNER_NAME="deno-arm64-runner" \
  -e RUNNER_TOKEN="${RUNNER_TOKEN}" \
  -e RUNNER_WORKDIR="/tmp/deno-arm64" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/deno-arm64:/tmp/deno-arm64 \
  myoung34/github-runner:latest
```

If the `myoung34/github-runner` image is not latest, sometimes the container will restart infinitely.
Killing it and pulling the latest fixes the problem.
