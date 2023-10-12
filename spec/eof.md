# "Mega EOF Endgame" Specification
###### tags: `EOF`

[toc]

## Preface

**This document describes all the changes which we previously dicussed titles as EOF 1.1 and EOF 2.0. Those changes do not have an EIP yet.**

This unified specification should be used as a guide to understand the various changes the EVM Object Format is proposing. The individual EIPs ~~still remain the official specification and should confusion arise those are to be consulted~~ are not fully updated yet, and this document serves as a main source of truth at the moment.

- 📃[EIP-3540](https://eips.ethereum.org/EIPS/eip-3540): EOF - EVM Object Format v1 [_history_](https://github.com/ethereum/EIPs/commits/master/EIPS/eip-3540.md)
- 📃[EIP-3670](https://eips.ethereum.org/EIPS/eip-3670): EOF - Code Validation [_history_](https://github.com/ethereum/EIPs/commits/master/EIPS/eip-3670.md)
- 📃[EIP-4200](https://eips.ethereum.org/EIPS/eip-4200): EOF - Static relative jumps [_history_](https://github.com/ethereum/EIPs/commits/master/EIPS/eip-4200.md)
- 📃[EIP-4750](https://eips.ethereum.org/EIPS/eip-4750): EOF - Functions [_history_](https://github.com/ethereum/EIPs/commits/master/EIPS/eip-4750.md)
- 📃[EIP-5450](https://eips.ethereum.org/EIPS/eip-5450): EOF - Stack Validation [_history_](https://github.com/ethereum/EIPs/commits/master/EIPS/eip-5450.md)
- 📃[EIP-6206](https://eips.ethereum.org/EIPS/eip-6206): EOF - JUMPF instruction [_history_](https://github.com/ethereum/EIPs/commits/master/EIPS/eip-6026.md)
- 📃[EIP-7480](https://eips.ethereum.org/EIPS/eip-7480): EOF - Data section access instructions [_history_](https://github.com/ethereum/EIPs/commits/master/EIPS/eip-7480.md)
- TBA: EOF - Contract Creation
- TBA: EOF - Restrict code and gas introspection
- 📃[EIP-663](https://eips.ethereum.org/EIPS/eip-663): Unlimited SWAP and DUP instructions [_history_](https://github.com/ethereum/EIPs/commits/master/EIPS/eip-663.md)
- 📃[EIP-7069](https://eips.ethereum.org/EIPS/eip-7069): Revamped CALL instructions (*does not require EOF*) [_history_](https://github.com/ethereum/EIPs/commits/master/EIPS/eip-7069.md)

## Container

EVM bytecode is traditionally an unstructured sequence of instructions. EOF introduces the concept of a container, which brings structure to byte code. The container consists of a header and then several sections.

```
container := header, body
header := 
    magic, version, 
    kind_types, types_size, 
    kind_code, num_code_sections, code_size+,
    [kind_container, num_container_sections, container_size+,]
    kind_data, data_size,
    terminator
body := types_section, code_section+, container_section*, data_section
types_section := (inputs, outputs, max_stack_height)+
```

_note: `,` is a concatenation operator, `+` should be interpreted as "one or more" of the preceding item, and `*` should be interpreted as "zero or more" of the preceding item._

While EOF is extensible, in this document we discuss the first version, EOFv1.

#### Header

| name              | length   | value  | description |
|-------------------|----------|---------------|-------------|
| magic             | 2 bytes  | 0xEF00        | EOF prefix  |
| version           | 1 byte   | 0x01          | EOF version |
| kind_types        | 1 byte   | 0x01          | kind marker for types size section |
| types_size        | 2 bytes  | 0x0004-0xFFFF | 16-bit unsigned big-endian integer denoting the length of the type section content |
| kind_code         | 1 byte   | 0x02          | kind marker for code size section |
| num_code_sections | 2 bytes  | 0x0001-0xFFFF | 16-bit unsigned big-endian integer denoting the number of the code sections |
| code_size         | 2 bytes  | 0x0001-0xFFFF | 16-bit unsigned big-endian integer denoting the length of the code section content |
| kind_container    | 1 byte   | 0x03          | kind marker for container size section |
| num_container_sections | 2 bytes  | 0x0001-0x00FF | 16-bit unsigned big-endian integer denoting the number of the container sections |
| container_size    | 2 bytes  | 0x0001-0xFFFF | 16-bit unsigned big-endian integer denoting the length of the container section content |
| kind_data         | 1 byte   | 0x04          | kind marker for data size section |
| data_size         | 2 bytes  | 0x0000-0xFFFF | 16-bit unsigned big-endian integer denoting the length of the data section content |
| terminator        | 1 byte   | 0x00          | marks the end of the header |

#### Body

| name          | length   | value  | description |
|---------------|----------|---------------|-------------|
| types_section | variable | n/a           | stores code section metadata |
| inputs        | 1 byte | 0x00-0x7F       | number of stack elements the code section consumes |
| outputs       | 1 byte | 0x00-0x80       | number of stack elements the code section returns or 0x80 for non-returning functions |
| max_stack_height | 2 bytes | 0x0000-0x03FF | maximum number of elements ever placed onto the stack by the code section |
| code_section  | variable | n/a           | arbitrary sequence of bytes |
| container_section | variable | n/a       | arbitrary sequence of bytes |
| data_section  | variable | n/a           | arbitrary sequence of bytes |

### Container Validation

The following validity constraints are placed on the container format:

- minimum valid header size is `15` bytes
- `version` must be `0x01`
- `types_size` is divisible by `4`
- the number of code sections must be equal to `types_size / 4`
- the number of code sections must not exceed 1024
- `code_size` may not be 0
- the number of container sections must not exceed 256
- `container_size` may not be 0, but container sections are optional
- the total size of the container without container sections must be `13 + 2*num_code_sections + types_size + code_size[0] + ... + code_size[num_code_sections-1] + data_size`
- the total size of the container with at least one container section must be `16 + 2*num_code_sections + types_size + code_size[0] + ... + code_size[num_code_sections-1] + data_size + 2*num_container_sections + container_size[0] + ... + container_size[num_container_sections-1]`

## Transaction Types

Introduce new transaction type `InitcodeTransaction` which extends EIP-1559 (type 2) transaction by adding a new field `initcodes: List[ByteList[MAX_INITCODE_SIZE], MAX_INITCODE_COUNT]`.

The `initcodes` can only be accessed via the `CREATE4` instruction (see below). 

We introduce a standardised creator contract (i.e. written in EVM, but existing at a known address, such as precompiles), which eliminates the need to have create transactions with empty `to`. Instead, `InitcodeTransaction`s will have to be sent to the creator contract. Deployment of creator contract will require irregular state change at EOF activation block. See the appendix below for creator contract code.

Under transaction validation rules `initcodes` are not validated for conforming to the EOF specification. They are only validated when accessed via `CREATE4`. This avoids potential DoS attacks of the mempool.

This data is similar to calldata for two reasons:
1) It must be fully transmitted in the transaction.
2) It is accessible to the EVM, but it can't be fully loaded into EVM memory.

For these reason we suggest the same cost as for calldata (16 gas for non-zero bytes, 4 for zero bytes -- see EIP-2028).

TODO initcode number limit should be defined
> Following the example of blobs limit in EIP-4844, we define `MAX_INITCODE_SIZE` and `MAX_INITCODE_COUNT` as `2**24`. We do not want to impose a limit on the container size here, because the calldata cost is a sufficient limiting factor. Excluding everything else in a transaction, it is possible to include 3.75MB of all-zero initcodes in the transaction using the 4 gas cost.

Creation transactions (any tranactions with empty `to`) are invalid in case `data` contains EOF code (starts with `EF00` magic).

## Execution Semantics

Code executing within an EOF environment will behave differently than legacy code. We can break these differences down into i) changes to existing behavior and ii) introduction of new behavior.

### Modified Behavior

- Execution starts at the first byte of code section 0, and `pc` is set to 0.
- `pc` is scoped to the executing code section
- The instructions `CALL`, `CALLCODE`, `DELEGATECALL`, `SELFDESTRUCT`, `JUMP`, `JUMPI`, `PC`, `CREATE`, `CREATE2`, `CODESIZE`, `CODECOPY`, `EXTCODESIZE`, `EXTCODECOPY`, `EXTCODEHASH`, `GAS` are deprecated and rejected by validation in EOF contracts. They are only available in legacy contracts.
- If the target account of `EXTCODECOPY` is an EOF contract, then it will copy 0 bytes.
- If the target account of `EXTCODEHASH` is an EOF contract, then it will return `0x9dbf3648db8210552e9c4f75c6a1c3057c0ca432043bd648be15fe7be05646f5` (the hash of `EF00`, as if that would be the code).
- If the target account of `EXTCODESIZE` is an EOF contract, then it will return 2.
- The instruction `JUMPDEST` is renamed to `NOP` and remains charging 1 gas without any effect.
    - Note: jumpdest-analysis is not performed anymore.
- EOF contract may not deploy legacy code
- Legacy contract may not deploy EOF code
- ~~If a `DELEGATECALL` crosses an EOF<>legacy boundary, then it returns 0 to signal failure (i.e. legacy->EOF and EOF->legacy `DELEGATECALL`s are disallowed).~~
- `DELEGATECALL` from an EOF contract to a legacy contract is disallowed, and it returns 0 to signal failure. We allow legacy to EOF path for existing proxy contracts to be able to use EOF upgrades.
- Introduce a replacement of `CALL`, `DELEGATECALL` and `STATICCALL` in EOF, with two differences to legacy:
    - The `gas_limit` input is removed.
    - The `output_offset` and `output_size` is removed.
    - The `gas_limit` will be set to `(gas_left / 64) * 63` (aka as if the caller used `gas()` in place of `gas_limit`).

### New Behavior

- `RJUMP (0xe0)` instruction
    - deduct 2 gas
    - read int16 operand `offset`, set `pc = offset + pc + 3`
- `RJUMPI (0xe1)` instruction
    - deduct 4 gas
    - pop one value, `condition` from stack
    - set `pc += 3`
    - if `condition != 0`, read int16 operand `offset` and set `pc += offset`
- `RJUMPV (0xe2)` instruction
    - deduct 4 gas
    - read uint8 operand `max_index`
    - pop one value, `case` from stack
    - set `pc += 2`
    - if `case > max_index` (out-of-bounds case), fall through and set `pc += (max_index + 1) * 2`
    - otherwise interpret 2 byte operand at `pc + case * 2` as int16, call it `offset`, and set `pc += (max_index + 1) * 2 + offset`
- introduce new vm context variables
    - `current_code_idx` which stores the actively executing code section index
    - new `return_stack` which stores the pairs `(code_section`, `pc`)`.
        - when instantiating a vm context, push an initial value to the *return stack* of `(0,0)`
- `CALLF (0xe3)` instruction
    - deduct 5 gas
    - read uint16 operand `idx`
    - if `1024 < len(stack) + types[idx].max_stack_height - types[idx].inputs`, execution results in an exceptional halt
    - if `1024 <= len(return_stack)`, execution results in an exceptional halt
    - push new element to `return_stack` `(current_code_idx, pc+3)`
    - update `current_code_idx` to `idx` and set `pc` to 0
- `RETF (0xe4)` instruction
    - deduct 4 gas
    - pops `val` from `return_stack` and sets `current_code_idx` to `val.code_section` and `pc` to `val.pc`
- `JUMPF (0xe5)` instruction
    - deduct 5 gas
    - read uint16 operand `section`
    - if `1024 < len(stack) + types[idx].max_stack_height - types[idx].inputs`, execution results in an exceptional halt
    - set `current_code_idx` to `section`
    - set `pc = 0`
- `CREATE3 (0xec)` instruction
    - deduct `32000` gas
    - read uint8 operand `initcontainer_index`
    - pops `value`, `salt`, `data_offset`, `data_size` from the stack
    - deduct `8 * ((initcontainer_size + 31) // 32)` gas (EIP-3860 + hashing charge)
    - load initcode EOF subcontainer at `initcontainer_index` in the container from which `CREATE3` is executed
    - validate the container
    - execute the container in "initcode-mode" and deduct gas for execution
        - calculate `new_address` as `keccak256(0xff || sender || salt || keccak256(initcontainer))[12:]`
        - an unsuccesful execution of initcode results in pushing `0` onto the stack
            - can populate returndata if execution `REVERT`ed
        - a successful execution ends with initcode executing `RETURNCONTRACT{deploy_container_index}(aux_data_offset, aux_data_size)` instruction (see below). After that:
            - load deploy EOF subcontainer at `deploy_container_index` in the container from which `RETURNCONTRACT` is executed
            - validate EOF format (header + total size) of the container
            - concatenate data section with `(aux_data_offset, aux_data_offset + aux_data_size)` memory segment and update data size in the header
            - validate EOF sections (e.g. types, codes)
            - set `state[new_address].code` to the updated deploy container
            - push `new_address` onto the stack
        - `RETURN` and `STOP` are not allowed in "initcode-mode" (abort execution)
    - deduct `200 * deployed_code_size` gas
    - if initcode container or deployed container is invalid, instruction's execution ends with the result 0 pushed on stack. The caller’s nonce remains increased and all creation gas is deducted.
- `CREATE4 (0xed)` instruction
    - Works the same as `CREATE3` except:
        - does not have `initcontainer_index` immediate
        - pops one more value from the stack (first argument): `tx_initcode_index`
        - takes  the initcode EOF container from the transaction `initcodes` array at `tx_initcode_index`
            - fails (returns 0 on the stack) if such initcode does not exist in the transaction
                - this is a "light" failure: caller's nonce is not updated and gas for initcode execution is not consumed
- `RETURNCONTRACT (0xee)` instruction
    - loads `uint8` immediate `deploy_container_index`
    - pops two values from the stack: `aux_data_offset`, `aux_data_size` referring to memory section that will be appended to deployed container's data
    - cost 0 gas + possible memory expansion for aux data
    - ends initcode frame execution and returns control to CREATE3/4 caller frame where `initcontainer_index` and aux data are used to construct deployed contract (see above)
    - instruction exceptionally aborts if invoked not in "initcode-mode"
- `DATALOAD (0xe8)` instruction
    - deduct 3 gas
    - pop one value, `offset`, from the stack
    - read `[offset, offset+32]` from the data section of the active container and push the value to the stack
    - exceptional abort if reading out of data bounds
- `DATALOADN (0xe9)` instruction
    - deduct 2 gas
    - like `DATALOAD`, but takes the offset as a 16-bit immediate value and not from the stack
- `DATASIZE (0xea)` instruction
    - deduct 2 gas
    - push the size of the data section of the active container to the stack
- `DATACOPY (0xeb)` instruction
    - deduct 3 gas
    - pops `mem_offset`, `offset`, `size` from the stack
    - perform memory expansion to `mem_offset + size` and deduct memory expansion cost
    - deduct `3 * ((size + 31) // 32)` gas for copying
    - read `[offset, offset+size]` from the data section of the active container and write it to memory starting at offset `mem_offset`
    - exceptional abort if reading out of data bounds
- `DUPN (0xe6)` instruction
    - deduct 3 gas
    - read uint8 operand `imm`
    - `n = imm + 1`
    - `n`‘th (1-based) stack item is duplicated at the top of the stack
    - Stack validation: `stack_height >= n`
- `SWAPN (0xe7)` instruction
    - deduct 3 gas
    - read uint8 operand `imm`
    - `n = imm + 1`
    - `n + 1`th stack item is swapped with the top stack item.
    - Stack validation: `stack_height >= n + 1`

## Code Validation

- no unassigned instructions used
- instructions with immediate operands must not be truncated at the end of a code section
- `RJUMP` / `RJUMPI` / `RJUMPV` operands must not point to an immediate operand and may not point outside of code bounds
- `RJUMPV` `count` cannot be zero
- `CALLF` and `JUMPF` operand may not exceed `num_code_sections`
- `CALLF` operand must not point to to a section with `0x80` as outputs (non-returning)
- `JUMPF` operand must point to a code section with equal or fewer number of outputs as the section in which it resides, or to a section with `0x80` as outputs (non-returning)
- no section may have more than 127 inputs or outputs
- section type is required to have `0x80` as outputs value, which marks it as non-returning, in case this section contains neither `RETF` instructions nor `JUMPF` into returning (`outputs <= 0x7f`) sections.
    - I.e. section having only `JUMPF`s to non-returning sections is non-returning itself.
- the first code section must have a type signature `(0, 0x80, max_stack_height)` (0 inputs non-returning function)
- `CREATE3` `initcontainer_index` must be less than `num_container_sections`
- `DATALOADN`'s `immediate + 32` must be within data section bounds
     - note that embedded initcontainer may have `DATALOADN` instructions referring to data in `aux_data` part that is appended during `CREATE3`/`CREATE4`, therefore it must be validated _after_ appending it.

## Stack Validation

- For each reachable instruction in the code the operand stack height is recorded.
- Validation procedure does not require actual operand stack implementation, but only to keep track of its height.
- The computational and space complexity is O(len(code)). Each instruction is visited at most once.
- `stack_height` below refers to the number of stack values accessible by this function, i.e. it does not take into account values of caller functions’ frames (but does include this function’s inputs).
- Each instruction's recorded`stack_height` is required to be the same for all possible code paths going through the instruction.
- No instruction may access more operand stack items than available, i.e. may not end with `stack_height < 0`
- Terminating instructions: `STOP`, `INVALID`, `RETURN`, `REVERT`, `RETURNCONTRACT`, `RETF`, `JUMPF`.
- During `CALLF`, the following must hold: `stack_height >= types[code_section_index].inputs`
- During `CALLF`, the following must hold: `stack_height + types[code_section_index].max_stack_height - types[code_section_index].inputs <= 1024`
- Stack validation of `JUMPF` depends on "no-returning" status of target section
    - `JUMPF` into returning section (can be only from returning section): `stack_height == type[current_section_index].outputs + type[code_section_index].inputs - type[code_section_index].outputs`
    - `JUMPF` into non-returning section: `stack_height >= type[code_section_index].inputs`
- During `JUMPF`, the following must hold: `stack_height + types[code_section_index].max_stack_height - types[code_section_index].inputs <= 1024`
- During `RETF`, the following must hold: `stack_height == types[current_code_index].outputs`
- During terminating instructions `STOP`, `INVALID`, `RETURN`, `REVERT`, `RETURNCONTRACT` operand stack may contain extra items below ones required by the instruction
- the last instruction may be a terminating instruction or `RJUMP`
- no instruction may be unreachable
- maximum data stack of a function must not exceed 1023
- `types[current_code_index].max_stack_height` must match the maximum stack height observed during validation
- Find full algorithm spec at https://eips.ethereum.org/EIPS/eip-5450#operand-stack-validation

## Appendix: Creator Contract

```solidity
{
/// Takes [index][salt][aux_data] as input,
/// creates contract and returns the address or failure otherwise

/// aux_data.length can be 0, but the first 2 words are mandatory
let size := calldatasize()
if lt(size, 64) { revert(0, 0) }

let tx_initcode_index := calldataload(0)
let salt := calldataload(32)

let aux_size := sub(size, 64)
calldatacopy(0, 64, aux_size)

let ret := create4(tx_initcode_index, callvalue(), salt, 0, aux_size)
if iszero(ret) { revert(0, 0) }

mstore(0, ret)
return(0, 32)

// Helper to compile this with existing Solidity (with --strict-assembly mode)
function create4(a, b, c, d, e) -> f {
    f := verbatim_5i_1o(hex"f7", a, b, c, d, e)
}
    
}
```

Bytecode (<small>Solidity 0.8.18 using `solc --strict-assembly --optimize creator_contract.yul`</small>): `60403610602a57600036603f1901806040833781602035348235f780156026578152602090f35b5080fd5b600080fd`