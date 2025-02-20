# EVM Circuit

<!-- toc -->

# Introduction

EVM circuit iterates over transactions included in proof to verify each execution step of transaction is valid. Basically the scale of a step is same as in EVM, so usually we handle one opcode per step, except those opcodes like `SHA3` or `CALLDATACOPY` that operate on variable size of memory would require multiple steps.

> The scale of a step somehow could be different depends on the approach, an extreme case is to implement a VM with reduced instruction set (like TinyRAM) to emulate EVM, which would have a much smaller step, but not sure how it compares to current approach.
> 
> **han**

To verify if a step is valid, we first enumerate all possible execution results of a step in the EVM including success and error cases, and then build the custom constraint to verify the step transition is correct for each execution result.

In each step, we constrain it to enable one of execution results, and specially constrain the first step to enable `BEGIN_TX`, then repeats the step to verify the full execution trace. Also each step is given access to next step to propagate the tracking information, by putting constraints like `assert next.program_counter == curr.program_counter + 1`.

# Concepts

## Execution result

It's intuitive to have each opcode as a branch in step. However, EVM has so rich opcodes that some of them are very similar like `{ADD,SUB}`, `{PUSH*}`, `{DUP*}` and `{SWAP*}` that seem to be handled by almost identical constraint with small tweak (to swap a value or automatically done due to linearity), it seems we could reduce our effort to only implement it once to handle multiple opcodes in single branch.

In addition, an EVM state transition could also contain serveral kinds of error cases, we also need to take them into consideration to be equivalent to EVM. It would be annoying for each opcode branch to handle their own error cases since it needs to halt the step and return the execution context to caller.

Fortunately, most error cases are easy to verify with some pre-built lookup table even they could happen to many opcodes, only some tough errors like out of gas due to dynamic gas usage need to be verified one by one. So we further unroll all kinds of error cases as kinds of execution result.

So we can enumerate [all possible execution results](https://github.com/appliedzkp/zkevm-specs/blob/83ad4ed571e3ada7c18a411075574110dfc5ae5a/src/zkevm_specs/evm/execution_result/execution_result.py#L4) and turn EVM circuit into a finite state machine like:

```mermaid
flowchart LR
    BEGIN[.] --> BeginTx;

    BeginTx --> |no code| EndTx;
    BeginTx --> |has code| EVMExecStates;
    EVMExecStates --> EVMExecStates;
    EVMExecStates --> EndTx;

    EndTx --> BeginTx;

    EndTx --> EndBlock;
    EndBlock --> EndBlock;
    EndBlock --> END[.];
```
```mermaid
flowchart LR
    subgraph A [EVMExecStates]
    BEGIN2[.] --> SuccessStep;
    BEGIN2[.] --> ReturnStep;
    SuccessStep --> SuccessStep;
    SuccessStep --> ReturnStep;
    ReturnStep --> |not is_root| SuccessStep;
    ReturnStep --> |not is_root| ReturnStep;
    ReturnStep --> |is_root| END2[.];
    end
```

- **BeginTx**:
    - beginning of a transaction.
- **EVMExecStates** = [ SuccessStep | ReturnStep ]
- **SuccessStep** = [ ExecStep | ExecMetaStep | ExecSubStep ]
    - set of states that suceed and continue the execution within the call.
- **ReturnStep** = [ ExplicitReturn | Error ]
    - set of states that halt the execution of a call and return to the caller
      or go to the next tx.
- **ExecStep**: 
    - 1-1 mapping with a GethExecStep for opcodes that map to a single gadget
      with a single step.  Example: `ADD`, `MUL`, `DIV`, `CREATE2`.
- **ExecMetaStep**: 
    - N-1 mapping with a GethExecStep for opcodes that share the same gadget
      (due to similarity) with a single step.  For example `{ADD, SUB}`,
      `{PUSH*}`, `{DUP*}` and `{SWAP*}`.
- **ExecSubStep**: 
    - 1-N mapping with a GethExecStep for opcodes that deal with dynamic size
      arrays for which multiple steps are generated.
        - `CALLDATACOPY` -> CopyToMemory
        - `RETURNDATACOPY` -> TODO
        - `CODECOPY` -> TODO
        - `EXTCODECOPY` -> TODO
        - `SHA3` -> TODO
        - `LOGN` -> CopyToLog
- **ExplicitReturn**: 
    - 1-1 mapping with a GethExecStep for opcodes that return from a call
      without exception.
- **Error** = [ ErrorEnoughGas | ErrorOutOfGas ]
    - set of states that are associated with exceptions caused by opcodes.
- **ErrorEnoughGas**: 
    - set of error states that are unrelated to out of gas.  Example:
      `InvalidOpcode`, `StackOverflow`, `InvalidJump`.
- **ErrorOutOfGas**: 
    - set of error states for opcodes that run out of gas.  For each opcode
      (sometimes group of opcodes) that has dynamic memory gas usage, there is
      a specific **ErrorOutOfGas** error state.
- **EndTx**
    - end of a transaction.
- **EndBlock**
    - end of a block (serves also as padding for the rest of the state step slots)


> In current implementation, we ask opcode implementer to also implement error cases, which seems to be redundant efforts. If we do this, they can focus more on opcode's success case. Also error cases are usually easier to verify, so I think it also reduces the overall implementation complexity.
> 
> **han**

## Random accessible data

In EVM, the interpreter has ability to do any random access to data like block context, account balance, stack and memory in current scope, etc... And some of them are read-write and others are read-only.

In EVM circuit, we leverage the concept [Circuit as a lookup table](#Circuit-as-a-lookup-table) to duplicate these random accessible data to other circuits in different layout and verify they are consistent and valid. After these random accessible data are verified, we can use them just like they are tables. [Here](https://github.com/appliedzkp/zkevm-specs/blob/83ad4ed571/src/zkevm_specs/evm/table.py#L108) are the tables currently used in EVM circuit.

For read-write accessible data, EVM circuit lookup State circuit with a sequentially `rw_counter` (read-write counter) to make sure the read-write access is chronological, and a flag `is_write` to have State circuit to check data is consistency between different write accesses.

For read-only accessible data, EVM circuit lookup Bytecode circuit, Tx circuit, Call circuit directly.

## State write reversion

In EVM, state write could be reverted if any call fails. There are many kinds of state write, a complete list can be found [here](https://github.com/ethereum/go-ethereum/blob/master/core/state/journal.go#L87-L141).

In EVM circuit, each call is attached with a flag`is_persistent` to know if it succeeds in the end or not, so ideally we only need to do reversion on those kinds of state write which affects future execution before reversion:

- `TxAccessListAccount`
- `TxAccessListStorageSlot`
- `AccountNonce`
- `AccountBalance`
- `AccountCodeHash`
- `AccountStorage`

Others we don't need to do reversion because they don't affect future execution before reversion, we only write them when `is_persistent` is `1`:

- `TxRefund`
- `AccountDestructed`

> Another tag is `TxLog`, which also doesn't affect future execution. It should be explained where to write such record to after we decide where to build receipt trie.
> 
> **han**

To enable state write reversion, we need some meta information of a call:

1. `is_persistent` - To know if we need reversion or not
2. `rw_counter_end_of_reversion` - To know at which point in the future we should revert
3. `state_write_counter` - To know how many state write we have done by now

Then at each state write, we first check if `is_persistent` is `0`, if so we do an extra state write at `rw_counter_end_of_reversion - state_write_counter` with the old value, which reverts the state write in a reverse order.

For more notes on state write reversion see:
- [Design Notes, State Write Reversion Note 1](../design/state-write-reversion.md)
- [Design Notes, State Write Reversion Note 2](../design/state-write-reversion2.md)

## Opcode fetching

In EVM circuit, there are 3 kinds of opcode source for execution or copy:

1. When contract interaction:
    Opcode is lookup from contract bytecode in Bytecode circuit by tuple `(code_hash, index, opcode)`
2. When contract creation in root call:
    Opcode is lookup from tx calldata in Tx circuit by tuple `(tx_id, TxTableTag.Calldata, index, opcode)`
3. When contract creation in internal call:
    Opcode is lookup from caller's memory in State circuit by tuple `(rw_counter, False, caller_id, index, opcode)`

Before we fetch opcode from any source, it checks if the index is in the given range, if not, it follows the behavior of current EVM to implicitly returning `0`.

## Internal call

EVM supports internal call triggered by opcodes. In EVM circuit, the opcodes (like `CALL` or `CREATE`) that trigger internal call will save its own `call_state` into State circuit, setup next call's context, and initialize next step's `call_state` to start a new environment. Then the opcodes (like `RETURN` or `REVERT`) and error cases that halt will restore caller's `call_state` and set it back to next step.

For a simple `CALL` example with illustration (many details are hided for simplicity):

![](./evm-circuit_internal-call.png)

# Constraints

## `main`

==TODO== Explain each execution result

# Implementation

- [spec](https://github.com/appliedzkp/zkevm-specs/blob/master/specs/evm-proof.md)
    - [python](https://github.com/appliedzkp/zkevm-specs/tree/master/src/zkevm_specs/evm)
- [circuit](https://github.com/appliedzkp/zkevm-circuits/tree/main/zkevm-circuits/src/evm_circuit)
