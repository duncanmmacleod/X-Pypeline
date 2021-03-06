.. _examples:

########################
xpipeline an explanation
########################

============
Introduction
============
The story of one “trial” of core X-Pipeline,

There is some timestamp considered as an “event time”

Some block of data from N detectors if obtain around this event time.

A time-frequency is made for a given fftlength, and based on time delays between detectors which is based on a potential sky location of the “source”, the tfmaps of N-1 detectors is phase shifted appropriately.

From these individual maps loud pixels are identified and then a “coherent” set of pixels is calculated from the overlap of these individually loud pixels.

a coherent energy is calculated and a threshold on loud coherent pixels is used,

These final loud enough coherent pixels are then turned from individual pixels into clusters using whatever clustering algorithm

these clusters then are assigned a variety of *likelihoods* these *likelihoods* for each cluster are based on the concept of translating the data in the dominant polarization frame i.e. plus and cross polarized time frequency maps instead of the standard time frequency maps considered above.

So the final important stat calculated is energy_of_cluster * likelihood_of_cluster for a given likelihood.

These likelihood rely largely on the ability to project the N stream of gravitational wave data in the relative antenna response pattern weighted time frequency space (so fplus fcross, fscalar, etc)

In order to calculate these antenna weighted tfmaps we need to re-weight the ASDs of the data streams appropriately. This re-weighted ASD is then multiplied with the time frequency map obtained normally above.


The Time-Frequency Map
----------------------
The following is all open data obtained via `LOSC <https://losc.ligo.org/>`_

.. ipython::

    In [1]: from xpipeline.core.xtimeseries import XTimeSeries

    In [2]: data = XTimeSeries.read('examples/GW150914.gwf', channels=['H1:GDS-CALIB_STRAIN','L1:GDS-CALIB_STRAIN']) 

    In [3]: asds = data.asd(1.0)

    In [4]: whitened_timeseries = data.whiten(asds)

    In [5]: tfmaps = whitened_timeseries.spectrogram(1. /64)

    In [6]: gps = 1126259462.427

    In [7]: plot = tfmaps.plot(figsize=[8, 4])

    In [8]: for ax in plot.axes:
       ...:     ax.set_xlim(gps - 0.15, gps + 0.05)
       ...:     ax.set_epoch(gps)
       ...:     ax.set_xlabel('Time [milliseconds]')
       ...:     ax.set_ylim(20, 500)

    @savefig plot-time-frequency-map.png
    In [9]: plot

    In [10]: plot = tfmaps.to_coherent().plot(figsize=[8, 4])

    In [11]: for ax in plot.axes:
        ...:     ax.set_xlim(gps - 0.15, gps + 0.05)
        ...:     ax.set_epoch(gps)
        ...:     ax.set_xlabel('Time [milliseconds]')
        ...:     ax.set_ylim(20, 500)
        ...:     

    @savefig plot-time-frequency-map-coherent.png
    In [12]: plot

The Dominant Polarization Frame
-------------------------------

.. ipython::

    In [13]: from xpipeline.core.xdetector import compute_antenna_patterns

    In [14]: phi = 0.7728; theta = 1.4323

    In [15]: antenna_patterns = compute_antenna_patterns(['H1', 'L1'], phi, theta, antenna_patterns=['f_plus', 'f_cross', 'f_scalar'])

    In [16]: projected_asds = asds.project_onto_antenna_patterns(antenna_patterns, to_dominant_polarization_frame=True)

    In [17]: projected_tfmaps = tfmaps.to_dominant_polarization_frame(projected_asds)

    In [18]: plot = projected_tfmaps['f_plus'].plot(figsize=[8, 4])

    In [19]: for ax in plot.axes:
        ...:     ax.set_xlim(gps - 0.15, gps + 0.05)
        ...:     ax.set_epoch(gps)
        ...:     ax.set_xlabel('Time [milliseconds]')
        ...:     ax.set_ylim(20, 500)

    @savefig plot-time-frequency-map-dpf-plus.png
    In [20]: plot

xpipeline likelihoods
---------------------
Now we have a basis to determine whether or not a particular cluster of pixels
can be considered likely was a gravitational wave

A gravitational wave not only should be coherent between the multiple data streams
but if it originated from a certain part of the sky the projection of the cluster onto
the plus and cross polarization plane (i.e. `projected_tfmaps` should also be large.

.. ipython::

    In [21]: from xpipeline.core.xlikelihood import XLikelihood

    In [22]: import numpy as np

    In [22]: mpp = projected_asds['f_plus'].to_m_ab()

    In [23]: mcc = projected_asds['f_cross'].to_m_ab()

    In [24]: wfptimefrequencymap = projected_tfmaps['f_plus'].to_coherent()

    In [25]: wfctimefrequencymap = projected_tfmaps['f_cross'].to_coherent()

    In [26]: frequencies = np.arange(0,513,64)

    In [27]: mpp = mpp[frequencies]

    In [28]: mcc = mcc[frequencies]

    In [29]: likelihood_map = XLikelihood.standard(mpp, mcc, wfptimefrequencymap, wfctimefrequencymap)

    In [30]: plot = likelihood_map.plot(figsize=[8, 4])

    In [31]: for ax in plot.axes:
        ...:     ax.set_xlim(gps - 0.15, gps + 0.05)
        ...:     ax.set_epoch(gps)
        ...:     ax.set_xlabel('Time [milliseconds]')
        ...:     ax.set_ylim(20, 500)

    @savefig plot-time-frequency-map-standard-likelihood.png
    In [32]: plot
