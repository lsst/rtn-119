##################################################################
Procedure for creating a butler repository at FrDF for DP1 release
##################################################################

.. abstract::

   In this note we document the process applied at the Rubin French Data Facility (FrDF) for creating a butler repository exposing the DP1 data that have been produced at the Rubin US Data Facility (USDF).

Introduction
============

The DP1 butler repository at FrDF is a community repository whose purpose is to make DP1 data (produced at USDF) available to the IN2P3 science community. Its content can be accessed directly from the FrDF interactive, computing and notebook platforms for analysis. Access can be configured in the same way as the "main" repo (https://doc.lsst.eu/tutorial/main_butler.html).

Input Datasets
==============

The process of registering and replicating the necessary input datasets is similar to the one described in RTN-103.
* SkyMap: we use the same ``skymap lsst_cells_v1.skymap.config`` as the one mentionned in RTN-103.
* Reference catalogs: ``https://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/releases/raw/refcat/the_monster_20250219``
* Raw exposures: we use the same raw exposures described in RTN-103 present in ``davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/instrument/raw/LSSTComCam/``. We extract the list of exposures to ingest from Rucio.
* Calibrations: the set of calibrations files used for DP1 have been replicated from USDF into ``davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/releases/dp1/LSSTComCam/calib/``.
* Products: the datasets generated from DP1 processing at USDF have been replicated into ``davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/releases/dp1/LSSTComCam/runs/``.

Creating and populating the repository
======================================

We present here the procedure we used for creating and populating the repository.

The location of the repository is referred using the environment variable ``$REPO``:

.. code-block:: bash

    $ export REPO='davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/groups/lsstfr/repos/dp1/'

The location of data to be ingested is defined using the environment variable ``$DATA``:

.. code-block:: bash

    $ export DATA='davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/releases/dp1/'


.. _create-empty-repository:

Create an empty repository
--------------------------

We use the seed configuration file ``butler-seed_dp1.yaml`` shown below to create a butler repository composed of a PostgreSQL registry database and a WebDAV datastore (the default):

.. code-block:: bash

    $ cat butler-seed_dp1.yaml
    datastore:
      root: "davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/groups/lsstfr/repos/dp1/"
      name: "dp1"
    registry:
      db: "postgresql://ccpglsstdev.in2p3.fr:6553/lsstfr"
      namespace: "dp1"


To create the repository at location ``$REPO`` we use the command:

.. code-block:: bash

    $ butler create --seed-config butler-seed_dp1.yaml --override $REPO


.. _register-instrument:

Register instrument
-------------------

To register the instrument for this repository we use the command below:

.. code-block:: bash

    $ butler register-instrument $REPO lsst.obs.lsst.LsstComCam


.. _register-sky-map:

Register SkyMap
----------------

To register the skymap configuration we use the command below:

.. code-block:: bash

    $ butler register-skymap --config-file lsst_cells_v1.skymap.config $REPO


.. _ingest-raw-exposures:

Ingest raw exposures
--------------------

We ingest the raw exposures in parallel using:

.. code-block:: bash

    $ xargs -a dp1_raws_urls.list -L 1 -P 32 butler ingest-raws --transfer direct $REPO


.. _define-visits:

Define visits
-------------

To define visits from the exposures previously ingested into the repository we use the command below:

.. code-block:: bash
    
    $ butler define-visits $REPO LSSTComCam --collections LSSTComCam/raw/all


.. _ingest-reference-catalog:

Ingest reference catalogs
-------------------------

The corresponding dataset type is registered with:

.. code-block:: bash

    $ butler register-dataset-type $REPO the_monster_20250219 SimpleCatalog htm7

Then the ingestion is done:

.. code-block:: bash

    $ butler ingest-files $REPO the_monster_20250219 refcats/DM-49042/the_monster_20250219 --prefix davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst//releases/raw/refcats/the_monster_20250219/ --transfer direct the_monster_20250219_frdf.ecsv


.. _add-instrument-calibrations:

Add instrument's curated calibrations
-------------------------------------

To ingest the known calibration data for LSSTComCam we use the command below:

.. code-block:: bash

    $ butler write-curated-calibrations $REPO lsst.obs.lsst.LsstComCam --label DM-49734


.. _ingest-calibration-data:

Ingest calibration data
-----------------------

To ingest calibration data we use the command below, for each dataset type:

.. code-block:: bash

    $ butler import -t direct --export-file export.yaml $REPO


.. _ingest-products:

Ingest products
---------------

To ingest products we use the same command, for each dataset type:

.. code-block:: bash

    $ butler import -t direct --export-file export.yaml $REPO






