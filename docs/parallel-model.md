# The Vectorized Programming Model

HPMPC maximizes resource utilization and minimizes thread synchronization by using a vectorized programming model.
Each operation performed on secret shares by a program is inherently parallelized by using SIMD instructions and multiple processes.
To illustrate the vectorized programming model, consider the following example:

```cpp
using A = Additive_Share<DATATYPE, Share>;
A x = A(1);
A y = A(2);
A z = x + y;
```

The code snippet looks like a simple addition of two shares without any parallelization.
However, the addition is performed multiple times in parallel by using vectorization and multiprocessing. Suppose the `DATTYPE=512`, `BITLENGTH=32`, and `NUM_PROCESSES=32`.
In this case, each Additive_Share object holds 16 independent secret shares, and the addition is performed on all 16 shares in parallel on each of the 32 processes. 
Thus, in total, 512 secret shares are added in parallel.
Clearly, under these circumstances, assigning the same value to all of the 512 shares as shown above would be a waste of resources.
Instead, parties can assign different values to each of the 16 independent shares and 32 processes to perform 512 unique additions in parallel.

```cpp
using A = Additive_Share<DATATYPE, Share>;
const int vectorization_factor=DATTYPE/BITLENGTH;
DATATYPE x_vectorized_values, y_vectorized_values;
UINT_TYPE x_values[vectorization_factor];
UINT_TYPE y_values[vectorization_factor];
for(int i=0; i<vectorization_factor; i++)
{
    x_values[i] = i+process_offset; // dummy value assignment
    y_values[i] = i+process_offset+vectorization_factor; // dummy value assignment
}
orthogonalize_arithmetic(x_values, x_vectorized_values,1);
orthogonalize_arithmetic(y_values, y_vectorized_values,1);
A x = A::get_share_from_public_dat(vectorized_values);
A y = A::get_share_from_public_dat(vectorized_values);
A z = x + y;
```

The code snippet above assigns a unique value to each of the 512 shares by using the `orthogonalize_arithmetic` function to convert an array of unsigned integers to a vectorized value that matches the register size specified by `DATTYPE`.
It also uses the globally available `process_offset` to determine the unique process ID and assign different values depending on the process ID. The `process_offset` also works correctly with SPLITROLES across executables.


### Vectorized Secret Sharing and Revealing

A full example would typically also involve secret sharing as opposed to setting all values from public data.
```cpp
using A = Additive_Share<DATATYPE, Share>;
const int vectorization_factor=DATTYPE/BITLENGTH;
DATATYPE x_vectorized_values, y_vectorized_values;

#if PARTY == P_0 // Code block only gets executed by party 0
for(int i=0; i<vectorization_factor; i++)
{
    UINT_TYPE x_values[vectorization_factor];
    x_values[i] = i+process_offset; // dummy value assignment
    orthogonalize_arithmetic(x_values, x_vectorized_values,1);
}
#elif PARTY == P_1 // Code block only gets executed by party 1
for(int i=0; i<vectorization_factor; i++)
{
    UINT_TYPE y_values[vectorization_factor];
    y_values[i] = i+process_offset+vectorization_factor; // dummy value assignment
    orthogonalize_arithmetic(y_values, y_vectorized_values,1);
}
#endif

A x,y;
x.template prepare_receive_from<P_0>(vectorized_values);
y.template prepare_receive_from<P_1>(vectorized_values);
Share::communicate();
x.template complete_receive_from<P_0>();
y.template complete_receive_from<P_1>();
A z = x + y;
```

The shares are now vectorized throughout the entire program, thus inherently benefitting by from all subsequent operations being parallelized. For full secrecy, the inputs provided in the code blocks that are exclusive to a party can come from an external source such as a local file.
When revealing the result, the shares are also vectorized and the result is revealed in a vectorized manner. The vectorized result can be converted back to an array of unsigned integers by using the `unorthogonalize_arithmetic` function.

```cpp
A z = x.prepare_mult(y);
Share::communicate();
z.complete_mult_without_trunc();
z.prepare_reveal_to_all();
Share::communicate();
DATATYPE vectorized_result = z.complete_reveal_to_all();
UINT_TYPE result_values[vectorization_factor];
unorthogonalize_arithmetic(vectorized_result, result_values,1);
for(int i=0; i<vectorization_factor; i++)
    std::cout << result_values[i] << std::endl;
```

### Minimizing Communication Rounds

To avoid overhead from parsing and interpreting circuit representations, HPMPC entrusts developers to trigger communication rounds explictly.
Note that implictly, messages are sent continuously between the parties whenever the `SEND_BUFFER` has been filled to interleave communication and computation. 
However, to explictly send messages even when the `SEND_BUFFER` is not full, the `Share::communicate()` function is used.
The function gurantees that all messages of a communication round get sent and should be called at the end of each communication round.


The previous examples illustrated how the vectorized programming model can be used to parallelize operations on secret shares.
However, in addition to vectorizing individual operations, multiple operations should also be grouped together to minimize the number of communication rounds between parties. 
Assume that x,y, and z are now arrays of size 100. Observe that the following code snippet requires the same number of communication rounds as the previous example, but performs 100 times as many operations. 

```cpp
for(int i=0; i<100; i++)
    z[i] = x[i].prepare_mult(y[i]);
Share::communicate(); // End of first communication round
for(int i=0; i<100; i++)
    z[i].complete_mult_without_trunc();

DATATYPE vectorized_result_values[100];
UINT_TYPE result_values[100][vectorization_factor];

for(int i=0; i<100; i++)
    z[i].prepare_reveal_to_all();
Share::communicate(); // End of second communication round
for(int i=0; i<100; i++)
    vectorized_result_values[i] = z[i].complete_reveal_to_all();
unorthogonalize_arithmetic(vectorized_result_values, result_values,100);

for(int i=0; i<100; i++)
    for(int j=0; j<vectorization_factor; j++)
        std::cout << result_values[i][j] << std::endl;
```

The predefined functions of HPMPC are designed to minimize the number of communication rounds between parties by letting the user provide contiguous arrays of secret shares as input along with the length and in some cases batch size. This way, the program can perform for instance multiple independent maximum operations on separate arrays in parallel without increasing the number of communication rounds compared to a single maximum operation.

