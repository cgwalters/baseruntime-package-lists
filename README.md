# Generating Base Runtime Lists

This repository provides a series of BASH and Python tools for creating
and updating lists and dist-git hashes for modules in the Fedora Project.

## Setup
Storage requirements:
* You may need up to 10GiB of free space for the repository data

These tools require the following pre-requisites:
* bash
* python 2
* python 3
* rsync
* mock
* rpmdevtools
* koji
* depchase (See below)
* perl
* perl(autodie), perl(Getopt::Std), perl(IPC::Open3), perl(List::Util),
  perl(Template), perl(Text::CSV_XS) and perl(Text::Wrap)

To hack on these tools, you may also need:
* argbash

The user that will be downloading the RPM repos to work with must have
the `mock` package installed on their system and be a member of the
'mock' group.  This can be accomplished by:
```
dnf install mock
usermod -a -G mock <yourloginname>
```

The user that will be operating on the override repositories must be a
member of the `modularity-wg` group in the Fedora Account system, that
your SSH keys are properly set up for use with Fedora infrastructure
and also that you have an identical group present in `/etc/group` such as:
```
modularity-wg:x:189842:<yourloginname>
```
This is so the sync tools can ensure that the files on Fedora People
are accessible to all members of the modularity-wg team. When you add
this, log out of your current session and back in to have the new group
take effect.

## Legend/Glossary
I will use the following terms or variables in this document:

`<release>`: The path on disk representing a Fedora release or
pre-release.  For example: Fedora 25 would be `25`, but Fedora 26 Beta
would be `test/26_beta`.

`<arch>`: The processor architecture. Supports
* `aarch64`
* `armv7hl`
* `i686`
* `ppc64`
* `ppc64le`
* `s390x`
* `x86_64`

(Note: some infrastructure tools use other architecture names for things;
these scripts will automatically translate from e.g. `armv7hl` to `armhfp`
where needed, so always use one of these for command-line arguments.)

### Installing depchase

```
$ git clone https://github.com/fedora-modularity/depchase.git
$ cd depchase
$ python3 setup.py install --user
```
This will install the `depchase` command as `~/.local/bin/depchase`

You may wish to add this to your default PATH variable in `~/.bashrc`
by doing:
```
$ echo "export PATH=$PATH:$HOME/.local/bin" >> ~/.bashrc
```

## Preparing the local repository data

Depchase (a tool we use for processing the data) requires local copies
of the repository metadata against which to operate. This repository
provides some convenient tools for retrieving this data.

### Syncing the existing repositories (the easy way)

Invoke `make -B repo/devel` to download or update your local copy of
the development tree repodata.  This operation may take several minutes.

### Syncing the existing repositories (the manual way)

Alternatively you may download the repodata tree manually.

Use the `download_repo.sh` tool located in the root of this repository
to retrieve the repository metadata and override repositories.

For example:

```
./download_repo.sh --release=devel \
                   --milestone=Beta \
                   --archful-srpm-file=./archful-srpms.txt \
                   --arch=aarch64 \
                   --arch=armv7hl \
                   --arch=i686 \
                   --arch=ppc64 \
                   --arch=ppc64le \
                   --arch=x86_64 \
                   --overrides
```

This will download the frozen Fedora 26 Beta repository metadata, the
overrides repository for that milestone and will automatically regenerate
those SRPMs that are known to have arch-specific `BuildRequires` and
merge them locally to the repo metadata. The packages will be stored in
the `repo/test/26_beta/` path relative to the root of this repository.

Use `--release=devel` for a snapshot of rawhide suitable for development
or `--release=rawhide` for the latest Rawhide compose.  Note Fedora 27
and newer also support s390x.  Not all `--release` arguments support
all other options.

This process may take a long time, as it will transfer approximately
1GiB of data per architecture to the local system. This tool uses rsync,
so subsequent calls to it will only download updated information (plus
regenerating the archful SRPMs).

### Adding content to the override repositories

Copy any binary (or noarch) RPMs for inclusion into
`repo/<release>/override/<arch>/os`. For source RPMs, copy them to the
`repo/<release>/override/<arch>/sources` path. Run 
`./repo/generate-repo-data.sh [<release>]` to automatically update 
the repo metadata and `./repo/rsync-push-repo.sh [<release>]` to rsync the
contents to the fedorapeople repository.

There is a helper utility in the root of the git repository called
`dl_pkgs.sh`.  This utility must be called with one or two arguments.
First, a path to a file containing one package NVR or Koji build-id
per line. If it is an NVR, the package must be an official build in
Koji. For a build-id, a scratch-build may work (but is untested). The
second argument should be `<release>` .  This tool should be used to
prep the override repository whenever a content change is made to the
module metadata, to keep the lists up-to-date.

Once completed, run `./repo/generate-repo-data.sh [<release>]` to 
regenerate all the repository information.  You can then run whatever 
tests you want.  When you are happy with the results, run 
`./repo/rsync-push-repo.sh [<release>]` to update the fedorapeople repository.

## Dealing with SRPMs with arch-specific BuildRequires

Some packages have differing BuildRequires depending on which platform
is being built. In these cases, in order to have the list generation
find the right values, we need to pull down those SRPMs and regenerate
them for every architecture that we process.

There is a helper utility in the root of the git repository called
`mock_wrapper.sh` which takes a single argument: a list of NVRs of
official Koji builds that should be regenerated. It will construct a mock
buildroot with limited packages (to reduce the risk that the output could
be different depending on the packages available in the host system),
check out each of the requested SRPM packages and regenerate them on
each architecture. It will then drop them into a directory `./output`
(clobbering any existing content!) relative to the working directory
that was executed.

Once that content is present, you can run `./populate_srpm_repo.sh
<release>/override` from the root of the git repository, which will move
all of the `./output` contents into the `repo/` hierarchy, after which you
can follow the "Pushing to the override repositories" instructions above.

This functionality is automatically run whenever the `download_repo.sh`
script is invoked with the `--with-archful-srpm-file` argument.

## How to generate package lists (the easy way)

To generate the relevant package lists for the development snapshot tree,
simply invoke `make`.  This will run depchase to resolve runtime and build
time dependencies for components listed in the toplevel package lists
and will also generate complete modulemd files for all supported modules.

## How to generate package lists (the manual way)

Ensure that the contents of `toplevel-binary-packages.txt` and `hints.txt`
in each of the `data/Fedora/$VERSION[_$MILESTONE]/$MODULE/$ARCH`
directories is correct.  These files must be present for each architecture
to be processed.

`toplevel-binary-packages.txt` contains the set of package names that
the base runtime must contain in order to provide the requisite APIs.

`hints.txt` is a manually-curated set of packages that is used to resolve
ambiguities in dependencies. For example, the `glibc` package `Requires:
glibc-langpack` and many packages may `Provides: glibc-langpack`,
so we include `glibc-minimal-langpack` in the hints.txt to resolve
this decision.

Then call `generate_module_lists.sh` like so:

```
./generate_module_lists.sh --version 26 \
                           --milestone=Beta \
                           --module=base-runtime \
                           --arch=aarch64 \
                           --arch=armv7hl \
                           --arch=i686 \
                           --arch=ppc64 \
                           --arch=ppc64le \
                           --arch=x86_64
```

This will produce a set of files in the
`data/Fedora/$VERSION[_$MILESTONE]/$MODULE/$ARCH` directories named:
* runtime-binary-packages-full.txt
* runtime-binary-packages-short.txt
* runtime-source-packages-full.txt
* runtime-source-packages-short.txt
* selfhosting-binary-packages-full.txt
* selfhosting-binary-packages-short.txt
* selfhosting-source-packages-full.txt
* selfhosting-source-packages-short.txt

To generate modulemd files, use `make_modulemd.pl` and point it to the
directory with package lists and modulemd templates.

To generate Host & Platform: `./make_modulemd.pl ./data/Fedora/devel/hp`

To generate Bootstrap: `./make_modulemd.pl ./data/Fedora/devel/bootstrap`

This will produce one or more modulemd files in the respective module
directories.  These files should be dist-git-ready, with no further
manual editing required.

## Making sure everything is okay

Run `make test` and resolve any issues before pushing the repodata or
any changes to this repository.

The simple test suite, under `./tests`, verifies that the data looks
sane -- such as that there are no empty package lists or that no package
appears in more than one version.

## And that's it!

Congratulations, you've made it to the end!

A few final words, however.  If you're hacking on Host & Platform,
please, file a pull request for `fedora-modularity/hp` so that your
content change be properly tracked and the toplevel lists & module data
files regenerated.

Host & Platform modules dist-git repositories include handy Makefiles
that allow for easy modulemd updates.  Just run `make update` after you
push your changes to the repo and your modulemd files will be synced.
