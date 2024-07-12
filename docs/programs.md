# Programs

`Programs` define high-level functions and workloads that can be executed in an MPC-generic manner by using the provided `Datatypes`. Out of the box, HPMPC provides the following types of programs:

- `Benchmarks`: Programs that are used to measure the performance of the MPC protocols and functions.
- `Functions`: Programs that implement high-level functions such as comparisons, ReLUs, and matrix multiplications.
- `Tests`: Programs that test the correctness of implemented functions and MPC primitives.
- `Tutorials`: Programs that provide examples on how to implement functions using the provided datatypes.

Especially, the `Functions` folder is relevant for users who want to implement their own program as multiple useful subroutines are already implemented by the the framework.
!!! Tip
    Before writing a new program, the tutorials in the `programs/tutorials` folder act as a starting point to understand how to implement functions using the provided datatypes.

## Import Existing Functions as Modules

The `Functions` folder contains a variety of high-level functions that are implemented in a protocol-agnostic manner. These functions can be imported as modules in a new program by including the corresponding header files. Below is an example of different functions that can be imported as modules.

| Header | Description |
|--------|-------------|
| `GEMM.hpp` | Functions for matrix multiplication and matrix-vector multiplications |
| `Comparison.hpp` | Functions for comparisons such as less than zero, or equal to zero |
| Different truncation headers | Functions for truncating fixed point numbers using approaches such as probabilistic rounding, or deterministic rounding |
| Different adder headers | Functions to compute the carry-out or the sum of two boolean shares |
| `ReLU.hpp` | Functions for computing the ReLU function |
| `prob_div.hpp` | Functions for dividing a fixed point share by a public value probabilistically |

Suppose a user wants to implement an average of two sets of numbers
The following code snippet demonstrates how to utilize the predfined `prob_div` function to compute the two batches using only a single communication round.
```cpp
using A = Additive_Share<DATATYPE, Share>;
A sum1 = A(0) + A(1) + A(2) + A(3) + A(4) + A(5) + A(6) + A(7) + A(8) + A(9);
A sum2 = A(1) + A(3) + A(5) + A(7) + A(9) + A(11) + A(13) + A(15) + A(17) + A(19);
A averages[] = {sum1, sum2};
prepare_prob_div(averages[0], 10); // output, denominator
prepare_prob_div(averages[1], 10);
Share::communicate();
complete_prob_div(averages, 2, 10); // output, batch size, denominator
```

Other functions follow a similar pattern. In order to reduce communication rounds, the user needs to provide a contiguous block of shares that are processed all at once.
Functions such as ReLU and comparisons also handle the share conversion between the boolean and arithmetic domains internally and provide both inputs and outputs in the arithmetic domain.
The tutorials show how the `pack_additive` function can be used to convert a contiguous array of `Additive_Share` to a `sint` container, perform the operation, and convert the result back to an array of `Additive_Shares`.


## Execute Predefined Programs

Predefined Programs can be executed by setting the `FUNCTION_IDENTIFIER` config option in the `Makefile` or `config.h` file. The `protocol_executer.hpp` file contains a mapping of function identifiers to the corresponding functions that are executed by an executable.
For instance, the following line in `protocol_executer.hpp` maps the function identifiers 0-7 to the benchmarks in the `bench_basic_primitives.hpp` file.
```cpp
#if FUNCTION_IDENTIFIER < 8
#include "programs/benchmarks/bench_basic_primitives.hpp"
```
By inspecting the `bench_basic_primitives.hpp` file, one can see that each function_identifer from 0-7 corresponds to a different benchmark function.

## Implement Custom Programs

To implement a custom program, the user needs to create a new header file in the `programs` folder and include the header file in the `protocol_executer.hpp` file when the function identifier is set to a chosen value. For additional details on how to implement a custom program, the tutorial `programs/tutorials/YourFirstProgram.hpp` should get users started and provides examples on how to import modules and datatypes to implement a custom program.


