.. _usage-workflows-probeParticles:

Probe Particles
---------------

.. sectionauthor:: Axel Huebl

Probe particles ("probes") can be used to record field quantities at selected positions over time.

As a geometric data-reduction technique, analyzing the discrete, regular field of a particle-in-cell simulation only at selected points over time can greatly reduce the need for I/O.
Such particles are often arranged at isolated points, regularly as along lines, in planes or in any other user-defined manner.

Probe particles are usually neutral, non-interacting test particles that are statically placed in the simulation or co-moving with along pre-defined path.
Self-consistently interacting particles are usually called :ref:`tracer particles <usage-workflows-tracerParticles>`.

Workflow
""""""""

* ``speciesDefinition.param``: create a stationary probe species, add ``probeE`` and ``probeB`` attributes to it for storing interpolated fields

.. code-block:: cpp

    using ParticleFlagsProbes = MakeSeq_t<
        particlePusher< particles::pusher::Probe >,
        shape< UsedParticleShape >,
        interpolation< UsedField2Particle >
    >;

    using Probes = Particles<
        PMACC_CSTRING( "probe" ),
        ParticleFlagsProbes,
        MakeSeq_t<
            position< position_pic >,
            probeB,
            probeE
        >
    >;

and add it to ``VectorAllSpecies``:

.. code-block:: cpp

   using VectorAllSpecies = MakeSeq_t<
       Probes,
       // ...
   >;

* ``density.param``: select in which cell a probe particle shall be placed, e.g. in each 4th cell per direction:

.. code-block:: cpp

   // put probe particles every 4th cell in X, Y(, Z)
   using ProbeEveryFourthCell = EveryNthCellImpl<
       mCT::UInt32<
           4,
           4,
           4
       >
   >;

* ``particle.param``: initialize the individual probe particles in-cell, e.g. always in the left-lower corner and only one per selected cell

.. code-block:: cpp

    namespace startPosition
    {
        //! Configuration of initial in-cell particle position
        struct OnePositionParameter
        {
            /** Maximum number of macro-particles per cell during density profile evaluation.
             *
             * Determines the weighting of a macro particle as well as the number of
             * macro-particles which sample the evolution of the particle distribution
             * function in phase space.
             *
             * unit: none
             */
            static constexpr uint32_t numParticlesPerCell = <PARTICLES_PER_CELL_TO_SPAWN>;

            /** each x, y, z in-cell position component in range [0.0, 1.0)
             *
             * @details in 2D the last component is ignored
             */
            static constexpr auto inCellOffset = float3_X(0., 0., 0.);
        };

        /** Definition of OnePosition start position functor that
         * places macro-particles at the initial in-cell position defined above.
         */
        using OnePosition = OnePositionImpl<OnePositionParameter>;
    } // namespace startPosition

* ``speciesInitialization.param``: initialize particles for the probe just as with regular particles

.. code-block:: cpp

   using InitPipeline = pmacc::mp_list<
       // ... ,
       CreateDensity<
           densityProfiles::ProbeEveryFourthCell,
           startPosition::OnePosition,
           Probes
       >
   >;

* ``fileOutput.param``: make sure the the probe particles are part of ``FileOutputParticles``

.. code-block:: cpp

   // either all via VectorAllSpecies or just select
   using FileOutputParticles = MakeSeq_t< Probes >;

Known Limitations
"""""""""""""""""

.. note::

   currently, only the electric field :math:`\vec E` and the magnetic field :math:`\vec B` can be recorded

.. note::

   we currently do not support time averaging

.. warning::

   If the probe particles are dumped in the file output, the instantaneous fields they recorded will be one time step behind the last field update (since our runOneStep pushed the particles first and then calls the field solver).
   Adding the attributes ``probeE`` or ``probeB`` to a species will increase the particle memory footprint only for the corresponding species by ``3 * sizeof(float_X)`` byte per attribute and particle.
