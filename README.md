# dobe

## A dockerized OpenWrt build environment

An isolated and fully self-contained Docker-based environment for operating the [OpenWrt build system][wiki-build-system].

The image is based Ubuntu 20.04 LTS and runs Bash shell with autocompletion by default.

The project source code can be found at <https://github.com/antichris/dobe>


## Prerequisites

- Docker (tested with 19.03+)
- A POSIX compliant shell environment
- GNU Coreutils (or an alternative providing `sha1sum`)


## Operation

### Building the image

Run the included [`bin/build` script][bin/build]:

```sh
bin/build
```

Or, if you cannot or wish not to use the script, you can manually build a Docker image with the [`docker/`] directory context:

```sh
docker build -t dobe:latest docker
```

If you are following the [OpenWrt's quickstart guide][wiki-quickstart], this has handled the [step 1 (prerequisites)][wiki-quickstart-s1] for you.


### Running the image

Run the [`bin/run` script][bin/run]:

```sh
bin/run
```

Alternatively, you can manually create a directory for the OpenWrt build system sources and execute the `docker run` command, mounting your local source directory (`build` in the following example) on `/src`:

```sh
mkdir build
docker run -it --rm -v "$(pwd)/build:/src" -e TERM="$TERM" dobe:latest
```

Note, though, that in this case the structure of your `build/` directory will not be [as described later on][`build/`], but will match that of the OpenWrt build system exactly.

It is also recommended that you pass your local `TERM` environment variable to the container for software running in there to figure out what terminal capabilities are supported.


### Building the firmware

Once your container is up and running a shell session, you should be in the default working directory for this Docker image, `/src`, which is intended for the OpenWrt sources. If for any reason you are not, run `cd /src` to get there.

For the [quickstart step 2 (getting the sources)][wiki-quickstart-s2] simply execute

```sh
/pull
```

and the [script][/pull] shall handle everything for you, including updating and installing package feeds. Running the above command in an empty directory (as it is when running the Docker image for the first time) will pull the [`master` branch][wiki-bleeding-edge] by default, the latest "bleeding edge" development version — the same as done in the quickstart guide.

If you actually wish to pull a specific version (see [tags][repo-tags] and [branches][repo-heads] in the OpenWrt repository), for example, the `v19.07.3` tag, set the `REF` environment variable for `/pull`, like so:

```sh
REF=v19.07.3 /pull
```

How you go about [step 3 (configuring your firmware)][wiki-quickstart-s3] is entirely up to you — you can run

```sh
make menuconfig
```

and manually configure your targets and build options as the guide suggests, or you can copy a [prepared config diff][wiki-build-configdiff] to `.config` in your [OpenWrt sources directory][`src/`], followed by an execution of `make defconfig`.

This Docker image includes a [wrapper script][/make] for the `make` utility, that executes `make` multi-threaded with reduced `nice` and `ionice` priorities.

Before starting [step 4][wiki-quickstart-s4], the actual build, [it is recommended][wiki-build-download] that you prefetch the source code for all the dependencies:

```sh
/make download
```

Finally, you are ready to build your firmware. Execute

```sh
/make
```

You should be able to just continue using your machine as normal, since the `/make` script [takes steps to reduce the priority][/make] of make jobs as much as possible.


### Building the firmware without attaching a shell session

All of the above can be accomplished by issuing the commands each running in a separate disposable container. Here is how you would do that, following the [quickstart guide][wiki-quickstart]:

1. #### Build prerequisites

	```sh
	bin/build
	```

2. #### Get the OpenWrt sources

	```sh
	bin/run /pull
	```

3. #### Configure the build system

	```sh
	bin/run /make menuconfig
	```

4. #### Build the firmware

	```sh
	bin/run /make download
	bin/run /make
	```

	Or, executing both commands in a single container:

	```sh
	bin/run '/make download && /make'
	```


## Directory structure

Whenever the phrase "mounted as" is used, it is to mean that the local host directory is mounted by the [`bin/run` script][bin/run] to the respective directory in the Docker container, and thus its contents are those of the corresponding directory in the OpenWrt build system.

The parentheses show the environment variable that can be set to a different local host directory to override the default value for the `bin/run` script.

- [`bin/`] — Utility scripts for building and running images.
	- [`version/`] — Files used by the [`build` script][bin/build] for image tag versioning.
- [`docker/`] — Docker build context.
- [`build/`] (`DOBE_DIR_ROOT`) — Default local build root.
	- [`cache/`] (`DOBE_DIR_CACHE`) — OpenWrt build artifacts.
		- [`build/`][cache/build] (`DOBE_DIR_BUILD`) — OpenWrt dependency build directory. Mounted as `/src/build_dir`.
		- [`dl/`] (`DOBE_DIR_DL`) — Downloaded dependency source archives. Mounted as `/src/dl`.
		- [`staging/`] (`DOBE_DIR_STAGING`) — OpenWrt staging directory. Mounted as `/src/staging_dir`.
		- [`tmp/`] (`DOBE_DIR_TMP`) — OpenWrt build temporary files. Mounted as `/src/tmp`.
	- [`feeds/`] (`DOBE_DIR_FEEDS`) — OpenWrt package feeds. Mounted as `/src/feeds`.
	- [`output/`] (`DOBE_DIR_OUTPUT`) — Built firmware images and packages. Mounted as `/src/bin`.
	- [`overlay/`] (`DOBE_DIR_OVERLAY`) — Firmware file system overlay. Mounted as `/src/files`.
	- [`src/`] (`DOBE_DIR_SRC`) — OpenWrt sources directory. Mounted as `/src`.
	- [`ssh/`] (`DOBE_DIR_SSH`) — Persistent SSH Configuration. Mounted as `/home/dobe.ssh`.


### Utility scripts: `bin`

[`bin/`]: #utility-scripts-bin "Utility scripts: `bin`"


#### `build`

[bin/build]: #build "The `build` utility script"

Builds Docker images.

It also tags each new image as `dobe:latest` and `dobe:v#`, where `#` is a number, that is incremented whenever a change in the `build` script or any of the [Docker build context files][`docker/`] is detected.


#### `cleanup`

Removes all but the latest two (distinguishing by the `:v#` tag numbers) of `dobe` Docker images.


#### `run`

[bin/run]: #run "The `run` utility script"

Spins up a Docker container from the `dobe:latest` image and passes all arguments to Docker.

It detects current user and group and passes them on as environment variables to Docker, where the [`entrypoint` script][entrypoint] creates the corresponding objects in the container. This can be overridden via host environment variables `DOBE_ID_USER` and `DOBE_ID_GROUP`.

It also mounts the various host directories into the container, creating missing ones as necessary. They too can be overridden via environment variables. To change, for example, the [local build root][`build/`], set `DOBE_DIR_ROOT`:

```sh
DOBE_DIR_ROOT=~/dev/OpenWrt bin/run
```

The other directory variables are: [`DOBE_DIR_CACHE`][`cache/`], [`DOBE_DIR_DL`][`dl/`], [`DOBE_DIR_BUILD`][cache/build], [`DOBE_DIR_STAGING`][`staging/`], [`DOBE_DIR_TMP`][`tmp/`], [`DOBE_DIR_FEEDS`][`feeds/`], [`DOBE_DIR_OUTPUT`][`output/`], [`DOBE_DIR_OVERLAY`][`overlay/`] and [`DOBE_DIR_SRC`][`src/`].

Environment variables are a minimalistic compromise when compared to passing the values as command line arguments on every invocation and introducing a specialized configuration system in a shell script.


#### Versioning: `version`

[`version/`]: #versioning-version "Versioning: `version`"

- `increment` script — Used by the [`build` script][bin/build] when generating version tags for the Docker images.
- `version.sum` — Stores the SHA-1 hash of hashes for the files relevant to image versioning.
- `version` — Stores current version number.

On every run, the `increment` script uses `sha1sum` to generate a list of SHA-1 hashes for the `build` script and the Docker build context files, and then a hash of this list. If the generated hash does not match the one stored in `version.sum`, it stores the new value and bumps the number in the `version` file.

Deleting the `version` file resets the version numbering.


### Docker build context: `docker`

[`docker/`]: #docker-build-context-docker "Docker build context: `docker`"

This directory contains the `Dockerfile` and additional files used during the Docker image build.


#### Docker entry point: `entrypoint`

[entrypoint]: #docker-entry-point-entrypoint "Docker entry point: `entrypoint`"

Detects user and group IDs from the directory that is mounted as the image working directory — `/src`, unless overridden by environment variables (`USER_ID` and `GROUP_ID`, respectively). It creates a group and user with the respective IDs, setting the user as a member of the group, and creates a symbolic link for the `/home/dobe.ssh` directory in the home directory of the user (as `~/.ssh`). It then yields control over via `exec` and `gosu` as the limited user it created previously.

If the first of the entry point arguments is recognized by the shell as a command (via `command -v`), it is `exec`ed with `gosu`, as stated above. Otherwise, if it is the only one, it is passed to the shell as a command string (an argument for `sh -c`); if there are other arguments, all are piped into the standard input of the shell to execute as a script (`sh -s`).

The primary motivation for this is to simplify the syntax for setting environment variables when running commands in container via [`bin/run`][bin/run], allowing to, for example, instead of

```sh
bin/run sh -c 'REF=v19.07.3 /pull'
```

type

```sh
bin/run REF=v19.07.3 /pull
```

This has the unfortunate side effect of the targeted command not running as the process ID 1, which may be undesirable in some specific cases. Although, when there is just a single command to take over the shell, that can be worked around by passing the entire command string as a single argument, prefixing the actual command with `exec`. For the above example that would be

```sh
bin/run 'REF=v19.07.3 exec /pull'
```


#### `make`

[/make]: #make "The `make` wrapper"

Wraps the `make` utility with "idle" `ionice` class and +19 niceness, which, according to the `nice` manual, is the least favorable to the process, and sets jobs to an autodetected value — one less than the number of available virtual cores from `getconf`, but no less than one.

The default values can be overridden via the environment like so:

```sh
IOCLASS=2 IOLEVEL=6 NICE=10 JOBS=3 /make
```

Not all of the values *have to* be set at the same time, of course — it will fall back to defaults where unspecified, and the order does not matter either.


#### `prompt-setup`

Locates the `PS1` prompt strings in a `.bashrc` file, enhancing them with the output of last exit/error status and current time, and adds `PROMPT_COMMAND` and `GIT_PS1_*` variables, to add Git status to the prompt and configure it.

It is only executed during the Docker image build stage to enhance `/etc/skel/.bashrc` which is used when creating new users.


#### `pull`

[/pull]: #pull "The `/pull` script"

Fetches the latest OpenWrt sources, updates and installs feeds.

It intelligently detects what you have checked out currently, checking out branches only if necessary, and rebasing local changes (if there are any) when you are on a branch. It also updates and installs package feeds, all by default.

What and how is updated can be controlled with environment variables, `FEEDS`, `REF`, `REMOTE`, `UPSTREAM` and `URL`. Of these `REF` is probably the one you are going to use the most, as it tells the script which branch, tag or commit you wish to checkout (and update, in the case of a branch).

For example, to pull the 19.07 development branch as `my-19.07` from the GitHub mirror, you can execute

```sh
bin/run \
	REF=my-19.07 \
	UPSTREAM=github/openwrt-19.07 \
	REMOTE=github \
	URL=https://github.com/openwrt/openwrt.git \
	/pull
```

in your host shell (or drop the `bin/run` part when doing this from within a container). This will create a new remote, `github`, set its URL to the given, checkout a new branch `my-19.07` at (and set its upstream to) `github/openwrt-19.07` and update and install all feeds. Later, when you want to update your branch with the latest commits from the GitHub mirror, it is enough to run just `/pull` on its own.

The URL for a remote will be reset to the default when `URL` is explicitly set to an empty value. For example, if you wanted the `github` origin from the above example to point to the default OpenWrt repository, you could achieve that by running

```sh
bin/run REMOTE=github URL= /pull
```

To update and install only a subset of feeds, set the corresponding environment variable, like so:

```sh
FEEDS='luci routing' /pull
```

You can set `FEEDS` to an empty value (e.g. `FEEDS= /pull` — note the space) to prevent updating and installation of the package feeds altogether.


### Build root: `build`

[`build/`]: #build-root-build "Build root: `build`"

Since by default the OpenWrt build system comes with everything tucked under the main source directory, the directory structure has been slightly reorganized here with the aim of simplifying navigation, resource location and cleanup tasks, as well as a potential introduction of versioning for the resources that are not covered by the build system (e.g. the file system overlay).

This and the following directories can be changed by setting environment variables for the [`bin/run` script][bin/run].


#### OpenWrt build artifacts: `build/cache`

[`cache/`]: #openwrt-build-artifacts-buildcache "OpenWrt build artifacts: `build/cache`"

The `cache/` directory holds the files that have been either downloaded or created during the OpenWrt build process.

##### `cache/build`

[cache/build]: #cachebuild "`cache/build`"

Used by the OpenWrt build system to store and compile the unpacked sources of build tools and dependencies.

Maps to `build_dir/` in the build system root.

##### `cache/dl`

[`dl/`]: #cachedl "`cache/dl`"

Contains the archived sources of build tools and dependencies.

You might want to preserve this even when you wipe out the rest of the `cache/` subdirectories, unless you are pressed for storage space, as doing so would reduce network traffic and time spent downloading during firmware builds.

Maps to `dl/` in the OpenWrt build system root.

##### `cache/staging`

[`staging/`]: #cachestaging "`cache/staging`"

Used by the OpenWrt build system to assemble ("install") the compiled tools and components, either for compilation of other components or for the actual firmware images.

Maps to `staging_dir/` in the build system root

##### `cache/tmp`

[`tmp/`]: #cachetmp "`cache/tmp`"

Contains various locks, caches and other temporary files created during the build process.

Maps to `tmp/` in the OpenWrt build system root.


#### Package feeds: `build/feeds`

[`feeds/`]: #package-feeds-buildfeeds "Package feeds: `build/feeds`"

Contains the sources of packages for the OpenWrt firmware.

Maps to `feeds/` in the OpenWrt build system root.


#### Build output: `build/output`

[`output/`]: #build-output-buildoutput "Build output: `build/output`"

This is the directory where you will find the final firmware images and packages produced by the OpenWrt build system.

Maps to `bin/` in the build system root.


#### File system overlay: `build/overlay`

[`overlay/`]: #file-system-overlay-buildoverlay "File system overlay: `build/overlay`"

The files and directories located here are copied into the firmware images that you build. Things like SSL/TLS certificates, SSH keys or UCI configuration files that you wish to incorporate into your firmware.

Maps to `files/` in the OpenWrt build system root.


#### OpenWrt sources: `build/src`

[`src/`]: #openwrt-sources-buildsrc "OpenWrt sources: `build/src`"

Contains the OpenWrt build system sources. This is also where the `.config` file resides, the one that determines what and how the build system will build when you run `make`.

Maps to the actual build system root directory.

#### Persistent SSH configuration: `build/ssh`

[`ssh/`]: #persistent-ssh-configuration-buildssh "Persistent SSH configuration: `build/ssh`"

This directory can be used to store SSH key pairs and configuration (as well as a `known_hosts` file). These are needed if you use the `git` protocol for connecting to remote repositories.

It is mounted as `/home/dobe.ssh`, but the [`entrypoint`][entrypoint] will create a symbolic link for it in the home directory of the user created in the container (as `~/.ssh`).


## License

This project is licensed under [Mozilla Public License Version 2.0][mpl]. See [LICENSE](LICENSE).


[wiki-build-system]: https://openwrt.org/docs/guide-developer/build-system/start
	"OpenWrt Project: The Build System"

[wiki-quickstart]: https://openwrt.org/docs/guide-developer/quickstart-build-images
	"OpenWrt Project: Quick Image Building Guide"

[wiki-quickstart-s1]: https://openwrt.org/docs/guide-developer/quickstart-build-images#prerequisites
	"1. Prerequisites — OpenWrt Project: Quick Image Building Guide"
[wiki-quickstart-s2]: https://openwrt.org/docs/guide-developer/quickstart-build-images#get_openwrt_source_code
	"2. Get OpenWrt source code — OpenWrt Project: Quick Image Building Guide"
[wiki-quickstart-s3]: https://openwrt.org/docs/guide-developer/quickstart-build-images#configure_target_and_options
	"3. Configure target and options — OpenWrt Project: Quick Image Building Guide"
[wiki-quickstart-s4]: https://openwrt.org/docs/guide-developer/quickstart-build-images#build_image
	"4. Build image — OpenWrt Project: Quick Image Building Guide"

[wiki-bleeding-edge]: https://openwrt.org/about/history#bleeding_edgemaster
	"Bleeding edge / master — OpenWrt Project: OpenWrt Version History"
[wiki-build-configdiff]: https://openwrt.org/docs/guide-developer/build-system/use-buildsystem#configure_using_config_diff_file
	"Configure using config diff file — OpenWrt Project: Build system – Usage"
[wiki-build-download]: https://openwrt.org/docs/guide-developer/build-system/use-buildsystem#download_sources_and_multi_core_compile
	"Download sources and multi core compile — OpenWrt Project: Build system – Usage"

[repo-tags]: https://git.openwrt.org/?p=openwrt/openwrt.git;a=tags
	"git.openwrt.org Git - openwrt/openwrt.git/tags"
[repo-heads]: https://git.openwrt.org/?p=openwrt/openwrt.git;a=heads
	"git.openwrt.org Git - openwrt/openwrt.git/heads"

[mpl]: https://www.mozilla.org/en-US/MPL/2.0/
	"Mozilla Public License, version 2.0"
