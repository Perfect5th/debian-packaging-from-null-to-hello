# Debian packaging from null to hello

This repository is a guide for packaging software for Debian-based Linux distributions. It's basically a slimmed-down version of https://wiki.debian.org/Packaging/Intro.

By following steps 1 to 7, you'll get a binary package suitable for direct installation.

By continuing on through to step 8, you'll get a binary package suitable for upload to a package archive such as those provide by [Launchpad PPAs](https://documentation.ubuntu.com/launchpad/en/latest/user/reference/packaging/ppas/ppa/).

The repository starts off with pretty much nothing except this README and a single script file, `hello-packaging`. You'll be adding everything needed to package and install that script.

If you want to cheat and see what the end result looks like, check out the `debian/latest` branch.

## Requirements

- A installation of a Linux distribution that has these packages installed:
  - `tar` - for creating archives
  - `devscripts` - for the Debian build tools
  - `debhelper-compat` - a build dependency that ensures a specific compatibility level for the build tools
  - `tree` - optional, used to visualize the directory tree
  
If you're using Ubuntu or Debian, you can install all of these like so:

``` sh
sudo apt install tar devscripts debhelper-compat tree
```

## The guide

### Step 1: Creating the upstream tarball

When you package for Debian, you start with an archive of the source code of the software you are packaging. This is called the "upstream tarball" or sometimes the "orig-tarball".

The upstream tarball always has a name with this format: `PACKAGENAME_VERSION.orig.tar.gz`.

To create one from the "source" of this repository, use `tar`:

``` sh
tar -czf ../hello-packaging_1.0.0.orig.tar.gz README.md hello-packaging
```

If you're not familiar with `tar`, `-czf` means "create (`c`) and compress (`z`) to this file (`f`)".

We put the upstream tarball in the directory above the current one because that's where the build tools expect it to be. This is because if it were in the current directory, it would be polluting our source code.

### Step 2: Add the Debian packaging directory

The build tools use a special directory, `debian`, for their configuration and as a working directory. You'll need one:

``` sh
mkdir debian
```

### Step 3: Add the Debian changelog

The first file we need in the `debian` directory is `debian/changelog`. This file indicates the version that we are releasing, who is releasing it, where we are releasing it to, and any relevant changes since the previous release.

We can use `dch` to create one for us, but you can also type one up manually if you like:

``` sh
export DEBEMAIL="youremail@example.com"
export DEBFULLNAME="Your Fullname"
dch --create --newversion 1.0.0-0nulltohello1 --package hello-packaging
```

This will ask you to select an editor. Pick whichever one you want except for `ed` (unless you're some kind of masochist).

This file will look like this:

``` text
hello-packaging (1.0.0-0nulltohello1) UNRELEASED; urgency=medium

  * Initial release. (Closes: #XXXXXX)

 -- Your Fullname <youremail@example.com>  Wed, 16 Jul 2025 22:58:43 +0000
```

Delete the `(Closes: #XXXXXX)` and save it. If you were releasing this to, for example, Ubuntu 24.04 Noble Numbat, you could change `UNRELEASED` to `noble`.

The reason we added `-0nulltohello1` to the version is because everything after the `-` is the "Debian revision".

The first digit, `0`, is usually reserved for packages that are actually in Debian. Here, we are pretending we are a downstream distributor of Debian packages called `nulltohello` and that this is the first version of upstream version `1.0.0` that we've packaged (hence `nulltohello1`).

### Step 4: Adding the Debian control file

The next file we need in the `debian` directory is `debian/control`. This file includes metadata about the package, including its build and installation dependencies.

Simply add these contents to `debian/control`:

``` text
Source: hello-packaging
Maintainer: Your Fullname <youremail@example.com>
Section: misc
Priority: optional
Standards-Version: 4.6.2
Build-Depends: debhelper-compat (= 13)

Package: hello-packaging
Architecture: all
Depends: ${misc:Depends}
Description: Demo Debian package
 hello-packaging serves as a quick demo for how to debianize software.
```

### Step 5: Adding the Debian rules file

The next file we need in the `debian` directory is `debian/rules`. This is a Makefile that is used by the build tools to build the Debian binary from the source.

A minimal version looks like this:

``` makefile
#!/usr/bin/make -f

%:
	dh $@
```

Which directs `make` to call `dh` with any recipe it is called with. It's a bit of magic that orchestrates the build process.

It should also be executable, so please: `chmod +x debian/rules`.

Because this package consists of a single shell script, we don't need to do any actual building, but this makes it easy to integrate existing build systems (`make`, `cmake`, language-specific compilers, _etc._) into Debian packaging.

### Step 6: Adding the install file

If you tried to build now, it would _almost_ work. We're still missing one thing—an indication of where the files produced by building the package should end up when installed.

One way to do this is to use an `install` file. This is the simplest way, in my opinion.

Add this file, `debian/install`:

``` text
hello-packaging /usr/bin
```

This directs the build tools that the `hello-packaging` file should be installed in `/usr/bin`.

### Step 7: Build!

To build the Debian binary, run:

``` sh
debuild -us -uc -i
```

`-us` means "unsigned source" and `-uc` means "unsigned changes". These prevent us from seeing errors related to key-signing the build artifacts, but when build a package for-serious, you should not use these.

`-i` means to ignore some commonly-ignored files when comparing the upstream tarball to the current directory. In this case, that means ignoring `.git`, as we are in a git repository.

Build artifacts are placed in the parent directory, list them with `ls ..`:

``` sh
$ ls ..
debian-packaging-from-null
hello-packaging_1.0.0-0nulltohello1.diff.gz
hello-packaging_1.0.0-0nulltohello1.dsc
hello-packaging_1.0.0-0nulltohello1_all.deb
hello-packaging_1.0.0-0nulltohello1_amd64.build
hello-packaging_1.0.0-0nulltohello1_amd64.buildinfo
hello-packaging_1.0.0-0nulltohello1_amd64.changes
hello-packaging_1.0.0.orig.tar.gz
```

### Intermission with installable deb

Before we test installing `hello-packaging_1.0.0-0nulltohello1_all.deb`, let's look at what went into it.

The Debian build tools produce a new directory named after the package, `debian/hello-packaging`, that contains the directory tree as it will be unpacked during installation:

``` sh
$ tree debian/hello-packaging
debian/hello-packaging
├── DEBIAN
│   ├── control
│   └── md5sums
└── usr
    ├── bin
    │   └── hello-packaging
    └── share
        └── doc
            └── hello-packaging
                └── changelog.Debian.gz

7 directories, 4 files
```

You can see that, per our `debian/install` file, the `hello-packaging.sh` file is in `usr/bin`. This means that when installed, it will be installed to `/usr/bin`. You can see that, per our n`debian/install` file, the `hello-packaging.sh` file is in `usr/bin`. This means that when installed, it will be installed to `/usr/bin`. It's useful to examine this tree after building to make sure things ended up in the intended places.

#### Test installation

If you want to, you can also install the `.deb` directly:

``` sh
sudo apt install ../hello-packaging_1.0.0-0nulltohello1_all.deb
```

Immediately after installation, you should be able to execute it:

``` sh
$ hello-packaging
Hello Debian packaging world!
```

### Step 8: Resolving lintian warnings

The Debian build tools include a linter called `lintian` that emits a number of warnings and errors at the end of the build process. Resolving these gets you a long way toward having a package ready for upload, but the output can be a bit mysterious:

``` text
E: hello-packaging: no-copyright-file
W: hello-packaging source: missing-debian-source-format
W: hello-packaging source: no-debian-copyright-in-source
W: hello-packaging: no-manual-page [usr/bin/hello-packaging.sh]
```

`E:` means error and `W:` means warning.

The first and only error is pretty obvious: we need a copyright file at `debian/copyright`. This is the copyright file for the _packaging_, not the source code. Copyright files need to have a copyright notice at the top, but can otherwise refer to a common license:

``` sh
Copyright 2025 Your Fullname <youremail@example.com>

/usr/share/common-licenses/GPL-3
```

The first warning means we're missing `debian/source/format`, which tells the build tools what format we are using to indicate Debian "patches". Patches are diff files that apply changes to the source code prior to build and are used when we want to apply our own changes to the upstream software. For example, if we wanted to backport a fix to a version that the upstream maintainers no longer support.

Create `debian/source/format` and add a single line:

``` text
3.0 (quilt)
```

The next warning, `no-debian-copyright-in-source` is taken care of by adding `debian/copyright`.

The next warning, `no-manual-page` tells us we should have a manual page in `/usr/share/man` for `hello-packaging`. Manual pages can be in a variety of formats, but we'll use the (ancient) `nroff` format just because it doesn't require any additional processing.

Save this as `debian/hello-packaging.1`:

``` text
.TH hello-packaging 1 "17 July 2025" "" ""
.SH NAME
\fBhello-packaging \fP- Demo Debian package
\fB
.SH SYNOPSIS
.nf
.fam C

\fBhello-packaging\fP [\fIoptions\fP]

.fam T
.fi
.fam T
.fi
.SH DESCRIPTION

\fBhello-packaging\fP serves as a quick demo for how to debianize software. The
actual binary is a single shell script that echoes a 'hello' message.
.SH EXAMPLES

To print the message
.SH I/O:

\fBhello-packaging\fP

.SH AUTHOR
Your Fullname <youremail@example.com>
```

And save this as `debian/manpages`, which tells the build tools where to find the manual page files:

``` text
debian/hello-packaging.1
```

### Step 9: Build again!

Now you should be able to build without any lintian errors or warnings:

```sh
debuild -us -uc -i
```

## Things you can do next

That's it, you've now got a package that you can install on a Debian-based Linux distribution!

Even so, there's a lot of things left out of this guide. Here's a few ideas for things to follow up on:

  - Using an email with which you have a GPG private key associated in the `debian/changelog` and removing the `-uc` and `-us` parameters from the `debuild` command. This means `debuild` will actually sign the resulting build artifacts, which is a pre-requisite to uploading the the package to a package repository.
  - Experimenting with (`quilt`)[https://wiki.debian.org/UsingQuilt] for applying patches to the upstream source code. This lets you make documented changes to your package.
  - [Uploading your package to a PPA](https://documentation.ubuntu.com/launchpad/en/latest/user/how-to/packaging/ppa-package-upload/#upload-a-package-to-a-ppa). One of the build artifacts is a `source.changes` file, which can be used to coordinate a `dput` upload to a package repository.
  - Taking a look at (`git-buildpackage`)[https://wiki.debian.org/PackagingWithGit?action=show&redirect=git-buildpackage], a tool for orchestrating Debian package builds in a git repository using branches and tags.
  - Seeing if there's a `dh-` package to use for your programming language of choice, and testing how it integrates with that language's build systems. An example is (`dh-python`/`pybuild`)[https://wiki.debian.org/Python/Pybuild], which can build your Python package and run `pytest`, `unittest`, or `tox` tests.
