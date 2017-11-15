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

<table>
  <tr>
    <th>Code Name</th>
    <th>Bands</th>
    <th>Visits</th>
  </tr>
  <tr>
    <td>singleFrameDriver.py</td>
    <td>HSC-G</td>
    <td>11382 and 11690</td>
  </tr>
  <tr>
    <td>coaddDriver.py</td>
    <td>HSC-R</td>
    <td>11442, 11446, 11450, 11470, 11476, 11478, 11506, 11508, 11532, 11534</td>
  </tr>
  <tr>
    <td> multiBandDriver.py </td>
    <td> HSC-G and HSC-R </td>
    <td> 9852, 9856, 9860, 9864, 9868, 9870, 9888, 9890, 9898, 9900, 9904, 9906, 9912, 11568, 11572, 11576, 11582, 11588, 11590, 11596, 11598, 11442, 11446, 11450, 11470, 11476, 11478, 11506, 11508, 11532, 11534 </td>
  </tr>
 </table>

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
