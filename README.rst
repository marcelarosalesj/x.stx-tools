=====
tools
=====

StarlingX Build Tools
---------------------

The StarlingX build process is tightly tied to CentOS in a number of
ways, doing the build inside a Docker container makes this much easier
on other flavors of Linux. Basically, the StarlingX ISO image creation
flow involves the following general steps.

1. Build the StarlingX docker image.
2. Package mirror creation.
3. Build packages/ISO creation.

Build the Starlingx docker image
--------------------------------

StarlingX docker image handles all steps related to StarlingX ISO
creation. This section describes how to customize the docker image
building process.

Container build image customization
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can start by customizing values for the StarlingX docker image
build process. There are a pair of useful files that help to do this.

- ``buildrc``
- ``localrc``

The ``buildrc`` file is a shell script that is used to set the default
configuration values. It is contained in the tbuilder repo and should
not need to be modified by users as it reads a ``localrc`` file that
will not be overwritten by tbuilder updates. This is where users should
alter the default settings. This is a sample of a ``localrc`` file:

.. code-block:: bash

    # tbuilder localrc
    MYUNAME=$USER
    PROJECT=starlingx
    HOST_PREFIX=$HOME/starlingx/workspace
    HOST_MIRROR_DIR=$HOME/starlingx/mirror

This project contains a Makefile that can be used to automate the build
lifecycle of a container. The Makefile will read the contents of the
``buildrc`` file.

StarlingX Builder container image are tied to your UID so image names
should include your username.

Build image
~~~~~~~~~~~

Once the configuration files have been customized, it is possible to build
the docker image. This process is automated by the ``tb.sh`` script.

.. code-block:: bash

    ./tb.sh create

NOTE:
~~~~~

-  Do NOT change the UID to be different from the one you have on your
   host or things will go poorly. i.e. do not change
   ``--build-arg MYUID=$(id -u)``

-  The Dockerfile needs MYUID and MYUNAME defined, the rest of the
   configuration is copied in via buildrc/localrc.

Package mirror creation
-----------------------

Once the StarlingX docker image has been built, you must create a mirror
before creating the ISO image. Basically, a mirror is a directory that
contains a series of packages. The packages are organized to be consumed
by the ISO creation scripts.

The ``HOST_MIRROR_DIR`` variable provides the path to the mirror. The
``buildrc`` file sets the value of this variable unless the ``localrc``
file has modified it.

The mirror creation involves a set of scripts and configuration files
required to download a group of RPMs, SRPMs, source code packages and
so forth. These tools live inside ``centos-mirror-tools`` directory.

.. code-block :: bash
    $ cd centos-mirror-tools

All items included in this directory must be visble inside the container
environment. Then the container shall be run from the same directory where
these tools are stored. Basically, we run a container with the previously
created StarlingX docker image, using the following configuration:

.. code-block :: bash

    $ docker run -it -v $(pwd):/localdisk <your_docker_image_name>:<your_image_version> bash

As ``/localdisk`` is defined as the workdir of the container, the same
folder name should be used to define the volume. The container will
start to run and populate ``logs`` and ``output`` folders in this
directory.

Run the ``download_mirror.sh`` script
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once inside the container, run the downloader script

.. code-block :: bash

    $ cd localdisk && bash download_mirror.sh

NOTE: in case there are some downloading failures due to network
instability or timeouts, you should download them manually, to assure
you get all RPMs listed in "rpms\_from\_3rd\_parties.lst" and
"rpms\_from\_centos\_repo.lst".

Copy the files to the mirror
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

After all downloads are complete, copy the downloaded files to mirror.

.. code-block :: bash

    $ find ./output -name "*.i686.rpm" | xargs rm -f
    $ chown  751:751 -R ./output
    $ cp -rf  output/stx-r1/ <your_mirror_folder>/

In this case, ``<your_mirror_folder>`` can be whatever folder you want to
use as mirror.

Tweaks in the StarlingX build system.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

NOTE: You do not need to do the following step if you've synced the latest codebase.

Go into the StarlingX build system (i.e. *another* container that hosts the
cgcs build system) and perform the following steps:

Debugging issues
~~~~~~~~~~~~~~~~

The ``download_mirror.sh`` script will create log files in the form of
``centos_rpms_*.txt``. After the download is complete, it's recommended
to check the content of these files to see if everything was downloaded
correctly.

A quick look into these files could be:

.. code-block :: bash

    $ cd logs
    $ cat *_missing_*log
    $ cat *_failmove_*log

Build packages/ISO creation
---------------------------

StarlingX ISO image creation required some customized packages. In this step,
a set of patches and customizations are applied to the source code to create
the RPM packages. We have an script called ``tb.sh`` that helps with
the process.

The ``tb.sh`` script is used to manage the run/stop lifecycle of working
containers. Copy it to somewhere on your ``PATH``, say ``$HOME/bin`` if
you have one, or maybe ``/usr/local/bin``.

The basic workflow is to create a working directory for a particular
build, say a specific branch or whatever. Copy the ``buildrc`` file from
the tbuilder repo to your work directory and create a ``localrc`` if you
need one. The current working directory is assumed to be this work
directory for all ``tb.sh`` commands. You switch projects by switching
directories.

By default ``LOCALDISK`` will be placed under the directory pointed to
by ``HOST_PREFIX``, which defaults to ``$HOME/starlingx``.

The ``tb.sh`` script uses sub-commands to select the operation: \*
``run`` - Runs the container in a shell. It will also create
``LOCALDISK`` if it does not exist. \* ``stop`` - Kills the running
shell. \* ``exec`` - Starts a shell inside the container.

You should name your running container with your username. tbuilder does
this automatically using the ``USER`` environment variable.

``tb.sh run`` will create ``LOCALDISK`` if it does not already exist
before starting the container.

Set the mirror directory to the shared mirror pointed to by
``HOST_MIRROR_DIR``. The mirror is LARGE, if you are on a shared machine
use the shared mirror. For example you could set the default value for
``HOST_MIRROR_DIR`` to ``/home/starlingx/mirror`` and share it.

Running the Container
~~~~~~~~~~~~~~~~~~~~~

Start the builder container:

.. code-block:: bash

    tb.sh run

or by hand:

.. code-block:: bash

    docker run -it --rm \
        --name ${TC_CONTAINER_NAME} \
        --detach \
        -v ${LOCALDISK}:${GUEST_LOCALDISK} \
        -v ${HOST_MIRROR_DIR}:/import/mirrors:ro \
        -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
        -v ~/.ssh:/mySSH:ro \
        -e "container=docker" \
        --security-opt seccomp=unconfined \
        ${TC_CONTAINER_TAG}

Running a Shell Inside the Container
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Since running the container does not return to a shell prompt the exec
into the container must be done from a different shell:

.. code-block:: bash

    tb.sh exec

or by hand:

.. code-block:: bash

    docker exec -it --user=${MYUNAME} ${USER}-centos-builder bash

Notes:
~~~~~~

-  The above will reusult in a running container in systemd mode. It
   will have NO login.
-  I tend to use tmux to keep a group of shells related to the build
   container
-  ``--user=${USER}`` is the default username, set ``MYUNAME`` in
   ``buildrc`` to change it.

Stop the Container
~~~~~~~~~~~~~~~~~~

.. code-block:: bash

    tb.sh stop

or by hand:

.. code-block:: bash

    docker kill ${USER}-centos-builder

What to do to build from WITHIN the container
---------------------------------------------

To make git cloning less painful
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

    $ eval $(ssh-agent)
    $ ssh-add

To start a fresh source tree
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Instructions
^^^^^^^^^^^^

Initialize the source tree.
---------------------------

.. code-block:: bash

    cd $MY_REPO_ROOT_DIR
    repo init -u https://opendev.org/starlingx/manifest.git -m default.xml
    repo sync

To generate cgcs-centos-repo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The cgcs-centos-repo is a set of symbolic links to the packages in the
mirror and the mock configuration file. It is needed to create these
links if this is the first build or the mirror has been updated.

.. code-block:: bash

    generate-cgcs-centos-repo.sh /import/mirrors/CentOS/pike

Where the argument to the script is the path of the mirror.

To build all packages:
~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

    $ cd $MY_REPO
    $ build-pkgs or build-pkgs --clean <pkglist>; build-pkgs <pkglist>

To generate cgcs-tis-repo:
~~~~~~~~~~~~~~~~~~~~~~~~~~

The cgcs-tis-repo has the dependency information that sequences the
build order; To generate or update the information the following command
needs to be executed after building modified or new packages.

.. code-block:: bash

    $ generate-cgcs-tis-repo

To make an iso:
~~~~~~~~~~~~~~~

.. code-block:: bash

    $ build-iso

First time build
~~~~~~~~~~~~~~~~

The entire project builds as a bootable image which means that the
resulting ISO needs the boot files (initrd, vmlinuz, etc) that are also
built by this build system. The symptom of this issue is that even if
the build is successful, the ISO will be unable to boot.

For more specific instructions on how to solve this issue, please the
README on ``installer`` folder in ``metal`` repository.

WARNING HACK WARNING
--------------------

-  Due to a lack of full udev support in the current build container,
   you need to do the following:

   .. code-block:: bash

       $ cd $MY_REPO
       $ rm build-tools/update-efiboot-image
       $ ln -s /usr/local/bin/update-efiboot-image $MY_REPO/build-tools/update-efiboot-image

-  if you see complaints about udisksctl not being able to setup the
   loop device or not being able to mount it, you need to make sure the
   build-tools/update-efiboot-image is linked to the one in
   /usr/local/bin

Troubleshooting
---------------

-  if you see:

   .. code-block:: bash

       Unit tmp.mount is bound to inactive unit dev-sdi2.device. Stopping, too.

-  it's a docker bug. just kill the container and restart the it using a
   different name.

   -  I usually switch between -centos-builder and -centos-builder2.
      It's some kind of timeout (bind?) issue.
