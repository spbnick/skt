skt - sonic kernel testing
==========================

Skt is a tool for automatically fetching, building, and testing kernel
patches published on Patchwork instances.

Dependencies
------------

Install dependencies needed for running skt like this:

    $ sudo dnf install python2 python2-junit_xml beaker-client

Dependencies needed to build kernels:

    $ sudo dnf builddep kernel-`uname -r`
    $ sudo dnf install bison flex

Extra dependencies needed for running the testsuite:

    $ sudo dnf install python2-mock

Run tests
---------

For running all tests write down:

    $ python -m unittest discover tests

For running some specific tests you can do this as following:

    $ python -m unittest tests.test_publisher

Usage
-----

The `skt` tool supports reading a configuration file, which can be specified
with the `--rc` option and is `~/.sktrc` by default. The several "commands"
`skt` implements can write their state to that configuration file as they
work, so that the other commands can take the workflow task over from them.

Some commands can receive that state from the command line, via options, but
some require some information stored in the configuration file, or discovering
the required command-line option values is difficult. For this reason, to
support a complete workflow, it is necessary to always make the commands
transfer their state via the configuration file. That can be done by passing
the global `--state` option with every command.

To separate the actual configuration from the specific workflow's state, and
to prevent separate tasks from interfering with each other, you can store your
configuration in a separate (e.g. read-only) file, copy it to a new file each
time you want to do something, then discard the file after the task is
complete. Note that reusing a configuration file with state added can break
some commands in unexpected ways. That includes repeating a previous command
after the next command in the workflow has already ran.

The following commands are supported by `skt`:

* `merge`
    - Fetch a kernel repository, checkout particular references, and
      optionally apply patches from patchwork instances.
* `build`
    - Build the kernel with specified configuration and put it into a tarball.
* `publish`
    - Publish (copy) the kernel tarball, configuration, and build information
      to the specified location, generating their resulting URLs, using the
      specified "publisher". Only "cp" and "scp" pusblishers are supported at
      the moment.
* `run`
    - Run tests on a built kernel using the specified "runner". Only
      "Beaker" runner is currently supported.
* `report`
    - Report build and/or test results using the specified "reporter".
      Currently results can be reported by e-mail or printed to stdout.
* `cleanup`
    - Remove the build information file, kernel tarball. Remove state
      information from the configuration file, if saving state was enabled
      with the global `--state` option, and remove the whole working directory,
      if the global `--wipe` option was specified.
* `all`
    - Run the following commands in order: `merge`, `build`, `publish`, `run`,
      `report` (if `--wait` option was specified), and `cleanup`.
* `bisect`
    - Bisect Git history between a known bad and a known good commit
      (defaulting to "master"), running tests to locate the offending commit.

The following is a walk-through through the process of checking out a kernel
commit, applying a patch from Patchwork, building the kernel, running the
tests, reporting the results, and cleaning up.

All the following commands use the `-vv` option to increase verbosity of the
command's output, so it's easier to debug problems. Remove the option for
quieter, shorter output.

You can make `skt` output junit-compatible results by adding a `--junit
<JUNIT_DIR>` option to any of the above commands. The results will be written
to the `<JUNIT_DIR>` directory.

### Merge

To checkout a kernel tree run:

    $ skt.py --rc <SKTRC> --state --workdir <WORKDIR> -vv \
             merge --baserepo <REPO_URL> --ref <REPO_REF>

Here `<SKTRC>` would be the configuration file to retrieve the configuration
and the state from, and store the updated state in. `<WORKDIR>` would be the
directory to clone and checkout the kernel repo to, `<REPO_URL>` would be the
source kernel Git repo URL, and `<REPO_REF>` would be the reference to
checkout.

E.g. to checkout "master" branch of the "net-next" repo:

    $ skt.py --rc skt-rc --state --workdir skt-workdir -vv \
             merge --baserepo git://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git \
                   --ref master

To apply a patch from Patchwork run:

    $ skt.py --rc <SKTRC> --state --workdir <WORKDIR> -vv \
             merge --baserepo <REPO_URL> \
                   --ref <REPO_REF> \
                   --pw <PATCHWORK_PATCH_URL>

Here, `<REPO_REF>` would be the reference to checkout, and to apply the patch
on top of, and `<PATCHWORK_PATCH_URL>` would be the URL pointing to a patch on
a Patchwork instance.

E.g. to apply a particular patch to a particular, known-good commit from the
"net-next" repo, run:

    $ skt.py --rc skt-rc --state --workdir skt-workdir -vv \
             merge --baserepo git://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git \
                   --ref a870a02cc963de35452bbed932560ed69725c4f2 \
                   --pw https://patchwork.ozlabs.org/patch/886637

### Build

And to build the kernel run:

    $ skt.py --rc <SKTRC> --state --workdir skt-workdir -vv \
             build -c `<CONFIG_FILE>`

Where `<CONFIG_FILE>` would be the kernel configuration file to build the
kernel with. The configuration will be applied with `make olddefconfig`, by
default.

E.g. to build with the current system's config file run:

    $ skt.py --rc skt-rc --state --workdir skt-workdir -vv \
             build -c /boot/config-`uname -r`

### Publish

To "publish" the resulting build using the simple "cp" (copy) publisher run:

    $ skt/skt.py --rc <SKTRC> --state --workdir skt-workdir -vv
                 publish -p cp <DIRECTORY> <URL_PREFIX>

Here `<DIRECTORY>` would be the location for the copied build artifacts, and
`URL_PREFIX` would be the string to add to prepend the filenames with
(together with a slash `/`) to construct the URLs the files will be reachable
at. The resulting URLs will be passed to other commands, such as `run`, via
the saved state in the configuration file.

E.g. to publish to the `/srv/builds` directory available at
`http://skt-server` run:

    $ skt/skt.py --rc skt-rc --state --workdir skt-workdir -vv
                 publish -p cp /srv/builds http://skt-server

License
-------
skt is distributed under GPLv2 license.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 2 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program. If not, see <http://www.gnu.org/licenses/>.
