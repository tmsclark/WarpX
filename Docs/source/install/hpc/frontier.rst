.. _building-frontier:

Frontier (OLCF)
===============

The `Frontier cluster (see: Crusher) <https://docs.olcf.ornl.gov/systems/crusher_quick_start_guide.html>`_ is located at OLCF.
Each node contains 4 AMD MI250X GPUs, each with 2 Graphics Compute Dies (GCDs) for a total of 8 GCDs per node.
You can think of the 8 GCDs as 8 separate GPUs, each having 64 GB of high-bandwidth memory (HBM2E).


Introduction
------------

If you are new to this system, **please see the following resources**:

* `Crusher user guide <https://docs.olcf.ornl.gov/systems/crusher_quick_start_guide.html>`_
* Batch system: `Slurm <https://docs.olcf.ornl.gov/systems/crusher_quick_start_guide.html#running-jobs>`_
* `Production directories <https://docs.olcf.ornl.gov/data/index.html#data-storage-and-transfers>`_:

  * ``$PROJWORK/$proj/``: shared with all members of a project, purged every 90 days (recommended)
  * ``$MEMBERWORK/$proj/``: single user, purged every 90 days (usually smaller quota)
  * ``$WORLDWORK/$proj/``: shared with all users, purged every 90 days
  * Note that the ``$HOME`` directory is mounted as read-only on compute nodes.
    That means you cannot run in your ``$HOME``.


Installation
------------

Use the following commands to download the WarpX source code and switch to the correct branch.
**You have to do this on Summit/OLCF Home/etc. since Frontier cannot connect directly to the internet**:

.. code-block:: bash

   git clone https://github.com/ECP-WarpX/WarpX.git $HOME/src/warpx
   git clone https://github.com/AMReX-Codes/amrex.git $HOME/src/amrex
   git clone https://github.com/ECP-WarpX/picsar.git $HOME/src/picsar
   git clone -b 0.14.5 https://github.com/openPMD/openPMD-api.git $HOME/src/openPMD-api

To enable HDF5, work-around the broken ``HDF5_VERSION`` variable (empty) in the Cray PE by commenting out the following lines in ``$HOME/src/openPMD-api/CMakeLists.txt``:
https://github.com/openPMD/openPMD-api/blob/0.14.5/CMakeLists.txt#L216-L220

We use the following modules and environments on the system (``$HOME/frontier_warpx.profile``).

.. literalinclude:: ../../../../Tools/machines/frontier-olcf/frontier_warpx.profile.example
   :language: bash
   :caption: You can copy this file from ``Tools/machines/frontier-olcf/frontier_warpx.profile.example``.

We recommend to store the above lines in a file, such as ``$HOME/frontier_warpx.profile``, and load it into your shell after a login:

.. code-block:: bash

   source $HOME/frontier_warpx.profile


Then, ``cd`` into the directory ``$HOME/src/warpx`` and use the following commands to compile:

.. code-block:: bash

   cd $HOME/src/warpx
   rm -rf build

   cmake -S . -B build   \
     -DWarpX_COMPUTE=HIP \
     -DWarpX_amrex_src=$HOME/src/amrex \
     -DWarpX_picsar_src=$HOME/src/picsar \
     -DWarpX_openpmd_src=$HOME/src/openPMD-api
   cmake --build build -j 32

The general :ref:`cmake compile-time options <building-cmake>` apply as usual.


.. _running-cpp-frontier:

Running
-------

.. _running-cpp-frontier-MI250X-GPUs:

MI250X GPUs (2x64 GB)
^^^^^^^^^^^^^^^^^^^^^

After requesting an interactive node with the ``getNode`` alias above, run a simulation like this, here using 8 MPI ranks and a single node:

.. code-block:: bash

   runNode ./warpx inputs

Or in non-interactive runs:

.. literalinclude:: ../../../../Tools/machines/frontier-olcf/submit.sh
   :language: bash
   :caption: You can copy this file from ``Tools/machines/frontier-olcf/submit.sh``.


.. _post-processing-frontier:

Post-Processing
---------------

For post-processing, most users use Python via OLCFs's `Jupyter service <https://jupyter.olcf.ornl.gov>`__ (`Docs <https://docs.olcf.ornl.gov/services_and_applications/jupyter/index.html>`__).

Please follow the same guidance as for :ref:`OLCF Summit post-processing <post-processing-summit>`.

.. _known-frontier-issues:

Known System Issues
-------------------

.. warning::

   May 16th, 2022 (OLCFHELP-6888):
   There is a caching bug in Libfrabric that causes WarpX simulations to occasionally hang on Frontier on more than 1 node.

   As a work-around, please export the following environment variable in your job scripts until the issue is fixed:

   .. code-block:: bash

      #export FI_MR_CACHE_MAX_COUNT=0  # libfabric disable caching
      # or, less invasive:
      export FI_MR_CACHE_MONITOR=memhooks  # alternative cache monitor

.. warning::

   Sep 2nd, 2022 (OLCFDEV-1079):
   rocFFT in ROCm 5.1+ tries to `write to a cache <https://rocfft.readthedocs.io/en/latest/library.html#runtime-compilation>`__ in the home area by default.
   This does not scale, disable it via:

   .. code-block:: bash

      export ROCFFT_RTC_CACHE_PATH=/dev/null
