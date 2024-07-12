# Datatypes

Datatypes provide a protocol-agnostic interface to implement high-level MPC functions.
For instance, a matrix multiplication can be implemented by using a dot product primitive of a protocol in a black box manner. For this purpose, the `Additive_Share` class provides generic interfaces that all protocols need to implement. Similarly, the `XOR_Share` class provides an interface for operations in the boolean domain.

The datatypes `sint` and `Bitset` are containers that are used by functions such as comparisons to efficiently switch between Bitslicing and Vectorization during share conversion.
In most cases, these datatypes do not need to be used directly, as functions, such as comparisons and ReLUs, are already implemented and can be imported as modules from the `programs/functions` folder.

Finally, the `FloatFixedConverter` class converts floating point numbers to fixed point numbers and vice versa. One conversion is typically required before secretly sharing a floating point number and the other conversion is required after revealing the result of a computation to convert the fixed point number back to a floating point number.

The following tables provide an overview of the datatypes and their functions.

| Datatype | Description | 
|----------|-------------|
| `Additive_Share`| A secret share in the arithmetic domain | 
| `XOR_Share`| A secret share in the boolean domain |
| `sint`| A container of size `BITLENGTH`storing `Additive_Shares` |
| `Bitset`| A container of size `BITLENGTH` storing `XOR_Shares` |
| `FloatFixedConverter`| A static struct to convert floating point numbers to fixed point numbers and vice versa |

## Implementing functions with Datatypes

The tutorials in the `programs/tutorials` folder provide examples of how to implement functions using the provided datatypes. Below is a quick example of how to operate on `Additive_Share` datatypes. As one can see, shares behave similarly to normal integers and can be used with STL containers such as `std::vector`.

```cpp
template <typname Share> 
void add_and_mult()
{
    using A = Additive_Share<DATATYPE, Share>; 
    A a[] = {A(2), A(3)}; // Initialize shares from public values
    A b[] = {A(4), A(5)};
    auto c = a[0] + b[0]; // Add two shares
    A d = c.prepare_mult(a[1] + b[1]); // Prepare multiplication
    Share::communicate();
    d.complete_mult(); // Complete multiplication
}
```

## Conversions between Bitslicing and Vectorization

As HPMPC uses Bitslicing for boolean operations and Vectorization for arithmetic operations, converting a single share from one domain to the other is not possible: A bitsliced variable with a register length of size `DATTYPE` stores `DATTYPE` bits from `DATTYPE` independent values, while a vectorized variable stores `DATTYPE` bits from `DATTYPE/BITLENGTH` independent values.
Thus a conversion between the two domains requires aggregating `BITLENGTH` shares in a container. 
This way, a container in both domains holds `DATTYPE*BITLENGTH` bits from `DATTYPE` independent values.

For this purpose, the `sint` and `Bitset` classes are provided.
Functions such as comparisons and ReLUs operate on these containers and are invoked using the `pack_additive` function that converts a contiguous array of `Additive_Share` to a `sint` container, performs the operation, and converts the result back to an array of `Additive_Shares`. In practice, users do not need to interact with the `sint` and `Bitset` containers directly but can use the pre-implemented functions from the `programs/functions` folder.


## Fixed Point Arithmetic

To efficiently handle decimal numbers, HPMPC uses fixed-point arithmetic. The `FloatFixedConverter` class provides functions to convert floating point numbers to fixed point numbers and vice versa. 
This conversion utilizes the `FRACTIONAL` config option that defines the number of bits used for the fractional part of a fixed point number. 
By increasing the `FRACTIONAL` config option, the precision of the fixed point numbers can be increased at the cost of a lower integer range.

After converting a floating point number to a fixed point number, the fixed point number can be secretly shared and after revealing the result, the fixed point number can be converted back to a floating point number.
The following example demonstrates how to convert a floating point number to a fixed point number and vice versa. 

```cpp
// Plaintext fixed point value with FRACTIONAL bits of precision:
UINT_TYPE fixed_point_value = FloatFixedConverter<float, INT_TYPE, UINT_TYPE, FRACTIONAL>::float_to_ufixed(3.5f); 

// Convert back:
float float_value = FloatFixedConverter<float, INT_TYPE, UINT_TYPE, FRACTIONAL>::ufixed_to_float(fixed_point_value);
```

!!!Tip
    Increasing the `BITLENGTH` config option can help to increase the range of fixed point numbers that can be represented. Also, different truncation approaches can be utilized. For instance, `TRUNC_APPROACH=0` requires a slack of a few bits to avoid truncation errors while `TRUNC_APPROACH=1` and `TRUNC_APPROACH=2` do not require a slack but introduce additional communication overhead.




