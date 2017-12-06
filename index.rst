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

Finding Memory Usages
=====================
Before delving into the respective codes and their data usage, it is important to discuss how the memory usages are found.  In this section, I will be discussing how the "sacct" slurm calls are invoked as well as how I extracted the memory usage from the /usr/bin/time code calls below. 

Simply put, the "sacct" slurm call allows us to look at the resources used by a code after it was run.  In my case, I used sacct in order to get the average memory usage of a job and reported that as the memory usage.  An example of such a call would be as follows:

.. code-block:: python
   :name: sacct

   sacct -u thrush --format=JobID,JobName,AveRSS,Elapsed 

Here, sacct looks for jobs associated with my username, and then gives me the job IDs, the job name that I assigned it, the average memory used for the number of cores used, and the time it took for the job to run.  Thus, since I'm using one core, I can say that the average memory reported is the memory used.    

The /usr/bin/time calls work slightly differently.  These calls (shown below) simply look at the real-time memory usage of the command-line code call by looking into the /usr/bin/time files and reporting back the time the job to be completed, the user name, the system used, and the memory utilized. 
  
.. code-block:: python
   :name: processCcd baseline

   /usr/bin/time -f "\n%E elapsed, \n%U user, \n%S system, \n%M memory\n" processCcd.py /datasets/hsc/repo --rerun private/thrush/RF --id visit=11382 ccd=0..8^10..103 --clobber-versions --clobber-config

Once the job is completed, the data from /usr/bin/time is printed on the screen, where it can be recorded.  Of course, this is done so as to record the memory usage of a job that is not using slurm to run, but instead is run on the command line. 

Baseline Memory
===============

processCcd.py Trial
-------------------
Prior to running the tests below, the baseline memory usage was established by running a similar trial as those below by using processCcd.py.  Of course, the processCcd.py code accomplishes the same task as the codes shown in the singleFrameDriver section, however processCcd.py is much more transparent with its memory usage and seems to have reasonable memory usage.  The code used in this case is that shown below:    

.. code-block:: python
   :name: processCcd baseline

   /usr/bin/time -f "\n%E elapsed, \n%U user, \n%S system, \n%M memory\n" processCcd.py /datasets/hsc/repo --rerun private/thrush/RF --id visit=11382 ccd=0..8^10..103 --clobber-versions --clobber-config

After running the code snippet above 5 times and averaging those results together, the average memory usage reported by /usr/bin/time was found to be 3,338,000 K.  However, after removing the --clobber statements from the code above, the memory usage of 5 runs averaged together came out to be 2,288,000 K.  

Different Number of Visits and CCDs
-----------------------------------
Before testing the methods discussed above, it is important to see first how memory scales with the number of visits and ccds that will be used.  In order to ascertain this scaling, three different trials were run for the batch-type method, the /usr/bin/time method and the homemade slurm script method discussed above ( the salloc method was not used as it give results almost identical to the batch-type method): one visit and one ccd, one visit with two ccds, and two visits with one ccd.  

Batch-Type trials
^^^^^^^^^^^^^^^^^
The batch-type method was run with the three code snippets shown below.  

.. code-block:: python
   :name: batch-type 1v1c

   singleFrameDriver.py /datasets/hsc/repo --rerun $DIR --batch-type slurm --mpiexec='-bind-to socket' --job Memtest --id visit=11382 ccd=0 --cores 1 --time 400 --clobber-versions


.. code-block:: python
   :name: batch-type 1v2c

   singleFrameDriver.py /datasets/hsc/repo --rerun $DIR --batch-type slurm --mpiexec='-bind-to socket' --job Memtest --id visit=11382 ccd=0^1 --cores 1 --time 400 --clobber-versions

.. code-block:: python
   :name: batch-type 2v1c

   singleFrameDriver.py /datasets/hsc/repo --rerun $DIR --batch-type slurm --mpiexec='-bind-to socket' --job Memtest --id visit=11382^11690 ccd=0 --cores 1 --time 400 --clobber-versions

Surprisingly, all three trials gave the same memory usage: 2592K, which seems to vastly underestimate the actual memory usage. As you can see, these results imply that the memory usage in this case does not scale at all with the number of ccds or visits. 

/usr/bin/time Trials
^^^^^^^^^^^^^^^^^^^^
For these runs, the following codes were used:

.. code-block:: python
   :name: usr/bin/time 1v1c

   /usr/bin/time -f "\n%E elapsed, \n%U user, \n%S system, \n%M memory\n" singleFrameDriver.py /datasets/hsc/repo --rerun $DIR --id visit=11382 ccd=0 --cores 1 --clobber-versions

.. code-block:: python
   :name: usr/bin/time 1v2c

   /usr/bin/time -f "\n%E elapsed, \n%U user, \n%S system, \n%M memory\n" singleFrameDriver.py /datasets/hsc/repo --rerun $DIR --id visit=11382 ccd=0^1 --cores 1 --clobber-versions

.. code-block:: python
   :name: usr/bin/time 2v1c

   /usr/bin/time -f "\n%E elapsed, \n%U user, \n%S system, \n%M memory\n" singleFrameDriver.py /datasets/hsc/repo --rerun $DIR --id visit=11382^11690 ccd=0 --cores 1 --clobber-versions

The first run gave a memory usage of 1,238,300K which seems high when compared to the /usr/bin/time trials shown below.  Similarly, the second run gave a memory usage of 1,373,300K, and the third run had a memory usage of 1,374,100K.  In this case, the memory usage does not scale linearly, but (as expected) it does scale with the number of ccd's and the number of visits. 

Homemade slurm script trials
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Finally, the following base codes was run:

.. code-block:: shell
   :name: slurm 1v1c
   
   #!/bin/bash -l
 
   #SBATCH -p debug
   #SBATCH -N 1
   #SBATCH -n 1
   #SBATCH -t 03:00:00
   #SBATCH -J test
 
   srun singleFrameDriver.py /datasets/hsc/repo --rerun private/thrush/RD --id ccd=0 visit=11382 --cores 1

When the code was ran as-is, sacct MaxRSS reported 398,400K in memory usage. When ccd=0^1 visit=11382, the memory usage jumped to 419,300K, as was also the case for the ccd=0 visit=11382^11690 run.

Like the /usr/bin/time trials above, the memory usage does not scale linearly as there seems to be a base memory usage that is needed. However, an increase in either visits or ccds produces roughly the same increase in memory usage. Additionally, as stated in previous sections, although this underestimates memory usage when compared to /usr/bin/time trials, this seems to be a step up from the memory reporting done by jobs who employ the --slurm method of invoking slurm.


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

Batch-type Results
------------------
The code shown for the first bullet point in this section was run 5 times, and the average memory for each run was found by using sacct and finding the AveRSS.  The average memory that was found with this method was found to be 306,500 K.  Of course, this is much lower than the baseline above.

/usr/bin/time Results
---------------------
Unlike the other three trials discussed in this section, this is most like the baseline trial from the previous section since both code calls do not use slurm.  The average memory usage found with the code as shown above was 2,725,000 K (averaged over 5 trials).  However, when the --clobber option was deleted, the memory usage jumped up to 3,155,000 K (averaged over 5 trials).  Of course, this is exactly opposite the tred suggested by processCcd.py.

Handmade Slurm Script Results
-----------------------------
The homemade slurm script had memory usage at 398,000 K (averaged over 5 trials) as reported by sacct for AveRSS after the trials were run.  Of course, this is slightly higher than expected from a code that is so similar to the Batch trial described above. 

salloc Results
--------------
Although this type of code call is slightly different from those described in the "Batch-type Results" described above, the results were very similar.  By looking into top.txt, the average memory usage was found to be 310,500 K.  Of course, this is not so shocking as salloc simply acts as an interactive slurm session, so although this call looks quite different from the batch-type results, they are essentially the same.    

Conclusion
----------
After searching through the literatures, it would seem that while the /usr/bin/time trials account for SWAP when it reports its memory usage, slurm does not.  Because of this, it is reasonable to say that the /usr/bin/time should be larger.  However, there could be some memory saving tricks employed by slurm that I am not accounting for which would make their memory reporting just as trustworty.  

coaddDriver Trials
==================
In order to investigate the memory usage of coaddDriver, I used three main methods:

-  tracking memory usage with /usr/bin/time.

.. code-block:: python
   :name: coaddDriver /usr/bin/time

   /usr/bin/time -f "\n%E elapsed, \n%U user, \n%S system, \n%M memory\n" coaddDriver.py /datasets/hsc/repo --rerun private/thrush/RD:private/thrush/RE --cores 1 --id tract=8766^8767 filter=HSC-R --selectId ccd=0..8^10..103 visit=11442^11446^11450^11470^11476^11478^11506^11508^11532^11534 

-  tracking the memory usage of a --slurm job with sacct after the job has run.

.. code-block:: python
   :name: coaddDriver --slurm
   
   coaddDriver.py  /datasets/hsc/repo --rerun private/thrush/RD:private/thrush/RF --batch-type=slurm --mpiexec='-bind-to socket' --job coaddWR --time 600 --cores 1 --id tract=8766^8767 filter=HSC-R --selectId ccd=0..8^10..103 visit=11442^11446^11450^11470^11476^11478^11506^11508^11532^11534 

-  tracking the memory usage of a hand-made slurm script with sacct after the job has run

.. code-block:: shell
   :name: coaddDriver trial

   #!/bin/bash -l

   #SBATCH -p debug
   #SBATCH -N 1
   #SBATCH -n 1
   #SBATCH -t 00:30:00
   #SBATCH -J coaddWRtest

   srun coaddDriver.py /datasets/hsc/repo --rerun private/thrush/RD:private/thrush/RG --id tract=8766^8767 filter=HSC-R --selectId ccd=0..8^10..103 visit=11442^11446^11450^11470^11476^11478^11506^11508^11532^11534 --cores 1
 

It should be noted that in order to set up the necessary files to run coaddDriver.py, I ran the following script, where I only used WideR visits in order to cut down on time.

.. code-block:: shell
   :name: beginning shell

   #!/bin/bash

   DIR=private/thrush/RD 

   export wideVisitsR=11442^11446^11450^11470^11476^11478^11506^11508^11532^11534

   makeSkyMap.py /datasets/hsc/repo --rerun $DIR

   singleFrameDriver.py /datasets/hsc/repo --rerun $DIR --batch-type slurm --mpiexec='-bind-to socket' --job WideR --id visit=$wideVisitsR ccd=0..8^10..103 --cores 112 --time 900 --clobber-versions

   mosaic.py /datasets/hsc/repo --rerun $DIR --numCoresForRead=12 --id tract=8766 ccd=0..8^10..103 visit=$wideVisitsR --diagnostics --diagDir=/scratch/thrush/anyPath/RC/mosaic_diag/R --clobber-versions
   mosaic.py /datasets/hsc/repo --rerun $DIR --numCoresForRead=12 --id tract=8767 ccd=0..8^10..103 visit=$wideVisitsR --diagnostics --diagDir=/scratch/thrush/anyPath/RC/mosaic_diag/R --clobber-versions

Results
-------

After running the /usr/bin/time trial, the memory usage was found to be approximately 912400 K.  However, the --slurm trial only reported a memory usage of 2592K, while the hand-made slurm script reported a memory usage of 371100 K.  All of these computations took approximately 20 minutes to complete, on average.  

These results mirror those of the singleFrameDriver trials above in that the largest memory usage belongs to the /usr/bin/time run, while the smallest memory usage belongs to the --slurm job. As stated in the singleFrameDriver section above, I believe that /usr/bin/time is more accurate in reporting its memory usage simply because it accounts for SWAP memory usage as well as normal memory usage, thus giving a more holistic view of the situation.

multiBandDriver Trials
======================

In order to set up the correct dataset that will be used for the multiBandDriver trials, the following code was run:

.. code-block:: shell
   :name: timecheckMBD trial

   #!/bin/bash


   DIR=private/thrush/RD  

   export wideVisitsG=9852^9856^9860^9864^9868^9870^9888^9890^9898^9900^9904^9906^9912^11568^11572^11576^11582^11588^11590^11596^11598
   export wideVisitsR=11442^11446^11450^11470^11476^11478^11506^11508^11532^11534
   makeSkyMap.py /datasets/hsc/repo --rerun $DIR

   singleFrameDriver.py /datasets/hsc/repo --rerun $DIR --batch-type slurm --mpiexec='-bind-to socket' --job WideG --id visit=$wideVisitsG ccd=0..8^10..103 --cores 112 --time 900 --clobber-versions
   singleFrameDriver.py /datasets/hsc/repo --rerun $DIR --batch-type slurm --mpiexec='-bind-to socket' --job WideR --id visit=$wideVisitsR ccd=0..8^10..103 --cores 112 --time 900 --clobber-versions

   mosaic.py /datasets/hsc/repo --rerun $DIR --numCoresForRead=12 --id tract=8766 ccd=0..8^10..103 visit=$wideVisitsG --diagnostics --diagDir=/scratch/thrush/anyPath/RC/mosaic_diag/G --clobber-versions
   mosaic.py /datasets/hsc/repo --rerun $DIR --numCoresForRead=12 --id tract=8767 ccd=0..8^10..103 visit=$wideVisitsG --diagnostics --diagDir=/scratch/thrush/anyPath/RC/mosaic_diag/G --clobber-versions
   mosaic.py /datasets/hsc/repo --rerun $DIR --numCoresForRead=12 --id tract=8766 ccd=0..8^10..103 visit=$wideVisitsR --diagnostics --diagDir=/scratch/thrush/anyPath/RC/mosaic_diag/R --clobber-versions
   mosaic.py /datasets/hsc/repo --rerun $DIR --numCoresForRead=12 --id tract=8767 ccd=0..8^10..103 visit=$wideVisitsR --diagnostics --diagDir=/scratch/thrush/anyPath/RC/mosaic_diag/R --clobber-versions

   coaddDriver.py  /datasets/hsc/repo --rerun $DIR --batch-type=slurm --mpiexec='-bind-to socket' --job coaddWG --time 200 --nodes 1 --procs 12  --id tract=8766^8767 filter=HSC-G --selectId ccd=0..8^10..103 visit=$wideVisitsG 
   coaddDriver.py  /datasets/hsc/repo --rerun $DIR --batch-type=slurm --mpiexec='-bind-to socket' --job coaddWR --time 200 --nodes 1 --procs 12 --id tract=8766^8767 filter=HSC-R --selectId ccd=0..8^10..103 visit=$wideVisitsR 

There are three main methods that I used in order to find the memory usage of one multiBandDriver job, where the G and R bands are combined for "wide" visits. In order to reduce runtime of the code, only 1 patch of the sky is used so as to reduce the computation time down to an hour.  The three methods include:

-  using the --slurm method

.. code-block:: python
   :name: MBD --slurm

   multiBandDriver.py /datasets/hsc/repo --rerun $DIR:/scratch/thrush/anyPath/RG --batch-type=slurm --mpiexec='-bind-to socket' --job mtWide --cores 1 --time 8000 --id tract=8766 patch=1,1 filter=HSC-G^HSC-R --clobber-versions


-  creating a handmade slurm script

.. code-block:: shell
   :name: MBD hand made slurm script

   #!/bin/bash -l

   #SBATCH -p debug
   #SBATCH -n 1
   #SBATCH -N 1
   #SBATCH -t 96:00:00
   #SBATCH -J mtWide_test

   srun multiBandDriver.py /datasets/hsc/repo --rerun private/thrush/RD:private/thrush/RH --cores 1 --id tract=8766 patch=1,1 filter=HSC-G^HSC-R --clobber-versions

-  running the multiBandDriver code with the /usr/bin/time method 

.. code-block:: python
   :name: MBD /usr/bin/time method

   /usr/bin/time -f "\n%E elapsed, \n%U user, \n%S system, \n%M memory\n" multiBandDriver.py /datasets/hsc/repo --rerun private/thrush/RD:/scratch/thrush/anyPath/RH --job mtWide_bin --cores 1 --id tract=8766 patch=1,1 filter=HSC-G^HSC-R
 
Usual multiBandDriver jobs combine more than 2 bands, but I cut down the number of bands in the interest of time.

Results
-------

The --slurm method used 2600 K of memory in order to work. However, the handmade slurm script used 365,512 K of memory in order to work. Both of these seem strangely low, but they did finish successfully.  On the other hand, the /usr/bin/time trial used 1,727,120K of memory.  Of course, these results echo those given above for the singleFrameDriver and coaddDriver codes.



.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
