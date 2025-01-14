NanoParticleTools tools is a python module that facilitates monte carlo simulation of Upconverting Nanoparticles (UCNP) using [RNMC](https://github.com/BlauGroup/RNMC).

# Using NanoParticleTools
NanoParticleTools provides functionality to generate inputs for running Monte Carlo Simulations on nanoparticles and analyzing outputs. Monte Carlo simulation uses NMPC within the [RNMC](https://github.com/BlauGroup/RNMC) package. While NanoParticleTools provides wrapper functions to run the C++ based simulator, [RNMC](https://github.com/BlauGroup/RNMC) must be installed to perform simulations.

## Installation
To install NanoParticleTools to a python environment, clone the repository and use one of the following commands from within the NanoParticleTools directory
```bash
python setup.py develop
```
or 
```bash
pip install .
```

### NixOS
A NixOS environment is also provided for an alternative setup method. This environment includes access to a compiled RNMC executable. To access the Nix development shell
```
nix develop
```
*Note: To use the NixOS environment, you must have root access on the system you are running on (i.e. This is usually not the case on supercomputers).*

## Running Simulations
An example of local execution can be seen below.

```python
from NanoParticleTools.flows.flows import get_npmc_flow
from NanoParticleTools.inputs.nanoparticle import SphericalConstraint

constraints = [SphericalConstraint(20)]
dopant_specifications = [(0, 0.1, 'Yb', 'Y'),
                         (0, 0.02, 'Er', 'Y')]

npmc_args = {'npmc_command': <NPMC_command>,
             'num_sims':2,
             'base_seed': 1000,
             'thread_count': 8,
             'simulation_length': 1000,
             }
spectral_kinetics_args = {'excitation_power': 1e12,
                          'excitation_wavelength':980}

flow = get_npmc_flow(constraints = constraints,
                     dopant_specifications = dopant_specifications,
                     doping_seed = 0,
                     spectral_kinetics_args = spectral_kinetics_args,
                     npmc_args = npmc_args,
                     output_dir = './scratch')
```

```python
from jobflow import run_locally
from maggma.stores import MemoryStore
from jobflow import JobStore

# Store the output data locally in a MemoryStore
docs_store = MemoryStore()
data_store = MemoryStore()
store = JobStore(docs_store, additional_stores={'trajectories': data_store})

responses = run_locally(flow, store=store, ensure_success=True)
```

In this example, the target `maggma.stores.MemoryStore` used to collect output is volatile and will be lost if the Store is reinitialized or the python kernel is restarted. Therefore, one may opt to use a MongoDB server to save calculation output to ensure data persistence. To integrate a MongoDB, use a MongoStore instead of a MemoryStore.
```
from maggma.stores.mongolike import MongoStore
docs_store = MongoStore(<mongo credentials or URI here>)
data_store = MongoStore(<mongo credentials or URI here>)
```
Refer to the maggma [Stores documentation](https://materialsproject.github.io/maggma/getting_started/stores/) for more information.

### High-throughput simulations


## Running the Builder
After running simulations, you may wish to average the outputs of trajectories obtained from the same recipe (using different dopant and simulation seeds). We have included a maggma builder in NanoParticleTools to easily group documents and perform the averaging. More information on builders can be found in the maggma [Builder documentation](https://materialsproject.github.io/maggma/reference/core_builder/)

An example of instantiating a builder is as follows:
```
from maggma.stores.mongolike import MongoStore

source_store = MongoStore(collection_name = "docs_npmc", <mongo credentials here>)
target_store = MongoStore(collection_name = "avg_npmc", <mongo credentials here>)

builder = UCNPBuilder(source_store, target_store, docs_filter={'data.simulation_time': {'$gte': 0.01}}, chunk_size=4)
```
Here, the `source_store` is a maggma Store which contains the trajectory documents produced from the SimulationReplayer analysis of NPMC runs. `target_store` is the Store in which you would like your averaged documents to be populated to. Optional arguments include a `docs_filter`, which is a pymongo query to target specific documents. `chunk_size` may also be specified and is dependent on the memory and speed of the machine executing the builder.

To execute the builder locally, use the `builder.run()` function. The builder may also be run in parallel or distributed mode, see the maggma ["Running a Builder Pipeline" documentation](https://materialsproject.github.io/maggma/getting_started/running_builders/)


# Contributing 
If you wish to make changes to NanoParticle tools, it may be wise to install the package in development mode. After cloning the package, use the following command.
```bash
python -m pip install -e .
```
Modifications should now be reflected when you run any functions in NanoParticleTools.

Further guidance on contributing via Pull Requests will be added in the near future.
