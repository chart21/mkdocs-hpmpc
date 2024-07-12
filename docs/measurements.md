# Measurements

Out of the box, HPMPC provides a set of benchmarks for different MPC primitives, high-level functions, and use cases. 
Most benchmarks are contained in `programs/benchmarks`. 
In order to run a benchmark, first obtain the `FUNCTION_IDENTIFIER` of the benchmark you want to run which is defined in the source file of the respective benchmark.
Typically, a benchmark relies on the `NUM_INPUTS` config option.
To compile a benchmark for a specific number of inputs and protocols, run the following command:
```bash
make -j PARTY=<party_id> FUNCTION_IDENTIFIER=<function_identifier> NUM_INPUTS=<num_inputs>
```
In the following lines, we explain how to run and interpret the results of a benchmark and to automate benchmarking of multiple configurations.

## Obtain Benchmark Results

As with other MPC programs, benchmarks usually benefit from vectorization and parallelization.
To obtain the best performance, it is recommended to run benchmarks with the maximum `DATTYPE` and `NUM_PROCESSES` that the hardware supports. In many cases, also `SPLITROLES` can be used to improve performance.
Since the introduced parallelism increases the number of performed operations it is important to consider the number of inputs and the parallelization factor when interpreting the results.
The parallelization factor can usually be calculated as follows:
> *parallelization_factor* $=$ `DATTYPE/BITLENGTH` $\cdot$ `NUM_PROCESSES` $\cdot$ `SPLITROLES_FACTOR` # Boolean (only) circuits do not require division by BITLENGTH

> *Total number of inputs processed* $=$ `NUM_INPUTS` $\cdot$ *parallelization_factor*

Note that Boolean operations that would usually be executed on single bits have a higher parallelization factor than arithemtic operations that would usually already be executed on registers with multiple bits (e.g. 32 for uint32). 
Thus, for a circuit such as AES, that only utilizes boolean operations, the parallelization factor is significantly higher than for a circuit that also uses arithmetic operations.
For mixed circuits, the parallelization factor of arithmetic operations applies as the program typically starts with and ends with arithmetic shares.

The SplitRoles factor is 1 for `SPLITROLES=0`, 6 for `SPLITROLES=1`, and 24 for `SPLITROLES=2` and `SPLITROLES=3`.
The reason for this is that SPLITROLES executes the same executable for all party permutations in parallel. For 3PC protocols, there are 6 permutations and for 4PC protocols, there are 24 permutations.


## Automate Benchmarking

To automate benchmarking of multiple configurations, we provide configuration files in `measurements/configs/`.
A configuration file contains a list of different arguments that are passed to the Makefile. 
Each combination of arguments can be then compiled and executed by the provided scripts.
The following `.conf` file tests 12 different protocols and 3 different input sizes for the benchmark with `FUNCTION_IDENTIFIER=1`. This results in 36 different configurations.
```py
FUNCTION_IDENTIFIER=1
PROTOCOL=1,2,3,4,5,6,7,8,9,10,11,12
PROCESS_NUM=32
DATTYPE=512
BITLENGTH=1
PRE=0
NUM_INPUTS=10000,100000,1000000
```

To run the benchmark for all configurations, execute the following command:
```bash
./measurements/run_config.py measurements/configs/<config_file>.conf
```
By default, the script will run the benchmark for all parties locally. The following options can be used to specify the party and IP addresses:

| Option | Description |
| --- | --- |
| -p | Party identifier |
| -a | IP address for party 0 |
| -b | IP address for party 1 |
| -c | IP address for party 2 |
| -d | IP address for party 3 |
| -s | `SPLITROLES` Identifier (0, 1, 2, or 3) |
| -g | Numbers of GPUs to use (0 for CPU only) |
| --override | Override config options (key=value) |
| -i | Number of iterations per run |

The results are stored in the `measurements/logs/` directory with the name of the test and a timestamp. 
Setting the number of iterations per run is useful to later compute means and standard deviations of the results. 
Note that the config file may set a DATTYPE that is not supported by the target hardware. In this case, the override option allows specifying a different value than the one in the config file.
The following command is an example for running a benchmark in a distributed setting on a specific hardware configuration with one GPU, AVX-2 (`DATTYPE=256`) support, 10 iterations per run, `SPLITROLES=1` (6 permutations), and `PROCESS_NUM=2,4`:

```bash
./measurements/run_config.py -p <party_id> -a <ip_address_a> -b <ip_address_b> -c <ip_address_c> -d <ip_address_d> -s 1 -g 1 -i 10 measurements/configs/<config_file>.conf --override DATTYPE=256 PROCESS_NUM=2,4
```

Finally, the results can be parsed using the `parse_logs.py` script. The script outputs a `.csv` table with the results of the benchmark and automatically computes useful stats such as the runtime in seconds, the size of messages sent and received in MB, and the throughput in Gbps. If applicable, the script also computes the number of operations per second, accuracy, and the number of tests that passed.
To compute the number of operations per second, the `.conf` file needs to provide `NUM_INPUTS`, `DATTYPE`, and `BITLENGTH` to compute the formula from above automatically.
The script can either be executed for a single log file or on a directory containing multiple log files.

```bash
./measurements/parse_logs.py measurements/logs/<log_file>.log
```

The neural network specific script `parse_layers.py` script collects statistics for each neural network layer and outputs a csv table with the results for each model. The script can be executed for a single log file or on a directory containing multiple log files.
The `run_pretrained.py` executes all pretrained neural network models from `Pygeon` with different configurations regarding truncation, bitlength, and the number of fractional bits.


