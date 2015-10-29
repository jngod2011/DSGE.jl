# DISCLAIMER

Copyright Federal Reserve Bank of New York.  You may reproduce, use, modify,
make derivative works of, and distribute and this code in whole or in part so
long as you keep this notice in the documentation associated with any
distributed works.   Neither the name of the Federal Reserve Bank of New York
(FRBNY) nor the names of any of the authors may be used to endorse or promote
works derived from this code without prior written permission.  Portions of the
code attributed to third parties are subject to applicable third party licenses
and rights.  By your use of this code you accept this license and any
applicable third party license.

OF ANY KIND, EITHER EXPRESS OR IMPLIED, INCLUDING WITHOUT LIMITATION ANY

FOR A PARTICULAR PURPOSE, EXCEPT TO THE EXTENT THAT THESE DISCLAIMERS ARE HELD
TO BE LEGALLY INVALID.  FRBNY IS NOT, UNDER ANY CIRCUMSTANCES, LIABLE TO YOU
FOR DAMAGES OF ANY KIND ARISING OUT OF OR IN CONNECTION WITH USE OF OR
INABILITY TO USE THE CODE, INCLUDING, BUT NOT LIMITED TO DIRECT, INDIRECT,
INCIDENTAL, CONSEQUENTIAL, PUNITIVE, SPECIAL OR EXEMPLARY DAMAGES, WHETHER

EQUITABLE THEORY, EVEN IF FRBNY HAS BEEN ADVISED OF THE POSSIBILITY OF SUCH
DAMAGES OR LOSS AND REGARDLESS OF WHETHER SUCH DAMAGES OR LOSS IS FORESEEABLE.

# FRBNY DSGE Model (Version 990.2)

This module, written in Julia, estimates the model discussed in the
Liberty Street Economics blog post "The FRBNY DSGE Model Forecast."
This object-oriented, Julia-language implementation reproduces the
MATLAB code included in that post. Code to forecast the model is under
development and will be released upon completion.

# Model Design

The Julia implementation of the FRBNY model is designed around a
single object - the Model Object - which centralizes all information
about the model's parameters, states, equilibrium conditions, and
settings in a single data structure. The Model Object also keeps track
of file locations for all I/O operations. 

The general structure and basic functionality of the model object is
implemented by the AbstractDSGEModel type in
`abstractdsgemodel.jl`. For a specific model specification SPEC, the
model object is constructed by the function `ModelSPEC()` in
`mSPEC.jl`, which is a subtype of `AbstractDSGEModel`.

The following objects define a model:

- __Parameters__: Have values, bounds, fixed-or-not status, priors. An
  instance of the `Param` type houses all information about a given
  parameter in a single data structure.
- __States__: Mappings of names to indices (e.g. "π_t" -> 1)
- __Equilibrium Conditions__: A function that takes parameters and model
  indices, then returns Γ0, Γ1, Ψ, and Π (which fully describe the model in cannonical form)

These are enough to define the model structure. _Everything else_ is
essentially a function of these basics, and we can get to a forecast by
this chain:

- (Parameters + Model Indices + Eqcond Function) -> (TTT + RRR)
- (TTT + RRR + Data) -> Estimation
- (Estimation + TTT + RRR + Data) -> Forecast      # not yet implemented


# Running the Code

## Running with Default Settings

So far, only the estimation step of the DSGE model has been
implemented. To run the estimation step from the Julia REPL, simply
create an instance of the model object and pass it to the `estimate`
function.

```julia
m = Model990()          # construct a model object
estimate(m)             # estimate the model
computeMoments(m)       # produce LaTeX tables of parameter moments
```


## Running with Modified Settings

To change defaults for estimation and forecasts, see `m990.jl`. There, you can modify

- **Estimation Parameters**
  - `reoptimize`: Whether to re-optimize and find the mode or use a saved mode
  - `recalculate_hessian`: Whether to re-compute the hessian or use saved.
  - `num_mh_simulations`: The number of posterior draws per block (see next item).
  - `num_mh_blocks`: The number of blocks to use in the Metropolis-Hastings MCMC sampling algorithm.
  - `num_mh_burn`: The number of blocks to discard as burn-in in Metropolis-Hastings.
  - `mh_thinning_step`: Metropolis-Hastings will save only every mh_thinning_step-th draw from the posterior distribution.
  - `num_mh_simulations_test`: `num_mh_simulations` when the argument `testing=true` is passed to `estimate`.
  - `num_mh_blocks_test`: `num_mh_blocks` when the argument `testing=true` is passed to `estimate`.
  - `num_mh_burn_test`: `num_mh_burn` when the argument `testing=true` is passed to `estimate`.

- **Forecast Parameters**
  - TBU 

You can also write a script that constructs a model object and then reassigns these fields in the model `m`.

# Directory Structure

The directory structure follows Julia module conventions. In the
top-level directory (DSGE), you will find the folling subdirectory
tree:

  - `docs/`: Helpful documentation, including this README
  - `save/`: 
     - `m990/`: Input/output files for the Model990 type. A model of
       type mSPEC (with `m.spec = SPEC`) will create its own save directory `mSPEC` at this
       level in the directory tree.
        - `input_data/`: Input mode, hessian, data. All input files are in HDF5 format.
     	- `output_data/`: Output data generated by the code (output
               mode, posterior draws, forecasts). All files in
               output_data are HDF5 files.
     	- `results/`
       	  - `plots/`
       	  - `tables/`: LaTeX tables of moments calculated by the `computeMoments` function
     	- `logs`
  - `src/`
     - `abstractdsgemodel.jl`: Defines the `AbstractDSGEModel` type.
     - `distributions_ext.jl`: Defines additional functions to return objects of type Distribution.
     - `DSGE.jl`: The main module file.
     - `estimate/`: Mode-finding and posterior sampling.
     - `models/m990/`: Contains code to define and initialize version 990 of the FRBNY DSGE model.
     - `parameters.jl`: Defines the `Param` type, a data type that stores information about
     - `solve/`: Solving the model; includes `gensys.jl` code.
  - `test/`: Module test suite.
   

# Implementation Details

This section describes important functions and implementation features in greater detail. If the user
is interested only in running the default model and reproducing the forecast
results, this section can be ignored.

This section focuses on what the code does and why, while the code itself
(including comments) provides detailed information regarding *how* these basic
procedures are implemented.

## The AbstractDSGEModel Type

TBU after merge

## The Model Object

TBU after merge

### Defining Indices 

We define several dictionaries that map variable names to indices in matrices
representing the model's equilibrium conditions and observables.

- `endogenous_states`: Endogenous states
- `exogenous_shocks`:  Exogenous shocks
- `expected_shocks`:  Expectation shocks
- `equilibrium_conditions`: Equation indices
- `endogenous_states_postgensys`: Endogenous states, after model solution and
    system augmentation
- `observables`:  Indices of named observables to use in measurement equation

- Since we don't care about the number, we only have to define the names.
- In this setup, adding states is easier, because we don't have to
  increment the index numbers of _everything_ when we add states.
- Super-automatic and less error prone; code focuses on the names just
  like we do.

## Parameters: The `AbstractParameter` Type

Subtypes of `AbstractParameter` implement our notion of a model parameter: a
time-invariant, unobserved value that has economic significance in the
model's equilibrium conditions. We estimate the model to find the
values of these parameters.

Though all parameters are time-invariant, each has different
features. Parameters whose values are scaled for use in computation
are implemented as `ScaledParameter` types, while those whose values
are not scaled are implemented as `UnscaledParameter`s. 

All parameters have the following fields:

-`value`: The transformed, scaled (for `ScaledParameter`s) value of the parameter
-`valuebounds`: The parameter's value is constrained to lie between these bounds
-`transbounds`: Bounds for the transformed parameter value
-`prior`: Prior probability distribution for the parameter
-`fixed`: Whether or not the parameter is fixed at a certain value. 
-`description`: Short description of the parameter's economic significance
-`texLabel`: Provided for printing tables of parameter values to LaTeX

`ScaledParameters` also have the following fields:

-`unscaledvalue`: Unscaled, transformed parameter value
-`scaling::Function`: The function used to scale the parameter 

Parameter values are accessed using 

## Estimation

**Main Function**: `estimate` in `src/estimate/estimate.jl`

**Purpose**: Finds modal parameter estimates and samples from posterior distribution. 

**Main Steps**: 

- *Initialization*: Read in and transform raw data from `save/m990/input_data/`. 

- *Find Mode*: The main program will call the `csminwel` optimization
  routine (located in `csminwel.jl`) to find modal parameter
  estimates. Can optionally start estimation from a starting parameter
  vector by specifying `save/m990/input_data/mode_in.h5` If the
  starting parameter vector is known to be optimized, the file should
  be called `mode_in_optimized.h5`

- *Sample from Posterior*: Posterior sampling begins from the computed
  mode, (or the provided mode if `reoptimize=false`), first computing
  the Hessian matrix to scale the proposal distribution in the
  Metropolis Hastings algorithm. Settings for the number of sampling
  blocks and the size of those blocks can be specified in
  `m990.jl` or from the REPL:

  ```julia
  m = Model990()			# Create model object
  m.num_mh_blocks = 10			# Reset sampling settings
  m.num_mh_simulations = 15000
  m.num_mh_burn = 1			
  estimate(m, verbose=true)		# Print each posterior draw to standard out
  ```

*Remark*: In addition to saving each `mh_thinning_step`-th draw of the parameter vector, the
estimation program also saves the resulting posterior value and transition
equation matrices implied by each draw of the parameter vector. This is to save
time in the forecasting step since that code can avoid recomputing those
matrices. In addition, to save space, all files in `save/input_data` and `save/output_data` are HDF5 files.

# Extending or Editing a Model

A particular model is defined by the files `mSPEC.jl` (which defines
the model object for model number SPEC), `eqcond.jl` (which defines
the equilibrium conditions) and `measurement.jl` (which defines the
mappings from states to observables). To add new parameters, equilibrium conditions, or
measurement equations, edit these files. The rest of the package
implements the machinery necessary to solve and estimate the model
parameters, and is model-agnostic.