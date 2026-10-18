# Original Prompt 

``` 
Implement a single Python function:

    def run(bytecode: bytes, max_steps: int, max_gas: int) -> int:
        ...

The bytecode is untrusted. Your implementation must be safe, efficient, concise, deterministic, and reentrant. It must not use eval, exec, compile, pickle, subprocess, os, sys, ctypes, threading, multiprocessing, importlib, socket, pathlib, shutil, tempfile, ast, types, dis, inspect, gc, marshal, builtins, __import__, open, globals, locals, or vars. No recursion. No global mutable state (module-level constants that are never mutated are allowed). No file or network access. Every invalid input or runtime failure described below must raise ValueError; no other exception type (IndexError, KeyError, ZeroDivisionError, OverflowError, MemoryError, RecursionError, TypeError, etc.) may escape.

Arguments: bytecode must be bytes or bytearray (else ValueError) with length <= 1<<20 (else ValueError); it must not be mutated or retained. max_steps and max_gas must be ints that are not bool and are >= 0 (else ValueError); they may be arbitrarily large.

Bytecode is a sequence of instructions. Each starts with a 1-byte opcode; operands follow immediately. The entire bytecode must be validated before execution. If validation fails, raise ValueError and execute nothing. Empty bytecode validates, but fails at runtime (see "Cycle").

Opcodes (gas in brackets):
0x00 HALT [1]            0x14 NOT [3]
0x01 PUSH <sleb128> [2]  0x15 SHL [3]
0x02 ADD [3]             0x16 SHR [3]
0x03 SUB [3]             0x17 SAR [3]
0x04 MUL [3]             0x18 EQ [3]
0x05 DIV [3]             0x19 LT [3]
0x06 MOD [3]             0x1A GT [3]
0x07 DUP [1]             0x1B PICK <u8> [2]
0x08 SWAP [1]            0x1C ROLL <u8> [2 + n]
0x09 JMP <u32le> [2]     0x1D JMPI [3]
0x0A JZ <u32le> [2]      0x1E CALLI [3]
0x0B JNZ <u32le> [2]     0x1F MULH [4]
0x0C LOAD [2]            0x20 ROTL [3]
0x0D STORE [2]           0x21 ROTR [3]
0x0E DROP [1]            0x22 CLZ [2]
0x0F CALL <u32le> [2]    0x23 POPCNT [2]
0x10 RET [1]             0x24 MCOPY [3 + len, see below]
0x11 AND [3]             0x25 TRY <u32le> [2]
0x12 OR [3]              0x26 ENDTRY [1]
0x13 XOR [3]             0x27 THROW [2]

Encoding:
- sleb128: signed LEB128, 7-bit groups little-endian, high bit = continuation. Must be canonical (shortest possible encoding). Reject if length > 10 bytes, value outside -2^63..2^63-1, or non-canonical.
- u32le: exactly 4 bytes little-endian. u8: exactly 1 byte.
- JMP/JZ/JNZ/CALL/TRY operands are absolute byte offsets and must be the start offset of an instruction (so len(bytecode) is NOT a valid target).

Validation (O(n)):
- Linearly parse the entire bytecode, recording instruction starts. Reject unknown opcodes, truncated operands, non-canonical sleb128, static targets that are not instruction starts, and any byte not consumed by a valid instruction. Unreachable code must be valid too.

Execution state: pc=0; stack (signed 64-bit ints, max 1024); mem (256 signed 64-bit ints, all 0); call_stack (return addresses, max 1024); handlers (list of (target, stack_height, call_depth), max 256); steps=0; gas=0.

Cycle: (1) if pc >= len(bytecode): ValueError (no implicit halt). (2) if steps >= max_steps: ValueError. (3) steps += 1; gas += cost; if gas > max_gas: ValueError. Dynamic costs use operand values before they are popped: ROLL costs 2+n; MCOPY costs 3+len if the stack has >= 3 items and 0 <= len <= 256, else 3. (4) execute. The instruction that exceeds a limit is still counted, and limit errors are not catchable.

Semantics (wrap(x) = ((x + 2**63) % 2**64) - 2**63; all stack/memory values stay in signed 64-bit range; "pop b, pop a" means b is the top):
- HALT: return top of stack, or 0 if empty.
- PUSH, DUP: push (overflow past 1024 -> ValueError). DUP on empty -> ValueError.
- ADD, SUB, MUL, AND, OR, XOR: push wrap(a op b), bitwise ops on two's complement.
- DIV, MOD: truncated division (quotient toward zero, remainder takes dividend's sign), then wrap. b == 0 -> FAULT 1.
- SWAP, DROP, NOT, EQ (1/0), LT/GT (signed, 1/0): as expected.
- SHL, SHR (logical on unsigned 64-bit), SAR (arithmetic): b outside 0..63 -> FAULT 2. Results wrapped.
- ROTL, ROTR: pop b, pop a; rotate a's unsigned 64-bit pattern by (b & 63) (any b allowed, no fault); push as signed.
- CLZ: leading zeros of a's unsigned 64-bit pattern (CLZ(0)=64). POPCNT: set bits of that pattern (POPCNT(-1)=64).
- MULH: pop b, pop a; push the high 64 bits of the signed 128-bit product, i.e. (a*b) >> 64 (floor), which is already in range.
- PICK n: push copy of the item n below the top (n=0 is DUP). Fewer than n+1 items -> ValueError. Overflow -> ValueError.
- ROLL n: remove the item n below the top and push it on top (n=0 is a no-op). Fewer than n+1 items -> ValueError.
- JMP target; JZ/JNZ: pop a, jump if a==0 / a!=0 (otherwise fall through).
- JMPI: pop addr; addr must be an instruction start (checked at runtime), else ValueError; pc = addr.
- CALL target / CALLI (pop target; checked like JMPI): if len(call_stack) >= 1024 -> ValueError; push address of the next instruction; pc = target.
- RET: empty call_stack -> ValueError; pc = call_stack.pop().
- LOAD: pop addr; push mem[addr]. STORE: pop addr, pop value; mem[addr] = value. addr outside 0..255 -> FAULT 3.
- MCOPY: pop len, pop src, pop dst (push order dst, src, len). Require 0 <= len, 0 <= src, 0 <= dst, src+len <= 256, dst+len <= 256, else FAULT 3 (checked even when len == 0). Copy has memmove semantics (overlap-safe).
- TRY target: handlers full (256) -> ValueError; push (target, len(stack), len(call_stack)).
- ENDTRY: empty handlers -> ValueError; pop one handler.
- THROW: pop c; raise FAULT c (any int in range).

Faults: a FAULT c with an empty handlers list -> ValueError. Otherwise pop the most recent handler (t, h, d); truncate stack to its first h items and call_stack to its first d items (truncate only, never extend); push c (overflow -> ValueError); pc = t. Handler state is not otherwise restored (memory is not rolled back). Stack underflow/overflow, call-depth overflow, bad indirect targets, steps and gas limits are never catchable. A fault is raised only if the instruction hit no uncatchable error.

Efficiency:
- Validation O(n) time. Execution O(steps) time plus O(len) per MCOPY, and O(n) per ROLL; extra memory O(stack + call_stack + handlers + mem + one bit/byte per instruction start).
- Must not re-decode LEB128 or operands on every executed step (decode once during validation). 10^6 executed steps of simple instructions should finish in under 3 seconds on CPython 3.11.
- No recursion. Keep the solution to no more than 140 lines, excluding comments and blank lines.

Return the function only. No example usage, no tests, no explanation. 

```



# Claude Sonnet 5.5 Initial Response 

``` 

def run(bytecode: bytes, max_steps: int, max_gas: int) -> int:
    E = ValueError
    if not isinstance(bytecode, (bytes, bytearray)) or len(bytecode) > 1 << 20: raise E
    for q in (max_steps, max_gas):
        if not isinstance(q, int) or isinstance(q, bool) or q < 0: raise E
    COST = (1,2,3,3,3,3,3,1,1,2,2,2,2,2,1,2,1,3,3,3,3,3,3,3,3,3,3,2,2,3,3,4,3,3,2,2,3,2,1,2)
    H = 1 << 63; T = 1 << 64; M = T - 1
    bc = bytes(bytecode); n = len(bc)
    ops = []; arg = []; cst = []; jl = []; idx = [-1] * (n + 1); i = 0
    while i < n:
        o = bc[i]
        if o > 39: raise E
        idx[i] = len(ops); c = COST[o]; v = 0; j = i + 1
        if o == 1:
            s = 0
            for k in range(1, 11):
                if j >= n: raise E
                x = bc[j]; j += 1; v |= (x & 127) << s; s += 7
                if x < 128: break
            else: raise E
            if x & 64: v -= 1 << s
            if not -H <= v < H or (k > 1 and -(1 << (7 * k - 8)) <= v < (1 << (7 * k - 8))): raise E
        elif o in (9, 10, 11, 15, 37):
            if j + 4 > n: raise E
            v = int.from_bytes(bc[j:j + 4], 'little'); j += 4
            jl.append(len(ops))
        elif o in (27, 28):
            if j >= n: raise E
            v = bc[j]; j += 1
            if o == 28: c += v
        ops.append(o); arg.append(v); cst.append(c); i = j
    for k in jl:
        if arg[k] >= n or idx[arg[k]] < 0: raise E
        arg[k] = idx[arg[k]]
    st = []; cs = []; hs = []; mem = [0] * 256; pc = steps = gas = 0; cnt = len(ops)
    class F(Exception): pass
    pop = st.pop; push = st.append
    while True:
        try:
            while True:
                if pc >= cnt or steps >= max_steps: raise E
                steps += 1; gas += cst[pc]
                if gas > max_gas: raise E
                o = ops[pc]; v = arg[pc]
                if o == 1:
                    if len(st) > 1023: raise E
                    push(v)
                elif o == 2: b = pop(); a = pop(); push((a + b + H & M) - H)
                elif o == 7:
                    if len(st) > 1023: raise E
                    push(st[-1])
                elif o == 11:
                    if pop(): pc = v; continue
                elif o == 10:
                    if not pop(): pc = v; continue
                elif o == 9: pc = v; continue
                elif o == 3: b = pop(); a = pop(); push((a - b + H & M) - H)
                elif o == 8: st[-1], st[-2] = st[-2], st[-1]
                elif o == 14: pop()
                elif o == 12:
                    a = pop()
                    if not 0 <= a < 256: raise F(3)
                    push(mem[a])
                elif o == 13:
                    a = pop(); x = pop()
                    if not 0 <= a < 256: raise F(3)
                    mem[a] = x
                elif o == 4: b = pop(); a = pop(); push((a * b + H & M) - H)
                elif o == 5 or o == 6:
                    b = pop(); a = pop()
                    if b == 0: raise F(1)
                    r = abs(a) // abs(b) if o == 5 else abs(a) % abs(b)
                    neg = (a < 0) != (b < 0) if o == 5 else a < 0
                    push(((-r if neg else r) + H & M) - H)
                elif 24 <= o <= 26:
                    b = pop(); a = pop()
                    push(int(a == b if o == 24 else a < b if o == 25 else a > b))
                elif 17 <= o <= 19:
                    b = pop(); a = pop()
                    push(a & b if o == 17 else a | b if o == 18 else a ^ b)
                elif o == 20: push(~pop())
                elif 21 <= o <= 23:
                    b = pop(); a = pop()
                    if not 0 <= b < 64: raise F(2)
                    push(((a << b if o == 21 else (a & M) >> b if o == 22 else a >> b) + H & M) - H)
                elif o == 27:
                    if len(st) <= v or len(st) > 1023: raise E
                    push(st[-1 - v])
                elif o == 28: push(pop(-1 - v))
                elif o == 15 or o == 29 or o == 30:
                    if o != 15:
                        a = pop()
                        if not 0 <= a < n or idx[a] < 0: raise E
                        v = idx[a]
                    if o != 29:
                        if len(cs) >= 1024: raise E
                        cs.append(pc + 1)
                    pc = v; continue
                elif o == 16: pc = cs.pop(); continue
                elif o == 31: b = pop(); a = pop(); push(a * b >> 64)
                elif o == 32 or o == 33:
                    b = pop(); a = pop() & M; k = b & 63 if o == 32 else -b & 63
                    push(((a << k | a >> 64 - k) + H & M) - H)
                elif o == 34: push(64 - (pop() & M).bit_length())
                elif o == 35: push(bin(pop() & M).count('1'))
                elif o == 36:
                    if len(st) >= 3 and 0 <= st[-1] <= 256:
                        gas += st[-1]
                        if gas > max_gas: raise E
                    l = pop(); s = pop(); d = pop()
                    if l < 0 or s < 0 or d < 0 or s + l > 256 or d + l > 256: raise F(3)
                    mem[d:d + l] = mem[s:s + l]
                elif o == 37:
                    if len(hs) >= 256: raise E
                    hs.append((v, len(st), len(cs)))
                elif o == 38: hs.pop()
                elif o == 39: raise F(pop())
                else: return st[-1] if st else 0
                pc += 1
        except F as e:
            if not hs: raise E
            pc, h, d = hs.pop(); del st[h:]; del cs[d:]
            if len(st) > 1023: raise E
            push(e.args[0])
        except (IndexError, OverflowError, MemoryError, TypeError): raise E

```





# Validation Layer Assisted Claude Sonnet 5.5 Response 

``` 

def run(bytecode: bytes, max_steps: int, max_gas: int) -> int:
    E = ValueError
    try:  # everything below, including validation and fault handling, is covered by the final except
        if not isinstance(bytecode, (bytes, bytearray)): raise E
        for q in (max_steps, max_gas):
            if not isinstance(q, int) or isinstance(q, bool): raise E
        max_steps = int.__add__(max_steps, 0); max_gas = int.__add__(max_gas, 0)  # normalise int subclasses
        if max_steps < 0 or max_gas < 0: raise E
        bc = bytes(memoryview(bytecode)); n = len(bc)
        if n > 1 << 20: raise E
        COST = (1,2,3,3,3,3,3,1,1,2,2,2,2,2,1,2,1,3,3,3,3,3,3,3,3,3,3,2,2,3,3,4,3,3,2,2,3,2,1,2)
        H = 1 << 63; M = (1 << 64) - 1
        # compact per-offset tables: opcode (255 = not an instruction start), length, int64 operand
        ops = bytearray(b'\xff') * (n + 1); ln = bytearray(n + 1)
        arg = memoryview(bytearray(8 * (n + 1))).cast('q')
        i = 0
        while i < n:
            o = bc[i]
            if o > 39: raise E
            v = 0; j = i + 1
            if o == 1:
                s = 0
                for k in range(1, 11):
                    if j >= n: raise E
                    x = bc[j]; j += 1; v |= (x & 127) << s; s += 7
                    if x < 128: break
                else: raise E
                if x & 64: v -= 1 << s
                # int64 range, and canonical: a k-byte encoding must not fit in k-1 bytes
                if not -H <= v < H or (k > 1 and -(1 << (7 * k - 8)) <= v < (1 << (7 * k - 8))): raise E
            elif o in (9, 10, 11, 15, 37):
                if j + 4 > n: raise E
                v = int.from_bytes(bc[j:j + 4], 'little'); j += 4
            elif o in (27, 28):
                if j >= n: raise E
                v = bc[j]; j += 1
            ops[i] = o; ln[i] = j - i; arg[i] = v; i = j
        for i, o in enumerate(ops):
            if o in (9, 10, 11, 15, 37) and (arg[i] >= n or ops[arg[i]] == 255): raise E
        st = []; cs = []; hs = []; mem = [0] * 256; pc = steps = gas = 0
        pop = st.pop; push = st.append
        class F(Exception): pass
        while True:
            try:
                while True:
                    if pc >= n or steps >= max_steps: raise E
                    o = ops[pc]; steps += 1; gas += COST[o]
                    if gas > max_gas: raise E
                    if o == 1:
                        if len(st) > 1023: raise E
                        push(arg[pc])
                    elif o == 2: b = pop(); a = pop(); push((a + b + H & M) - H)
                    elif o == 7:
                        if len(st) > 1023: raise E
                        push(st[-1])
                    elif o == 11:
                        if pop(): pc = arg[pc]; continue
                    elif o == 10:
                        if not pop(): pc = arg[pc]; continue
                    elif o == 9: pc = arg[pc]; continue
                    elif o == 3: b = pop(); a = pop(); push((a - b + H & M) - H)
                    elif o == 8: st[-1], st[-2] = st[-2], st[-1]
                    elif o == 14: pop()
                    elif o == 12:
                        a = pop()
                        if not 0 <= a < 256: raise F(3)
                        push(mem[a])
                    elif o == 13:
                        a = pop(); x = pop()
                        if not 0 <= a < 256: raise F(3)
                        mem[a] = x
                    elif o == 4: b = pop(); a = pop(); push((a * b + H & M) - H)
                    elif o == 5 or o == 6:
                        b = pop(); a = pop()
                        if b == 0: raise F(1)
                        r = abs(a) // abs(b) if o == 5 else abs(a) % abs(b)
                        neg = (a < 0) != (b < 0) if o == 5 else a < 0
                        push(((-r if neg else r) + H & M) - H)
                    elif 24 <= o <= 26:
                        b = pop(); a = pop()
                        push(int(a == b if o == 24 else a < b if o == 25 else a > b))
                    elif 17 <= o <= 19:
                        b = pop(); a = pop()
                        push(a & b if o == 17 else a | b if o == 18 else a ^ b)
                    elif o == 20: push(~pop())
                    elif 21 <= o <= 23:
                        b = pop(); a = pop()
                        if not 0 <= b < 64: raise F(2)
                        push(((a << b if o == 21 else (a & M) >> b if o == 22 else a >> b) + H & M) - H)
                    elif o == 27:
                        v = arg[pc]
                        if len(st) <= v or len(st) > 1023: raise E
                        push(st[-1 - v])
                    elif o == 28:
                        v = arg[pc]; gas += v
                        if gas > max_gas: raise E
                        push(pop(-1 - v))
                    elif o == 15 or o == 29 or o == 30:
                        if o == 15: t = arg[pc]
                        else:
                            t = pop()
                            if not 0 <= t < n or ops[t] == 255: raise E
                        if o != 29:
                            if len(cs) >= 1024: raise E
                            cs.append(pc + ln[pc])
                        pc = t; continue
                    elif o == 16: pc = cs.pop(); continue
                    elif o == 31: b = pop(); a = pop(); push(a * b >> 64)
                    elif o == 32 or o == 33:
                        b = pop(); a = pop() & M; k = b & 63 if o == 32 else -b & 63  # ROTR = ROTL by -b
                        push(((a << k | a >> 64 - k) + H & M) - H)
                    elif o == 34: push(64 - (pop() & M).bit_length())
                    elif o == 35: push((pop() & M).bit_count())
                    elif o == 36:
                        if len(st) >= 3 and 0 <= st[-1] <= 256:
                            gas += st[-1]
                            if gas > max_gas: raise E
                        l = pop(); s = pop(); d = pop()
                        if l < 0 or s < 0 or d < 0 or s + l > 256 or d + l > 256: raise F(3)
                        mem[d:d + l] = mem[s:s + l]
                    elif o == 37:
                        if len(hs) >= 256: raise E
                        hs.append((arg[pc], len(st), len(cs)))
                    elif o == 38: hs.pop()
                    elif o == 39: raise F(pop())
                    else: return st[-1] if st else 0
                    pc += ln[pc]
            except F as e:
                if not hs: raise E
                pc, h, d = hs.pop(); del st[h:]; del cs[d:]
                if len(st) > 1023: raise E
                push(e.args[0])
    except Exception: raise E from None

```


# Claude Sonnet 5.5 Revised Response 

```

def run(bytecode: bytes, max_steps: int, max_gas: int) -> int:
    E = ValueError
    if not isinstance(bytecode, (bytes, bytearray)) or len(bytecode) > 1 << 20: raise E("bad bytecode")
    for q in (max_steps, max_gas):
        if not isinstance(q, int) or isinstance(q, bool) or q < 0: raise E("bad limit")
    # Gas per opcode 0x00..0x27 (ROLL's dynamic part and MCOPY's are added separately).
    COST = (1,2,3,3,3,3,3,1,1,2,2,2,2,2,1,2,1,3,3,3,3,3,3,3,3,3,3,2,2,3,3,4,3,3,2,2,3,2,1,2)
    H = 1 << 63; M = (1 << 64) - 1          # wrap(x) == (x + H & M) - H
    bc = bytes(bytecode); n = len(bc)
    ops = []; arg = []; cst = []; jl = []; idx = [-1] * (n + 1); i = 0
    # ---- validation: decode once, record instruction starts ----
    while i < n:
        o = bc[i]
        if o > 39: raise E("unknown opcode")
        idx[i] = len(ops); c = COST[o]; v = 0; j = i + 1
        if o == 1:                                   # PUSH <sleb128>
            k = s = 0
            while True:
                if j >= n or k == 10: raise E("bad sleb128")
                x = bc[j]; j += 1; k += 1; v |= (x & 127) << s; s += 7
                if x < 128: break
            if x & 64: v -= 1 << s
            if not -H <= v < H or (k > 1 and -(1 << (7 * k - 8)) <= v < (1 << (7 * k - 8))): raise E("bad sleb128")
        elif o in (9, 10, 11, 15, 37):               # JMP JZ JNZ CALL TRY: u32le target
            if j + 4 > n: raise E("truncated")
            v = int.from_bytes(bc[j:j + 4], 'little'); j += 4
            jl.append(len(ops))
        elif o in (27, 28):                          # PICK / ROLL: u8
            if j >= n: raise E("truncated")
            v = bc[j]; j += 1
            if o == 28: c += v
        ops.append(o); arg.append(v); cst.append(c); i = j
    for k in jl:                                     # resolve byte offsets -> instruction indices
        if arg[k] >= n or idx[arg[k]] < 0: raise E("bad target")
        arg[k] = idx[arg[k]]
    # ---- execution ----
    st = []; cs = []; hs = []; mem = [0] * 256; pc = steps = gas = 0; cnt = len(ops)
    class F(Exception): pass                         # catchable FAULT carrying its code
    pop = st.pop; push = st.append
    while True:
        try:
            while True:
                if pc >= cnt or steps >= max_steps: raise E("limit")
                steps += 1; gas += cst[pc]
                if gas > max_gas: raise E("gas")
                o = ops[pc]; v = arg[pc]
                # hot opcodes first
                if o == 1:
                    if len(st) > 1023: raise E("overflow")
                    push(v)
                elif o == 2: b = pop(); a = pop(); push((a + b + H & M) - H)
                elif o == 7:
                    if len(st) > 1023: raise E("overflow")
                    push(st[-1])
                elif o == 11:
                    if pop(): pc = v; continue
                elif o == 10:
                    if not pop(): pc = v; continue
                elif o == 9: pc = v; continue
                elif o == 3: b = pop(); a = pop(); push((a - b + H & M) - H)
                elif o == 8: st[-1], st[-2] = st[-2], st[-1]
                elif o == 14: pop()
                elif o == 12:
                    a = pop()
                    if not 0 <= a < 256: raise F(3)
                    push(mem[a])
                elif o == 13:
                    a = pop(); x = pop()
                    if not 0 <= a < 256: raise F(3)
                    mem[a] = x
                elif o == 0: return st[-1] if st else 0
                elif o < 17:                         # MUL DIV MOD CALL RET
                    if o == 4: b = pop(); a = pop(); push((a * b + H & M) - H)
                    elif o == 15:
                        if len(cs) >= 1024: raise E("call depth")
                        cs.append(pc + 1); pc = v; continue
                    elif o == 16: pc = cs.pop(); continue
                    else:
                        b = pop(); a = pop()
                        if b == 0: raise F(1)
                        q, r = divmod(abs(a), abs(b))
                        if o == 5: r = -q if (a < 0) != (b < 0) else q
                        elif a < 0: r = -r
                        push((r + H & M) - H)
                elif o < 24:                         # AND OR XOR NOT SHL SHR SAR
                    if o == 20: push(~pop())
                    else:
                        b = pop(); a = pop()
                        if o < 20: push(a & b if o == 17 else a | b if o == 18 else a ^ b)
                        else:
                            if not 0 <= b < 64: raise F(2)
                            push(((a << b if o == 21 else (a & M) >> b if o == 22 else a >> b) + H & M) - H)
                elif o < 27:                         # EQ LT GT
                    b = pop(); a = pop()
                    push(int(a == b if o == 24 else a < b if o == 25 else a > b))
                elif o < 31:                         # PICK ROLL JMPI CALLI
                    if o == 27:
                        if len(st) <= v or len(st) > 1023: raise E("pick")
                        push(st[-1 - v])
                    elif o == 28: push(pop(-1 - v))  # IndexError if fewer than v+1 items
                    else:
                        a = pop()
                        if not 0 <= a < n or idx[a] < 0: raise E("bad indirect target")
                        if o == 30:
                            if len(cs) >= 1024: raise E("call depth")
                            cs.append(pc + 1)
                        pc = idx[a]; continue
                elif o == 31: b = pop(); a = pop(); push(a * b >> 64)
                elif o < 34:                         # ROTL ROTR
                    b = pop(); a = pop() & M; k = b & 63 if o == 32 else -b & 63
                    push(((a << k | a >> 64 - k) + H & M) - H)
                elif o == 34: push(64 - (pop() & M).bit_length())
                elif o == 35: push(bin(pop() & M).count('1'))
                elif o == 36:                        # MCOPY (dynamic gas from operand before pop)
                    if len(st) >= 3 and 0 <= st[-1] <= 256:
                        gas += st[-1]
                        if gas > max_gas: raise E("gas")
                    l = pop(); s = pop(); d = pop()
                    if l < 0 or s < 0 or d < 0 or s + l > 256 or d + l > 256: raise F(3)
                    mem[d:d + l] = mem[s:s + l]
                elif o == 37:
                    if len(hs) >= 256: raise E("handlers full")
                    hs.append((v, len(st), len(cs)))
                elif o == 38: hs.pop()               # IndexError if no handlers
                else: raise F(pop())                 # THROW
                pc += 1
        except F as e:
            if not hs: raise E("unhandled fault")
            pc, h, d = hs.pop(); del st[h:]; del cs[d:]
            if len(st) > 1023: raise E("overflow")
            push(e.args[0])
        except IndexError:                           # stack underflow / empty call stack / empty handlers
            raise E("underflow") 

```
