========
Examples
========

Introduction
============

This section demonstrate SIMPLE-NN with examples. 
Example files are in :code:`SIMPLE-NN/examples/`.
In this example, snapshots from 500K MD trajectory of 
amorphous SiO\ :sub:`2`\  (60 atoms) are used as training set.  

.. Note::

    Since we set the relative path for reference file in :code:`str_list`, 
    You need to move to the directory indicated in each section below to run the examples.

.. _Generate NNP:

Generate NNP
============

To generate NNP using symmetry function and neural network, 
you need three types of input file (input.yaml, str_list, params_XX) 
as described in :doc:`/tutorials/tutorial` section.
The example files except params_Si and params_O are introduced below.
Detail of params_Si and params_O can be found in :doc:`/features/features` section.
Input files introduced in this section can be found in 
:code:`SIMPLE-NN/examples/SiO2/generate_NNP`.

::

    # input.yaml
    generate_features: true
    preprocess: true
    train_model: true
    atom_types:
      - Si
      - O

    symmetry_function:
      params:
        Si: params_Si
        O: params_O
       
    neural_network:
      method: Adam
      nodes: 30-30
      batch_size: 10
      total_epoch: 50000
      learning_rate: 0.001

::

    # str_list
    ../ab_initio_output/OUTCAR_comp ::10

With this input file, SIMPLE-NN calculate feature vectors and its derivatives (:code:`generate_features`), 
generate training/validation dataset (:code:`preprocess`) and optimize the network (:code:`train_model`).
Sample VASP OUTCAR file (the file is compressed to reduce the file size) is in :code:`SIMPLE-NN/examples/SiO2/ab_initio_output`.
In MD trajectory, snapshots are sampled in the interval of 10 MD steps.
In this example, 70 symmetry functions consist of 8 radial symmetry functions per 2-body combination 
and 18 angular symmetry functions per 3-body combination.
Thus, this model uses 70-30-30-1 network for both Si and O. 
The network is optimized by Adam optimizer with the 0.001 of learning rate and batch size is 10. 

Output files can be found in :code:`SIMPLE-NN/examples/SiO2/generate_NNP/outputs`.
In the folder, generated dataset is stored in :code:`data` folder
and execution log and energy/force RMSE are stored in :code:`LOG`. 

Potential test
==============

.. _gen_test_data:

Generate test dataset
---------------------
Generating a test dataset is same as generating a training/validation dataset.
In this example, we use same VASP OUTCAR to generate test dataset.
Input files introduced in this section can be found in 
:code:`SIMPLE-NN/examples/SiO2/generate_test_data`.

::

    # input.yaml
    generate_features: true
    preprocess: true
    train_model: false
    atom_types:
      - Si
      - O

    symmetry_function:
      params:
        Si: params_Si
        O: params_O
      valid_rate: 0.

In this case, :code:`train_model` is set to :code:`false` 
because training process is not required to generate test dataset.
In addition, valid_rate also set to 0.
:code:`str_list` is same as `Generate NNP`_ section.

.. Note::

    To prevent overwriting of the existing training/validation dataset,
    create a new folder and create a test dataset.


.. _test_mode:

Error check
-----------

To check the error for test dataset, use the setting below.
And for running test mode, you need to copy the :code:`train_list` 
file generated in :ref:`gen_test_data` section
to this folder and change filename to :code:`test_list`.
Input files introduced in this section can be found in 
:code:`SIMPLE-NN/examples/SiO2/error_check`.

::

    # input.yaml
    generate_features: false
    preprocess: false
    train_model: true
    atom_types:
      - Si
      - O

    symmetry_function:
      params:
        Si: params_Si
        O: params_O
       
    neural_network:
      method: Adam
        nodes: 30-30
      batch_size: 10
      train: false
      test: true
      continue: true

.. Note::
  You need to change the filename from :code:`SAVER_epochXXXX.*` to :code:`SAVER.*` to use the option :code:`continue: true`
  and modify the checkpoints file (remove '_epochXXXX' in the text). 
  If you use the option :code:`continue: weights`, 
  change the filename from :code:`potential_saved_epochXXXX` to :code:`potential_saved`.

After running SIMPLE-NN with the setting above, 
new output file named :code:`test_result` is generated. 
The file is pickle format and you can open this file with python code of below::

    from six.moves import cPickle as pickle

    with open('test_result') as fil:
        res = pickle.load(fil) # For Python 2
        # res = pickle.load(fil, encoding='latin1') # For Python 3

In the file, DFT energies/forces, NNP energies/forces are included.

Molecular dynamics
==================
Please check in :doc:`/tutorials/tutorial` section for detailed LAMMPS script writing.


Principal component analysis
============================

SIMPLE-NN provides principal component analysis (PCA) as a preprocessing method of input descriptor vector.
Input descriptor vectors often have high correlation between components, including Behler-type symmetry functions.
In that case, decorrelating input descriptor vector using PCA before feeding it to a machine-learning model can give faster convergence.

In order to use PCA, add following lines in :code:`input.yaml` before you do preprocess.
For detailed descriptions of input parameters, see :ref:`here <models/hdnn/hdnn:PCA-related parameters>`.

.. code:: yaml

   neural_network:
      pca: true
      pca_whiten: true
      pca_min_whiten_level: 1.0e-8

A pickle file named :code:`pca` will be generated during the preprocessing. You need to copy :code:`pca` file to where you run SIMPLE-NN with trained model, just like :code:`scale_factor` file.


Parameter tuning
================

GDF
---
GDF [#f1]_ is used to reduce the force errors of the sparsely sampled atoms. 
To use GDF, you need to calculate the :math:`\rho(\mathbf{G})` 
by adding the following lines to the :code:`symmetry_function` section in :code:`input.yaml`.
SIMPLE-NN supports automatic parameter generation scheme for :math:`\sigma` and :math:`c`.
Use the setting :code:`sigma: Auto` to get a robust :math:`\sigma` and :math:`c` (values are stored in LOG file).
Input files introduced in this section can be found in 
:code:`SIMPLE-NN/examples/SiO2/parameter_tuning_GDF`.

::

    #symmetry_function:
      #continue: true # if individual pickle file is not deleted
      atomic_weights:
        type: gdf
        params:
          sigma: Auto
          # for manual setting
          #  Si: 0.02 
          #  O: 0.02


:math:`\rho(\mathbf{G})` indicates the density of each training point.
After calculating :math:`\rho(\mathbf{G})`, histograms of :math:`\rho(\mathbf{G})^{-1}` 
are also saved as in the file of :code:`GDFinv_hist_XX.pdf`.

.. Note::
  If there is a peak in high :math:`\rho(\mathbf{G})^{-1}` region in the histogram, 
  increasing the Gaussian weight(:math:`\sigma`) is recommended until the peak is removed.
  On the contrary, if multiple peaks are shown in low :math:`\rho(\mathbf{G})^{-1}` region in the histogram,
  reduce :math:`\sigma` is recommended until the peaks are combined. 

In the default setting, the group of :math:`\rho(\mathbf{G})^{-1}` is scaled to have average value of 1. 
The interval-averaged force error with respect to the :math:`\rho(\mathbf{G})^{-1}` 
can be visualized with the following script.


::

    from simple_nn.utils import graph as grp

    grp.plot_error_vs_gdfinv(['Si','O'], 'test_result')

where :code:`test_result` is generated after :ref:`test_mode` as the output file. 
The graph of interval-averaged force errors with respect to the 
:math:`\rho(\mathbf{G})^{-1}` is generated as :code:`ferror_vs_GDFinv_XX.pdf`

.. .. image:: /images/ref_forceerror

If default GDF is not sufficient to reduce the force error of sparsely sampled training points, 
One can use scale function to increase the effect of GDF. In scale function, 
:math:`b` controls the decaying rate for low :math:`\rho(\mathbf{G})^{-1}` and 
:math:`c` separates highly concentrated and sparsely sampled training points.
To use the scale function, add following lines to the :code:`symmetry_function` section in :code:`input.yaml`.

::

    #symmetry_function:
      weight_modifier:
        type: modified sigmoid
        params:
          Si:
            b: 0.02
            c: 3500.
          O:
            b: 0.02
            c: 10000.

For our experience, :math:`b=1.0` and automatically selected :math:`c` shows reasonable results. 
To check the effect of scale function, use the following script for visualizing the 
force error distribution according to :math:`\rho(\mathbf{G})^{-1}`. 
In the script below, :code:`test_result_noscale` is the test result file from the training without scale function and 
:code:`test_result_wscale` is the test result file from the training with scale function.

::

    from simple_nn.utils import graph as grp

    grp.plot_error_vs_gdfinv(['Si','O'], 'test_result_noscale', 'test_result_wscale')




.. [#f1] `W. Jeong, K. Lee, D. Yoo, D. Lee and S. Han, J. Phys. Chem. C 122 (2018) 22790`_

.. _W. Jeong, K. Lee, D. Yoo, D. Lee and S. Han, J. Phys. Chem. C 122 (2018) 22790: https://pubs.acs.org/doi/abs/10.1021/acs.jpcc.8b08063
