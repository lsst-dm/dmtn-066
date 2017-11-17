..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

.. note::

   **This technote is not yet published.**

   Pipeline driver tasks such as singleFrameDriver, coaddDriver, and multiBandDriver have been tested to see what their memory usage is.  This technote will detail how these memory tests were ran and what results were found.

.. Add content here.

Introduction
============

This tech note discusses the memory usage of the reprocessing code on example
cases in order to give insight into the true memory needs of a typical 
reprocessing job.  This is a summary of DM-11818, DM-12051, and DM-12198.

Dataset Information
===================
The data used here is from the LSST GPFS storage that is located at 
/datasets/hsc/repo.  The first two sets of trials (the singleFrameDriver and 
coaddDriver trials) discussed here are only in the HSC-G band, while the 
multiBandDriver trials use the HSC-G and HSC-R bands.

Due to our desire to keep run-times low while using one core for most of our 
trials, we used very small datasets. The table below details what 
visits and bands are used for each type of trial discussed in this technote.

.. _table-label:

.. table:: Visits and bands used for each code trial

    +---------------------+-------------+-------------------------------------+
    | Code Name           | Bands       | Visits                              |
    +=====================+============++=========+++++++++++++++++++++++++++=+
    | singleFrameDriver.py| HSC-G       | 11382                               |
    |                     |             | and                                 |
    |                     |             | 11690                               |
    +---------------------+-------------+-------------------------------------+
    | coaddDriver.py      | HSC-R       | 11442, 11446, 11450, 11470, 11476,  |
    |                     |             | 11478  11506, 11508, 11532, 11534   |
    +---------------------+-------------+-------------------------------------+
    | multiBandDriver.py  | HSC-G, HSC-R| 9852, 9856, 9860, 9864, 9868, 9870, |
    |                     |             | 9888, 9890, 9898, 9900, 9904, 9906, |
    |                     |             | 9912, 11568, 11572, 11576, 11582,   |
    |                     |             | 11588, 11590, 11596, 11598, 11442,  |
    |                     |             | 11446, 11450, 11470, 11476, 11478,  |
    |                     |             | 11506, 11508, 11532, 11534          | 
    +---------------------+-------------+-------------------------------------+

Hardware and Software
=====================
The LSST'S lsst-dev01 system was used to process this data, along with SLURM for some of the trials.  More details can be found in DMTR-31.

For these trials, the w_2017_30 version of LSST's software stack was used. 

singleFrameDriver Trials
========================
In order to find the memory usage of a singleFrameDriver.py job, and how it scales with the number of visits and the number of cores, four main trials were run:

-  singleFrameDriver.py was submitted to slurm by the --batch-type slurm method and the memory was found via sacct

.. code-block:: python
   :name: normal --batch-type slurm method
     
   singleFrameDriver.py /datasets/hsc/repo --rerun private/thrush/RF --batch-type slurm --mpiexec='-bind-to socket' --job Memtest --id visit=11382 ccd=0..8^10..103 --cores 1 --clobber-versions

-  the memory usage was found by running singleFrame.py without slurm and extracting the memory information with /usr/bin/time

.. code-block:: python
   :name: /usr/bin/time method

   /usr/bin/time -f "\n%E elapsed, \n%U user, \n%S system, \n%M memory\n" singleFrameDriver.py /datasets/hsc/repo --rerun private/thrush/RF --id visit=11382 ccd=0..8^10..103 --cores 1 --clobber-versions

-  a salloc session was obtained on slurm and a normal singleFrameDriver.py trial was run without the --batch-type slurm option discussed in the first point

.. code-block:: python
   :name: salloc method

   # asking for the allocation on slurm:
   salloc -t 03:00:00 -N 1 -n 1
 
   # once the allocation is given, run the python executable in the background 
   # so that you can invoke top:
   singleFrameDriver.py /datasets/hsc/repo --rerun private/thrush/RF --id ccd=0..8^10..103 visit=11382 --cores 1 &
 
   # call top -b to take system information periodically so that memory usage 
   # can be tracked
   top -b > top.txt

-  singleFrameDriver.py will be run with a hand-made slurm script.

.. code-block:: python
   :name: handmade slurm script

   #!/bin/bash -l

   #SBATCH -p debug
   #SBATCH -N 1
   #SBATCH -n 1
   #SBATCH -t 03:00:00
   #SBATCH -J test

   srun singleFrameDriver.py /datasets/hsc/repo --rerun private/thrush/RF --id ccd=0..8^10..103 visit=11382 --cores 1

Baseline Memory
---------------
Prior to running the tests above, the baseline memory usage was established by running a similar trial as those above by using processCcd.py.  Of course, the processCcd.py code accomplishes the same task as the codes shown above, however processCcd.py is much more transparent with its memory usage and seems to have reasonable memory usage.  The code used in this case is that shown below:    

.. code-block:: python
   :name: processCcd baseline

   /usr/bin/time -f "\n%E elapsed, \n%U user, \n%S system, \n%M memory\n" processCcd.py /datasets/hsc/repo --rerun private/thrush/RF --id visit=11382 ccd=0..8^10..103 --clobber-versions --clobber-config

After running the code snippet above 5 times and averaging those results together, the average memory usage reported by /usr/bin/time was found to be 3,338,000 K.  However, after removing the --clobber statements from the code above, the memory usage of 5 runs averaged together came out to be 2,288,000 K.  

Batch-type Results
------------------
The code shown for the first bullet point in this section was run 5 times, and the average memory for each run was found by using sacct and finding the AveRSS.  The average memory that was found with this method was found to be 306,500 K.  Of course, this is much lower than the baseline above.

/usr/bin/time Results
---------------------


Handmade Slurm Script Results
-----------------------------


salloc Results
--------------


coaddDriver Trials
==================


multiBandDriver Trials
======================

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
