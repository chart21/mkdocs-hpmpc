# HPMPC: High Performance Secure Multi-Party Computation

HPMPC is a C++ framework for Secure Multi-Party Computation (MPC) that provides various tweaks and design choices to scale MPC programs to large-scale workloads.
The framework contains the following software components.

| Component | Description | Dependencies |
| --- | --- | --- |
| [`Core`](core.md) | Implements networking, cryptographic primitives, and hardware acceleration techniques such as Bitslicing, Vectorization, and GPU acceleration. | - |
| [`Protocols`](protocols.md) | Implements various MPC protocols with a fixed interface for each primitive. | Utilizes high-level interfaces from `Core` for networking and cryptographic primitives. |
| [`Datatypes`](datatypes.md) | Provides datatypes for arithmetic circuits, boolean circuits, and fixed-point arithmetic. | Maps each operation on secret shares to the primitives implemented by a `Protocol`. |
| [`Programs`](programs.md) | Implements high-level functions, routines, use cases, tests, and benchmarks. | Utilizes `Datatypes` to offer a generic programming interface for MPC. |
| [`Measurements`](measurements.md) | Provides config files and scripts to automate benchmarking and testing of MPC workloads. | Uses implemented `Programs` to run benchmarks and tests. |
| [`Neural Networks`](nn.md) | Implements neural network inference with MPC. | Combines functions implemented in `Programs` with `PIGEON`, a templated neural network inference engine to perform private inference |

The framework is designed to be modular and extensible, allowing users to implement new MPC protocols and functions without in-depth knowledge of other components.

!!! Tip
    Click on the components to navigate to their respective detailed documentation sites. 
    HPMPC also provides an extensive set of configuration options to customize MPC programs, optimize their performances, and support various hardware architectures. For more details, refer to [Configurations](configurations.md).


## [Protocols](protocols.md)

Out of the box, the framework provides multiple MPC protocols that support different primitives. The basic primitives cover secret sharing revealing, addition, and multiplication, which enables computing functions such as AES that do not require share conversion. Protocols supporting all primitives can also evaluate mixed circuits and fixed-point arithmetic. 

Protocols can be selected with the `PROTOCOL` configuration option when compiling an executable. For instance, setting `PROTOCOL=5` will compile the executable with Protocol 5 (Trio). 
The preprocessing phases allow shifting some computation and communication to an input-independent phase before the online phase. This phase is optional and can be activated using the `PRE=1` config option when compiling an executable.
Maliciously secure protocols print a terminal message when a hash-based consistency check fails to indicate aborting the computation.


| Protocol | Adversary Model | Preprocessing | Supported Primitives |
| --- | --- | --- | --- |
| `1` Sharemind (3PC) | Semi-Honest | ✘ | Basic
| `2` Replicated (3PC) | Semi-Honest |✘  | Basic
| `3` ASTRA (3PC) | Semi-Honest | ✔ | Basic
| `4` ABY2 Dummy (2PC) | Semi-Honest | ✔ | Basic
| `5` Trio (3PC) | Semi-Honest | ✔ | All
| `6` Trusted Third Party (3PC) | Semi-Honest | ✘ | All
| `7` Trusted Third Party (4PC) | Semi-Honest | ✘ | All
| `8` Tetrad (4PC) | Malicious | ✔ | Basic
| `9` Fantastic Four (4PC) | Malicious | ✘ | Basic
| `10` Quad (4PC) | Malicious | ✘ | All
| `11` Quad: Het (4PC) | Malicious | ✔ | All
| `12` Quad (4PC) | Malicious | ✔ | All

!!! Tip
    For most applications `PROTOCOL=5` and `PROTOCOL=12` are recommended.

## [Programs](programs.md)

Each predefined program is mapped to a `FUNCTION_IDENTIFIER` that can be set when compiling an executable. 
The `protocol_executer.hpp` file contains the mapping of `FUNCTION_IDENTIFIER` to their corresponding source files.
For instance, `FUNCTION_IDENTIFIER` 0-7 are reserved for benchmarking different basic primitives. 
Programs can import `Datatypes` and existing `Functions` such as comparisons to program MPC workloads.

!!! Note
    There are multiple tutorials in `programs/tutorial` that demonstrate how to implement MPC programs using the framework's interface.

## [Configurations](configurations.md)

`Programs` can be compiled with various configurations. For instance, a program might use a certain `BITLENGTH`, `PROTOCOL`, and number of `FRACTIONAL` bits for fixed-point arithmetic.
These config options are fetched from the `config.h` file which contains extensive configurations for details and tweaks to run MPC programs.
Changes can be made permanently in the `config.h` file or temporarily by specifying the option as an argument to the `make` command.
```bash
make -j PARTY=<party_id> FUNCTION_IDENTIFIER=<function_identifier> BITLENGTH=<bitlength>
```

## [The Vectorized Programming Model](parallel-model.md)

The framework uses the register length specified by the `DATTYPE` config option for all secret shares.
This means that each arithmetic secret share is inherently a vector of `DATTYPE/BITLENGTH` elements and each boolean secret share is a vector of `DATTYPE` elements.
When setting `NUM_PROCESSES` to a value greater than 1, the framework will additionally execute the same program on multiple processes in parallel. 
Applications such as neural network inference, or AES computations especially benefit from this parallelization approach as they require operations on independent batches of data.
Additionally, the `SPLITROLES` config option can be used to execute the same program for all party permutations in parallel to achieve load balancing across parties.

To turn off any kind of inherent parallelization, set `DATTYPE=BITLENGTH` (or `DATTYPE=1` for boolean circuits), `NUM_PROCESSES=1`, and `SPLITROLES=0`.
This will result in a single-threaded execution of the program without any vectorization.

!!! Warning
    Not all `DATTYPE` values are supported by all hardware architectures. Additionally, some `BITLENGTH`s are not supported by certain DATTYPEs. Learn more [here](core.md#arch). 
    Understanding the vectorized programming model is crucial to writing correct programs. Refer to the tutorials and [detailed documentation](parallel-model.md) for more information.

## Compiling and Running Programs

You can use the provided Dockerfile or set up the project manually. The only dependency is OpenSSL. Neural Networks and other functions with matrix operations also require the Eigen library. Install on your target system, for instance via ```apt install libssl-dev libeigen3-dev```. 

To use GPU acceleration for matrix multiplication and convolutions, you need an NVIDIA GPU and the NVCC compiler. You also need a copy of the [CUTLASS](https://github.com/NVIDIA/cutlass) library. You can then set up the project as follows.

### Setup
```bash
# General dependencies
sudo apt install libssl-dev libeigen3-dev

# Dependencies for GPU acceleration (optional)
git clone https://github.com/NVIDIA/cutlass.git


```

### Basic Setting

Programs can be compiled and executed for a specific party in distributed settings or for all parties locally.

```bash
# Compile and run for all parties
make -j PARTY=all FUNCTION_IDENTIFIER=<function_identifier> PROTOCOL=<protocol_id>
scripts/run.sh -p all -n <num_parties>
```

```bash
# Compile and run for P0, repeat for other parties analogously
make -j PARTY=0 FUNCTION_IDENTIFIER=<function_identifier> PROTOCOL=<protocol_id>
scripts/run.sh -p 0 -a <ip_address_p0> -b <ip_address_p1> -c <ip_address_p2> -d <ip_address_p3>
```

### SplitRoles

For additional load balancing, `SPLITROLES` can be set to 1 to execute a 3PC program for all 6 party permutations in parallel, 2 to execute a 3PC program on four nodes for all 24 party permutations, and 3 to execute a 4PC program on four nodes for all 24 party permutations.

```bash
# Compile and run for all parties
make -j PARTY=all FUNCTION_IDENTIFIER=<function_identifier> PROTOCOL=<protocol_id> SPLITROLES=<splitroles_id>
scripts/run.sh -p all -n <num_parties> -s <splitroles_id>
```

```bash
# Compile and run for P0, repeat for other parties analogously
make -j PARTY=0 FUNCTION_IDENTIFIER=<function_identifier> PROTOCOL=<protocol_id> SPLITROLES=<splitroles_id>
scripts/run.sh -p 0 -a <ip_address_p0> -b <ip_address_p1> -c <ip_address_p2> -d <ip_address_p3> -s <splitroles_id>
```

### GPU Acceleration

The framework supports GPU acceleration for matrix multiplication by using CUDA.
This requires to first compile the standalone CUDA executables.

```bash
# Cutlass is required for GPU acceleration
git clone https://github.com/NVIDIA/cutlass.git
cd core/cuda
# Replace with your architecture, nvcc path, and CUTLASS path:
make -j arch=sm_89 CUDA_PATH=/usr/local/cuda CUTLASS_PATH=/home/user/cutlass 
```

The framework implements different approaches to accelerate convolutions and matrix multiplications `USE_CUDA_GEMM=2` or `USE_CUDA_GEMM=4` are good starting points for most applications.
```bash
# Compile and run for all parties
make -j PARTY=all FUNCTION_IDENTIFIER=<function_identifier> PROTOCOL=<protocol_id> USE_CUDA_GEMM=2```
```

```bash
# Compile and run for P0, repeat for other parties analogously
make -j PARTY=0 FUNCTION_IDENTIFIER=<function_identifier> PROTOCOL=<protocol_id> USE_CUDA_GEMM=2```
```

#### Multi-GPU Acceleration
If each node has access to multiple GPUs these can be utilized with the `-g` option at runtime.
```bash
# Run SplitRoles executable for P0 and utilize 6 GPUs for convolutions and matrix multiplications
scripts/run.sh -p 0 -a <ip_address_p0> -b <ip_address_p1> -c <ip_address_p2> -d <ip_address_p3> -s 1 -g 6
```

## [Neural Networks](nn.md)

The `PIGEON` (Private Inference of Neural Networks) submodule is a templated neural network inference engine that can be used to run neural networks using MPC. 
The second submodule, `PyGEON`, enables training and exporting models and datasets in PyTorch and importing them to `PIGEON` for inference.
The `NN` program orchestrates the execution of `PIGEON` and the MPC backend. 

### Setup

```bash
# Load Submodules for Neural Networks
git submodule update --init --recursive

# Dependencies for training and exporting models and datasets
pip install torch torchvision 

# Dependencies for downloading pretrained models and datasets
pip install gdown 
```

### Run neural network inference

The `programs/NN.hpp` file contains a mapping of various neural network architectures to a `FUNCTION_IDENTIFIER`. 
The following command compiles a neural network inference of `VGG16` on the CIFAR-10 dataset (`FUNCTION_IDENTIFIER=74`). 
The batch size in the following example is `NUM_INPUTS` $\cdot$ `DATTYPE`/`BITLENGTH` $\cdot$ `PROCESS_NUM` $\cdot$ `SPLITROLES_FACTOR` $=$ 1 $\cdot$ 256/32 $\cdot$ 4 $\cdot$ 6 $=$ 192.

```bash
`SPLITROLES_FACTOR 
make -j PARTY=<party_id> FUNCTION_IDENTIFIER=74 BITLENGTH=32 FRACTIONAL=5 DATTYPE=256 SPLITROLES=1 PROCESS_NUM=4 PROTOCOL=5 NUM_INPUTS=1 USE_CUDA_GEMM=2
```

The command above uses dummy weights and datasets. To use real weights and datasets, the `MODELOWNER` and `DATAOWNER` config options must be set to the party that holds the model and the party that holds the dataset respectively. Additionally, pretrained models need to be available for importing.

```bash
cd nn/Pygeon
# Option 1: Train a VGG16 model and export it to PyGEON 
 python main.py --action train --export_model --export_dataset --transform standard --model VGG16 --num_classes 10 --dataset_name CIFAR-10 --modelpath ./models/alexnet_cifar --num_epochs 30 --lr 0.01 --criterion CrossEntropyLoss --optimizer Adam
# Option 2: Download a pretrained VGG16 model and CIFAR10 dataset
python download_pretrained.py single_model datasets

# Set environment variables for the party holding the model parameters
export MODEL_DIR=nn/Pygeon/models/pretrained
export MODEL_FILE=VGG16_CIFAR-10_standard.bin #adjust path if needed

# Set environment variables for the party holding the dataset
export DATA_DIR=nn/Pygeon/data/datasets
export SAMPLES_FILE=CIFAR-10_standard_test_images.bin
export LABELS_FILE=CIFAR-10_standard_test_labels.bin #adjust path if needed

#Compile the program, specify the party that holds the model and the dataset
make -j PARTY=<party_id> MODELOWNER=P_0 DATAOWNER=P_1 FUNCTION_IDENTIFIER=74 

# Run the program
scripts/run.sh -p <party_id> -a <ip_address_a> -b <ip_address_b> -c <ip_address_c> -d <ip_address_d> 
```

!!! Tip
    There is an extensive set of configuration options to optimize the performance of Neural Network inference such as different truncation and adder approaches. For more details, refer to [Neural Network Settings](configurations.md#neural-network-settings).
    



## [Measurements](measurements.md)

To automate benchmarking of multiple configurations, the framework provides configuration files in `measurements/configs/`. By using the associated `measurements/run_config.py` and `measurements/parse_logs.py` scripts, programs can be compiled and executed with various configurations while results are stored in the `measurements/logs/` directory. 
The following command is an example of running a benchmark and parsing the results.
```bash
./measurements/run_config.py -p <party_id> -a <ip_address_a> -b <ip_address_b> -c <ip_address_c> -d <ip_address_d> -i 10 measurements/configs/<config_file>.conf 
./measurements/parse_logs.py measurements/logs/<log_file>.log
```

!!! Note
    The --override option allows specifying a different value than the one in the config file. This might be necessary to adjust a `.conf` file to a specific target hardware. Learn more [here](measurements.md#automating-benchmarking).


