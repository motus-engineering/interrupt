---
title: STM32 Zephyr Reference App in a Docker Devcontainer
description:
  The post introduces and guides you through setting up a Docker Devcontainer for a Zephyr project on an STM32 device.
author: andrew
tags: [zephyr, build-system]
---

In the past, embedded system development environments were only provided by vendors such as Keil, IAR, CodeWarrior or chip vendors like Microchip or Atmel. These systems were only available on Windows. Many useful embedded tools were also only available on Windows as well. Somehow that paradigm of using a Windows host persisted even with the advent of Ecplise based IDEs. Flash forward to today and you see VSCode making heavy inroads as the preferred editor of choice in embedded. At the same time, the industry is eschewing vendor out of the box build systems as they don't support automated builds and tests well. Tools like Github Actions, or Circle CI are being adopted. What all this means, is more flexibility is needed in the embedded space, just like application or web software has enjoyed for years.

7 years ago I started pursuing this more flexible development environment using docker containers. In January of 2022 Microsoft started the devcontainer [spec on github](https://github.com/devcontainers/spec), which is also now integrated into VSCode. This has made using devcontainers more mainstream for software development and real joy to develop embedded firmware with. Of course about a year ago, I began using Zephyr and lamented not having a usable devcontainer at hand. So, I set out to build one with the hopes of sharing it back to the community.

<!-- excerpt start -->

This post introduces a Zephyr reference app for an STM32 device, which leverages a devcontainer within VSCode and Docker. It lays the foundation for creating a development environment that won't pollute your host machine and ensures everyone's build and the automated CI builds are the same thing, every single time.

<!-- excerpt end -->

After reading this post, you will be able to easily get a new Zephyr app up and running by using the reference repo in Github as a starting point. You will also be able to run a Zephyr app on an STM32 Cortex-M device.

{% include newsletter.html %}

{% include toc.html %}

## Reference Project Introduction
Let's get right to it. The reference project can be found on Github here:

```bash
https://github.com/motus-engineering/zephyr-app
```

The README file will take you step by step through how to setup the environment, compile, program and debug the app. This article will walk through the various pieces of the setup in more detail and provide insights that might not otherwise be obvious from just the README.

## Overview
The key software required to make all this work are [WSL2](https://learn.microsoft.com/en-us/windows/wsl/setup/environment), [Docker](https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-containers), [VSCode](https://code.visualstudio.com/) and [USBIPD](https://github.com/dorssel/usbipd-win).

The following diagram helps visualize how all the pieces fit together. Throughout the article it can be useful to come back to this to better help digest what is going on.

<p align="center">
  <img src="{% img_url stm32-zephyr-devcontainer/block-diagram.drawio.svg %}" alt="Block Diagram" />
</p>

## Devcontainers and VSCode Integration
### Container Image
The central piece to all of this is the container. It provides the consistent environment for anyone working on your project. It ensures repeatability and avoids issues with developer's machines being different. The integration VSCode provides for containers is through the `devcontainer.json` file found in:

```bash
.vscode/devcontainer.json
```
This file allows you to specify what container image to use, how it should be configured, and run. For a Zephyr project, the CI image from [https://github.com/zephyrproject-rtos/docker-image](https://github.com/zephyrproject-rtos/docker-image) is leveraged. The CI image provides all the necessary dependencies for a Zepher project.

Inside the `devcontainer.json` file you will see a Dockerfile is specified to build the container image along with a Zephyr container version argument.

```json
"build": {
  "dockerfile": "Dockerfile",
  "args": { "ZEPHYR_CONTAINER_VERSION": "v0.26.5"}
}
```

By providing a Dockerfile, shown below, it is possible to use the latest Zephyr CI image or the version provided from the devcontainer file. The Dockerfile also allows you to customize the environment beyond what the Zephyr CI image provides if needed. For example, the Python `semver` module is installed using the `RUN` command below. This approach allows you to control when you move to newer Zephyr versions in your development environment and any other dependencies your project needs.

```docker
ARG ZEPHYR_CONTAINER_VERSION="latest"
FROM ghcr.io/zephyrproject-rtos/ci:${ZEPHYR_CONTAINER_VERSION}

USER user

RUN pip install semver
```

### Container Run Arguments
The `devcontainer.json` file supports customizing how the image is run through the `runArgs` parameter. Only one run argument is currently necessary to ensure USB debugging is possible.

```json
	"runArgs": ["--privileged"]
```

Obviously, running a container in privileged mode is not a secure practice for things like web applications, but in this case it is useful to access our USB debugger easily.

### Shared Mounts
The `devcontainer.json` file also allows you to specify mounts between WSL and the container. A few shared mounts are necessary to support debugging.

```json
	"mounts": [
		// Allows for USB devices to be recognized when attached after the devcontainer is running
		{"type": "bind", "source": "/dev/bus/usb", "target": "/dev/bus/usb"},
		// Allows USB STLink debuggers to be accessed properly
		{"type": "bind", "source": "/etc/udev/rules.d", "target": "/etc/udev/rules.d"},
		// General Share folder
		{"type": "bind", "source": "${localEnv:HOME}/share", "target": "/mnt/share"}
	]
```

1. Sharing `/dev/bus/usb` allows the container to recognize the USB debugger even if it is attached after the container is running.
1. Sharing the udev rules allows the container to access the USB debugger without `sudo`.
1. A generic share folder is created as a helper location. It is used in a script described later to install the ST Command Line Tools.

### Lifecycle Scripts
When starting up a container, you can use [lifecycle scripts](https://containers.dev/implementors/json_reference/#lifecycle-scripts) to customize various aspects of the environment.

As a convenience, the `postCreateCommand` runs `west init` once the devcontainer is created and started if needed.

```json
	"postCreateCommand": {
		"west_init": "if ! [ -d .west ]; then west init -l app; fi" // run west init if not done already
	},
```

When using a lifecycle script, it is important to know when and where in the startup of the container it is executing. For example, `initializeCommand` is run on the host machine during initialization, while the `onCreateCommand` is run inside the container immediately after it has started for the first time.

### VSCode Customizations
The last section in the `devcontainer.json` file allows for customizations to the VSCode editor itself. This is used to setup specific toolchain paths, file associations, formatting tools, and install required extensions. This is great way to ensure common IDE setups for critical things like coding or styling standards are used by everyone.

```json
	// Configure tool-specific properties.
	"customizations": {
		// Configure properties specific to VS Code.
		"vscode": {
			// Set *default* container specific settings.json values on container create.
			"settings": {
				"cortex-debug.armToolchainPrefix": "arm-zephyr-eabi",
				"cortex-debug.armToolchainPath": "/opt/toolchains/zephyr-sdk-0.16.3/arm-zephyr-eabi/bin/",
				"files.associations": {
					"*.h": "c",
					"*.c": "c"
				},
				"C_Cpp.clang_format_style": "file:${containerWorkspaceFolder}/.vscode/.clang-format",
				"C_Cpp.clang_format_path": "/usr/local/bin/clang-format", // use the system version supplied with container
				"cmake.configureOnOpen": false
		},
			// Add the IDs of extensions you want installed when the container is created.
			"extensions": [
				"ms-vscode.cpptools-extension-pack",
				"marus25.cortex-debug",
				"github.vscode-github-actions",
				"GitHub.vscode-pull-request-github",
				"cschlosser.doxdocgen"]
		}
	}
```

## Git Configuration
In order to make the git workflow smooth, you can configure WSL to use the Git Credential Manager installed on your Windows host. This only needs to be done once per WSL distro instance you have, and you may only ever have one. Note also, that you should `git clone` any repo you are working with inside your WSL file space. If you place the repo in the Windows file space (e.g. `C:\my_repo`), there will be a significant [performance](https://learn.microsoft.com/en-us/windows/wsl/filesystems#file-storage-and-performance-across-file-systems) hit that will noticeably slow your builds down.

The README file directs you to set this up by first installing Git for Windows, then in WSL, configure your global name and email and login to github with the Github CLI tool `gh`.

Doing all this ensures your credentials will also be available inside any devcontainer you subsequently run from that WSL instance. Features in VSCode like Source Control, or Extensions like GitHub Actions and GitHub Pull Requests will now work properly along with `git` on the command line.

## West Workspace
Zephyr's meta tool `west` supports three project [topologies](https://docs.zephyrproject.org/latest/develop/west/workspaces.html#topologies-supported). This project uses the T2: Star topology, where the application is the manifest repository as shown below.

```
west-workspace/
│
├── app/                 # your .git project
│   ├── CMakeLists.txt   # project CMake file
│   ├── prj.conf         # never modified by west
│   ├── main.c           # application main()
│   └── west.yml         # main manifest with required modules and zephyr repositories
│
├── modules/
│   └── cmsis/           # .git/ project from either the main manifest or some import.
│   └── hal_stm32/       # .git/ project from either the main manifest or some import.
│
└── zephyr/              # .git/ project
    └── west.yml         # This can be partially imported with lower precedence or ignored.
                         # Only the 'manifest-rev' version can be imported.
```

## Configuring and Building Firmware

At this point, the reference app is ready to be configured and built using Zephyr's `west` tool.

As noted earlier, the devcontainer file has already run `west init` if needed for you. Use the following commands to finish the configuration and build.

```bash
west update
```

`west update` will use the `app/west.yml` file to download the required module and Zephyr repositories. As this is an STM32 project, the west file narrows the list of imported modules to `cmsis` and `hal_stm32` only. This keeps your working directory distraction free of unneeded modules. The `west.yml` file also allows you to specify the version of Zephyr you want to use.

```bash
west build -d build -b nucleo_l476rg app
```

`west build` does the actual work to finally give you the that `.elf` file we have been working towards.
- `-d build` specifies to put all the build artifacts in a folder called `build`.
- `-b nucleo_l476rg` specifies to build the app for an [STM Nucleo-64 STM32L476RG](https://www.st.com/en/evaluation-tools/nucleo-l476rg.html) development board.
- `app` specifies your application to build (i.e. the folder where the application lives with a `west.yml` file)

## Debugging
In all my travails with containerized development, this was a blocking point for some time. It was next to impossible to get the USB debugger exposed properly to the container. However, [dorssel](https://github.com/dorssel) created the [usbipd-win](https://github.com/dorssel/usbipd-win) project and Microsoft contributed to help make it work with WSL2. Finally, there was a solution to get a USB debugger working with a container.

That said, there are still a few hurdles before we can hit that magic debug button in VSCode.

First, you need to create the `udev` rules in your WSL instance for non-root access USB debuggers. As this is an STM32 focused project, I chose to support STLINK. The easiest way to do this is by running:

```bash
sudo apt install stlink-tools
```

This handles setting up those `udev` rules for you. You only need to do this once per WSL instance. If you prefer to do this yourself, you will need add the appropriate `.rules` file for the debugger you are using. The following rules files are installed with `stlink-tools`.

```bash
ubuntu:/etc/udev/rules.d$ ls -l
-rw-r--r-- 1 root root   245 Jun 11  2021 49-stlinkv1.rules
-rw-r--r-- 1 root root   439 Jun 11  2021 49-stlinkv2-1.rules
-rw-r--r-- 1 root root   253 Jun 11  2021 49-stlinkv2.rules
-rw-r--r-- 1 root root  1368 Jun 11  2021 49-stlinkv3.rules
```

In the earlier devcontainer [Shared Mounts](#shared-mounts) section, we laid the groundwork for these `udev` rules to work with containers by sharing them with the container from WSL.

Second, the needed tools (e.g. `gdbserver`) to support STLINK debugging need to be installed inside the container. The choice of supporting STLINK debuggers comes with some issues in fully automating the setup inside the container, namely that ST has you agree to their waivers when downloading tools. As a result, this step is partially manual.

Download the [STM32CubeCLT](https://www.st.com/en/development-tools/stm32cubeclt.html) package from ST. From your Windows context, it should be placed in `\\wsl.localhost\Ubuntu\home\<username>\share`. Because one of the previous shared mounts specified in the `devcontainer.json` file, the downloaded file will be available in your container as well at `/mnt/share`.

From the bash terminal of your container in VSCode, running the following custom `west` script handles unpacking the archive and installing the needed pieces.

```bash
west st-clt /mnt/share/<SMT32CubeCLT filename>
```

The script is located at `app/scripts/west_commands_st-clt.py` if you are interested in the details of what is happening.

Once done, you can now plug in your STLINK debugger, [bind it and attach it](https://learn.microsoft.com/en-us/windows/wsl/connect-usb#attach-a-usb-device) with `usbipd` in PowerShell, then start your debugging session in VSCode leveraging the [Cortex-Debug](https://marketplace.visualstudio.com/items?itemName=marus25.cortex-debug) extension from marus25 that was automatically added to VSCode for you from the list of extensions in the `devcontainer.json` file.

You can review the debugging configuration in `.vscode/launch.json`. In this file you will also note there is a JLink configuration which can be used if that is your preferred debugger. You will need to install the JLink `udev` rules and drivers similarly to how the STMCubeCLT was done.

## Additional Project Features
I endeavour this reference project to provide useful features for modern embedded development. You will find below some brief introduction to those features such as unit testing, code formatting and a CI Pipeline using Github Actions. These will evolve over time incorporating feedback and new features aiming to be a go to for the embedded community as a "how to". Your input and contributions are encouraged and welcome.

### Example Application
State Machine
STM32 specifics

### Unit Tests and Code Coverage
The reference project contains an example unit test found `/tests/supervisor`.

Unit tests leverage the Zephyr `ZTEST` framework and the `native_posix` board. Each test is thus compiled and run natively. This ensures fast unit tests locally and also the ability to run them as part of a CI build.

Unit tests are run using `twister` provided with Zephyr, which can also be run through `west`. However, just running unit tests is not good enough. We also want to know our test coverage. To accomplish this a custom `west` command was scripted.

```bash
west unit
```

This command handles running `twister` along with the needed options to generate a code coverage report which can be found in `build/twister/coverage/`. The report is generated in `xml` and text format. It will even be added to a Pull Request.

### Code Formatting
Any good project will have consistent styling. There is no better way to ensure this happens on a project than with formatting tools. VSCode comes with built in clang-format support, so this reference project uses that. The `.clang-format` file is located in `.vscode/`. You can customize this file to match your style preferences.

### CI Pipeline With Github Actions and devcontainers
One of the big advantages for using devcontainers is the benefit that your local environment is the same as the CI build. 

This project contains a Github Actions Workflow file at `.github/workflows/ci.yml`.

The workflow will use the Zephyr CI docker image and perform the following:

1. Format check using the clang-format file
1. Configure and build the firmware using west
1. Run unit tests
1. If Pull Request, publishes the Code Coverage report to the PR
1. If nightly build, saves build artifacts
1. If release build, tags the release and publishes the build artifacts to a Release on Github

## Conclusion
TBD

## Where to go next
* further resources
* planned next steps for the project
* how someone can contribute

<!-- Interrupt Keep START -->
{% include newsletter.html %}

{% include submit-pr.html %}
<!-- Interrupt Keep END -->

{:.no_toc}

## References

<!-- prettier-ignore-start -->
[^reference_key]: [Post Title](https://example.com)
<!-- prettier-ignore-end -->
