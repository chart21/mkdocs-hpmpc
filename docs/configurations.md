# Configurations

HPMPC provides extensive configuration options to customize MPC programs, optimize their performances and support various hardware architectures. Understanding the configuration options is crucial to achieve the best performance for a specific use case. The following sections provide an overview of the most important configuration options and tips on how to set them.
A complete list of configuration options can be found in the `config.h` file.
By default, the `config.h` file contains tweaks that should work well for most use cases.

To change a configuration option, specify the option in the Makefile for temporary changes or in the `config.h` file for permanent changes that also apply to all future compilations.
By default, the Makefile sets the configuration options to the ones in `config.h`. Every configuration option can be overridden by the Makefile by specifying the option as an argument to the `make` command. 
```bash
make -j PARTY=<party_id> FUNCTION_IDENTIFIER=<function_identifier> NUM_INPUTS=<num_inputs>
```

## Basic Settings

| Configuration Option | Description | Tips |
| --- | --- | --- |
| `PROTOCOL` | Protocol to use for the MPC computation. | Protocol 5 (Trio) is recommended for 3PC, Protocol 12 (Quad) for 4PC. |
| `PRE` | Use a preprocessing phase? | Preprocessing is supported by Protocols 3, 5, 8, 11, 12. |
| `PARTY` | Party ID (starting from 0). | For local execution, `PARTY=all` can be set by the `Makefile`. |
| `BITLENGTH` | Bitlength of integers. | Lower BITLENGTHs can improve performance. Boolean circuits can use `BITLENGTH=1`. |
| `FRACTIONAL` | Fractional bits to use for fixed-point arithmetic. | Higher values can improve precision perform worse with probabilistic truncation. |
| `FUNCTION_IDENTIFIER` | Identifier of the function to run. | Refer to `protocol_executer.hpp` for a list of different predefined functions. |
| `NUM_INPUTS` | Number of inputs. | Used by benchmarking functions or neural networks to set the number of inputs processed. |

## Concurrency Settings

| Configuration Option | Description | Tips |
| --- | --- | --- |
| `DATTYPE` | Register size to use for SIMD parallelization. | Use the maximum supported by the hardware. Note that some BITLENGTHs are not supported by all DATTYPEs. |
| `PROCESS_NUM` | Number of parallel processes to use. | Experiment with different values to find the optimal performance. |


## Hardware Acceleration Settings

| Configuration Option | Description | Tips |
| --- | --- | --- | 
| `RANDOM_ALGORITHM` | Random number generation algorithm. 2 uses VAES or AES-NI, 1 uses Bitslicing. | Prefer 2 for performance. |
| `USE_SSL_AES` | Use OPENSSL's AES implementation. | Prefer 0 for performance. |
| `ARM` | ARM processor support. | Set to 1 for ARM processors, 0 otherwise. |
| `USE_CUDA_GEMM` | Use CUDA for matrix multiplication. | Prefer 2 or 4 for performance. |

## Tweaks

| Configuration Option | Description | Tips |
| --- | --- | --- |
| `SEND_BUFFER` | Number of messages to buffer before sending. | Experiment with different values to find the optimal performance. |
| `RECV_BUFFER` | Number of receiving messages to buffer. | Experiment with different values to find the optimal performance. |
| `VERIFY_BUFFER` | Number of messages to buffer before hashing. | The lower the better, but must be at least 512/DATTYPE for correctness. |

## Neural Network Settings

| Configuration Option | Description | Tips |
| --- | --- | --- |
| `MODELOWNER` | Who holds the model parameters? | Use `P_0` for party 0, `P_1` for party 1, etc. Use -1 to use dummy model parameters. |
| `DATAOWNER` | Who holds the data? | Use `P_0` for party 0, `P_1` for party 1, etc. Use -1 to use dummy data. |
| `TRUNC_THEN_MULT` | Truncate before or after multiplication. | Set to 1 for higher accuracy in most cases. |
| `TRUNC_APPROACH` | Truncation approach. Use 0 for probabilistic truncation, 1 for reduced slack truncation, 2 for exact truncation | Prefer 0 for performance and 1/2 for accuracy. |
| `TRUNC_DELAYED` | Delay CONV truncation until next ReLU. | Must be set to 1 for truncation approaches 1 and 2. |
| `COMPUTE_ARGMAX` | Compute final argmax during inference. | Set to 0 to reveal class probabilities. |
| `PUBLIC_WEIGHTS` | Use public weights. | Set to 1 to use public weights for improved performance. |
| `COMPRESS` | Use 8-bit ReLUs for all layers. | Set to 1 to benchmark ReLUs with reduced bitlength. |
| `REDUCED_BITLENGTH_k` | cut off BITLENGTH-k least significant bits. | May reduce accuracy significantly. |
| `REDUCED_BITLENGTH_m` | cut off m most significant bits. | May reduce accuracy significantly. |
| `IS_TRAINING` | Training or inference phase. | Training is not supported yet. |

For different adder approaches there are also the `ONLINE_OPTIMIZED` and `BANDWIDTH_OPTIMIZED` options, where
`BANDWIDTH_OPTIMIZED=1` uses a Ripple Carry Adder (RCA) approach, `BANDWIDTH_OPTIMIZED=0` and `ONLINE_OPTIMIZED=0` uses a Parallel Prefix Adder (PPA) approach, and `BANDWIDTH_OPTIMIZED=0` and `ONLINE_OPTIMIZED=1` uses a 4-Way parallel prefix adder approach (PPA4).
For the predefined neural networks a `FUNCTION_IDENTIFIER<100` uses RCA, `FUNCTION_IDENTIFIER<200` uses PPA, and `FUNCTION_IDENTIFIER<300` uses PPA4. For instance `FUNCTION_IDENTIFIER=74` executes VGG16 with RCA, `FUNCTION_IDENTIFIER=174` with PPA, and `FUNCTION_IDENTIFIER=274` with PPA4.



