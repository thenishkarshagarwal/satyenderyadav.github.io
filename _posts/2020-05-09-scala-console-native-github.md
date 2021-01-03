---
title: Scala to native code with GraalVM and GitHub Actions
date: 2020-05-09 18:42:00 -0500
categories: [Continuous Integration]
tags: [Scala, GraalVM, GitHub, CICD, JVM]
image:
  path: /assets/img/2020-05-09-scala-console-native-github/banner.png
seo:
  date_modified: 2020-05-20 13:06:32 -0400
---

If you've ever wanted to write a fancy new console app in Scala, but found yourself torn between engaging your innate desire to write functionally beautiful code, and the perversion of handing your users a JAR file, this post is for you.

Using [GraalVM](https://github.com/oracle/graal)'s `native-image` tool and [GitHub Actions](https://github.com/features/actions), we'll see how to configure a CI/CD workflow to automatically cross-compile a typical Scala console application to native executables for macOS, Linux, and Windows.

This approach assumes your project is on GitHub.

![Logos for GraalVM, GitHub Actions, and Scala united around a native executable icon]({{ "/assets/img/2020-05-09-scala-console-native-github/banner.png" | relative_url }})

### Step overview
1. Create a Scala project, using SBT.
2. Add the `sbt-assembly` plugin. See [the official SBT Assembly setup instructions](https://github.com/sbt/sbt-assembly#setup) for details.
3. Use GitHub Actions Workflow to invoke GraalVM's `native-image` on macOS, Linux, and Windows to transform the JAR into a native executable for each (see below).
4. (optional) Upload native binaries as a GitHub Release.

# GraalVM
Before we continue, a bit of background on GraalVM:

> [GraalVM](https://github.com/oracle/graal) is a universal virtual machine for running applications written in JavaScript, Python, Ruby, R, JVM-based languages like Java, Scala, Clojure, Kotlin, and LLVM-based languages such as C and C++.

Most notably for our purposes, it also includes a tool known as `native-image`.

> [GraalVM Native Image](https://www.graalvm.org/docs/reference-manual/native-image/) allows you to ahead-of-time compile Java code to a standalone executable, called a native image. This executable includes the application classes, classes from its dependencies, runtime library classes from JDK and statically linked native code from JDK.

Tl;dr `native-image` is capable of compiling Java bytecode to a standalone executable (no JVM required!) for macOS, Windows, and Linux.

Typically, you can expect the resulting binary to be around `10 MB` in size for a Scala console application. Compared to a compiled C program, this is quite large, but (IMO) not unreasonable.

# GitHub Actions
If you haven't heard of it, GitHub Actions is a [CI/CD](https://en.wikipedia.org/wiki/CI/CD) platform, which allows you to build, test, release (you name it) your code in response to platform events/triggers (such as new code getting pushed).

In our case, we need it to build our Scala project into a JAR, and then to invoke `native-image` for each platform. 

One limitation of `native-image` is that it's only able to produce native executables for the host on which it's installed. Luckily, GitHub Actions supports multi-platform builds.

In the sections to follow, we'll look at some code snippets from a [sample YAML GitHub Actions workflow definition which builds a Scala project to native code for each platform, using GraalVM](https://github.com/kevinhartman/srb2-soctool/blob/master/.github/workflows/ci.yml). This sample comes from a small (and fun!) Scala console app called [soctool](https://github.com/kevinhartman/srb2-soctool).

## Build Scala as JAR
{% raw  %}
```yml
jobs:
  build:
    name: sbt assembly
    runs-on: ubuntu-latest

    # Perform this job's steps inside a JDK 11 container, with SBT installed.
    container:
      image: eed3si9n/sbt:jdk11-alpine
    steps:
    # Check-out project sources.
    - uses: actions/checkout@v2

    # Use sbt-assembly to build, run unit tests on, and package the result as a fat JAR
    # named './target/soctool.jar'
    - name: sbt assembly
      run: sbt 'set assemblyOutputPath in assembly := new File("./target/soctool.jar")' assembly

    # Upload the JAR as an intermediate artifact, which can be consumed by subsequent jobs.
    - uses: actions/upload-artifact@v2
      with:
        path: target/soctool.jar
```
{% endraw %}

The snippet above defines a new job `build` that uses a Docker container to build the project's Scala sources into a fat JAR, which is then uploaded as an intermediate artifact.

With the JAR uploaded, we can consume it from parallel `native-image` jobs, one for each platform.

## Cross-compile to native macOS and Linux
{% raw  %}
```yml
  release_nix:
    name: native-image-nix

    # only create native executables for Git tag triggers (1)
    if: startsWith(github.ref, 'refs/tags/')

    # declare dependency on JAR build.
    needs: build

    # run this job on the current template instantiation
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        # instantiate this job twice, once for each OS
        os: [ubuntu-18.04, macos-10.15]
    steps:

    # install GraalVM, using a community GitHub Action
    - uses: DeLaGuardo/setup-graalvm@3
      with:
        graalvm-version: '20.0.0.java11'

    # use GraalVM's package manager to add native-image extension
    - name: Install GraalVM's native-image extension
      run: gu install native-image

    # download the JAR from job 'build'
    - uses: actions/download-artifact@v2
      with:
        path: ./

    # invoke native-image on the JAR, transforming it to native executable 'soctool'
    - name: Create native soctool
      run: native-image --verbose -jar ./artifact/soctool.jar soctool

    # tar-up the executable
    - name: Create tarball
      run: tar -zcvf "soctool-${{ matrix.os }}.tar.gz" soctool

    # upload it to the release corresponding to the current Git tag (2)
    - name: Create GitHub release
      uses: softprops/action-gh-release@v1
      with:
        files: 'soctool-${{ matrix.os }}.tar.gz'
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
{% endraw %}

The snippet above handles invoking `native-image` on our JAR for macOS and Linux (Windows is handled separately, below).

Other than what's noted as inline comments, there are a few things to call out:

1. We only run this job if the event that triggered it was the push of a Git tag. I tend to find this to be a clean approach for simple projects: any tag is considered to be a release, and thus should trigger the release mechanism.
2. We use `softprops/action-gh-release` instead of the official GitHub release Action, since it is idempotent by tag. If it's invoked multiple times within the same build (i.e. by each native target), only a single release will be created (with the name of the tag, by default), but the artifacts of each invocation will be appended to the release.

## Cross-compile for Windows
{% raw  %}
```yml
  release_win:
    name: native-image-win

    # same as before, only run for Git tags
    if: startsWith(github.ref, 'refs/tags/')

    # declare dependency on JAR build (1)
    needs: build
    runs-on: windows-2019
    steps:
    - uses: DeLaGuardo/setup-graalvm@3
      with:
        graalvm-version: '20.0.0.java11'

    # we need to provide the full path, plus a .cmd extension
    # to the 'gu' utility (2)
    - name: Install GraalVM's native-image extension
      run: ${{ env.JAVA_HOME }}\bin\gu.cmd install native-image

    # download the JAR from job 'build'
    - uses: actions/download-artifact@v2
      with:
        path: ./

    # configure native x64 dev env, and invoke native-image on JAR (3).
    - name: Create native soctool
      run: >-
        "C:\Program Files (x86)\Microsoft Visual Studio\2019\Enterprise\VC\Auxiliary\Build\vcvars64.bat" &&
        ${{ env.JAVA_HOME }}\bin\native-image.cmd --verbose -jar .\artifact\soctool.jar soctool
      shell: cmd

    # zip-up the executable ("zip is hip" for windows)
    - name: Create zip
      run: 7z a soctool-windows.zip soctool.exe

    # upload it to the release
    - name: Create GitHub release
      uses: softprops/action-gh-release@v1
      with:
        files: 'soctool-windows.zip'
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
{% endraw %}

Finally, the above snippet of course does the same for Windows. This is done separately, since GraalVM's toolchain operates a little differently here.

A few call-outs:

1. Even though this is an entirely separate job from the matrix-based Linux and macOS job(s), it'll still run in parallel with each of them. This is because it only declares dependency on job `build`.
2. GraalVM's Windows tools don't seem to work in the PATH, so we must supply the full path for each invocation. Additionally, we append `.cmd` for each, since they are wrapper batch files.
3. On Windows, `native-image` needs to know about the Visual C++ compiler toolchain. We invoke `vcvars64.bat` to accomplish this first, which prepares the current environment for its use. Note that this time, the resulting executable name will ***automatically*** be terminated with `.exe`.

## Result
![The sample project's GitHub release page, showing resulting binaries for macOS, Windows, and Linux]({{ "/assets/img/2020-05-09-scala-console-native-github/release.png" | relative_url }})

Now, by pushing a Git tag, we'll automatically get a GitHub Release with native executables for macOS, Windows, and Linux :)

# Quick start
To do this for your own Scala console app, I'd recommend grabbing the [sample workflow](https://github.com/kevinhartman/srb2-soctool/blob/master/.github/workflows/ci.yml) YAML, and customizing it for your project.