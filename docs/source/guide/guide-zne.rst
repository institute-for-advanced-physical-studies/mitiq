.. mitiq documentation file

*********************************************
Zero Noise Extrapolation
*********************************************
Zero noise extrapolation has two main components: noise scaling and then extrapolation.

======================================
Digital noise scaling: Unitary Folding
======================================
Zero noise extrapolation has two main components: noise scaling and then extrapolation.
Unitary folding is a method for noise scaling that operates directly at the gate level.
This makes it easy to use across platforms. It is especially appropriate when
your underlying noise should scale with the depth and/or the number of gates in your
quantum program. More details can be found in :cite:`Giurgica_Tiron_2020_arXiv`
where the unitary folding framework was introduced.

At the gate level, noise is amplified by mapping gates (or groups of gates) `G` to

.. math::
  G \mapsto G G^\dagger G .

This makes the circuit longer (adding more noise) while keeping its effect unchanged (because
:math:`G^\dagger = G^{-1}` for unitary gates).  We refer to this process as
*unitary folding*. If `G` is a subset of the gates in a circuit, we call it `local folding`.
If `G` is the entire circuit, we call it `global folding`.

In ``mitiq``, folding functions input a circuit and a *scale factor* (or simply *scale*), i.e., a floating point value
which corresponds to (approximately) how much the length of the circuit is scaled.
The minimum scale factor is one (which corresponds to folding no gates). A scale factor of three corresponds to folding
all gates locally. Scale factors beyond three begin to fold gates more than once.

---------------------
Local folding methods
---------------------

For local folding, there is a degree of freedom for which gates to fold first. The order in which gates are folded can
have an important effect on how the noise is caled. As such, ``mititq`` defines several local folding methods.

We introduce three folding functions:

    1. ``mitiq.zne.scaling.fold_gates_from_left``
    2. ``mitiq.zne.scaling.fold_gates_from_right``
    3. ``mitiq.zne.scaling.fold_gates_at_random``

The ``mitiq`` function ``fold_gates_from_left`` will fold gates from the left (or start) of the circuit
until the desired scale factor is reached.


.. doctest:: python

    >>> import cirq
    >>> from mitiq.zne.scaling import fold_gates_from_left

    # Get a circuit to fold
    >>> qreg = cirq.LineQubit.range(2)
    >>> circ = cirq.Circuit(cirq.ops.H.on(qreg[0]), cirq.ops.CNOT.on(qreg[0], qreg[1]))
    >>> print("Original circuit:", circ, sep="\n")
    Original circuit:
    0: ───H───@───
              │
    1: ───────X───

    # Fold the circuit
    >>> folded = fold_gates_from_left(circ, scale_factor=2.)
    >>> print("Folded circuit:", folded, sep="\n")
    Folded circuit:
    0: ───H───H───H───@───
                      │
    1: ───────────────X───

In this example, we see that the folded circuit has the first (Hadamard) gate folded.

.. note::
    ``mitiq`` folding functions do not modify the input circuit.

Because input circuits are not modified, we can reuse this circuit for the next example. In the following code,
we use the ``fold_gates_from_right`` function on the same input circuit.

.. doctest:: python

    >>> from mitiq.zne.scaling import fold_gates_from_right

    # Fold the circuit
    >>> folded = fold_gates_from_right(circ, scale_factor=2.)
    >>> print("Folded circuit:", folded, sep="\n")
    Folded circuit:
    0: ───H───@───@───@───
              │   │   │
    1: ───────X───X───X───

We see the second (CNOT) gate in the circuit is folded, as expected when we start folding from the right (or end) of
the circuit instead of the left (or start).

Finally, we mention ``fold_gates_at_random`` which folds gates according to the following rules.

    1. Gates are selected at random and folded until the input scale factor is reached.
    2. No gate is folded more than once for any ``scale_factor <= 3``.
    3. "Virtual gates" (i.e., gates appearing from folding) are never folded.

All of these local folding methods can be called with any ``scale_factor >= 1``.

------------------------------------
Any supported circuits can be folded
------------------------------------

Any program types supported by ``mitiq`` can be folded, and the interface for all folding functions is the same. In the
following example, we fold a Qiskit circuit.

.. note::
    This example assumes you have Qiskit installed. ``mitiq`` can interface with Qiskit, but Qiskit is not
    a core ``mitiq`` requirement and is not installed by default.

.. doctest:: python

    >>> import qiskit
    >>> from mitiq.zne.scaling import fold_gates_from_left

    # Get a circuit to fold
    >>> qreg = qiskit.QuantumRegister(2)
    >>> circ = qiskit.QuantumCircuit(qreg)
    >>> _ = circ.h(qreg[0])
    >>> _ = circ.cnot(qreg[0], qreg[1])
    >>> print("Original circuit:", circ, sep="\n") # doctest: +SKIP +NORMALIZE_WHITESPACE
    Original circuit:
           ┌───┐
    q31_0: ┤ H ├──■──
           └───┘┌─┴─┐
    q31_1: ─────┤ X ├
                └───┘


This code (when the print statement is uncommented) should display something like:


We can now fold this circuit as follows.

.. doctest:: python

    >>> folded = fold_gates_from_left(circ, scale_factor=2.)
    >>> print("Folded circuit:", folded, sep="\n") # doctest: +SKIP +NORMALIZE_WHITESPACE
    Folded circuit:
         ┌───┐┌───┐┌───┐
    q_0: ┤ H ├┤ H ├┤ H ├──■──
         └───┘└───┘└───┘┌─┴─┐
    q_1: ───────────────┤ X ├
                        └───┘

By default, the folded circuit has the same type as the input circuit. To return an internal ``mitiq`` representation
of the folded circuit (a Cirq circuit), one can use the keyword argument ``return_mitiq=True``.


^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Folding gates by fidelity
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In local folding methods, gates can be folded according to custom fidelities by passing the keyword argument
``fidelities`` into a local folding method. This argument should be a dictionary where each key is a string which
specifies the gate and the value of the key is the fidelity of that gate. An example is shown below where we set the
fidelity of all single qubit gates to be 1.0, meaning that these gates introduce no errors in the computation.

.. doctest:: python

    from cirq import Circuit, LineQubit, ops
    from mitiq.zne.scaling import fold_gates_at_random

    qreg = LineQubit.range(3)
    circ = Circuit(
        ops.H.on_each(*qreg),
        ops.CNOT.on(qreg[0], qreg[1]),
        ops.T.on(qreg[2]),
        ops.TOFFOLI.on(*qreg)
    )
    print(circ)
    # 0: ───H───@───@───
    #           │   │
    # 1: ───H───X───@───
    #               │
    # 2: ───H───T───X───


    folded = fold_gates_at_random(
        circ, scale_factor=3., fidelities={"single": 1.0,
                                           "CNOT": 0.99,
                                           "TOFFOLI": 0.95}
    )
    print(folded)
    # 0: ───H───@───@───@───@───@───@───
    #           │   │   │   │   │   │
    # 1: ───H───X───X───X───@───@───@───
    #                       │   │   │
    # 2: ───H───T───────────X───X───X───


We can see that only the two-qubit gates and three-qubit gates have been folded in the folded circuit.

Specific gate keys override the global "single", "double", or "triple" options. For example, the dictionary
``fidelities = {"single": 1.0, "H": 0.99}`` sets all single qubit gates to fidelity one except the Hadamard gate.


A full list of string keys for gates can be found with ``help(fold_method)`` where ``fold_method`` is a valid local
folding method. Fidelity values must be between zero and one.


--------------
Global folding
--------------

As mentioned, global folding methods fold the entire circuit instead of individual gates. An example using the same Cirq
circuit above is shown below.


.. doctest:: python

    >>> import cirq
    >>> from mitiq.zne.scaling import fold_global

    # Get a circuit to fold
    >>> qreg = cirq.LineQubit.range(2)
    >>> circ = cirq.Circuit(cirq.ops.H.on(qreg[0]), cirq.ops.CNOT.on(qreg[0], qreg[1]))
    >>> print("Original circuit:", circ, sep="\n")
    Original circuit:
    0: ───H───@───
              │
    1: ───────X───

    # Fold the circuit
    >>> folded = fold_global(circ, scale_factor=3.)
    >>> print("Folded circuit:", folded, sep="\n")
    Folded circuit:
    0: ───H───@───@───H───H───@───
              │   │           │
    1: ───────X───X───────────X───

Notice that this circuit is still logically equivalent to the input circuit, but the global folding strategy folds
the entire circuit until the input scale factor is reached. As with local folding methods, global folding can be called
with any ``scale_factor >= 3``.


----------------------
Custom folding methods
----------------------

Custom folding methods can be defined and used with ``mitiq`` (e.g., with ``mitiq.execute_with_zne``. The signature
of this function must be as follows.

.. doctest:: python

    import cirq
    from mitiq.zne.scaling import converter

    @converter
    def my_custom_folding_function(circuit: cirq.Circuit, scale_factor: float) -> cirq.Circuit:
        # Insert custom folding method here
        return folded_circuit

.. note::

    The ``converter`` decorator makes it so ``my_custom_folding_function`` can be used with any supported circuit type,
    not just Cirq circuits. The body of the ``my_custom_folding_function`` should assume the input circuit is a Cirq
    circuit, however.

This function can then be used with ``mitiq.execute_with_zne`` as an option to scale the noise:

.. doctest:: python

    # Variables circ and scale are a circuit to fold and a scale factor, respectively
    zne = mitiq.execute_with_zne(circuit, executor, scale_noise=my_custom_folding_function)


====================================================
Classical fitting and extrapolation: Factory Objects
====================================================

A :class:`.Factory` object is a self-contained representation of an error mitigation method.

This representation is not just hardware-agnostic, it is even *quantum-agnostic*,
in the sense that it mainly deals with classical data: the classical input and the classical output of a
noisy computation. Nonetheless, a factory can easily interact with a quantum system via its ``self.run`` method
which is the only interface between the "classical world" of a factory and the "quantum world" of a circuit.

The typical tasks of a factory are:

1. Record the result of the computation executed at the chosen noise level;

2. Determine the noise scale factor at which the next computation should be run;

3. Given the history of noise scale factors and results, evaluate the associated zero-noise extrapolation.

The structure of the :class:`.Factory` class is adaptive by construction, since the choice of the next noise
level can depend on the history of these values. Obviously, non-adaptive
methods are supported too and they actually represent the most common choice. Non-adaptive factories are instances
of :class:`.BatchedFactory` objects. Adaptive factories are instances of :class:`.AdaptiveFactory` objects.

Specific classes derived from the abstract class :class:`.Factory` represent different zero-noise extrapolation
methods. All the built-in factories can be found in the module :py:mod:`mitiq.zne.inference` and
are summarized in the following table.

.. _built-in-factories:

   .. autosummary::
      :nosignatures:

      mitiq.zne.inference.LinearFactory
      mitiq.zne.inference.RichardsonFactory
      mitiq.zne.inference.PolyFactory
      mitiq.zne.inference.ExpFactory
      mitiq.zne.inference.PolyExpFactory
      mitiq.zne.inference.AdaExpFactory


Once instantiated, a factory can be passed as an argument to the high-level functions contained in the module :py:mod:`mitiq.zne`.
Alternatively, a factory can be directly used to implement a zero-noise extrapolation procedure in a fully self-contained way.

To clarify this aspect, we now perform the same zero-noise extrapolation with both methods.

----------------------------------------------------------
Using a factory object with the :py:mod:`mitiq.zne` module
----------------------------------------------------------

Let us consider an ``executor`` function which is similar to the one used in
the :ref:`getting started <guide-getting-started>` section.

.. testcode::

   import numpy as np
   from cirq import Circuit, depolarize, DensityMatrixSimulator

   # initialize a backend
   SIMULATOR = DensityMatrixSimulator()
   # 5% depolarizing noise
   NOISE = 0.05

   def executor(circ: Circuit) -> float:
      """Executes a circuit with depolarizing noise and
      returns the expectation value of the projector |0><0|."""
      circuit = circ.with_noise(depolarize(p=NOISE))
      rho = SIMULATOR.simulate(circuit).final_density_matrix
      obs = np.diag([1, 0])
      expectation = np.real(np.trace(rho @ obs))
      return expectation

.. note::

   In this example we used *Cirq* but other quantum software platforms can be used,
   as shown in the :ref:`getting started <guide-getting-started>` section.

We also define a simple quantum circuit whose ideal expectation value is by construction equal to
``1.0``.

.. testcode::

   from cirq import LineQubit, X, H

   qubit = LineQubit(0)
   circuit = Circuit(X(qubit), H(qubit), H(qubit), X(qubit))
   expval = executor(circuit)
   exact = 1.0
   print(f"The ideal result should be {exact}")
   print(f"The real result is {expval:.4f}")
   print(f"The abslute error is {abs(exact - expval):.4f}")

.. testoutput::

   The ideal result should be 1.0
   The real result is 0.8794
   The abslute error is 0.1206


Now we are going to initialize three factory objects, each one encapsulating a different
zero-noise extrapolation method.

.. testcode::

   from mitiq.zne.inference import LinearFactory, RichardsonFactory, PolyFactory

   # method: scale noise by 1 and 2, then extrapolate linearly to the zero noise limit.
   linear_fac = LinearFactory(scale_factors=[1.0, 2.0])

   # method: scale noise by 1, 2 and 3, then evaluate the Richardson extrapolation.
   richardson_fac = RichardsonFactory(scale_factors=[1.0, 2.0, 3.0])

   # method: scale noise by 1, 2, 3, and 4, then extrapolate quadratically to the zero noise limit.
   poly_fac = PolyFactory(scale_factors=[1.0, 2.0, 3.0, 4.0], order=2)

The previous factory objects can be passed as arguments to the high-level functions
in ``mitiq.zne``. For example:

.. testcode::

   from mitiq.zne.zne import execute_with_zne

   zne_expval = execute_with_zne(circuit, executor, factory=linear_fac)
   print(f"Error with linear_fac: {abs(exact - zne_expval):.4f}")

   zne_expval = execute_with_zne(circuit, executor, factory=richardson_fac)
   print(f"Error with richardson_fac: {abs(exact - zne_expval):.4f}")

   zne_expval = execute_with_zne(circuit, executor, factory=poly_fac)
   print(f"Error with poly_fac: {abs(exact - zne_expval):.4f}")

.. testoutput::

   Error with linear_fac: 0.0291
   Error with richardson_fac: 0.0070
   Error with poly_fac: 0.0110

We can also specify the number of shots to use for each noise-scaled circuit.

.. testcode::

   from mitiq.zne.inference import LinearFactory

   # Specify the number of shots for each scale factor.
   factory_with_shots = LinearFactory(scale_factors=[1.0, 2.0], shot_list=[100, 200])

In this case the factory will pass the number of shots from the `shot_list` to the `executor`. Accordingly, the
`executor` should support a `shots` keyword argument, otherwise the shot values will go unused.

---------------------------------------------
Directly using a factory for error mitigation
---------------------------------------------

Zero-noise extrapolation can also be implemented by directly using the methods ``self.run``
and ``self.reduce`` of a :class:`.Factory` object.

The method ``self.run`` evaluates different expectation values at different noise levels
until a sufficient amount of data is collected.

The method ``self.reduce`` instead returns the final zero-noise extrapolation which, in practice,
corresponds to a statistical inference based on the measured data.

.. testcode::

   # we import one of the built-in noise scaling function
   from mitiq.zne.scaling import fold_gates_at_random

   linear_fac.run(circuit, executor, scale_noise=fold_gates_at_random)
   zne_expval = linear_fac.reduce()
   print(f"Error with linear_fac: {abs(exact - zne_expval):.4f}")

   richardson_fac.run(circuit, executor, scale_noise=fold_gates_at_random)
   zne_expval = richardson_fac.reduce()
   print(f"Error with richardson_fac: {abs(exact - zne_expval):.4f}")

   poly_fac.run(circuit, executor, scale_noise=fold_gates_at_random)
   zne_expval = poly_fac.reduce()
   print(f"Error with poly_fac: {abs(exact - zne_expval):.4f}")

.. testoutput::

   Error with linear_fac: 0.0291
   Error with richardson_fac: 0.0070
   Error with poly_fac: 0.0110

Behind the scenes, a factory object collects different expectation values at different scale factors.
After running a factory, this information can be accessed with appropriate *get* methods. For example:

.. testcode::

   scale_factors = poly_fac.get_scale_factors()
   print("Scale factors:", scale_factors)
   exp_values = poly_fac.get_expectation_values()
   print("Expectation values:", np.round(exp_values, 2))

.. testoutput::

   Scale factors: [1. 2. 3. 4.]
   Expectation values: [0.88 0.79 0.72 0.67]

If the user has manually evaluated a list of expectation values associated to a list of scale factors, the
simplest way to estimate the corresponding zero-noise limit is to directly call the static `extrapolate` method of the 
desired factory class (in this case initializing a factory object is unnecessary).  For example:

.. testcode::

   zero_limit = PolyFactory.extrapolate(scale_factors, exp_values, order=2)
   print(f"Error with PolyFactory.extrapolate method: {abs(exact - zero_limit):.4f}")

.. testoutput::

   Error with PolyFactory.extrapolate method: 0.0110

Both the zero-noise value and the optimal parameters of the fit can be returned from `extrapolate` by specifying `full_output = True`.

---------------------------------------------
Advanced usage of a factory
---------------------------------------------

.. note::
   This section can be safely skipped by all the readers who are interested
   in a standard usage of ``mitiq``.
   On the other hand, more experienced users and ``mitiq`` contributors
   may find this content useful to understand how a factory object actually
   works at a deeper level.

In this advanced section we present a *low-level usage* and a *very-low-level usage* of a factory.
Again, for simplicity, we solve the same zero-noise extrapolation problem that we have just considered
in the previous sections.

Eventually we will also discuss how the user can easily define a custom factory class.


^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Low-level usage: the ``run_classical`` method.
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The ``self.run`` method takes as arguments a circuit and other "quantum" objects.
On the other hand, the core computation performed by any factory corresponds to
a some classical computation applied to the measurement results.

At a lower level, it is possible to clearly separate the quantum and the
classical steps of a zero-noise extrapolation procedure.
This can be done by defining a function which maps a noise scale factor to the
corresponding expectation value.

.. testcode::

   def noise_to_expval(scale_factor: float) -> float:
      """Function returning an expectation value for a given scale_factor."""
      # apply noise scaling
      scaled_circuit = fold_gates_at_random(circuit, scale_factor)
      # return the corresponding expectation value
      return executor(scaled_circuit)

.. note::
   The body of the previous function contains the execution of a quantum circuit.
   However, if we see it as a "black-box", it is just a classical function mapping real
   numbers to real numbers.

The function ``noise_to_expval`` encapsulate the "quantum part" of the problem. The "classical
part" of the problem can be solved by passing ``noise_to_expval``
to the ``self.run_classical`` method of a factory.
This method will repeatedly call ``noise_to_expval`` for different
noise levels, so one can view ``self.run_classical`` as the classical counterpart of the quantum method
``self.run``.

.. testcode::

   linear_fac.run_classical(noise_to_expval)
   zne_expval = linear_fac.reduce()
   print(f"Error with linear_fac: {abs(exact - zne_expval):.4f}")

   richardson_fac.run_classical(noise_to_expval)
   zne_expval = richardson_fac.reduce()
   print(f"Error with richardson_fac: {abs(exact - zne_expval):.4f}")

   poly_fac.run_classical(noise_to_expval)
   zne_expval = poly_fac.reduce()
   print(f"Error with poly_fac: {abs(exact - zne_expval):.4f}")

.. testoutput::

   Error with linear_fac: 0.0291
   Error with richardson_fac: 0.0070
   Error with poly_fac: 0.0110

.. note::
   With respect to ``self.run``, the ``self.run_classical`` method is much more flexible and
   can be applied whenever the user is able to autonomously scale the noise level associated
   to an expectation value. Indeed, the function ``noise_to_expval`` can represent any experiment
   or any simulation in which noise can be artificially increased. The scenario
   is therefore not restricted to quantum circuits but can be easily extended to
   annealing devices or to gates which are controllable at a pulse level. In principle,
   one could even use the ``self.run_classical`` method to mitigate experiments which are
   unrelated to quantum computing.


^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Defining a custom factory
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If necessary, the user can modify an existing extrapolation methods by subclassing
one of the :ref:`built-in factories <built-in-factories>`.

Alternatively, a new adaptive extrapolation method can be derived from the abstract class :class:`.Factory`.
In this case its core methods must be implemented:
``self.next``, ``self.push``, ``self.is_converged``, ``self.reduce``, etc.
Typically, the ``self.__init__`` method must be overridden.

A new non-adaptive method can instead be derived from the abstract :class:`.BatchedFactory` class.
In this case it is usually sufficient to override only the ``self.__init__`` and
the ``self.reduce`` methods, which are responsible for the initialization and for the
final zero-noise extrapolation, respectively.

---------------------------------------------
Example: a simple custom factory
---------------------------------------------

Assume that, from physical considerations, we know that the ideal expectation value
(measured by some quantum circuit) must always be within two limits: ``min_expval`` and ``max_expval``.
For example, this is a typical situation whenever the measured observable has a bounded
spectrum.

We can define a linear non-adaptive factory which takes into account this information
and clips the result if it falls outside its physical domain.

.. testcode::

   from typing import Iterable
   from mitiq.zne.inference import BatchedFactory, mitiq_polyfit
   import numpy as np

   class MyFactory(BatchedFactory):
      """Factory object implementing a linear extrapolation taking
      into account that the expectation value must be within a given
      interval. If the zero-noise limit falls outside the
      interval, its value is clipped.
      """

      def __init__(
            self,
            scale_factors: Iterable[float],
            min_expval: float,
            max_expval: float,
         ) -> None:
         """
         Args:
            scale_factors: The noise scale factors at which
                           expectation values should be measured.
            min_expval: The lower bound for the expectation value.
            min_expval: The upper bound for the expectation value.
         """
         super(MyFactory, self).__init__(scale_factors)
         self.min_expval = min_expval
         self.max_expval = max_expval

      def reduce(self) -> float:
         """
         Fit a linear model and clip its zero-noise limit.

         Returns:
            The clipped extrapolation to the zero-noise limit.
         """
         # Fit a line and get the intercept
         _, intercept = mitiq_polyfit(
            self.get_scale_factors(), self.get_expectation_values(), deg=1
        )

         # Return the clipped zero-noise extrapolation.
         return np.clip(intercept, self.min_expval, self.max_expval)

.. testcleanup::

   fac = MyFactory([1, 2, 3], min_expval=0.0, max_expval=2.0)
   fac.run_classical(noise_to_expval)
   assert np.isclose(fac.reduce(), 1.0, atol=0.1)
   # Linear model with a large zero-noise limit
   noise_to_large_expval = lambda x : noise_to_expval(x) + 10.0
   fac.run_classical(noise_to_large_expval)
   # assert the output is clipped to 2.0
   assert np.isclose(fac.reduce(), 2.0)

This custom factory can be used in exactly the same way as we have
shown in the previous section. By simply replacing ``LinearFactory``
with ``MyFactory`` in all the previous code snippets, the new extrapolation
method will be applied.

-------------------------------------------------
Regression tools in :py:mod:`mitiq.zne.inference`
-------------------------------------------------

In the body of the previous ``MyFactory`` example, we imported and used the :py:func:`.mitiq_polyfit` function.
This is simply a wrap of :py:func:`numpy.polyfit`, slightly adapted to the notion and to the error types
of ``mitiq``. This function can be used to fit a polynomial ansatz to the measured expectation values. This function performs
a least squares minimization which is **linear** (with respect to the coefficients) and therefore admits an algebraic solution.

Similarly, from :py:mod:`mitiq.zne.inference` one can also import :py:func:`.mitiq_curve_fit`,
which is instead a wrap of :py:func:`scipy.optimize.curve_fit`. Differently from :py:func:`.mitiq_polyfit`,
:py:func:`.mitiq_curve_fit` can be used with a generic (user-defined) ansatz.
Since the fit is based on a numerical **non-linear** least squares minimization, this method may fail to converge
or could be subject to numerical instabilities.
