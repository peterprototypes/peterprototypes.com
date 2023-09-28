+++
title = "Rust Dockerfile Boilerplate"
date = 2023-09-28
description = "A Dockerfile boilerplate for quickly building size optimized images with cached dependencies"
+++

At this point, Docker has become so deeply entrenched in our workflows, in order to remove it we need to format the planet, become fish, and try this evolution business again.

![one thousand forty floppy drives](cover.png)

In this post, I'll share a Dockerfile boilerplate for quickly building slim images. We achieve this via caching and [multi-stage builds](https://docs.docker.com/build/building/multi-stage/) builds. Usually, you might start with the following Dockerfile:<!-- more -->

```Dockerfile
FROM rust:latest
WORKDIR /app
COPY . .
RUN cargo build --release
CMD ["target/release/my-app"]
```

This works and has the added advantage of being short and simple to understand (an underrated trait these days). Unfortunately, it will download and compile all dependencies each time the image is built. It will also contain the build cache and rust toolchain.

Just the `FROM rust:latest` line is equivalent to one thousand forty floppy drives:
```bash
$ docker inspect rust:latest | jq '.[0].Size' | numfmt --to=iec-i --suffix=B
1.5GiB
```

Compiling an empty application with `actix-web` as a single dependency bumps that up one and a half thousand floppy drives:
```bash
$ docker run -w /app rust:latest /bin/bash -c "cargo init && cargo add actix-web && cargo build"
    Created binary (application) package
    Updating crates.io index
    ✂️ snip ✂️
    Compiling app v0.1.0 (/app)
    Finished dev [unoptimized + debuginfo] target(s) in 10.35s
$ docker inspect --size 551e1c78b939 | jq '.[0].SizeRootFs' | numfmt --to=iec-i --suffix=B
2.1GiB
```

Shipping that is not good craftsmanship. If you're using a registry to store build images it will take longer to transfer them between CI runners and deployment environments. The time it takes a commit to reach a dev server can kill a productive developer-qa pair working on an issue. It may also incur more [storage and data transfer costs](https://aws.amazon.com/ecr/pricing/).

This can be solved by using [multi-stage builds](https://docs.docker.com/build/building/multi-stage/). We can use the big fat `rust:latest` image to build our application, and then only copy the resulting binary to a second slim image. For now, I've picked up `debian:12-slim` for that. We'll build the dependencies early in the Dockerfile in an empty Rust project. This will cache them and speed up subsequent runs.

Here's how the resulting [`Dockerfile`](./Dockerfile.txt) looks:
```Dockerfile
# Use a base image with the latest version of Rust installed
FROM rust:latest as builder

# Set the working directory in the container
WORKDIR /app

# Create a blank project
RUN cargo init

# Copy only the dependencies
COPY Cargo.toml Cargo.lock .

# A dummy build to get the dependencies compiled and cached
RUN cargo build --release

# Copy the real application code into the container
COPY . .

# Build the application
RUN cargo build --release

# (Optional) Remove debug symbols
RUN strip target/release/example-app

# Use a slim image for running the application
FROM debian:12-slim as runtime

# Copy only the compiled binary from the builder stage to this image
COPY --from=builder /app/target/release/example-app /bin/example-app

# Specify the command to run when the container starts
CMD ["/bin/example-app"]
```

The produced image is only 4% the size of the one used for building:

```bash
$ docker inspect 261be2 | jq '.[0].Size' | numfmt --to=iec-i --suffix=B
81MiB
```

### Over Doing It

Let's say you *need* more. Only plebs can be satisfied with a 96% improvement. Let's see how much going to [Alpine](https://www.alpinelinux.org/) Linux for a runtime gives you.

We have to introduce cross-compilation in our builder stage since Alpine ships with [musl libc](https://www.musl-libc.org/) for a standard library. Even their website is skinny at 18kb.

Here's the Dockerfile needed to make that happen:
```Dockerfile
# Use a base image with the latest version of Rust installed
FROM rust:latest as builder

# Set the working directory in the container
WORKDIR /app

# Install the linux-musl build target
RUN rustup target add x86_64-unknown-linux-musl

# Create a blank project
RUN cargo init

# Copy only the dependencies
COPY Cargo.toml Cargo.lock .

# A dummy build to get the dependencies compiled and cached
RUN cargo build --target x86_64-unknown-linux-musl --release

# Copy the real application code into the container
COPY . .

# Build the application
RUN cargo build --target x86_64-unknown-linux-musl --release

# (Optional) Remove debug symbols
RUN strip target/x86_64-unknown-linux-musl/release/example-app

# Use a slim image for running the application
FROM alpine as runtime

# Copy only the compiled binary from the builder stage to this image
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/example-app /bin/example-app

# Specify the command to run when the container starts
CMD ["/bin/example-app"]

```

Let's see how we did:
```bash
$ docker inspect --size 7f3d27543e29 | jq '.[0].SizeRootFs' | numfmt --to=iec-i --suffix=B
7.5MiB
```

Woah, that's a big drop! From 2.1GiB to 7.5MiB. Over 99.7% improvement 🤝 🥇 🥇 🤝 (pat self on the back emoji)

But that's not entirely fair. Let's include `actix-web` as a dependency:
```bash
$ cargo add actix-web && docker-compose build
✂️ snip ✂️
5.885   running: "musl-gcc" "-O3" "-ffunction-sections" "-fdata-sections" "-fPIC" "-m64" "-I" "zstd/lib/" "-I" "zstd/lib/common" "-I" "zstd/lib/legacy" "-fvisibility=hidden" "-DZSTD_LIB_DEPRECATED=0" "-DXXH_PRIVATE_API=" "-DZSTDLIB_VISIBILITY=" "-DZDICTLIB_VISIBILITY=" "-DZSTDERRORLIB_VISIBILITY=" "-DZSTD_LEGACY_SUPPORT=1" "-o" "/app/target/x86_64-unknown-linux-musl/release/build/zstd-sys-d5ce7566f728ee39/out/zstd/lib/common/debug.o" "-c" "zstd/lib/common/debug.c"
5.885 
5.885   --- stderr
5.885 
5.885 
5.885   error occurred: Failed to find tool. Is `musl-gcc` installed?
5.885 
5.885 
------
failed to solve: process "/bin/sh -c cargo build --target x86_64-unknown-linux-musl --release" did not complete successfully: exit code: 101
```

Sigh. So not only do we have to cross-compile our Rust application, but we also have to cross-compile every single C/C++ dependency in our tree. At this point, you should step back from the keyboard and keep your hands where the over-optimization police can see them. You have to weigh the benefits of those last 70MiB you're trying to shave against the complexity you're introducing. You have to decide, after achieving 96% improvement, if those last few megabytes are worth your time.

To fix the above error, we need to install `musl-tools` in our build stage:

```Dockerfile
...
WORKDIR /app

# Install C/C++ musl toolchain
RUN apt-get update && apt-get install -y musl-tools

# Include the musl target
...
```

I had to add a minimal `actix-web` main function. Otherwise, it gets optimized out. The final image size is 13MiB:
```bash
docker inspect --size 5e2be5ccb691 | jq '.[0].SizeRootFs' | numfmt --to=iec-i --suffix=B
13MiB
```

Sometimes it's also worth checking if you can remove C/C++ dependencies via feature flags. E.g. from `native-tls` to `rustls-tls`:
```diff
- reqwest = { version = "0.11", features = ["native-tls"] }
+ reqwest = { version = "0.11", features = ["rustls-tls"] }
```

Here's the final Alpine [`Dockerfile`](./Dockerfile-cross.txt) with cross-compilation support for both Rust and C/C++:
```Dockerfile
# Use a base image with the latest version of Rust installed
FROM rust:latest as builder

# Set the working directory in the container
WORKDIR /app

# Install C/C++ musl toolchain (a lot of crates may need "clang" as well)
RUN apt-get update && apt-get install -y musl-tools

# Install the linux-musl build target
RUN rustup target add x86_64-unknown-linux-musl

# Create a blank project
RUN cargo init

# Copy only the dependencies
COPY Cargo.toml Cargo.lock .

# A dummy build to get the dependencies compiled and cached
RUN cargo build --target x86_64-unknown-linux-musl --release

# Copy the real application code into the container
COPY . .

# Build the application
RUN cargo build --target x86_64-unknown-linux-musl --release

# (Optional) Remove debug symbols
RUN strip target/x86_64-unknown-linux-musl/release/example-app

# Use a slim image for running the application
FROM alpine as runtime

# Copy only the compiled binary from the builder stage to this image
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/example-app /bin/example-app

# Specify the command to run when the container starts
CMD ["/bin/example-app"]
```

### References

- [Docker Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [ECR Pricing](https://aws.amazon.com/ecr/pricing/)
- [Alpine Linux](https://www.alpinelinux.org/)
- [musl libc](https://www.musl-libc.org/)