Hawkes processes simulation
===========================

This example illustrates the use of `tick.simulation.SimuHawkes`, and its kernels
`tick.simulation.hawkes_kernels.HawkesKernelExp`,
`tick.simulation.hawkes_kernels.HawkesKernelPowerLaw` and
`tick.simulation.hawkes_kernels.HawkesKernelTimeFunc`

.. contents::
    :depth: 3
    :backlinks: none


.. testsetup:: *

    from tick.simulation import SimuHawkes, HawkesKernelExp, HawkesKernelPowerLaw, \
        HawkesKernelTimeFunc
    import matplotlib.pyplot as plt
    from pylab import rcParams
    from tick.base import TimeFunction
    import numpy as np
    import itertools
    np.set_printoptions(precision=4)


.. warning::

    The spectral radius of a stable Hawkes process is < 1.
    DO NOT perform simulation of a non stable Hawkes process,
    as this will not work as it might last forever.

One dimensional
---------------

Exponential kernel
~~~~~~~~~~~~~~~~~~

Here we plot the track recorded intensity along with the ticks

.. plot:: modules/examples/simulation/code_samples/hawkes_1d_simu.py
    :include-source:


Multi-dimensional
-----------------

Exponential kernel
~~~~~~~~~~~~~~~~~~

Here we use a second technique to generate matrix of exponential
kernels :math:`\alpha \beta e^{-\beta x}`. We just need to give, when creating
the SimuHawkes object a matrix of :math:`\alpha`, a matrix of :math:`\beta` and
the vector of :math:`\mu` (actually you could set the mus using the set_mu
method)

.. plot:: modules/examples/simulation/code_samples/hawkes_multidim_simu.py
    :include-source:


Truncated power law kernels
~~~~~~~~~~~~~~~~~~~~~~~~~~~

A truncated power law kernel has the form
kernel(x)=α(x+δ)−β,  ∀x∈[0,support]

The support can be specified directly or by specifying a
truncature error using

support=kernel−1(error)

In the following example we shall build a 2-dimensional Hawkes process
with one kernel used on the diagonal and another one for 2→12→1
cross-influence (cross-influence 1→21→2 will be 0)

Let us first build and look at the diagonal kernel

.. testcode:: [hawkes_power_law]

    kernel_diag = HawkesKernelPowerLaw(1, 1, 3,error=1e-5)

.. doctest:: [hawkes_power_law]

    >>> print("Kernel support is %g" % kernel_diag.get_support())
    Kernel support is 45.4159


Then the cross kernel

.. testcode:: [hawkes_power_law]

    kernel_cross = HawkesKernelPowerLaw(2, 1, 2, error=1e-5)


Let's define the SimuHawkes object. Here we use the matrix
notation to build the SimuHawkes object.

.. testcode:: [hawkes_power_law]

    hawkes = SimuHawkes(kernels =
                    [[kernel_diag, kernel_cross],
                    [0, kernel_diag]],
                    baseline=[0.5, 0.5],
                    verbose=False)


We compute now the mean intensity vector

.. doctest:: [hawkes_power_law]

    >>> mi=hawkes.mean_intensity()
    >>> print("The expected intensity is : %s" % mi)
    The expected intensity is : [ 4.9706  0.9995]


We are ready for the Hawkes simulation.

.. testcode:: [hawkes_power_law]

    run_time = 200

    # We activate intensity tracking and launch the process during t = runtime
    dt = 0.01
    hawkes.track_intensity(dt)
    hawkes.end_time = run_time
    hawkes.simulate()

    # We obtain all the jumps from the process for each coordinate
    timestamps = hawkes.timestamps

    # And the evolution of its intensity over time for each coordinate
    intensity = hawkes.tracked_intensity
    intensity_times = hawkes.intensity_tracked_times


.. doctest:: [hawkes_power_law]

    >>> print("hawkes simulated with %i points" % hawkes.n_total_jumps) # doctest: +SKIP
    hawkes simulated with 891 points
    >>> ai = [len(t)/run_time for t in timestamps]
    >>> print("The realized averaged intensity is %s" % ai) # doctest: +SKIP
    The realized averaged intensity is [12.92, 2.525]

Let's summarize in a plot

.. plot:: modules/examples/simulation/code_samples/hawkes_power_law.py
    :include-source:


Time Functions and small negative kernels
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. plot:: modules/examples/simulation/code_samples/hawkes_time_func_simu.py
    :include-source:
