##################################################################
Procedure for creating a butler repository at FrDF for DP1 release
##################################################################

.. abstract::

   In this note we document the process applied at the Rubin French Data Facility (FrDF) for creating a butler repository exposing the DP1 data that have been produced at the Rubin US Data Facility (USDF).

Introduction
============

The DP1 butler repository at FrDF is a community repository whose purpose is to make DP1 data (produced at USDF) available to the IN2P3 science community. Its content can be accessed directly from the FrDF interactive, computing and notebook platforms for analysis. Access can be configured in the same way as the "main" repo, as documented `here <https://doc.lsst.eu/tutorial/main_butler.html>`__.


Input Datasets
==============

The process of registering and replicating the necessary input datasets is mostly similar to the one described in `RTN-103 <https://rtn-103.lsst.io>`__.

.. _import-sky-map:

SkyMap
------

We use the same ``skymap lsst_cells_v1.skymap.config`` as the one mentionned in `RTN-103 <https://rtn-103.lsst.io>`__.

.. _import-raw-exposures:

Raw images
----------

We use the LSSTComCam raw exposures located in ``davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/instrument/raw/LSSTComCam/`` as described in `RTN-103 <https://rtn-103.lsst.io>`__. We extract the list of exposures to ingest from Rucio (give more details?).

.. _import-calibration-data:

Calibration data
----------------

The set of calibrations files used for DP1 have been replicated from USDF into ``davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/releases/dp1/LSSTComCam/calib/``.

.. _import-reference-catalog:

Reference catalogs
------------------

We use the reference catalogs replicated from USDF into ``https://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/releases/raw/refcat/the_monster_20250219``.

.. _import-products:

Products
--------

The datasets generated from the DP1 processing at USDF (catalogs and images) have been replicated into ``davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/releases/dp1/LSSTComCam/runs/``. We consider all datasets, excepted from the tasks metadata, configuration, and logs.


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

The reference catalogs are ingested with:

.. code-block:: bash

    $ butler import -t direct --export-file export.yaml $REPO davs://ccdavrubinint.in2p3.fr:2880/pnfs/in2p3.fr/lsst/releases/raw/


The export file is generated at USDF with:

.. code-block:: bash

    $ export-datasets --root file:///sdf/data/rubin/shared --filename export_refcat.yaml --collections LSSTComCam/DP1 dp1 the_monster_20250219


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

    $ butler import -t direct --export-file export.yaml $REPO $DATA


The export file is generated at USDF with:

.. code-block:: bash

    $ export-datasets --filename export.yaml --collections LSSTComCam/DP1 dp1 $dataset


for each dataset type. Calibration dataset types are identified in this `reference file <https://github.com/lsst-uk/lsst-uk-butler/blob/main/dp1_auto_ingest/allDataTypesUSDF.list>`__. We consider the following data set types:

* All data set types with ``isCalibration=True`` 
* ``fgcmLookUpTable``
* ``standard_passband``


.. _ingest-products:

Ingest products
---------------

To ingest products we use the same command, for each dataset type:

.. code-block:: bash

    $ butler import -t direct --export-file export.yaml $REPO $DATA


Similarly, the export files are generated at USDF for each dataset type. We consider all dataset types excepted:

* The ones that have already been previously ingested.
* All tasks related auxilliary files, i.e. ``TaskMetadata``, ``ButlerLogRecords`` and ``Config`` storage class datasets.



.. _create-collection:

Create chained collection
-------------------------

Finally, we define a DP1 collection containing all collections previously defined:

.. code-block:: bash

    $ butler collection-chain dp1 LSSTComCam/DP1 LSSTComCam/runs/DRP/DP1/DM-51335,LSSTComCam/runs/DRP/DP1/v29_0_0/DM-50260/20250419T073356Z,LSSTComCam/runs/DRP/DP1/v29_0_0/DM-50260/20250417T034317Z,LSSTComCam/runs/DRP/DP1/v29_0_0/DM-50260/20250416T185152Z,LSSTComCam/raw/all,LSSTComCam/calib/DM-48955/illumCorr/illuminationCorrection.20250224a,LSSTComCam/calib/DM-48520/DP1/flat-y.20250207a,LSSTComCam/calib/DM-48520/DP1/flat-z.20250207a,LSSTComCam/calib/DM-48520/DP1/flat-i.20250207a,LSSTComCam/calib/DM-48520/DP1/flat-r.20250207a,LSSTComCam/calib/DM-48520/DP1/flat-g.20250207a,LSSTComCam/calib/DM-48520/DP1/flat-u.20250207a,LSSTComCam/calib/DM-48520/DP1/dark.20250207a,LSSTComCam/calib/DM-48520/DP1/bias.20250207a,LSSTComCam/calib/DM-48520/DP1/cti.20250207a,LSSTComCam/calib/DM-48520/DP1/defects.20250207a,LSSTComCam/calib/DM-47365/addManualDefects/defects.20241211a,LSSTComCam/calib/DM-47741/twiflat/flat-y.20241120a,LSSTComCam/calib/DM-47547/twiflat/flat-z.20241113a,LSSTComCam/calib/DM-47547/twiflat/flat-r.20241113a,LSSTComCam/calib/DM-47547/twiflat/flat-g.20241113a,LSSTComCam/calib/DM-47499/twiflat/flat-u.20241110a,LSSTComCam/calib/DM-47447/gainFixup/flat-g.20241107a,LSSTComCam/calib/DM-47447/gainFixup/flat-i.20241107a,LSSTComCam/calib/DM-47447/gainFixup/flat-r.20241107a,LSSTComCam/calib/DM-47447/gainFixup/dark.20241107a,LSSTComCam/calib/DM-47447/gainFixup/bias.20241107a,LSSTComCam/calib/DM-47447/gainFixup/ptc.20241107a,LSSTComCam/calib/DM-47197/pseudoFlat/flat-r.20241028d,LSSTComCam/calib/DM-47197/pseudoFlat/flat-i.20241028d,LSSTComCam/calib/DM-46360/isrTaskLSST/flat-i.20240926a,LSSTComCam/calib/DM-46360/isrTaskLSST/flat-r.20240926a,LSSTComCam/calib/DM-46360/isrTaskLSST/flat-g.20240926a,LSSTComCam/calib/DM-46360/isrTaskLSST/dark.20240926a,LSSTComCam/calib/DM-46360/isrTaskLSST/bias.20240926a,LSSTComCam/calib/DM-46360/isrTaskLSST/bfk.20240926a,LSSTComCam/calib/DM-46360/isrTaskLSST/ptc.20240926a,LSSTComCam/calib/DM-46360/isrTaskLSST/linearizer.20240926a,LSSTComCam/calib/DM-46360/isrTaskLSST/defects.20240926a,LSSTComCam/calib/DM-47498/fallbackFlats/flat-all.20241112a,LSSTComCam/calib/DM-49734,LSSTComCam/calib/DM-49734/unbounded,refcats/DM-49042/the_monster_20250219,skymaps,LSSTComCam/calib/fgcmcal/DM-48089,LSSTComCam/calib/fgcmcal/DM-48089/standard_passbands

