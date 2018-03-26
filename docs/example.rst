
Example - Spring Mass System
=============================

.. _simulation-layout:

.. figure:: images/spring_mass_diagram.png
    :align: center
    :width: 2in

    Spring-mass system


Step 1: Define target random variable and initialize model
-----------------------------------------------------------


.. code-block:: python

  #Random variable for spring stiffness
  stiffness_rv = BetaRandomVariable(alpha=3.,beta=2.,shift=1.,scale=2.5)


.. code-block:: python
    
  #Specify spring-mass system and initialize model:
  m = 1.5                           #deterministic mass
  state0 = [0., 0.]                 #initial conditions at rest
  t_grid = np.arange(0., 10., 0.1)  #time discretization
  model = SpringMass_1D(m, state0, t_grid)


Step 2: Construct SROM for the input
-------------------------------------

.. code-block:: python
    
  #Generate SROM for random stiffness
  sromsize = 10
  dim = 1
  input_srom = SROM(sromsize, dim)
  input_srom.optimize(stiffness_rv)

  #Compare SROM vs target stiffness distribution:
  pp_input = Postprocessor(input_srom, stiffness_rv)
  pp_input.compare_CDFs()


Step 3: Propagate uncertainty using SROM
-----------------------------------------

Approach a) Estimate output statistics directly
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

  #Run model to get max disp for each SROM stiffness sample
  srom_disps = np.zeros(sromsize)
  (samples, probs) = input_srom.get_params()
  for i, stiff in enumerate(samples):
      srom_disps[i] = model.get_max_disp(stiff)

  #Form new SROM for the max disp. solution using samples from the model   
  output_srom = SROM(sromsize, dim)
  output_srom.set_params(srom_disps, probs)

  #Compare solutions
  pp_output = Postprocessor(output_srom, mc_solution)
  pp_output.compare_CDFs()


Approach b) Form SROM surrogate model for output
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

  #Get perturbed input srom samples to run through model for FD
  stepsize = 1e-12
  samples_fd = FD.get_perturbed_samples(samples,perturb_vals=[stepsize])

  #Run model to get perturbed outputs for FD calc.
  perturbed_disps = np.zeros(sromsize)
  for i, stiff in enumerate(samples_fd):
      perturbed_disps[i] = model.get_max_disp(stiff)
  gradient = FD.compute_gradient(srom_disps, perturbed_disps,[stepsize])


.. code-block:: python

  #Form SROM surrogate and draw samples from it:
  surrogate_PWL = SROMSurrogate(input_srom, srom_disps,gradient)
  stiffness_samples = stiffness_rv.draw_random_sample(5000)
  output_samples = surrogate_PWL.sample(stiffness_samples)
  solution_PWL = SampleRV(output_samples)

  #Compare SROM piecewise linear solution to Monte Carlo
  pp_pwl = Postprocessor(solution_PWL, mc_solution)
  pp_pwl.compare_CDFs()


