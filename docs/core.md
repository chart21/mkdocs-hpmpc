# Core

The Core contains the backbone for networking, cryptographic operations, and hardware acceleration techniques such as Bitslicing, GPU acceleration, and Vectorization.

The Core is divided into the following sub-components:

|SUB-COMPONENT|DESCRIPTION|
|---|---|
|[Arch](#arch)|Contains the architecture-specific headers for Bitslicing and Vectorization|
|[Crypto](#crypto)|Contains implementations of cryptographic operations such as AES and SHA|
|[CUDA](#cuda)|Contains standalone CUDA implementations for matrix multiplications and convolutions|
|[Networking](#networking)|Handles socket communication and SSL/TLS connections between parties|
|Other|Contains microbenchmarks for cryptographic operations and basic operations, utilities for printing and debugging, and SSL certificates used by the nodes to establish secure connections|

!!! Tip
    We provide different configuration options to support various architectures. These options can be set in the `Makefile` or `config.h` file. It is worth experimenting with different configurations to find the best performance for a specific use case.


## Arch

Bitslicing and Vectorization become more effective with larger register sizes. However, not all CPUs support wider registers and not all functions require high parallelism. 
Arch contains headers for common X86 architectures such as SSE and AVX512.
The headers contain efficient conversions for Bitslicing and Vectorization and map architecture-specific operations to generic functions.
This way a user can write code that is architecture-agnostic, and compile the code with the appropriate flags to increase or decrease the level of parallelism.
The `DATTYPE` config option defines the register size. For instance, setting `DATTYPE` to 512 will use AVX512 registers for Bitslicing and Vectorization, while setting it to 32 will use uint32_t registers.
The following options are available.

| Register Size | Requirements            | Supported BITLENGTH             | Config Option                  |
|------|-------------------------|---------------------------------|--------------------------------------|
| 512  | AVX512         | 16, 32, 64                      | `DATTYPE=512`                |
| 256  | AVX2           | 16, 32, (64 with AVX512)         | `DATTYPE=256`                |
| 128  | SSE                     | 16, 32, (64 with AVX512)         | `DATTYPE=128`                |
| 64   | None                    | 64                              | `DATTYPE=64`                 |
| 32   | None                    | 32                              | `DATTYPE=32`                 |
| 16   | None                    | 16                              | `DATTYPE=16`                 |
| 8    | None                    | 8 (Does not support all arithmetic instructions) | `DATTYPE=8`                  |
| 1    | None                    | 8,16,32,64 (Use only for boolean circuits)  | `DATTYPE=1`                  |

The vectorization factor for a `DATTYPE` can be calculated as `DATTYPE`/ `BITLENGTH`. For instance, if `DATTYPE` is 256 and `BITLENGTH` is 32, each arithmetic instruction will be executed on 256/32=8 inputs in parallel, and each boolean operation will be executed 256 times in parallel. `DATTYPE=32` and `BITLENGTH=32` will not vectorize arithmetic operations and `DATTYPE=1` and `BITLENGTH=32`will not bitslice boolean operations.
!!! Warning
    Some BITLENGTHs may not be supported for all DATTYPEs. For instance, AVX2 does not support arithmetic optations on packed 64-bit integers. Check the Table above for supported BITLENGTHs for each DATTYPE. All nodes must use the same DATTYPE and BITLENGTH to ensure compatibility.

!!! References
    The architecture-specific headers for vectorization and Bitslicing are adapted from [USUBA](https://github.com/usubalang/usuba), [MIT LICENSE](https://raw.githubusercontent.com/usubalang/usuba/main/LICENSE).

## Crypto

Crypto contains cryptographic implementations for AES and SHA.
The AES implementation is used to generate shared random numbers between parties and the SHA implementation is used to compare the views of the parties to ensure consistent messages.
We support three different AES implementations for different architectures and two different SHA implementations.


| Implementation          | Description                                                                                              | Config Option                  |
|-----------------|----------------------------------------------------------------------------------------------------------|--------------------------------------|
| AES (X86)       | Uses the AES-NI or VAES instruction set for AES encryption and decryption. This option is usually the fastest if the CPU supports AES-NI or VAES. VAES uses wider registers to increase parallelism. | `RANDOM_ALGORITHM=2`                |
| AES (Bitslicing)| A Bitsliced implementation of AES that does not require any special instruction set.                     | `RANDOM_ALGORITHM=1`                |
| AES (OPENSSL)   | Uses the OpenSSL library for AES encryption and decryption.                                              | `USE_SSL_AES=1` and `RANDOM_ALGORITHM=2` |
| SHA (X86)       | Uses the SHA-NI instruction set for SHA hashing if available. This option is usually the fastest if the CPU supports SHA-NI. IF SHA-NI is not available this option falls back to a non-accelerated implementation of SHA. | `USE_ARM=0`                     |
| SHA (ARM)   | Uses SHA instruction set for ARM CPUs. | `USE_ARM=1`                     |
!!! Tip
    The `VERIFY_BUFFER` sets how many messages get accumulated before a hash is computed. Setting `VERIFY_BUFFER=0` will buffer all messages and compute a single hash at the end of the protocol.
    Usually, a smaller buffer provides the best results. Note that to ensure correctness, the following equation must hold $VERIFY\_BUFFER \cdot DATTYPE \ge 512$, since the SHA function requires at least 512 bits of inputs to compute a hash. Setting `VERIFY_BUFFER` to `512/DATTYPE` usually achieves the best performance.

!!! References
    The AES-NI implementation is adapted from [AES-Brute-Force](https://github.com/sebastien-riou/aes-brute-force), [Apache 2.0 LICENSE](https://raw.githubusercontent.com/sebastien-riou/aes-brute-force/master/LICENSE)

    The bitsliced AES implementation is adapted from [USUBA](https://github.com/usubalang/usuba), [MIT LICENSE](https://raw.githubusercontent.com/usubalang/usuba/main/LICENSE).

    The SHA-256 implementation is adapted from [SHA-Intrinsics](https://github.com/noloader/SHA-Intrinsics/tree/master), No License.



## CUDA

In the CUDA folder, we provide standalone implementations of matrix multiplications and convolutions. These need to be compiled separately and will then be linked automatically by setting the appropriate flags in the project's `Makefile` or `config.h`. 
To compile, the node requires a CUDA-compatible GPU and the CUDA toolkit installed. The implementations also require the CUTLASS library that provides templated CUDA kernels.
The executables can be built as follows.
```sh
# Dependencies for GPU acceleration
git clone https://github.com/NVIDIA/cutlass.git
# Compile standalone executable for GPU acceleration
cd core/cuda
make -j arch=sm_89 CUDA_PATH=/usr/local/cuda CUTLASS_PATH=/home/user/cutlass # Replace with you architecture, nvcc path and CUTLASS path
```

Note that some target architectures may not support datatypes such as uint16_t on the GPU. Thus, by default, some of these are commented out in our source files. If you want to use a BITLENGTH of 16 with GPU acceleration you can uncomment the appropriate lines in the `.cu` source files of the CUDA directory.

We provide four different options for accelerating matrix multiplications and convolutions on the GPU. 

| Approach          | Description                                                                                              | Config Option                  |
|-----------------|----------------------------------------------------------------------------------------------------------|--------------------------------------|
| CPU Matrix Multiplication | The matrix multiplication is handled on the CPU but utilizes optimizations such as cache tiling and transposing. | `USE_CUDA_GEMM=0`                |
| GPU Matrix Multiplication | Accelerates matrix multiplications on the GPU. Convolutions are split up in im2col layout changes handled by the CPU and only the matrix multiplication itself is outsourced.| `USE_CUDA_GEMM=1`                |
| Optimized GPU Matrix Multiplication | Similiar to `USE_CUDA_GEMM=1` but provides improved data transfer and scheduling on the GPU that should be faster on most architectures.| `USE_CUDA_GEMM=3`                |
| NCHW GPU Convolution | Handles the whole convolution operation on the GPU. This is usually faster than the previous approaches. Separate matrix multiplications are handled the same way as in `USE_CUDA_GEMM=3`.| `USE_CUDA_GEMM=2`                |
| CHWN GPU Convolution | Similiar to `USE_CUDA_GEMM=2` but uses a different layout for the input and output tensors. This layout may be faster on some architectures.| `USE_CUDA_GEMM=4`                |

!!! References
    CUDA GEMM and Convolution implementations are adapted from [Cutlass](https://github.com/NVIDIA/cutlass), [LICENSE](https://raw.githubusercontent.com/NVIDIA/cutlass/main/LICENSE.txt) and [Piranha](https://github.com/ucbrise/piranha/tree/main), [MIT LICENSE](https://raw.githubusercontent.com/ucbrise/piranha/main/LICENSE).

## Networking

In the networking folder, we provide implementations for socket communication and TCP/TLS connections between parties. 
Each party has a sending and receiving thread for each party it communicates with. The threads have the following responsibilities.

- Sending Threads: The sending threads wait until the main thread signals a condition variable by invoking the `send()` method.
This is either done manually or if the `SEND_BUFFER` is full.
- Receiving Threads: The receiving threads continuously buffer incoming messages and signal the main thread whenever the `RECV_BUFFER` is full.
Whenever the main thread requires a new chunk of messages it calls the `receive` method which consumes the data if it is already buffered or blocks until the receiving thread buffered the required data.

This execution model ensures that all communication between parties is utilizing parallelism and the main thread is only blocked when it needs to wait for a specific message to arrive before proceeding.

The following configuration options are available for networking.

| Option          | Description                                                                                              | Config Option                  |
|-----------------|----------------------------------------------------------------------------------------------------------|--------------------------------------|
| Encrypted Communication | Encrypts messages between the parties using TLS implemented by OpenSSL | `USE_SSL=1`                |
| Send Buffer Size | Buffers `x` elements before sending a message. Setting `x=0` waits until all elements for a communication round are available. Each element is of size `DATTYPE` bits. | `SEND_BUFFER=x`                |
| Receive Buffer Size | Buffers `x` elements before signaling the main thread. Setting `x=0` only signals the main thread when all elements for a communication round are available. Each element is of size `DATTYPE` bits. | `RECV_BUFFER=y`                |

!!! Tip
    Experimenting with different buffer sizes can improve performance significantly. The default option of `SEND_BUFFER=10000` and `RECV_BUFFER=10000` showed good performance in various experiments.


