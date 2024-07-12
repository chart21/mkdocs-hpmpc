# Protocols

The `Protocols` folder contains multiple MPC protocols that implement MPC primitives on a high level by utilizing templating and functions provided by the `Core`.

Each protocol requires an init-Protocol and an actual protocol. The init-Protocol only describes the communication pattern of each primitive and the actual protocol implements the computation. Since HPMPC supports different register sizes in a generic way, the protocols operate on templated datatypes. Protocols that require a preprocessing phase can also implement a post-protocol. The post-protocol is executed after the preprocessing phase.

The protocols only need to implement a few basic primitives such as secret sharing, revealing, addition, share conversion, dot products, and multiplications. 
HPMPC implements several functions on top of these basic primitives to implement several protocols such as comparisons or matrix multiplication, and provides unit tests to ensure that the protocols are implemented correctly. 


## Adding a new protocol

The init protocol, actual protocol, and optional post protocol need to be defined in separate classes. 
Whether to implement a joint class for each party or separate classes for each party depends on the difference between the computation of each party.
A minimal example requires implementing the following functions for the Init and the actual protocol.

- `prepare_receive_from()` and `complete_receive_from()` to secretly share a value
- `prepare_reveal_to_all()` and `complete_reveal_to_all()` to reveal a secret

With these primitives in place, secret sharing and revealing can be tested.
The other primitives can then be implemented and tested step by step.

!!! Note
    All protocols require identical function signatures to be later used as a template. The function signatures are defined by all previous protocols. HPMPC uses compile-time polymorphism using templates for improved performance and thus does not provide a base class for the protocols.

After implementing the source files for a protocol, the files need to be referenced in `Protocols.h` and a `FUNCTION_IDENTIFIER` needs to be defined. In case a party also requires the implementation of a post protocol, the post protocol needs to be referenced in `generic_share.hpp`.

!!! Tip
    Examples of several protocols can be found in the `Protocols` folder. Especially the `Replicated` and `Sharemind` protocols are good starting points for new protocols.

## Implementation

A protocol is written from the perspective of an individual share.
For instance a protocol where $P_i$ holds shares $v_i$ and $v_{i+1}$ of a value $v$ might look like this:

```cpp
template <typename Datatype>
class ProtocolXYZ_Share{
private:
    Datatype i;
    Datatype ip1;
}
```

Each primitive such as multiplication and addition can then be implemented as a function of this class. Note that local operations are also provided by templates since the `+,-,*` overloads are not provided for all possible datatypes.
```cpp
template <typename func_add>
ProtocolXYZ_Share Add(ProtocolXYZ_Share b, func_add ADD) const {
return ProtocolXYZ_Share{ADD(i,b.i), ADD(ip1,b.ip1)};
}
```

Primitives can utilize functions provided by the `Core` to invoke cryptographic operations and network communication. The following function might implement a multiplication protocol for our fictional protocol:
```cpp
template <typename func_add, func_sub, func_mult>
ProtocolXYZ_Share prepare_mult(ProtocolXYZ_Share b, func_add ADD, func_sub SUB, func_mult MULT) const {
Datatype msg = ADD(MULT(i,b.i),getRandomVal(PPREV));
send_to_live(PNEXT, i);
return ProtocolXYZ_Share{msg, 
}

template <typename func_add, func_sub>
void complete_mult(func_add ADD, func_sub SUB) {
Datatype msg = recv_from_live(PPREV);
ip1 = SUB(ip1, msg)
}
```

The following table provides an overview of the available operations.

| Operation | Description | Example |
| --- | --- | --- |
| `send_to_live(PID,val)` | Sends a message to a party in the online phase. | `send_to_live(PNEXT, val)` -> sends a message to the next party. |
| `pre_send_to_live(PID,val)` | Sends a message to a party in the preprocessing phase. | `pre_send_to_live(P_1, val)` -> sends a message to party 1 in the preprocessing phase. |
| `recv_from_live(PID)` | Receives a message from a party in the online phase. | `recv_from_live(P_2)` -> receives a message from party 2. |
| `pre_recv_from_live(PID)` | Receives a message from a party in the preprocessing phase. | `pre_recv_from_live(PPREV)` -> receives a message from the previous party in the preprocessing phase. |
| `getRandomVal(PID)` | Returns a random value. Can be invoked with different macros. For instance `getRandomVal(PSELF)` returns a random value from a seed held only by the party invoking the operation while `getRandomVal(PNEXT)` returns a shared random value with the seed held by both the party invoking the operation and the next party. | `getRandomVal(P_012)` -> random value held by P_0, P_1, P_2. |
| `store_compare_view(PID, val)` | Stores the value to compare its view with other parties at the end of the protocol. | `store_compare_view(P_1, val)` -> Compare the view of `val` with party 1 at the end of the protocol. |


### Init Protocols

Init protocols only describe the communication pattern of the actual protocol. They require the same function signatures as the actual protocols. For instance, the earlier defined Share might have an init protocol that looks like this:
```cpp
template <typename Datatype>
class ProtocolXYZ_Init{
private:
// no data required
public:
template <typename func_add>
ProtocolXYZ_Init Add(ProtocolXYZ_Init b, func_add ADD) const {
return ProtocolXYZ_Init();
}
template <typename func_add, func_sub, func_mult>
ProtocolXYZ_Init prepare_mult(ProtocolXYZ_Init b, func_add ADD, func_sub SUB, func_mult MULT) const {
send_to_(PNEXT);
return ProtocolXYZ_Init();
}
template <typename func_add, func_sub>
void complete_mult(func_add ADD, func_sub SUB) {
receive_from_(PPREV);
}
};
```

### Post protocols

Some protocols offer a preprocessing phase. If the config option `pre=1` is set, the preprocessing phase is separated from the online phase. The online phase can then be implemented in the post-protocol. The post-protocol requires the same function signatures as the preprocessing protocol.
Parties that only receive data in the preprocessing phase do not require a post-protocol. Instead, they automatically wait in the preprocessing phase until all data is received.

!!! Warning
    If the config option `pre=0` is set, the post protocol is not executed and the offline and online phases are executed in the same protocol.

!!! Tip
    Only start implementing the post protocol after the actual protocol is implemented and tested with the option `pre=0`.


