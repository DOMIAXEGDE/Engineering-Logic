# Documentation Bundle

- **Root:** `C:\Users\dacoo\OneDrive\configure\edition4\OS\version2\TCP\services\omicron`
- **Generated:** `2026-06-15T23:26:51`
- **Included files:** `14`
- **Max text file bytes:** `5000000`
- **Ignored dirs:** `.git, .hg, .idea, .runtime, .svn, .venv, .vscode, __pycache__, build, dist, doc, doc_export, node_modules, vendor, venv`

## Filesystem Tree (included paths)

```text
omicron
├── 0.py
├── 1.c
├── 10.py
├── 11.py
├── 12.md
├── 2.c
├── 3.c
├── 4.cpp
├── 5.cpp
├── 6.cpp
├── 7.php
├── 8.py
├── 9.py
└── dictionary.md
```

## Table of Contents

1. [`0.py`](#file-1)
2. [`1.c`](#file-2)
3. [`10.py`](#file-3)
4. [`11.py`](#file-4)
5. [`12.md`](#file-5)
6. [`2.c`](#file-6)
7. [`3.c`](#file-7)
8. [`4.cpp`](#file-8)
9. [`5.cpp`](#file-9)
10. [`6.cpp`](#file-10)
11. [`7.php`](#file-11)
12. [`8.py`](#file-12)
13. [`9.py`](#file-13)
14. [`dictionary.md`](#file-14)

## File Contents

<a id="file-1"></a>
### [1] `0.py`

- **Bytes:** `19171`
- **Type:** `text`

```python
#!/usr/bin/env python3
"""
Canonical Truth-Table Colour Encoding (CTCE)

A deterministic mapping from a Boolean function's complete truth-table signature
to a reproducible colour value.

Core idea:
    Boolean behaviour -> canonical truth-table bits -> integer/hash -> colour

This file supports:
    - constants
    - unary gates
    - all 16 two-input Boolean functions
    - arbitrary n-input Boolean functions
    - simple combinational circuit composition
    - bounded sequential transition-table encoding
    - deterministic RGB/HEX colours
    - optional multi-swatch colour signatures for very large functions

Important limitation:
    A single 24-bit RGB colour has only 16,777,216 possible values.
    Therefore, for large Boolean functions, RGB can be reproducible but not
    globally unique. The canonical signature/hash remains the unique identifier.
"""

from __future__ import annotations

from dataclasses import dataclass
from hashlib import sha256
from itertools import product
from typing import Callable, Iterable, Mapping, Sequence


Bit = int
Bits = tuple[int, ...]
BoolFunc = Callable[..., int]


# ---------------------------------------------------------------------
# Basic validation
# ---------------------------------------------------------------------

def as_bit(x: int | bool) -> int:
    """Convert an integer/bool value to a strict Boolean bit."""
    return 1 if bool(x) else 0


def require_bits(bits: str) -> str:
    """Validate a truth-table bit string."""
    if not bits:
        raise ValueError("Truth-table bit string cannot be empty.")
    if any(ch not in "01" for ch in bits):
        raise ValueError("Truth-table bit string must contain only 0 and 1.")
    return bits


def infer_input_count_from_table(bits: str) -> int:
    """
    Infer n from a truth table of length 2^n.

    Example:
        len(bits) = 4  -> n = 2
        len(bits) = 8  -> n = 3
        len(bits) = 16 -> n = 4
    """
    bits = require_bits(bits)
    length = len(bits)

    if length & (length - 1) != 0:
        raise ValueError("Truth-table length must be a power of 2.")

    return length.bit_length() - 1


def input_rows(n: int) -> list[Bits]:
    """
    Canonical input ordering.

    For n = 2:
        00, 01, 10, 11

    For n = 3:
        000, 001, 010, 011, 100, 101, 110, 111
    """
    if n < 0:
        raise ValueError("Input count cannot be negative.")
    return [tuple(row) for row in product((0, 1), repeat=n)]


# ---------------------------------------------------------------------
# Colour conversion
# ---------------------------------------------------------------------

def int_to_rgb(value: int) -> tuple[int, int, int]:
    """Map an integer into 24-bit RGB by using its low 24 bits."""
    value = value & 0xFFFFFF
    return ((value >> 16) & 255, (value >> 8) & 255, value & 255)


def rgb_to_hex(rgb: tuple[int, int, int]) -> str:
    """Convert an RGB triple to #RRGGBB."""
    r, g, b = rgb
    return f"#{r:02X}{g:02X}{b:02X}"


def hash_to_rgb(data: str | bytes, salt: str = "CTCE-RGB-v1") -> tuple[int, int, int]:
    """
    Deterministically map arbitrary data to RGB.

    This is reproducible, but not globally unique for all possible functions,
    because RGB has only 24 bits.
    """
    if isinstance(data, str):
        data = data.encode("utf-8")

    digest = sha256(salt.encode("utf-8") + b":" + data).digest()
    value = int.from_bytes(digest[:3], "big")
    return int_to_rgb(value)


def truth_bits_to_direct_rgb(bits: str) -> tuple[int, int, int]:
    """
    Direct numeric colour mapping.

    This works especially cleanly for small truth tables. For the 16 two-input
    functions, it spreads the IDs 0..15 across the 24-bit RGB range.
    """
    bits = require_bits(bits)
    function_id = int(bits, 2)
    max_id = (1 << len(bits)) - 1

    if max_id == 0:
        return (0, 0, 0)

    scaled = round((function_id / max_id) * 0xFFFFFF)
    return int_to_rgb(scaled)


def truth_bits_to_hash_rgb(bits: str) -> tuple[int, int, int]:
    """Hash-based colour mapping for arbitrary-size truth tables."""
    bits = require_bits(bits)
    return hash_to_rgb(bits)


def truth_bits_to_multiswatch(bits: str, swatches: int = 4) -> list[str]:
    """
    Produce a longer deterministic colour signature.

    This reduces visual collision risk by giving several colours rather than
    one colour. It is useful for large circuits, PLAs, FPGAs, ASIC blocks, and
    bounded sequential machines.
    """
    bits = require_bits(bits)
    if swatches <= 0:
        raise ValueError("swatches must be positive.")

    digest = sha256(("CTCE-MULTI-v1:" + bits).encode("utf-8")).digest()
    needed = swatches * 3

    while len(digest) < needed:
        digest += sha256(digest).digest()

    colours: list[str] = []
    for i in range(swatches):
        chunk = digest[i * 3:(i + 1) * 3]
        colours.append(rgb_to_hex(tuple(chunk)))  # type: ignore[arg-type]

    return colours


# ---------------------------------------------------------------------
# BooleanFunction object
# ---------------------------------------------------------------------

@dataclass(frozen=True)
class BooleanFunction:
    """
    Canonical representation of a Boolean function.

    The truth table is stored in canonical row order:
        0...00, 0...01, ..., 1...11
    """
    name: str
    input_count: int
    truth_bits: str

    def __post_init__(self) -> None:
        require_bits(self.truth_bits)
        expected = 1 << self.input_count
        if len(self.truth_bits) != expected:
            raise ValueError(
                f"{self.name}: expected truth table length {expected}, "
                f"got {len(self.truth_bits)}."
            )

    @property
    def function_id(self) -> int:
        return int(self.truth_bits, 2)

    @property
    def canonical_signature(self) -> str:
        return f"CTCE:v1:n={self.input_count}:bits={self.truth_bits}"

    @property
    def canonical_hash(self) -> str:
        return sha256(self.canonical_signature.encode("utf-8")).hexdigest()

    @property
    def direct_hex(self) -> str:
        return rgb_to_hex(truth_bits_to_direct_rgb(self.truth_bits))

    @property
    def hash_hex(self) -> str:
        return rgb_to_hex(truth_bits_to_hash_rgb(self.canonical_signature))

    @property
    def multiswatch(self) -> list[str]:
        return truth_bits_to_multiswatch(self.canonical_signature, swatches=4)

    def evaluate(self, *inputs: int | bool) -> int:
        if len(inputs) != self.input_count:
            raise ValueError(
                f"{self.name} expects {self.input_count} inputs, "
                f"got {len(inputs)}."
            )

        bits = tuple(as_bit(x) for x in inputs)
        index = int("".join(str(x) for x in bits), 2) if bits else 0
        return int(self.truth_bits[index])

    def table_rows(self) -> list[tuple[Bits, int]]:
        return [(row, self.evaluate(*row)) for row in input_rows(self.input_count)]


def from_callable(name: str, input_count: int, func: BoolFunc) -> BooleanFunction:
    """Create a BooleanFunction from a Python callable."""
    outputs: list[str] = []

    for row in input_rows(input_count):
        outputs.append(str(as_bit(func(*row))))

    return BooleanFunction(name=name, input_count=input_count, truth_bits="".join(outputs))


def from_truth_bits(name: str, bits: str) -> BooleanFunction:
    """Create a BooleanFunction directly from a truth-table bit string."""
    n = infer_input_count_from_table(bits)
    return BooleanFunction(name=name, input_count=n, truth_bits=bits)


# ---------------------------------------------------------------------
# Primitive gates
# ---------------------------------------------------------------------

GATES_0_INPUT: dict[str, BooleanFunction] = {
    "FALSE/0": BooleanFunction("FALSE/0", 0, "0"),
    "TRUE/1": BooleanFunction("TRUE/1", 0, "1"),
}

GATES_1_INPUT: dict[str, BooleanFunction] = {
    "BUFFER/IDENTITY": from_callable("BUFFER/IDENTITY", 1, lambda a: a),
    "NOT/INVERTER": from_callable("NOT/INVERTER", 1, lambda a: not a),
}

# All 16 two-input Boolean functions in canonical order:
# input rows are 00, 01, 10, 11
GATES_2_INPUT: dict[str, BooleanFunction] = {
    "FALSE": from_truth_bits("FALSE", "0000"),
    "AND": from_truth_bits("AND", "0001"),
    "A_AND_NOT_B": from_truth_bits("A_AND_NOT_B", "0010"),
    "A": from_truth_bits("A", "0011"),
    "NOT_A_AND_B": from_truth_bits("NOT_A_AND_B", "0100"),
    "B": from_truth_bits("B", "0101"),
    "XOR": from_truth_bits("XOR", "0110"),
    "OR": from_truth_bits("OR", "0111"),
    "NOR": from_truth_bits("NOR", "1000"),
    "XNOR/EQUIVALENCE": from_truth_bits("XNOR/EQUIVALENCE", "1001"),
    "NOT_B": from_truth_bits("NOT_B", "1010"),
    "A_OR_NOT_B": from_truth_bits("A_OR_NOT_B", "1011"),
    "NOT_A": from_truth_bits("NOT_A", "1100"),
    "NOT_A_OR_B": from_truth_bits("NOT_A_OR_B", "1101"),
    "NAND": from_truth_bits("NAND", "1110"),
    "TRUE": from_truth_bits("TRUE", "1111"),
}

ALIASES: dict[str, str] = {
    "IMPLY": "NOT_A_OR_B",
    "A_IMPLIES_B": "NOT_A_OR_B",
    "NON_IMPLY": "A_AND_NOT_B",
    "A_NOT_IMPLIES_B": "A_AND_NOT_B",
    "CONVERSE_IMPLY": "A_OR_NOT_B",
    "B_IMPLIES_A": "A_OR_NOT_B",
    "CONVERSE_NON_IMPLY": "NOT_A_AND_B",
    "B_NOT_IMPLIES_A": "NOT_A_AND_B",
    "PROJECTION_LEFT": "A",
    "PROJECTION_RIGHT": "B",
    "NOT_A_GATE": "NOT_A",
    "NOT_B_GATE": "NOT_B",
}


def get_gate(name: str) -> BooleanFunction:
    """Fetch a primitive gate by name or alias."""
    key = name.strip().upper().replace(" ", "_").replace("-", "_")

    for table in (GATES_0_INPUT, GATES_1_INPUT, GATES_2_INPUT):
        if key in table:
            return table[key]

    if key in ALIASES:
        return GATES_2_INPUT[ALIASES[key]]

    raise KeyError(f"Unknown gate: {name}")


# ---------------------------------------------------------------------
# Combinational circuit composition
# ---------------------------------------------------------------------

@dataclass(frozen=True)
class CircuitNode:
    """
    A node in a combinational circuit.

    inputs:
        Each input is either:
            - an integer external-input index, e.g. 0 for x0, 1 for x1
            - a node name referring to an earlier node
    """
    name: str
    gate: BooleanFunction
    inputs: tuple[int | str, ...]


@dataclass(frozen=True)
class CombinationalCircuit:
    """
    A simple acyclic circuit evaluator.

    Nodes must be topologically ordered:
        every node can only depend on external inputs or earlier nodes.
    """
    name: str
    input_count: int
    nodes: tuple[CircuitNode, ...]
    outputs: tuple[int | str, ...]

    def evaluate(self, *external_inputs: int | bool) -> Bits:
        if len(external_inputs) != self.input_count:
            raise ValueError(
                f"{self.name} expects {self.input_count} inputs, "
                f"got {len(external_inputs)}."
            )

        ext = tuple(as_bit(x) for x in external_inputs)
        memory: dict[str, int] = {}

        def resolve(ref: int | str) -> int:
            if isinstance(ref, int):
                return ext[ref]
            return memory[ref]

        for node in self.nodes:
            args = tuple(resolve(ref) for ref in node.inputs)
            memory[node.name] = node.gate.evaluate(*args)

        return tuple(resolve(ref) for ref in self.outputs)

    def output_truth_bits(self) -> str:
        """
        Encode multi-output circuits by concatenating output bits per row.

        Example for 2 outputs:
            row 00 -> 01
            row 01 -> 11
            row 10 -> 10
            row 11 -> 00

            encoded bits -> 01111000
        """
        bits: list[str] = []

        for row in input_rows(self.input_count):
            output_bits = self.evaluate(*row)
            bits.extend(str(bit) for bit in output_bits)

        return "".join(bits)

    def colour_encoding(self) -> dict[str, object]:
        bits = self.output_truth_bits()
        signature = f"CTCE:v1:circuit={self.name}:n={self.input_count}:out={len(self.outputs)}:bits={bits}"
        return {
            "name": self.name,
            "input_count": self.input_count,
            "output_count": len(self.outputs),
            "truth_bits": bits,
            "canonical_signature": signature,
            "canonical_hash": sha256(signature.encode("utf-8")).hexdigest(),
            "hash_hex": rgb_to_hex(hash_to_rgb(signature)),
            "multiswatch": truth_bits_to_multiswatch(signature, swatches=4),
        }


# ---------------------------------------------------------------------
# Useful module constructors
# ---------------------------------------------------------------------

def make_half_adder() -> CombinationalCircuit:
    return CombinationalCircuit(
        name="HALF_ADDER",
        input_count=2,
        nodes=(
            CircuitNode("sum", GATES_2_INPUT["XOR"], (0, 1)),
            CircuitNode("carry", GATES_2_INPUT["AND"], (0, 1)),
        ),
        outputs=("sum", "carry"),
    )


def make_full_adder() -> CombinationalCircuit:
    return CombinationalCircuit(
        name="FULL_ADDER",
        input_count=3,
        nodes=(
            CircuitNode("a_xor_b", GATES_2_INPUT["XOR"], (0, 1)),
            CircuitNode("sum", GATES_2_INPUT["XOR"], ("a_xor_b", 2)),
            CircuitNode("a_and_b", GATES_2_INPUT["AND"], (0, 1)),
            CircuitNode("cin_and_axorb", GATES_2_INPUT["AND"], (2, "a_xor_b")),
            CircuitNode("carry", GATES_2_INPUT["OR"], ("a_and_b", "cin_and_axorb")),
        ),
        outputs=("sum", "carry"),
    )


def make_mux_2_to_1() -> CombinationalCircuit:
    """
    2-to-1 multiplexer:
        inputs: a, b, select
        output: a if select=0 else b
    """
    return CombinationalCircuit(
        name="MUX_2_TO_1",
        input_count=3,
        nodes=(
            CircuitNode("not_s", GATES_1_INPUT["NOT/INVERTER"], (2,)),
            CircuitNode("a_path", GATES_2_INPUT["AND"], (0, "not_s")),
            CircuitNode("b_path", GATES_2_INPUT["AND"], (1, 2)),
            CircuitNode("out", GATES_2_INPUT["OR"], ("a_path", "b_path")),
        ),
        outputs=("out",),
    )


# ---------------------------------------------------------------------
# Bounded sequential encoding
# ---------------------------------------------------------------------

@dataclass(frozen=True)
class TransitionMachine:
    """
    Bounded sequential-machine representation.

    A sequential device is not fully described by a single ordinary
    combinational truth table unless its state is included.

    This representation encodes:
        current_state bits + input bits -> next_state bits + output bits

    That transition table can then be colour-encoded.
    """
    name: str
    state_bit_count: int
    input_count: int
    output_count: int
    transition: Callable[[Bits, Bits], tuple[Bits, Bits]]

    def transition_bits(self) -> str:
        bits: list[str] = []

        for state in input_rows(self.state_bit_count):
            for inputs in input_rows(self.input_count):
                next_state, outputs = self.transition(state, inputs)

                if len(next_state) != self.state_bit_count:
                    raise ValueError("Invalid next-state width.")

                if len(outputs) != self.output_count:
                    raise ValueError("Invalid output width.")

                bits.extend(str(as_bit(x)) for x in next_state)
                bits.extend(str(as_bit(x)) for x in outputs)

        return "".join(bits)

    def colour_encoding(self) -> dict[str, object]:
        bits = self.transition_bits()
        signature = (
            f"CTCE:v1:machine={self.name}:"
            f"state={self.state_bit_count}:in={self.input_count}:"
            f"out={self.output_count}:bits={bits}"
        )
        return {
            "name": self.name,
            "state_bit_count": self.state_bit_count,
            "input_count": self.input_count,
            "output_count": self.output_count,
            "transition_bits": bits,
            "canonical_signature": signature,
            "canonical_hash": sha256(signature.encode("utf-8")).hexdigest(),
            "hash_hex": rgb_to_hex(hash_to_rgb(signature)),
            "multiswatch": truth_bits_to_multiswatch(signature, swatches=4),
        }


def make_t_flip_flop() -> TransitionMachine:
    """
    T flip-flop:
        state q
        input t
        next q = q XOR t
        output q_next
    """
    def transition(state: Bits, inputs: Bits) -> tuple[Bits, Bits]:
        q = state[0]
        t = inputs[0]
        q_next = q ^ t
        return (q_next,), (q_next,)

    return TransitionMachine(
        name="T_FLIP_FLOP",
        state_bit_count=1,
        input_count=1,
        output_count=1,
        transition=transition,
    )


# ---------------------------------------------------------------------
# Reporting
# ---------------------------------------------------------------------

def describe_boolean_function(fn: BooleanFunction) -> dict[str, object]:
    return {
        "name": fn.name,
        "input_count": fn.input_count,
        "truth_bits": fn.truth_bits,
        "function_id": fn.function_id,
        "direct_hex": fn.direct_hex,
        "hash_hex": fn.hash_hex,
        "canonical_hash": fn.canonical_hash,
        "multiswatch": fn.multiswatch,
    }


def print_primitive_gate_table() -> None:
    print("name.truth_bits.function_id.direct_hex.hash_hex")

    for gate in (
        list(GATES_0_INPUT.values())
        + list(GATES_1_INPUT.values())
        + list(GATES_2_INPUT.values())
    ):
        print(
            f"{gate.name}."
            f"{gate.truth_bits}."
            f"{gate.function_id}."
            f"{gate.direct_hex}."
            f"{gate.hash_hex}"
        )


def print_encoding(title: str, encoding: Mapping[str, object]) -> None:
    print(title)
    for key, value in encoding.items():
        print(f"{key}: {value}")
    print()


# ---------------------------------------------------------------------
# Demonstration
# ---------------------------------------------------------------------

def main() -> None:
    print("CANONICAL TRUTH-TABLE COLOUR ENCODING")
    print()

    print("Primitive gates using '.' as delimiter:")
    print_primitive_gate_table()
    print()

    # Arbitrary Boolean function from callable:
    majority_3 = from_callable(
        "MAJORITY_3",
        3,
        lambda a, b, c: (a + b + c) >= 2,
    )

    print("Example arbitrary Boolean function:")
    print(describe_boolean_function(majority_3))
    print()

    # Composed circuit modules:
    half_adder = make_half_adder()
    full_adder = make_full_adder()
    mux = make_mux_2_to_1()

    print_encoding("Half-adder encoding:", half_adder.colour_encoding())
    print_encoding("Full-adder encoding:", full_adder.colour_encoding())
    print_encoding("2-to-1 MUX encoding:", mux.colour_encoding())

    # Bounded sequential device:
    tff = make_t_flip_flop()
    print_encoding("T flip-flop transition encoding:", tff.colour_encoding())


if __name__ == "__main__":
    main()

```

<a id="file-2"></a>
### [2] `1.c`

- **Bytes:** `4669`
- **Type:** `text`

```text
/**
 * @file main.c
 * @brief Permutation Generator in C, architected to safety-critical standards.
 *
 * @author Dominic Alexander Cooper (Original Algorithm)
 * @author Gemini (Architectural Refactoring)
 *
 * @details
 * This program generates all possible character combinations for a given length,
 * based on a predefined character set. It is a complete rewrite of the original
 * concept to align with principles of safety-critical software design.
 */
#include <stdio.h>
#include <stdint.h> // For fixed-width integer types like uint64_t
#include <stdbool.h> // For bool type
#include <limits.h> // For UINT64_MAX
//==============================================================================
// 1. CONFIGURATION AND DATA DEFINITIONS
//==============================================================================
#define MAX_PERMUTATION_LENGTH 10
static const char ALPHABET[] = {
    'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm',
    'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z',
    'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M',
    'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z',
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', ' ', '\n'
};
static const uint64_t ALPHABET_SIZE = sizeof(ALPHABET) / sizeof(ALPHABET[0]);
//==============================================================================
// 2. ABSTRACT INTERFACE FOR OUTPUT (OutputSink)
//==============================================================================
typedef struct {
    void* context;
    bool (*write)(void* context, const char* buffer, uint64_t size);
    bool (*write_char)(void* context, char c);
} OutputSink;
//==============================================================================
// 3. CORE LOGIC (Generator)
//==============================================================================
static bool safe_uint64_power(uint64_t base, uint64_t exp, uint64_t* result) {
    *result = 1;
    for (uint64_t i = 0; i < exp; ++i) {
        if (*result > UINT64_MAX / base) {
            return false;
        }
        *result *= base;
    }
    return true;
}
static bool generate_permutations(uint64_t length, OutputSink* sink) {
    uint64_t num_combinations;
    if (!safe_uint64_power(ALPHABET_SIZE, length, &num_combinations)) {
        return false;
    }
    char current_perm[MAX_PERMUTATION_LENGTH];
    for (uint64_t i = 0; i < num_combinations; ++i) {
        uint64_t temp_row = i;
        for (int64_t j = length - 1; j >= 0; --j) {
            uint64_t char_index = temp_row % ALPHABET_SIZE;
            current_perm[j] = ALPHABET[char_index];
            temp_row /= ALPHABET_SIZE;
        }
        if (!sink->write(sink->context, current_perm, length)) {
            return false;
        }
        if (!sink->write_char(sink->context, '\n')) {
            return false;
        }
    }
    return true;
}
//==============================================================================
// 4. CONCRETE IMPLEMENTATION OF OUTPUTSINK (FileSink)
//==============================================================================
static bool file_sink_write(void* context, const char* buffer, uint64_t size) {
    FILE* p = (FILE*)context;
    return fwrite(buffer, sizeof(char), size, p) == size;
}
static bool file_sink_write_char(void* context, char c) {
    FILE* p = (FILE*)context;
    return fputc(c, p) != EOF;
}
static bool file_sink_init(OutputSink* sink, const char* filename) {
    FILE* p = fopen(filename, "w");
    if (p == NULL) {
        return false;
    }
    *sink = (OutputSink){
        .context = p,
        .write = file_sink_write,
        .write_char = file_sink_write_char
    };
    return true;
}
static void file_sink_close(OutputSink* sink) {
    if (sink && sink->context) {
        fclose((FILE*)sink->context);
        sink->context = NULL;
    }
}
//==============================================================================
// 5. SYSTEM ASSEMBLER (main)
//==============================================================================
int main(void) {
    const uint64_t permutation_length = 4;
    if (permutation_length == 0 || permutation_length > MAX_PERMUTATION_LENGTH) {
        return 1;
    }
    OutputSink file_sink;
    if (!file_sink_init(&file_sink, "system_safe.txt")) {
        perror("Error opening file");
        return 1;
    }
    bool success = generate_permutations(permutation_length, &file_sink);
    file_sink_close(&file_sink);
    if (!success) {
        return 1;
    }
    return 0;
}
```

<a id="file-3"></a>
### [3] `10.py`

- **Bytes:** `9296`
- **Type:** `text`

````python
#!/usr/bin/env python3
from __future__ import annotations

import argparse
import datetime as _dt
import os
import re
from pathlib import Path
from typing import Dict, Iterable, List, Optional, Tuple, Union

# ---- CONFIG ----
TEXT_EXTENSIONS = {
    ".css", ".cmd", ".js", ".php", ".hpp", ".cpp", ".md", ".py", ".txt", ".ps1", ".html", ".h", ".json", ".c", ".fabric", ".facts", ".rules"
}
IMAGE_EXTENSIONS = {".png"}

SPECIAL_FILENAMES = {".env", ".gitignore", ".htaccess"}  # extensionless-but-important

IGNORE_DIRS = {
    ".git", ".svn", ".hg",
    "node_modules", "vendor",
    "venv", ".venv", "__pycache__",
    "dist", "build", "doc_export",
    ".idea", ".vscode",
    "doc", ".runtime",  # prevent bundling the bundle output folder itself
}

MAX_TEXT_FILE_BYTES = 5_000_000  # 5MB cap for *text* files


# ---- HELPERS ----
def is_probably_binary(path: Path, sample_size: int = 4096) -> bool:
    try:
        with path.open("rb") as f:
            chunk = f.read(sample_size)
        return b"\x00" in chunk
    except Exception:
        return True


def is_text_included(path: Path) -> bool:
    if path.name in SPECIAL_FILENAMES:
        return True
    return path.suffix.lower() in TEXT_EXTENSIONS


def is_png(path: Path) -> bool:
    return path.suffix.lower() == ".png"


def is_included_any(path: Path) -> bool:
    return is_text_included(path) or (path.suffix.lower() in IMAGE_EXTENSIONS)


def iter_included_paths(root: Path) -> Iterable[Path]:
    for dirpath, dirnames, filenames in os.walk(root):
        # prune ignored dirs
        dirnames[:] = [d for d in dirnames if d not in IGNORE_DIRS]

        for name in filenames:
            p = Path(dirpath) / name
            if p.is_file() and is_included_any(p):
                yield p


def read_text(path: Path) -> Tuple[str, str]:
    """
    Returns (content, note). note is "" when OK.
    """
    try:
        size = path.stat().st_size
        if size > MAX_TEXT_FILE_BYTES:
            return "", f"Skipped (too large): {size} bytes > {MAX_TEXT_FILE_BYTES}"
        if is_probably_binary(path):
            return "", "Skipped (binary detected)"
        return path.read_text(encoding="utf-8", errors="replace"), ""
    except Exception as e:
        return "", f"Error reading: {e}"


def detect_code_lang(path: Path) -> str:
    ext = path.suffix.lower()
    return {
        ".py": "python",
        ".js": "javascript",
        ".ts": "typescript",
        ".json": "json",
        ".html": "html",
        ".css": "css",
        ".ps1": "powershell",
        ".cmd": "bat",
        ".cpp": "cpp",
        ".hpp": "cpp",
        ".php": "php",
        ".md": "markdown",
        ".txt": "text",
        "": "text",
    }.get(ext, "text")


def fenced_block(content: str, lang: str) -> str:
    """
    Create a fenced code block that won't break if the content contains ``` already.
    """
    # Find the longest run of backticks in the content
    runs = [len(m.group(0)) for m in re.finditer(r"`+", content)]
    fence_len = max(runs) + 1 if runs else 3
    fence = "`" * max(3, fence_len)
    return f"{fence}{lang}\n{content}\n{fence}\n"


def relpath_posix(from_dir: Path, to_path: Path) -> str:
    rel = os.path.relpath(to_path, start=from_dir)
    return Path(rel).as_posix()


def png_dimensions(path: Path) -> Optional[Tuple[int, int]]:
    """
    Parse PNG IHDR to get (width, height) without external dependencies.
    Returns None if unreadable/not PNG.
    """
    try:
        with path.open("rb") as f:
            header = f.read(24)
        if len(header) < 24:
            return None
        sig = header[:8]
        if sig != b"\x89PNG\r\n\x1a\n":
            return None
        chunk_type = header[12:16]
        if chunk_type != b"IHDR":
            return None
        w = int.from_bytes(header[16:20], "big")
        h = int.from_bytes(header[20:24], "big")
        return (w, h)
    except Exception:
        return None


# ---- TREE BUILD + RENDER ----
TreeNode = Dict[str, Union["TreeNode", None]]  # None means leaf file


def build_tree(relpaths: List[Path]) -> TreeNode:
    root: TreeNode = {}
    for rp in relpaths:
        cur = root
        parts = rp.parts
        for i, part in enumerate(parts):
            is_last = (i == len(parts) - 1)
            if is_last:
                cur.setdefault(part, None)
            else:
                nxt = cur.get(part)
                if nxt is None:
                    cur[part] = {}
                cur = cur[part]  # type: ignore[assignment]
    return root


def render_tree(tree: TreeNode, prefix: str = "") -> List[str]:
    """
    ASCII tree with deterministic ordering:
    directories first, then files, each group alphabetically.
    """
    lines: List[str] = []
    items = list(tree.items())

    def is_dir(item):
        _, v = item
        return isinstance(v, dict)

    dirs = sorted([it for it in items if is_dir(it)], key=lambda x: x[0].lower())
    files = sorted([it for it in items if not is_dir(it)], key=lambda x: x[0].lower())
    ordered = dirs + files

    for idx, (name, node) in enumerate(ordered):
        last = (idx == len(ordered) - 1)
        branch = "└── " if last else "├── "
        lines.append(prefix + branch + name)

        if isinstance(node, dict):
            extension = "    " if last else "│   "
            lines.extend(render_tree(node, prefix + extension))

    return lines


# ---- MAIN OUTPUT ----
def create_doc_md(root: Path, outpath: Path) -> Path:
    files = sorted(iter_included_paths(root), key=lambda p: p.relative_to(root).as_posix().lower())
    relpaths = [p.relative_to(root) for p in files]

    tree = build_tree(relpaths)
    tree_lines = [root.name or root.as_posix()] + render_tree(tree)

    outpath.parent.mkdir(parents=True, exist_ok=True)

    now = _dt.datetime.now().isoformat(timespec="seconds")

    with outpath.open("w", encoding="utf-8") as out:
        # Header
        out.write("# Documentation Bundle\n\n")
        out.write(f"- **Root:** `{root}`\n")
        out.write(f"- **Generated:** `{now}`\n")
        out.write(f"- **Included files:** `{len(files)}`\n")
        out.write(f"- **Max text file bytes:** `{MAX_TEXT_FILE_BYTES}`\n")
        out.write(f"- **Ignored dirs:** `{', '.join(sorted(IGNORE_DIRS))}`\n\n")

        # Tree view
        out.write("## Filesystem Tree (included paths)\n\n")
        out.write(fenced_block("\n".join(tree_lines), "text"))
        out.write("\n")

        # TOC
        out.write("## Table of Contents\n\n")
        for i, rp in enumerate(relpaths, start=1):
            out.write(f"{i}. [`{rp.as_posix()}`](#file-{i})\n")
        out.write("\n")

        # File contents
        out.write("## File Contents\n\n")
        for i, path in enumerate(files, start=1):
            rp = path.relative_to(root).as_posix()
            out.write(f'<a id="file-{i}"></a>\n')
            out.write(f"### [{i}] `{rp}`\n\n")

            size = path.stat().st_size
            out.write(f"- **Bytes:** `{size}`\n")

            if is_png(path):
                out.write("- **Type:** `png`\n")
                dims = png_dimensions(path)
                if dims:
                    out.write(f"- **Dimensions:** `{dims[0]}×{dims[1]}`\n")
                elif size == 0:
                    out.write("- **Dimensions:** `unknown (empty file)`\n")
                else:
                    out.write("- **Dimensions:** `unknown`\n")

                # Embed image (path relative to the markdown file location)
                md_rel = relpath_posix(outpath.parent, path)
                out.write(f"- **Path (from doc):** `{md_rel}`\n\n")
                out.write(f"![{rp}]({md_rel})\n\n")
                continue

            # Text file
            out.write("- **Type:** `text`\n")
            content, note = read_text(path)
            if note:
                out.write(f"- **NOTE:** {note}\n")
            out.write("\n")

            if content:
                lang = detect_code_lang(path)
                out.write(fenced_block(content, lang))
                out.write("\n")
            else:
                out.write("_No content (skipped or empty)._ \n\n")

    return outpath


def main() -> int:
    parser = argparse.ArgumentParser(
        description="Bundle repository documentation into doc/doc.md (including .png images)."
    )
    parser.add_argument(
        "--root",
        type=Path,
        default=Path(__file__).resolve().parent,
        help="Root folder to scan (default: script directory).",
    )
    parser.add_argument(
        "--out",
        type=Path,
        default=None,
        help="Output markdown path (default: <root>/doc/doc.md).",
    )
    args = parser.parse_args()

    root = args.root.resolve()
    out = (args.out.resolve() if args.out else (root / "doc" / "doc.md"))

    outpath = create_doc_md(root, out)
    print(f"Documentation updated: Created '{outpath}'")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())

````

<a id="file-4"></a>
### [4] `11.py`

- **Bytes:** `12424`
- **Type:** `text`

```python
#!/usr/bin/env python3
from __future__ import annotations

import argparse
import re
import shutil
import sys
from dataclasses import dataclass
from pathlib import Path
from typing import List, Optional, Sequence, Tuple


HEADER_RE = re.compile(
    r'^<a id="file-(?P<a_id>\d+)"></a>\s*\n'
    r'### \[(?P<num>\d+)\] `(?P<path>[^`\n]+)`\s*\n',
    re.MULTILINE,
)

TYPE_RE = re.compile(r'^- \*\*Type:\*\* `(?P<type>[^`]+)`\s*$', re.MULTILINE)
PATH_FROM_DOC_RE = re.compile(r'^- \*\*Path \(from doc\):\*\* `(?P<path>[^`]+)`\s*$', re.MULTILINE)
NOTE_RE = re.compile(r'^- \*\*NOTE:\*\* (?P<note>.+?)\s*$', re.MULTILINE)

NO_CONTENT_MARKER = "_No content (skipped or empty)._"


@dataclass(frozen=True)
class ExportedFile:
    index: int
    relative_path: str
    file_type: str
    target_path: Path
    status: str
    warning: Optional[str] = None


@dataclass(frozen=True)
class ParsedSection:
    index: int
    relative_path: str
    file_type: str
    body: str


def _normalise_relative_path(raw_path: str) -> Path:
    """
    Convert the POSIX relative paths emitted by bundle_documentation.py into a safe
    platform-local relative Path.

    Rejects absolute paths, drive paths, and any path containing '..' so a malicious
    or edited doc.md cannot write outside the selected export directory.
    """
    raw = raw_path.strip().replace("\\", "/")
    if not raw:
        raise ValueError("empty path")

    # Reject POSIX absolute and Windows drive / UNC forms before Path conversion.
    if raw.startswith("/") or raw.startswith("//") or re.match(r"^[A-Za-z]:/", raw):
        raise ValueError(f"unsafe absolute path: {raw_path!r}")

    parts = [part for part in raw.split("/") if part not in ("", ".")]
    if not parts:
        raise ValueError(f"empty path after normalisation: {raw_path!r}")
    if any(part == ".." for part in parts):
        raise ValueError(f"unsafe parent-directory segment in path: {raw_path!r}")

    return Path(*parts)


def _find_code_fence(section_body: str) -> Optional[Tuple[str, str, int, int]]:
    """
    Return (fence, language, content_start, content_end) for the first fenced code
    block in a file section. The bundler selects a backtick fence longer than any
    run of backticks in the content, then writes:

        <fence><lang>
        <content>
        <fence>

    Therefore the exporter removes exactly one final line break before the closing
    fence to recover the original text content.
    """
    opening_re = re.compile(r"^(`{3,})([^\r\n`]*)\r?\n", re.MULTILINE)

    for opening in opening_re.finditer(section_body):
        fence = opening.group(1)
        language = opening.group(2).strip()
        closing_re = re.compile(rf"^{re.escape(fence)}[ \t]*\r?$", re.MULTILINE)
        closing = closing_re.search(section_body, opening.end())
        if closing:
            content_start = opening.end()
            content_end = closing.start()
            return fence, language, content_start, content_end

    return None


def _extract_text_content(section: ParsedSection) -> Tuple[str, Optional[str]]:
    """
    Extract text file contents from the bundled Markdown section.

    Returns (content, warning). Empty original files and skipped files both appear
    without a fenced block in doc.md. If a NOTE says the file was skipped, the
    exporter creates an empty file and reports a warning because the original bytes
    are not present in the bundle.
    """
    fence_info = _find_code_fence(section.body)
    note_match = NOTE_RE.search(section.body)
    note = note_match.group("note") if note_match else None

    if fence_info is None:
        warning = None
        if note:
            warning = f"{section.relative_path}: content was not embedded in doc.md ({note}); created empty file"
        elif NO_CONTENT_MARKER in section.body:
            warning = None  # Most commonly an originally empty file.
        else:
            warning = f"{section.relative_path}: no fenced content found; created empty file"
        return "", warning

    _, _, start, end = fence_info
    content = section.body[start:end]

    # bundle_documentation.py always writes one extra newline between the content
    # and the closing fence. Remove exactly that sentinel newline.
    if content.endswith("\r\n"):
        content = content[:-2]
    elif content.endswith("\n"):
        content = content[:-1]

    return content, None


def _try_copy_png(section: ParsedSection, doc_path: Path, output_path: Path, asset_root: Optional[Path]) -> Tuple[bool, Optional[str]]:
    """
    PNG files are linked by bundle_documentation.py, not embedded. Try to recover
    them from the linked path beside doc.md, then from --asset-root when provided.
    """
    candidates: List[Path] = []

    linked_match = PATH_FROM_DOC_RE.search(section.body)
    if linked_match:
        linked = linked_match.group("path")
        candidates.append((doc_path.parent / linked).resolve())

    if asset_root is not None:
        try:
            rel = _normalise_relative_path(section.relative_path)
            candidates.append((asset_root / rel).resolve())
        except ValueError:
            pass

    for candidate in candidates:
        if candidate.is_file():
            output_path.parent.mkdir(parents=True, exist_ok=True)
            shutil.copy2(candidate, output_path)
            return True, None

    return False, (
        f"{section.relative_path}: PNG bytes are not embedded in doc.md and no source image was found; "
        "use --asset-root <original-project-root> if the original PNG files still exist"
    )


def parse_bundle_doc(doc_text: str) -> List[ParsedSection]:
    matches = list(HEADER_RE.finditer(doc_text))
    sections: List[ParsedSection] = []

    for pos, match in enumerate(matches):
        start = match.end()
        end = matches[pos + 1].start() if pos + 1 < len(matches) else len(doc_text)
        body = doc_text[start:end]

        type_match = TYPE_RE.search(body)
        file_type = type_match.group("type").strip().lower() if type_match else "text"

        sections.append(
            ParsedSection(
                index=int(match.group("num")),
                relative_path=match.group("path"),
                file_type=file_type,
                body=body,
            )
        )

    return sections


def export_doc_md(
    doc_path: Path,
    output_dir: Path,
    *,
    asset_root: Optional[Path] = None,
    clean: bool = False,
    strict: bool = False,
) -> List[ExportedFile]:
    doc_path = doc_path.resolve()
    output_dir = output_dir.resolve()
    asset_root = asset_root.resolve() if asset_root is not None else None

    if not doc_path.is_file():
        raise FileNotFoundError(f"doc.md not found: {doc_path}")

    if clean and output_dir.exists():
        shutil.rmtree(output_dir)

    output_dir.mkdir(parents=True, exist_ok=True)

    doc_text = doc_path.read_text(encoding="utf-8", errors="replace")
    sections = parse_bundle_doc(doc_text)
    if not sections:
        raise ValueError("No bundled file sections were found. Expected headings like: ### [1] `relative/path`")

    exported: List[ExportedFile] = []
    warnings: List[str] = []

    for section in sections:
        try:
            rel_path = _normalise_relative_path(section.relative_path)
        except ValueError as exc:
            warning = f"{section.relative_path}: skipped unsafe path ({exc})"
            warnings.append(warning)
            exported.append(
                ExportedFile(
                    index=section.index,
                    relative_path=section.relative_path,
                    file_type=section.file_type,
                    target_path=output_dir,
                    status="skipped",
                    warning=warning,
                )
            )
            continue

        target_path = output_dir / rel_path

        if section.file_type == "png" or target_path.suffix.lower() == ".png":
            copied, warning = _try_copy_png(section, doc_path, target_path, asset_root)
            status = "copied" if copied else "missing-binary"
            if warning:
                warnings.append(warning)
            exported.append(
                ExportedFile(
                    index=section.index,
                    relative_path=section.relative_path,
                    file_type="png",
                    target_path=target_path,
                    status=status,
                    warning=warning,
                )
            )
            continue

        content, warning = _extract_text_content(section)
        target_path.parent.mkdir(parents=True, exist_ok=True)
        target_path.write_text(content, encoding="utf-8", newline="")

        if warning:
            warnings.append(warning)

        exported.append(
            ExportedFile(
                index=section.index,
                relative_path=section.relative_path,
                file_type=section.file_type,
                target_path=target_path,
                status="written",
                warning=warning,
            )
        )

    if strict and warnings:
        warning_text = "\n".join(f"- {w}" for w in warnings)
        raise RuntimeError(f"Export completed with warnings under --strict:\n{warning_text}")

    return exported


def _default_doc_path() -> Path:
    """
    bundle_documentation.py defaults to <root>/doc/doc.md, but users sometimes copy
    doc.md beside the exporter. Support both without requiring flags.
    """
    cwd = Path.cwd()
    default_bundle_path = cwd / "doc" / "doc.md"
    if default_bundle_path.is_file():
        return default_bundle_path
    return cwd / "doc.md"


def _print_report(exported: Sequence[ExportedFile], output_dir: Path) -> None:
    written = sum(1 for item in exported if item.status == "written")
    copied = sum(1 for item in exported if item.status == "copied")
    missing = sum(1 for item in exported if item.status == "missing-binary")
    skipped = sum(1 for item in exported if item.status == "skipped")
    warnings = [item.warning for item in exported if item.warning]

    print(f"Export directory: {output_dir.resolve()}")
    print(f"Text files written: {written}")
    print(f"Binary/assets copied: {copied}")
    print(f"Missing binary/assets: {missing}")
    print(f"Skipped unsafe paths: {skipped}")

    if warnings:
        print("\nWarnings:")
        for warning in warnings:
            print(f"- {warning}")


def build_arg_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        description=(
            "Rebuild the file tree exported by bundle_documentation.py from its doc.md file. "
            "Text files are restored from fenced code blocks. PNG files are copied from their "
            "linked source path or from --asset-root because PNG bytes are not embedded in doc.md."
        )
    )
    parser.add_argument(
        "--doc",
        type=Path,
        default=None,
        help="Path to the bundled Markdown file. Default: ./doc/doc.md if present, otherwise ./doc.md.",
    )
    parser.add_argument(
        "--out",
        type=Path,
        default=Path("doc_export"),
        help="Target export directory. Default: ./doc_export.",
    )
    parser.add_argument(
        "--asset-root",
        type=Path,
        default=None,
        help="Optional original project root used to copy PNG assets by their bundled relative paths.",
    )
    parser.add_argument(
        "--clean",
        action="store_true",
        help="Delete the target export directory before rebuilding it.",
    )
    parser.add_argument(
        "--strict",
        action="store_true",
        help="Return a non-zero exit code if any file cannot be fully recovered.",
    )
    return parser


def main(argv: Optional[Sequence[str]] = None) -> int:
    parser = build_arg_parser()
    args = parser.parse_args(argv)

    doc_path = args.doc if args.doc is not None else _default_doc_path()

    try:
        exported = export_doc_md(
            doc_path=doc_path,
            output_dir=args.out,
            asset_root=args.asset_root,
            clean=args.clean,
            strict=args.strict,
        )
    except Exception as exc:
        print(f"export-doc-md.py: error: {exc}", file=sys.stderr)
        return 1

    _print_report(exported, args.out)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())

```

<a id="file-5"></a>
### [5] `12.md`

- **Bytes:** `25376`
- **Type:** `text`

21738557678957308158741642336353824280332427554930462924011189994109105105399130047664661471458793302528156074488074768125325552059117162238556309527112834936402188626268571709336456942027967402833880797056936225132017725976586481814385289638616659987888585358981544385034633067849657986738201604757978062292371686684217177466600615729617645101051031690971779321891869993134927346734231792233347686484069782002834172768598042396958673057024399135260926405602215071853002342426851734483123545556826674122796095857548185081234803807946323363606417211054601458552928534493033118150115619161555693770241770550554499529737582114074182431380120714772840707852159395088269493664238748745057279742776132574469624384875689653304074128944534892250540907769326668500295952921861344379476626657472972460434308726160421244365335397697991399868234227796537523829637677472673791857345957699155594276736748554297537094263224276050288150034334304031603438181473074767607273403307773584325417137737701477971846397404610374556796495675702713103440740370821135337497894686402417831163286658604172754902404505248797999513552439537747437062838507349149574411692954309118305515784336212767794896245651850429003622360203311195922091308745264247426275664262903368105963447993872982404221129113611430590164096178677862328636765023730827518584894411268112192363775644648336736326326479630754040396646252127483728213670434324620680059698198565531856618854574849997490986236923049595190473493695005187298119535664213576804121208841824101804888938999642372559539985031810776912598576904119926445418173787501765032036514255215576012122651442924710742174620825852533362603916493693733437565515864571515256852947599965158993347461724161145290991367516751601939399361015051015476243808784998657546465004465440461390458547052497114606476940456260906729550091872493138826237398790698959157679058963489175666984749164078898161434695126103149071723157060029746609588021401401847155108865608161639019318658604769954311193690364864744215987464347352343141936023872305181698856134904018283044900995041154705664645787949986348462697467626808122222495750887730457969446897899444006679033913072706748099943675543189461141357225260171682472427925018950980984220501517226327607778798851466147373957921044143564455174320126079978097464623700636831069918558462276346545528188423171655991225299338055897976273677346076559344292753631730540690790372575915129667162451021403370829028435175282052783329387908742148806434204104584916324704159327692576628626966143320003654982288625267475848569846970807081753389615010585911564268587665475349755696498578806539863779352181616810326297502207343490753542635768604211845147056939493961296723130208253304056628652592001346954336454040797972493069703646326671026143247996691554289142216805305419070058131841159213030912196664626769201105184502933216712498716891145667825079785007777996562196855349701504610197431256464612630658920347877380480727937735839843304096956095076206968035589856410251937214337732402795582277883521197148138051136128015507723128684979645234073445263776002166561061497415663347442891874206885307785487965252964175848782263666349914384100348780316444724000991924583932314469658092479151937296568286263324765298414745146473940713648168231997229208733237018871681675333225983168201088300122761264779075664800830563657951847335951228125174279824737146539718373566040936565184725881675021710153236918334078594452738996957603176166100665500548935077043097401653393611129250985001816359841689497143448607973544672752864200343792809669074241377372336560894310727300730467809888052921237145057271017389271177141612339851232714688099298171914024446958230519724587779094170476012059666212074943871737982354029149673215711964737449814272272273983247924042083863279274154380509418733001015984358985741224927228265085884438288667739962013337901809725099065245906867431204375489020349595878759526270043987382718764837721117503622974080735805330852504947565558841642079011345530247173908920605609860234246013148102581541475767761601824537593754710386295058544114834014011324152608722172696035621674423313477587301824634441484357520805762693717037296883955604150388365434876873426153569994626042623600885140627353305644715990011607387165350063605104244023690105558808677540583445884211088421732073920268810173937898670637727684337721813682806507142230622788714185953629286276229330242619504014044379624503675971628399724575929820599897342285046886515922242765628549182509843863342298113797089900466021407084367296739894613560856064146419291321904937153737213570150699055762095664052833681980335402525406029470646329821469986018320406982087815763613021215625075500624970737387289607749988931318355748184824354836687340021190885606391690248862033093741664236950559092518449404094944200880466536190364505771030571473106580239128169809411543823085909945916334768114352907357042400263855642408474706572169208521291204510159028800098284419824550103703866758193930239827906219738600335670534586258940392205848771393952691396224637124204326030247206252653835900197978424578964601532223600661271604856650799938334940259746418654090194570423653276836579563833052677075844611111375727759499902531894590801161474915902584030891567762250316662578393742610902450167095629726552464640161400709101907489120317265710604125819667312675667083780623632242838986647098143658152092152419406922450462489892540002936717052922480888327808298523307178077994081014625678528415086267778466947777142080569691094775976156340169402382643718254828470004921141687541022747113123201664994077834182535137926562751205674470887552669357462501424471742277886556275408124285802338309882809232784245074060424414768443813634371188989234545147758364629197138643995297607176074770599162239991995116684051089222492220621777876901887021180221589645989056004524046798016261336363126191408197756259835238990339125512010305840667466160350938237456245635832920763483566597743214439042428972830283709088454118195354985719097596478635049280448787702028527928700021483384147291766642143539964858651313977216068128261765929808603165226198865647625305706970345876338093517518320590824871773707719139899297972308780480518685777968373623144207958662245666688590202817222008725569721723583736259846527707852715636785990617620106074636963992221028786416626417278058079039613189200722472921760890987780841648501099531669384264786416569470099299695756065470328562925314641166484408067926691338190226398462664641352558772454524700081493289101439355346755205057827175591852975023524063991704638404750132701489555843939090917905864088513227573234243165486961403870932041006024400710074544277139237828079144357544370660961502226810152731840622018455818804742324652493038943016020740720103829847847927447842125505601829423329435072683100087786087704158861536383590786858455795030823432378289107803090188694867642985320699721074095892042800084614415787372817386895059661302848173258105313779754912053839913361643268122160717738553580893027070930877154597957548513429979224943670471591611736431439156893966586257189591726800302369525873486217877552057462402722686672320548829570760371863624579563058476705334625815923487374263606084963300830542433575304840760246795489054506463011993504994183230939008561484792786116880453360000849231023731431422293164310182279848548629617401861450112876161517640793297436886634604891507377317809051075118201450552819858757299618304864199017461288769796272893729474165432616211635550879158378858187425985265862376943881900491177694333381919019560710594332306227007902422911796281609259338544740087186689590731413988938710866996486388699172434968604506839339593308383024608753629397430197493638672845557809973853293500374490230085282347798951067302002137301744453473767922653884187277050887698385726461540006770309760905114500562877035626973614638479565981887013013114515603403864094170303871137916659506184429534684803059162683529279867202653734831667737273885963871675085701814136658409723617385958885393375630614966896177872376238412777020726983597379542504818414707989196136247872422957353935256228721857329640946630438182620085804642994239128649538594354919348076270262538944319461627741356378515337820986103194078492045231199497606711150257102219353292836955038703146067099387801927110939099023295552463662013857796074017897704117556002766400496617484255567661256442883321712442149841028876169025581854196119402305272931228533901269091444837046250202010128122201622444857025615541099588979793903185309290487761635509729801008616268027853652988802666362046340070208428256739302070004701585729913692040283161537765888029195986447038982229354030350901129545928453827858356150478866056301283839622254193264759969222756151108290370774433326893982888818447309872524175829885836928973102836229041936248447112584429146384052433010201141686355166749188239976805181310992918252209217660281930313197812078944243514724448037070014910969905376733402689738944926987751781585532480646020941494342802946468865243238189802417576833581173963742081504497352473523557085293468377567788608246477865717280236802636879884687017896793377324577679445966568481034783188083822993192695745271446633023970665039889223914537716207722056464517620347701038584495606495504681863397037978197037130968122344969698594687372917121711245616811542811858410126766429732695859096599289736363286758035119623511203849102799185231821052164885855130782249845659914838048578055641105975082019681849642573763325571972748430236241101112059429040944990156633734042751706974949845071378744972012160695150591827275066392047305391940130425210836998485158217014949231471249483271216535911842597091341068163515024165583480763843237744611619141203470818656847561935110925095592616782022145173904503709657429376934184737522528799121336638513912599683969426703638798566042465044635867410103703734066819043490108557304499803589012785029343228115806187104895090298362469697921922328963012776108911235199832729287898595172909705496987295996690242525112639968120692003912586537107422498308844045534291212342813852430716732786154626120193516120132872901877972627708596323515387441695389351089469778661603665837033972199970004234123674921120884513960185594377423890660638266926477927416191273078300095785895173850252595417254787220009104870794010896673279085324554354587309420214740443327296578434144258130702682188456439344124227553636507190100578207910365232823489232978140363519400212479759763877178300118024431413198365981611647062157975634523058428713352562358479161524397639068827922592639503162319154630754979908060008005730601790371785695305723569050315303946064655659991596672444035806529212860701537490133582427709209687534427886812902437353501479374662312936273390213398175743925470746720330309591301351522523134706641770202928387615477963987387279508226105336005754192931714797960535725442436856048855129123732244694870344061013873111527022492760449657931764844653357052112523879349312863896489806153245755429328601621892171143245663125529616452885610651989033590049932226934654681693297044860469880240589911308338390474223658460104029148135294197794427729665405049329294679093485359175994102163647303967419633266066415575721010653522703239697009538079054718777014463703184068311553637541978100222847258080655333795190597126884037898373306347466102624681085929045082086006850059927717864242531920311077248146495032693348041317220155888994575930127329087008561050626979059963991680197422723795993823233208866675929487820384794631551504273023390829612928681315391378597286689898684131977596390983024023242426808720724260322432143297391140887156368999533416558592651641800433328483854470009029401490024409718262303910636894293991698969039494001157430280805490127486929861131654040383067440012941339312008267854741516201444287790022987130999363501785060704244342919458658608834459392083462128399605354858323603144610555421931358753377246279537110349536667328368033727028135889709125295466175737018980820067088036667554056814639411592599449011610200090327870228615750773781835311948011070794948185907116102765322951236791511729997110806081844673657911587868791889466308869252293341708538784799215786919165638823446400527471847102717458438871547690462392257615101602363608008023541880457458407265493132815585599946710119960183649421096700519967452152030380939845964053382084193214569075111065715106877767195817543222323089722578776171853750738739667259732216617791450127961697457600363887004543012351782042313461705470572420624335013671100069995510203698966630968244254675557363638992775681000670772907933067211696878690699829992264055279293701097164457427375405614724226544013073054912832907446841841020988416611533441467033536279030851923764795124982178600596560161476786196584707246491091361311899689585252263474522422658441973818617330963803290035213509579294385055264581808790076712264443046865786849027304924353407764737634323639377115271601993886211022211444229042430465035535788625682553822656657405216703930712110695776744740536040759808300734143196199139619363499552206049353289423159262894531161764033029931083200791657211688902951547625148119568259334736755546791352908164666043799476663977214072433445884674172095227618463195528911573671369992514025006831936428689486746159512081714423406256357637813484223302406572698182920319465966863661195020431768479210774337611652186684708564814984270069787130984359326714735988344220081571691302751524616913335579648820073420506205063050680852128808735269958939420385372798590431970117160233213830788243452443617963674125235650479225099888512690895491827241822925271115819111017631100541251397650786393037395799908902580844383689153255691279999039032216334807612869276692721059486369212348668878044707724666351557554956984025929946192909160925205847250261901590764578349188295523553356914999651364919278621370568269476108725406272448795806466161702029623376155752428205768994879849326579275698210551772819534621318983464003540946378743668335817036821247412254326433534674212074039634474446308805023621286424919537296768874226127075281887995134371402509789983143628287207003093361299266178108175275684330548006089435313630182134464460261274577028683137502410528344049429432638780914458125277027563211645993248743565429597700952959168666725636749045326222219567568808764529926159403493188741643252916878510403857066354264843209635243131211532297877256448034479826448357371609030166535793341933732442290536524874860273308359552145338280624986607526704904854411060764459814028329648381539926461044246824563147031616732016743136068446387814674465596267515073811622220203920555275509658911481353380664118847662295795856379842923420854243364202852916297590255878021870549755732701124198976524212219972440587942989331692328949755339021610294787682254038405317615908077863631449595444335242705076902808714461694930105766495171939309573687622874648491968518967175042050550014617605155396225215916223530033356222629902053244829176291150028862498089937110738830121812078880905996710247255609297697362528345304922622501680362662050573202297695043611039547981188508256443067461164986937220648293824911274904021925880091147686850196501560380285423891916899304514830418851982155863055844735006979430235376359514083912508415291096230596107632988762425440779677532675381716184855537768176934842415959957683374298343232766365718962833961658388141456392139184301077401692624356017408941594493965836787217321831270447073287648483320606525113931192205212682102230410654804282553872179904402482074228416173365655467808494987302481494183664967141289166472896395519613817698881006058555576551074735437444930889445481621723617997902286918137848927448644656590907873811915091017181460563288813614274055761445865300627824314743218754698324913987143613555489838432945171478839692934100669063509960062759623068697626104744944367650515681590523960806580652029596120874437918526114146770081647890486699909675322489273953883492394827872303246914586941914635485780848136289559090575313748946089122071828440082116336683465966207581537895312702395745073126645996547304296924423402253184010603477076222621580283654593501321400511749094962132272966495933458778733891029534092820056339359138750724034015588179221359111059806767993479322281871394819174015556531117495518914365782295455919767342329196326111185092553280471346533335439181724400865690211918050763549532022350816659176482526586996292443340787097977793785265939776074518287192576996572546113574447259918376810673445385365344816429911921404231597859175769163907381166608307429481788859468024060171995767900655393784619544102270554461018596982916600242601055517209350425167779996719766028069268876093268785586289697867614216368442780643686668787770553333453997242948720993911481377052731250119374969819449754783901019742088735008967471903300588797258254670722264428424785606865805928081466300552309326168704009168631312461135546043874932517152459909474672518245494071070079114478117826431600093900664182383823847937978269931363996027479425749761845785292138067268833666331492117097157414246242715910564575474295034029524080529051714553350961314730001763694729329068680666405607313498980485689965228696292936289772866319285809486095533922204963988386894356894340014252847298112522495889128735438244879837729564292618660743348547531042807500940824242995019029472244806995301666193082113964234743407831996126178813605990402041896212979870639623063267843987816421044533338252784640543525392457158345339084623572540891178202335327362799766063195295188701432670196475027350557358522393655731300469624328755189287113889678691304768654809898717488301637181461656966031184071683459532884128896277914200716319546270269379304282737422970803882848206147971297042454428733728556464127379112682358321571655275801008924671346984950852343342187213216692249593546120334016043298380397247921462539496661623523892988579289402165184130512802207892840923234841534974398577917680454699434478224843671195583400384354042754775587307332352261984231012195714765165636707857482806194947322269737601534578814837853578699988452609685340133522285134152432595118613231733796149263637832333325606471064393717624573279887918492415595488843675183514171294511672484024675799712248127243687039597811817776626781637657118102974300850429703188184389515626488105969856944869872980945832147756685672511399650892165964096652610617182863181974106117630504397551429192988602623250116133312124944204775805011663625184174120895793953316551068839990123327263866877127862911247685866416790332327844988245734509506476136269335560874708746708331724999645059372684290523679295961189138143925748466733732984030254295887438023260226059583560293828558681155453734784603964178954105651822547232326732336525352924798341859162970887498844766439643264428284908273538990945746698978604968318477039637351576520331632459325704119697849029384891909080228260414529010149619726311220633142857984141452655140268909938646926431483980032755408401728502879990153163002676024279475315271178989674655437298707304897548539323474650825960868704269788214032280316839127280132001150270860264468166084824080453141831312260634591216813598924864521709214097448288668912652066000224284235192776804597782485842558132932799136596639567373858376012193104547124002934256730462559528535811786147466834804070846146478994468318948927507106298479179833631966826286339068049947345371752463850908997765517991888742367423897065581756199352376716002411966788372693710530205845920279652131877025185074710605220290448395026504730466513775174966587590021404846894243611278281098437796309363519858105973790256300667345712316351034993651247200192142124566978636194662213503059949887481299067527153714899443947185458706135320991144840466672482198267454513817983662397038696846167857502526794501546307874108355320017887290359490128582852836111195412164646897882804154797625245290317419728421685550334452381025464166123389820881003529197949694280834002392441641742923620004113029675154546526994267343709174779075016710083503502034465701854299835680782537614065933132655586649252723817130324585288281492226097985117942023396848707889266831422148880240857673440282649611112649073935929893980560095010203965840987528554049873462704426192883720605581020685060076755280348572396472026772624873189153027689041293269290204806599324655663069125763046885146170726903532596578913459800083514340750813750797603448378027724637529271111549521952381709879721643466508693420761784998017001533606180974534276948351693059411334044228923124943147145282636127050514217812429973934865642651357328562462037912600342732436656796780100293977404950459476903382248319259535411896602105725585098335254153670000196708874433639437700337656555417491678906240483279201934861952502296499873807093265381118908328568809754818940887476104983352354218521762312139357981

The Elements

Dominic Alexander Cooper
```
000    I exist
001    I feel pain
002    My pain can be increased by incompetence
003    My pain can be decreased by competence
004    I can learn
005    Learning increases competence
006    Contradiction increases incompetence
007    Proof by contradiction does not enable learning
008    Proof by error-free construction is learning
009    Proof by contradiction is not learning
010    Your level of competence is quantifiable
011    Your level of incompetence is quantifiable
012    Context changes your current level of competence
013    The basis span of your competence can be optimized for all contexts
014    Everyone is innocent
015    Competence is a functional axial n-dimensional space
016    Incompetence is not a functional axial n-dimensional space

017    Since I exist, I can be treated as a living information-bearing system
018    Since I feel pain, pain is a valid system signal
019    Since incompetence can increase pain, incompetence is a negative system state
020    Since competence can decrease pain, competence is a positive system state
021    Since I can learn, my system state can be updated
022    Since learning increases competence, learning is a valid update operation
023    Since contradiction increases incompetence, contradiction is a corruption signal
024    Since proof by error-free construction is learning, construction is the preferred proof engine
025    Since proof by contradiction is not learning, contradiction is not a preferred learning engine
026    Since competence is quantifiable, competence can be measured
027    Since incompetence is quantifiable, incompetence can be measured
028    Since context changes competence, competence is context-indexed
029    Since competence can be optimized for all contexts, competence has a basis-span
030    Since everyone is innocent, correction must not be treated as guilt
031    Since competence is functional axial n-dimensional space, competence has dimensions
032    Since incompetence is not functional axial n-dimensional space, incompetence lacks stable constructive basis

033    The mind can ingest information through perception, language, memory, and action
034    The mind can encode information as concepts, habits, images, words, and procedures
035    The mind can index information by association, label, context, emotion, and purpose
036    The mind can query stored information through attention, recall, questioning, and tool-use
037    The mind can update stored information through learning
038    The mind can validate stored information through error-free construction
039    The mind can reject or quarantine contradictory information when it increases incompetence
040    The mind can externalize information into tools, notes, diagrams, calendars, code, and environment
041    The human environment can serve as auxiliary storage
042    Human-made tools can serve as auxiliary memory, index, query, and validation systems
043    The body can store competence as trained procedure
044    Repetition can strengthen retrieval pathways when it increases competence
045    Reflection can detect incompetence when pain, error, or contradiction appears
046    Deliberate practice can transform incompetence into competence
047    A context can be treated as a coordinate region inside competence-space
048    A skill can be treated as an axis inside competence-space
049    A habit can be treated as a repeated function over time
050    A tool can be treated as an external operator that extends mental function
051    A note can be treated as persistent external memory
052    A diagram can be treated as spatial indexing of thought
053    A calendar can be treated as temporal indexing of intention
054    A checklist can be treated as procedural compression
055    A question can be treated as a query operation
056    An answer can be treated as a retrieved or constructed record
057    A mistake can be treated as a detected incompetence marker
058    Correction can be treated as a competence-restoring update
059    Optimization can be treated as basis-span improvement across contexts
060    The mental human-centric database is the union of mind, body, environment, and tools
061    The purpose of the mental human-centric database is to reduce pain by increasing competence
062    The governing rule of the database is to learn by error-free construction
063    The final enforced target is complete competence under {014 + 015}
064    Once true, always, true
065    What is not true, cannot be done
```

<a id="file-6"></a>
### [6] `2.c`

- **Bytes:** `2128`
- **Type:** `text`

```text
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <limits.h>
#define x0 10
static const char x1[] = {
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9'
};
static const uint64_t x2 = sizeof(x1) / sizeof(x1[0]);
typedef struct {
    void* x4;
    bool (*x5)(void* x4, const char* x6, uint64_t x7);
    bool (*x8)(void* x4, char x9);
} x3;
static bool x10(uint64_t x11, uint64_t x12, uint64_t* x13) {
    *x13 = 1;
    for (uint64_t x14 = 0; x14 < x12; ++x14) {
        if (*x13 > UINT64_MAX / x11) {
            return false;
        }
        *x13 *= x11;
    }
    return true;
}
static bool x15(uint64_t x16, x3* x17) {
    uint64_t x18;
    if (!x10(x2, x16, &x18)) {
        return false;
    }
    char x19[x0];
    for (uint64_t x14 = 0; x14 < x18; ++x14) {
        uint64_t x20 = x14;
        for (int64_t x21 = x16 - 1; x21 >= 0; --x21) {
            uint64_t x22 = x20 % x2;
            x19[x21] = x1[x22];
            x20 /= x2;
        }
        if (!x17->x5(x17->x4, x19, x16)) {
            return false;
        }
        if (!x17->x8(x17->x4, '\n')) {
            return false;
        }
    }
    return true;
}
static bool x23(void* x4, const char* x6, uint64_t x7) {
    FILE* x24 = (FILE*)x4;
    return fwrite(x6, sizeof(char), x7, x24) == x7;
}
static bool x25(void* x4, char x9) {
    FILE* x24 = (FILE*)x4;
    return fputc(x9, x24) != EOF;
}
static bool x26(x3* x17, const char* x27) {
    FILE* x24 = fopen(x27, "w");
    if (x24 == NULL) {
        return false;
    }
    *x17 = (x3){
        .x4 = x24,
        .x5 = x23,
        .x8 = x25
    };
    return true;
}
static void x28(x3* x17) {
    if (x17 && x17->x4) {
        fclose((FILE*)x17->x4);
        x17->x4 = NULL;
    }
}
int main(void) {
    const uint64_t x30 = 7;
    if (x30 == 0 || x30 > x0) {
        return 1;
    }
    x3 x31;
    if (!x26(&x31, "x33.txt")) {
        perror("Error opening file");
        return 1;
    }
    bool x32 = x15(x30, &x31);
    x28(&x31);
    if (!x32) {
        return 1;
    }
    return 0;
}
```

<a id="file-7"></a>
### [7] `3.c`

- **Bytes:** `25230`
- **Type:** `text`

```text
/*
 * configurator_v2.c
 *
 * A user-specifiable C generative intelligence fabric.
 *
 * Internal C identifiers intentionally preserve the xN object naming style used
 * by configurator.c. Runtime object names, input names, flow names, and output
 * names are user-specified through CLI flags, config files, or the micro UI.
 *
 * Build:
 *     cc -std=c11 -Wall -Wextra -pedantic -O2 configurator_v2.c -o configurator_v2
 *
 * Examples:
 *     ./configurator_v2 --object-name x33 --input-name x1 --input-value 01 \
 *         --input-width 3 --flow-name x15 --flow-type cartesian \
 *         --output-name x31 --output-target x33.txt
 *
 *     ./configurator_v2 --sample-config x33.fabric
 *     ./configurator_v2 --config x33.fabric
 *     ./configurator_v2 --ui
 *
 * Config format:
 *     object_name=x33
 *     input_name=x1
 *     input_value=0123456789
 *     input_width=7
 *     flow_name=x15
 *     flow_type=cartesian
 *     output_name=x31
 *     output_target=x33.txt
 *     separator=\n
 *     end
 *
 * Supported flow_type values:
 *     cartesian  - generate every fixed-width combination from input_value
 *     literal    - write input_value once
 *     repeat     - write input_value input_width times
 *     reverse    - write input_value reversed once
 */

#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <limits.h>
#include <string.h>
#include <stdlib.h>
#include <ctype.h>

#define x0 64
#define x1 512
#define x2 32
#define x200 4096

typedef struct {
    void* x4;
    bool (*x5)(void* x4, const char* x6, uint64_t x7);
    bool (*x8)(void* x4, char x9);
    bool x10;
} x3;

typedef enum {
    x11 = 0,
    x12 = 1,
    x13 = 2,
    x14 = 3
} x15;

typedef struct {
    bool x16;
    char x17[x0];
    char x18[x0];
    char x19[x1];
    uint64_t x20;
    char x21[x0];
    x15 x22;
    char x23[x0];
    char x24[x1];
    char x25[x0];
    char x26[x1];
    char x27[x1];
} x28;

typedef struct {
    x28 x29[x2];
    uint64_t x30;
} x31;

static bool x32(char* x33, uint64_t x34, const char* x35) {
    size_t x36;

    if (x33 == NULL || x35 == NULL || x34 == 0) {
        return false;
    }

    x36 = strlen(x35);
    if (x36 >= x34) {
        return false;
    }

    memcpy(x33, x35, x36 + 1);
    return true;
}

static int x37(int x38) {
    return tolower((unsigned char)x38);
}

static bool x39(const char* x40, const char* x41) {
    if (x40 == NULL || x41 == NULL) {
        return false;
    }

    while (*x40 && *x41) {
        if (x37(*x40) != x37(*x41)) {
            return false;
        }
        ++x40;
        ++x41;
    }

    return *x40 == '\0' && *x41 == '\0';
}

static char* x42(char* x43) {
    char* x44;
    size_t x45;

    if (x43 == NULL) {
        return NULL;
    }

    while (*x43 && isspace((unsigned char)*x43)) {
        ++x43;
    }

    x45 = strlen(x43);
    while (x45 > 0 && isspace((unsigned char)x43[x45 - 1])) {
        x43[x45 - 1] = '\0';
        --x45;
    }

    x44 = strchr(x43, '#');
    if (x44 != NULL) {
        *x44 = '\0';
        x45 = strlen(x43);
        while (x45 > 0 && isspace((unsigned char)x43[x45 - 1])) {
            x43[x45 - 1] = '\0';
            --x45;
        }
    }

    return x43;
}

static bool x46(char* x47, char** x48, char** x49) {
    char* x50;

    if (x47 == NULL || x48 == NULL || x49 == NULL) {
        return false;
    }

    x50 = strchr(x47, '=');
    if (x50 != NULL) {
        *x50 = '\0';
        *x48 = x42(x47);
        *x49 = x42(x50 + 1);
        return **x48 != '\0';
    }

    x50 = x47;
    while (*x50 && !isspace((unsigned char)*x50)) {
        ++x50;
    }

    if (*x50 == '\0') {
        *x48 = x42(x47);
        *x49 = NULL;
        return **x48 != '\0';
    }

    *x50 = '\0';
    ++x50;
    *x48 = x42(x47);
    *x49 = x42(x50);

    return **x48 != '\0';
}

static bool x51(const char* x52, uint64_t* x53) {
    char* x54;
    unsigned long long x55;

    if (x52 == NULL || x53 == NULL || *x52 == '\0') {
        return false;
    }

    while (*x52 && isspace((unsigned char)*x52)) {
        ++x52;
    }

    if (*x52 == '-') {
        return false;
    }

    x55 = strtoull(x52, &x54, 10);
    if (x54 == x52 || *x42(x54) != '\0') {
        return false;
    }

    if (x55 > UINT64_MAX) {
        return false;
    }

    *x53 = (uint64_t)x55;
    return true;
}

static bool x56(char* x57, uint64_t x58, const char* x59) {
    uint64_t x60 = 0;

    if (x57 == NULL || x59 == NULL || x58 == 0) {
        return false;
    }

    while (*x59) {
        char x61 = *x59++;

        if (x61 == '\\') {
            x61 = *x59++;
            if (x61 == '\0') {
                x61 = '\\';
                --x59;
            } else if (x61 == 'n') {
                x61 = '\n';
            } else if (x61 == 't') {
                x61 = '\t';
            } else if (x61 == 'r') {
                x61 = '\r';
            } else if (x61 == '\\') {
                x61 = '\\';
            }
        }

        if (x60 + 1 >= x58) {
            return false;
        }

        x57[x60++] = x61;
    }

    x57[x60] = '\0';
    return true;
}

static void x62(x28* x63) {
    if (x63 == NULL) {
        return;
    }

    memset(x63, 0, sizeof(*x63));
    x63->x16 = true;
    (void)x32(x63->x17, sizeof(x63->x17), "x33");
    (void)x32(x63->x18, sizeof(x63->x18), "x1");
    (void)x32(x63->x19, sizeof(x63->x19), "0123456789");
    x63->x20 = 7;
    (void)x32(x63->x21, sizeof(x63->x21), "x15");
    x63->x22 = x11;
    (void)x32(x63->x23, sizeof(x63->x23), "x31");
    (void)x32(x63->x24, sizeof(x63->x24), "x33.txt");
    (void)x32(x63->x25, sizeof(x63->x25), "\n");
    (void)x32(x63->x26, sizeof(x63->x26), "");
    (void)x32(x63->x27, sizeof(x63->x27), "");
}

static void x64(x31* x65) {
    if (x65 != NULL) {
        memset(x65, 0, sizeof(*x65));
    }
}

static bool x66(x31* x67, const x28* x68) {
    if (x67 == NULL || x68 == NULL || x67->x30 >= x2) {
        return false;
    }

    x67->x29[x67->x30++] = *x68;
    return true;
}

static bool x69(const char* x70, x15* x71) {
    if (x70 == NULL || x71 == NULL) {
        return false;
    }

    if (x39(x70, "cartesian") || x39(x70, "product") || x39(x70, "combinations")) {
        *x71 = x11;
        return true;
    }

    if (x39(x70, "literal") || x39(x70, "echo")) {
        *x71 = x12;
        return true;
    }

    if (x39(x70, "repeat")) {
        *x71 = x13;
        return true;
    }

    if (x39(x70, "reverse")) {
        *x71 = x14;
        return true;
    }

    return false;
}

static bool x72(void* x4, const char* x6, uint64_t x7) {
    FILE* x73 = (FILE*)x4;
    return fwrite(x6, sizeof(char), (size_t)x7, x73) == x7;
}

static bool x74(void* x4, char x9) {
    FILE* x73 = (FILE*)x4;
    return fputc((unsigned char)x9, x73) != EOF;
}

static bool x75(x3* x76, const char* x77) {
    FILE* x73;

    if (x76 == NULL || x77 == NULL || *x77 == '\0') {
        return false;
    }

    if (x39(x77, "stdout") || x39(x77, "-")) {
        *x76 = (x3){
            .x4 = stdout,
            .x5 = x72,
            .x8 = x74,
            .x10 = false
        };
        return true;
    }

    x73 = fopen(x77, "w");
    if (x73 == NULL) {
        return false;
    }

    *x76 = (x3){
        .x4 = x73,
        .x5 = x72,
        .x8 = x74,
        .x10 = true
    };

    return true;
}

static void x78(x3* x79) {
    if (x79 != NULL && x79->x4 != NULL && x79->x10) {
        fclose((FILE*)x79->x4);
        x79->x4 = NULL;
        x79->x10 = false;
    }
}

static bool x80(x3* x81, const char* x82) {
    uint64_t x83;

    if (x81 == NULL || x82 == NULL) {
        return false;
    }

    x83 = (uint64_t)strlen(x82);
    if (x83 == 0) {
        return true;
    }

    return x81->x5(x81->x4, x82, x83);
}

static bool x84(uint64_t x85, uint64_t x86, uint64_t* x87) {
    uint64_t x88;

    if (x87 == NULL || x85 == 0) {
        return false;
    }

    *x87 = 1;
    for (x88 = 0; x88 < x86; ++x88) {
        if (*x87 > UINT64_MAX / x85) {
            return false;
        }
        *x87 *= x85;
    }

    return true;
}

static bool x89(const x28* x90) {
    uint64_t x91;

    if (x90 == NULL) {
        return false;
    }

    if (x90->x17[0] == '\0' || x90->x18[0] == '\0' ||
        x90->x21[0] == '\0' || x90->x23[0] == '\0' ||
        x90->x24[0] == '\0') {
        return false;
    }

    if (x90->x20 == 0) {
        return false;
    }

    if (x90->x22 == x11) {
        if (x90->x20 > x0) {
            return false;
        }
        if (strlen(x90->x19) == 0) {
            return false;
        }
        if (!x84((uint64_t)strlen(x90->x19), x90->x20, &x91)) {
            return false;
        }
    }

    return true;
}

static bool x92(x3* x93, const x28* x94, const char* x95, uint64_t x96) {
    if (x93 == NULL || x94 == NULL || x95 == NULL) {
        return false;
    }

    if (!x80(x93, x94->x26)) {
        return false;
    }

    if (x96 > 0 && !x93->x5(x93->x4, x95, x96)) {
        return false;
    }

    if (!x80(x93, x94->x27)) {
        return false;
    }

    if (!x80(x93, x94->x25)) {
        return false;
    }

    return true;
}

static bool x97(const x28* x98, x3* x99) {
    uint64_t x100;
    uint64_t x101;
    uint64_t x102;
    char x103[x0 + 1];

    if (x98 == NULL || x99 == NULL) {
        return false;
    }

    x100 = (uint64_t)strlen(x98->x19);
    if (!x84(x100, x98->x20, &x101)) {
        return false;
    }

    for (x102 = 0; x102 < x101; ++x102) {
        uint64_t x104 = x102;
        int64_t x105;

        for (x105 = (int64_t)x98->x20 - 1; x105 >= 0; --x105) {
            uint64_t x106 = x104 % x100;
            x103[x105] = x98->x19[x106];
            x104 /= x100;
        }

        x103[x98->x20] = '\0';

        if (!x92(x99, x98, x103, x98->x20)) {
            return false;
        }
    }

    return true;
}

static bool x107(const x28* x108, x3* x109) {
    if (x108 == NULL || x109 == NULL) {
        return false;
    }

    return x92(x109, x108, x108->x19, (uint64_t)strlen(x108->x19));
}

static bool x110(const x28* x111, x3* x112) {
    uint64_t x113;

    if (x111 == NULL || x112 == NULL) {
        return false;
    }

    for (x113 = 0; x113 < x111->x20; ++x113) {
        if (!x92(x112, x111, x111->x19, (uint64_t)strlen(x111->x19))) {
            return false;
        }
    }

    return true;
}

static bool x114(const x28* x115, x3* x116) {
    size_t x117;
    char x118[x1];
    size_t x119;

    if (x115 == NULL || x116 == NULL) {
        return false;
    }

    x117 = strlen(x115->x19);
    if (x117 >= sizeof(x118)) {
        return false;
    }

    for (x119 = 0; x119 < x117; ++x119) {
        x118[x119] = x115->x19[x117 - 1 - x119];
    }
    x118[x117] = '\0';

    return x92(x116, x115, x118, (uint64_t)x117);
}

static bool x120(const x28* x121) {
    x3 x122;
    bool x123;

    if (!x89(x121)) {
        fprintf(stderr, "Invalid object specification: %s\n", x121 ? x121->x17 : "(null)");
        return false;
    }

    if (!x75(&x122, x121->x24)) {
        perror("Error opening output target");
        return false;
    }

    if (x121->x22 == x11) {
        x123 = x97(x121, &x122);
    } else if (x121->x22 == x12) {
        x123 = x107(x121, &x122);
    } else if (x121->x22 == x13) {
        x123 = x110(x121, &x122);
    } else if (x121->x22 == x14) {
        x123 = x114(x121, &x122);
    } else {
        x123 = false;
    }

    x78(&x122);
    return x123;
}

static bool x124(const x31* x125) {
    uint64_t x126;

    if (x125 == NULL || x125->x30 == 0) {
        return false;
    }

    for (x126 = 0; x126 < x125->x30; ++x126) {
        if (!x120(&x125->x29[x126])) {
            return false;
        }
    }

    return true;
}

static bool x127(x28* x128, const char* x129, const char* x130) {
    uint64_t x131;
    x15 x132;

    if (x128 == NULL || x129 == NULL || x130 == NULL) {
        return false;
    }

    if (x39(x129, "object") || x39(x129, "object_name") || x39(x129, "name")) {
        return x32(x128->x17, sizeof(x128->x17), x130);
    }

    if (x39(x129, "input_name")) {
        return x32(x128->x18, sizeof(x128->x18), x130);
    }

    if (x39(x129, "input") || x39(x129, "input_value") || x39(x129, "alphabet")) {
        return x56(x128->x19, sizeof(x128->x19), x130);
    }

    if (x39(x129, "input_width") || x39(x129, "width") ||
        x39(x129, "length") || x39(x129, "depth")) {
        if (!x51(x130, &x131)) {
            return false;
        }
        x128->x20 = x131;
        return true;
    }

    if (x39(x129, "flow_name")) {
        return x32(x128->x21, sizeof(x128->x21), x130);
    }

    if (x39(x129, "flow") || x39(x129, "flow_type") || x39(x129, "processing_flow")) {
        if (!x69(x130, &x132)) {
            return false;
        }
        x128->x22 = x132;
        return true;
    }

    if (x39(x129, "output_name")) {
        return x32(x128->x23, sizeof(x128->x23), x130);
    }

    if (x39(x129, "output") || x39(x129, "output_target") || x39(x129, "file")) {
        return x32(x128->x24, sizeof(x128->x24), x130);
    }

    if (x39(x129, "separator") || x39(x129, "sep")) {
        return x56(x128->x25, sizeof(x128->x25), x130);
    }

    if (x39(x129, "prefix")) {
        return x56(x128->x26, sizeof(x128->x26), x130);
    }

    if (x39(x129, "suffix")) {
        return x56(x128->x27, sizeof(x128->x27), x130);
    }

    return false;
}

static bool x133(const char* x134, x31* x135) {
    FILE* x136;
    char x137[x200];
    uint64_t x138 = 0;
    x28 x139;
    bool x140 = false;

    if (x134 == NULL || x135 == NULL) {
        return false;
    }

    x136 = fopen(x134, "r");
    if (x136 == NULL) {
        return false;
    }

    x64(x135);
    x62(&x139);

    while (fgets(x137, sizeof(x137), x136) != NULL) {
        char* x141;
        char* x142;
        char* x143;

        ++x138;
        x141 = x42(x137);

        if (*x141 == '\0') {
            continue;
        }

        if (x39(x141, "end")) {
            if (!x66(x135, &x139)) {
                fclose(x136);
                return false;
            }
            x62(&x139);
            x140 = false;
            continue;
        }

        if (!x46(x141, &x142, &x143) || x143 == NULL) {
            fprintf(stderr, "Invalid config line %llu\n", (unsigned long long)x138);
            fclose(x136);
            return false;
        }

        if (x39(x142, "object") || x39(x142, "object_name")) {
            if (x140) {
                if (!x66(x135, &x139)) {
                    fclose(x136);
                    return false;
                }
                x62(&x139);
            }
            x140 = true;
        } else {
            x140 = true;
        }

        if (!x127(&x139, x142, x143)) {
            fprintf(stderr, "Invalid config key or value on line %llu: %s\n",
                    (unsigned long long)x138, x142);
            fclose(x136);
            return false;
        }
    }

    if (x140) {
        if (!x66(x135, &x139)) {
            fclose(x136);
            return false;
        }
    }

    fclose(x136);
    return x135->x30 > 0;
}

static bool x144(const char* x145, char* x146, uint64_t x147, const char* x148) {
    char x149[x200];
    size_t x150;

    if (x145 == NULL || x146 == NULL || x147 == 0 || x148 == NULL) {
        return false;
    }

    printf("%s [%s]: ", x145, x148);
    fflush(stdout);

    if (fgets(x149, sizeof(x149), stdin) == NULL) {
        return false;
    }

    x150 = strlen(x149);
    if (x150 > 0 && x149[x150 - 1] == '\n') {
        x149[x150 - 1] = '\0';
    }

    if (x149[0] == '\0') {
        return x32(x146, x147, x148);
    }

    return x32(x146, x147, x149);
}

static bool x151(const char* x152, uint64_t* x153, uint64_t x154) {
    char x155[x0];
    char x156[x0];

    if (x152 == NULL || x153 == NULL) {
        return false;
    }

    snprintf(x156, sizeof(x156), "%llu", (unsigned long long)x154);

    if (!x144(x152, x155, sizeof(x155), x156)) {
        return false;
    }

    return x51(x155, x153);
}

static bool x157(x31* x158) {
    x28 x159;
    char x160[x0];
    bool x161 = true;

    if (x158 == NULL) {
        return false;
    }

    x64(x158);

    while (x161 && x158->x30 < x2) {
        x62(&x159);

        if (!x144("object_name", x159.x17, sizeof(x159.x17), x159.x17)) {
            return false;
        }
        if (!x144("input_name", x159.x18, sizeof(x159.x18), x159.x18)) {
            return false;
        }
        if (!x144("input_value", x159.x19, sizeof(x159.x19), x159.x19)) {
            return false;
        }
        if (!x151("input_width", &x159.x20, x159.x20)) {
            return false;
        }
        if (!x144("flow_name", x159.x21, sizeof(x159.x21), x159.x21)) {
            return false;
        }
        if (!x144("flow_type cartesian|literal|repeat|reverse", x160, sizeof(x160), "cartesian")) {
            return false;
        }
        if (!x69(x160, &x159.x22)) {
            fprintf(stderr, "Unknown flow_type: %s\n", x160);
            return false;
        }
        if (!x144("output_name", x159.x23, sizeof(x159.x23), x159.x23)) {
            return false;
        }
        if (!x144("output_target file|stdout|-", x159.x24, sizeof(x159.x24), x159.x24)) {
            return false;
        }
        if (!x144("separator", x160, sizeof(x160), "\\n")) {
            return false;
        }
        if (!x56(x159.x25, sizeof(x159.x25), x160)) {
            return false;
        }
        if (!x144("prefix", x160, sizeof(x160), "")) {
            return false;
        }
        if (!x56(x159.x26, sizeof(x159.x26), x160)) {
            return false;
        }
        if (!x144("suffix", x160, sizeof(x160), "")) {
            return false;
        }
        if (!x56(x159.x27, sizeof(x159.x27), x160)) {
            return false;
        }

        if (!x66(x158, &x159)) {
            return false;
        }

        if (!x144("add another object? y|n", x160, sizeof(x160), "n")) {
            return false;
        }

        x161 = x39(x160, "y") || x39(x160, "yes");
    }

    return x158->x30 > 0;
}

static bool x162(const char* x163) {
    FILE* x164;

    if (x163 == NULL) {
        return false;
    }

    x164 = fopen(x163, "w");
    if (x164 == NULL) {
        return false;
    }

    fprintf(x164,
            "# configurator_v2 fabric config\n"
            "object_name=x33\n"
            "input_name=x1\n"
            "input_value=0123456789\n"
            "input_width=3\n"
            "flow_name=x15\n"
            "flow_type=cartesian\n"
            "output_name=x31\n"
            "output_target=x33.txt\n"
            "separator=\\n\n"
            "prefix=\n"
            "suffix=\n"
            "end\n"
            "\n"
            "object_name=x34\n"
            "input_name=x35\n"
            "input_value=emerge\n"
            "input_width=1\n"
            "flow_name=x36\n"
            "flow_type=reverse\n"
            "output_name=x37\n"
            "output_target=stdout\n"
            "separator=\\n\n"
            "end\n");

    fclose(x164);
    return true;
}

static void x165(const char* x166) {
    fprintf(stderr,
            "Usage:\n"
            "  %s --ui\n"
            "  %s --config <path>\n"
            "  %s --sample-config <path>\n"
            "  %s [object flags]\n"
            "\n"
            "Object flags:\n"
            "  --object-name <name>       runtime object name\n"
            "  --input-name <name>        runtime input name\n"
            "  --input-value <value>      object input payload/alphabet\n"
            "  --input-width <n>          width/count/depth\n"
            "  --flow-name <name>         runtime processing-flow name\n"
            "  --flow-type <type>         cartesian|literal|repeat|reverse\n"
            "  --output-name <name>       runtime output name\n"
            "  --output-target <target>   file path, stdout, or -\n"
            "  --separator <text>         supports \\\\n, \\\\t, \\\\r, \\\\\\\\\n"
            "  --prefix <text>            supports escapes\n"
            "  --suffix <text>            supports escapes\n"
            "\n"
            "Aliases:\n"
            "  -n <name>  -i <value>  -w <n>  -f <type>  -o <target>\n",
            x166, x166, x166, x166);
}

static int x167(int x168, char** x169, x31* x170, bool* x171) {
    int x172;
    x28 x173;
    bool x174 = false;

    if (x170 == NULL || x171 == NULL) {
        return 1;
    }

    *x171 = false;
    x64(x170);

    if (x168 <= 1) {
        x165(x169[0]);
        return 0;
    }

    for (x172 = 1; x172 < x168; ++x172) {
        if (x39(x169[x172], "--help") || x39(x169[x172], "-h")) {
            x165(x169[0]);
            return 0;
        }

        if (x39(x169[x172], "--ui")) {
            if (!x157(x170)) {
                return 1;
            }
            *x171 = true;
            return 0;
        }

        if (x39(x169[x172], "--config")) {
            if (x172 + 1 >= x168) {
                x165(x169[0]);
                return 1;
            }
            if (!x133(x169[x172 + 1], x170)) {
                perror("Error loading config");
                return 1;
            }
            *x171 = true;
            return 0;
        }

        if (x39(x169[x172], "--sample-config")) {
            if (x172 + 1 >= x168) {
                x165(x169[0]);
                return 1;
            }
            if (!x162(x169[x172 + 1])) {
                perror("Error writing sample config");
                return 1;
            }
            *x171 = false;
            return 0;
        }
    }

    x62(&x173);

    for (x172 = 1; x172 < x168; ++x172) {
        const char* x175 = x169[x172];
        const char* x176 = NULL;

        if (x172 + 1 < x168) {
            x176 = x169[x172 + 1];
        }

        if (x176 == NULL) {
            x165(x169[0]);
            return 1;
        }

        if (x39(x175, "--object-name") || x39(x175, "-n")) {
            if (!x127(&x173, "object_name", x176)) {
                return 1;
            }
        } else if (x39(x175, "--input-name")) {
            if (!x127(&x173, "input_name", x176)) {
                return 1;
            }
        } else if (x39(x175, "--input-value") || x39(x175, "-i")) {
            if (!x127(&x173, "input_value", x176)) {
                return 1;
            }
        } else if (x39(x175, "--input-width") || x39(x175, "-w")) {
            if (!x127(&x173, "input_width", x176)) {
                return 1;
            }
        } else if (x39(x175, "--flow-name")) {
            if (!x127(&x173, "flow_name", x176)) {
                return 1;
            }
        } else if (x39(x175, "--flow-type") || x39(x175, "-f")) {
            if (!x127(&x173, "flow_type", x176)) {
                return 1;
            }
        } else if (x39(x175, "--output-name")) {
            if (!x127(&x173, "output_name", x176)) {
                return 1;
            }
        } else if (x39(x175, "--output-target") || x39(x175, "-o")) {
            if (!x127(&x173, "output_target", x176)) {
                return 1;
            }
        } else if (x39(x175, "--separator")) {
            if (!x127(&x173, "separator", x176)) {
                return 1;
            }
        } else if (x39(x175, "--prefix")) {
            if (!x127(&x173, "prefix", x176)) {
                return 1;
            }
        } else if (x39(x175, "--suffix")) {
            if (!x127(&x173, "suffix", x176)) {
                return 1;
            }
        } else {
            fprintf(stderr, "Unknown flag: %s\n", x175);
            x165(x169[0]);
            return 1;
        }

        ++x172;
        x174 = true;
    }

    if (!x174) {
        x165(x169[0]);
        return 0;
    }

    if (!x66(x170, &x173)) {
        return 1;
    }

    *x171 = true;
    return 0;
}

int main(int x177, char** x178) {
    x31 x179;
    bool x180;
    int x181;

    x181 = x167(x177, x178, &x179, &x180);
    if (x181 != 0) {
        return x181;
    }

    if (!x180) {
        return 0;
    }

    if (!x124(&x179)) {
        return 1;
    }

    return 0;
}
```

<a id="file-8"></a>
### [8] `4.cpp`

- **Bytes:** `14184`
- **Type:** `text`

```cpp
/*
 * configure_1.cpp
 *
 * Risk-free C++ replacement for the original broad-character permutation
 * generator. It emits a stable cppdb source-record format that can represent
 * embedded whitespace and newlines without corrupting database ingestion.
 *
 * Build:
 *   c++ -std=c++17 -Wall -Wextra -pedantic -O2 configure_1.cpp -o configure_1
 *
 * Examples:
 *   ./configure_1 --help
 *   ./configure_1 --execute --limit 1000 --output system_safe.cppdb.txt
 *   ./configure_1 --execute --alphabet safe --width 3 --all --allow-large
 */

#include <cctype>
#include <cstdio>
#include <cstdint>
#include <fstream>
#include <iomanip>
#include <iostream>
#include <limits>
#include <sstream>
#include <stdexcept>
#include <string>
#include <string_view>
#include <vector>

namespace {

constexpr std::uint64_t kDefaultWidth = 4;
constexpr std::uint64_t kDefaultLimit = 10000;
constexpr std::uint64_t kUnapprovedRecordLimit = 100000;
constexpr std::uint64_t kMaxWidth = 16;

struct Options {
    std::uint64_t width = kDefaultWidth;
    std::uint64_t start = 0;
    std::uint64_t limit = kDefaultLimit;
    bool all = false;
    bool allow_large = false;
    bool execute = false;
    bool overwrite = false;
    std::string alphabet_name = "original";
    std::string output_path = "system_safe.cppdb.txt";
};

class OutputSink {
public:
    virtual ~OutputSink() = default;
    virtual void write(std::string_view value) = 0;
};

class StreamSink final : public OutputSink {
public:
    explicit StreamSink(std::ostream& stream) : stream_(stream) {}

    void write(std::string_view value) override {
        stream_.write(value.data(), static_cast<std::streamsize>(value.size()));
        if (!stream_) {
            throw std::runtime_error("Failed while writing to stream.");
        }
    }

private:
    std::ostream& stream_;
};

class FileSink final : public OutputSink {
public:
    explicit FileSink(const std::string& path) : file_(std::fopen(path.c_str(), "wb")) {
        if (file_ == nullptr) {
            throw std::runtime_error("Unable to open output file: " + path);
        }
    }

    ~FileSink() override {
        if (file_ != nullptr) {
            std::fclose(file_);
        }
    }

    FileSink(const FileSink&) = delete;
    FileSink& operator=(const FileSink&) = delete;

    void write(std::string_view value) override {
        if (!value.empty() &&
            std::fwrite(value.data(), 1, value.size(), file_) != value.size()) {
            throw std::runtime_error("Failed while writing output file.");
        }
    }

private:
    std::FILE* file_;
};

class Writer {
public:
    explicit Writer(OutputSink& sink) : sink_(sink) {}

    Writer& operator<<(char value) {
        sink_.write(std::string_view(&value, 1));
        return *this;
    }

    Writer& operator<<(std::string_view value) {
        sink_.write(value);
        return *this;
    }

    Writer& operator<<(const std::string& value) {
        sink_.write(value);
        return *this;
    }

    Writer& operator<<(const char* value) {
        sink_.write(value == nullptr ? std::string_view{} : std::string_view(value));
        return *this;
    }

    template <typename T>
    Writer& operator<<(const T& value) {
        std::ostringstream out;
        out << value;
        sink_.write(out.str());
        return *this;
    }

private:
    OutputSink& sink_;
};

std::string to_lower(std::string value) {
    for (char& c : value) {
        c = static_cast<char>(std::tolower(static_cast<unsigned char>(c)));
    }
    return value;
}

bool parse_u64(std::string_view text, std::uint64_t& out) {
    if (text.empty()) {
        return false;
    }

    std::uint64_t value = 0;
    for (char c : text) {
        if (!std::isdigit(static_cast<unsigned char>(c))) {
            return false;
        }
        const std::uint64_t digit = static_cast<std::uint64_t>(c - '0');
        if (value > (std::numeric_limits<std::uint64_t>::max() - digit) / 10) {
            return false;
        }
        value = value * 10 + digit;
    }

    out = value;
    return true;
}

std::uint64_t require_u64(const std::vector<std::string>& args, std::size_t& i) {
    if (i + 1 >= args.size()) {
        throw std::runtime_error("Missing value for " + args[i]);
    }

    std::uint64_t value = 0;
    if (!parse_u64(args[++i], value)) {
        throw std::runtime_error("Invalid unsigned integer for " + args[i - 1] + ": " + args[i]);
    }
    return value;
}

std::string require_string(const std::vector<std::string>& args, std::size_t& i) {
    if (i + 1 >= args.size()) {
        throw std::runtime_error("Missing value for " + args[i]);
    }
    return args[++i];
}

bool safe_power(std::uint64_t base, std::uint64_t exp, std::uint64_t& out) {
    if (base == 0) {
        return false;
    }

    std::uint64_t value = 1;
    for (std::uint64_t i = 0; i < exp; ++i) {
        if (value > std::numeric_limits<std::uint64_t>::max() / base) {
            return false;
        }
        value *= base;
    }

    out = value;
    return true;
}

std::string original_alphabet() {
    std::string alphabet;
    for (char c = 'a'; c <= 'z'; ++c) {
        alphabet.push_back(c);
    }
    for (char c = 'A'; c <= 'Z'; ++c) {
        alphabet.push_back(c);
    }
    for (char c = '0'; c <= '9'; ++c) {
        alphabet.push_back(c);
    }
    alphabet.push_back(' ');
    alphabet.push_back('\n');
    return alphabet;
}

std::string alphabet_for_name(const std::string& name) {
    const std::string normalized = to_lower(name);
    if (normalized == "original") {
        return original_alphabet();
    }
    if (normalized == "safe") {
        return "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_";
    }
    if (normalized == "digits") {
        return "0123456789";
    }
    if (normalized == "lower") {
        return "abcdefghijklmnopqrstuvwxyz";
    }
    throw std::runtime_error("Unknown alphabet: " + name);
}

std::string escape_field(std::string_view value) {
    std::ostringstream out;
    out << std::hex << std::uppercase << std::setfill('0');

    for (unsigned char c : value) {
        if (c == '\\') {
            out << "\\\\";
        } else if (c == '\n') {
            out << "\\n";
        } else if (c == '\r') {
            out << "\\r";
        } else if (c == '\t') {
            out << "\\t";
        } else if (c < 32 || c == 127) {
            out << "\\x" << std::setw(2) << static_cast<unsigned int>(c);
        } else {
            out << static_cast<char>(c);
        }
    }

    return out.str();
}

std::uint64_t fnv1a_64(std::string_view value) {
    std::uint64_t hash = 14695981039346656037ull;
    for (unsigned char c : value) {
        hash ^= static_cast<std::uint64_t>(c);
        hash *= 1099511628211ull;
    }
    return hash;
}

std::string hex64(std::uint64_t value) {
    std::ostringstream out;
    out << std::hex << std::nouppercase << std::setfill('0') << std::setw(16) << value;
    return out.str();
}

std::string make_record_value(std::uint64_t ordinal, std::uint64_t width, std::string_view alphabet) {
    std::string value(static_cast<std::size_t>(width), '\0');
    const std::uint64_t base = static_cast<std::uint64_t>(alphabet.size());

    for (std::uint64_t pos = 0; pos < width; ++pos) {
        const std::uint64_t reversed_index = width - 1 - pos;
        const std::uint64_t alphabet_index = ordinal % base;
        value[static_cast<std::size_t>(reversed_index)] = alphabet[static_cast<std::size_t>(alphabet_index)];
        ordinal /= base;
    }

    return value;
}

bool file_exists(const std::string& path) {
    std::FILE* file = std::fopen(path.c_str(), "rb");
    if (file == nullptr) {
        return false;
    }
    std::fclose(file);
    return true;
}

void print_help(const char* executable) {
    std::cout
        << "Usage:\n"
        << "  " << executable << " --execute [options]\n"
        << "  " << executable << " [options]              # dry-run plan only\n\n"
        << "Options:\n"
        << "  --width <n>           record width, default 4, maximum 16\n"
        << "  --alphabet <name>     original|safe|digits|lower, default original\n"
        << "  --start <n>           first ordinal to emit, default 0\n"
        << "  --limit <n>           maximum records to emit, default 10000\n"
        << "  --all                 emit all records after --start\n"
        << "  --allow-large         allow more than 100000 records\n"
        << "  --output <path|- >    output file, or - for stdout\n"
        << "  --overwrite           allow replacing an existing output file\n"
        << "  --execute             actually write output\n"
        << "  --help                show this help\n\n"
        << "Output format:\n"
        << "  comment metadata lines beginning with '#'\n"
        << "  record<TAB>ordinal<TAB>role<TAB>width<TAB>fnv1a64<TAB>escaped_value\n";
}

Options parse_options(int argc, char** argv) {
    Options options;
    std::vector<std::string> args(argv, argv + argc);

    for (std::size_t i = 1; i < args.size(); ++i) {
        const std::string& arg = args[i];
        if (arg == "--help" || arg == "-h") {
            print_help(argv[0]);
            std::exit(0);
        } else if (arg == "--width" || arg == "-w") {
            options.width = require_u64(args, i);
        } else if (arg == "--alphabet" || arg == "-a") {
            options.alphabet_name = require_string(args, i);
        } else if (arg == "--start") {
            options.start = require_u64(args, i);
        } else if (arg == "--limit") {
            options.limit = require_u64(args, i);
        } else if (arg == "--all") {
            options.all = true;
        } else if (arg == "--allow-large") {
            options.allow_large = true;
        } else if (arg == "--output" || arg == "-o") {
            options.output_path = require_string(args, i);
        } else if (arg == "--overwrite") {
            options.overwrite = true;
        } else if (arg == "--execute") {
            options.execute = true;
        } else {
            throw std::runtime_error("Unknown option: " + arg);
        }
    }

    if (options.width == 0 || options.width > kMaxWidth) {
        throw std::runtime_error("Width must be between 1 and 16.");
    }

    return options;
}

std::uint64_t requested_count(const Options& options, std::uint64_t total) {
    if (options.start >= total) {
        return 0;
    }

    const std::uint64_t remaining = total - options.start;
    return options.all || options.limit > remaining ? remaining : options.limit;
}

void validate_safety(const Options& options, std::uint64_t count) {
    if (!options.allow_large && count > kUnapprovedRecordLimit) {
        throw std::runtime_error(
            "Refusing to emit more than 100000 records without --allow-large.");
    }
}

void write_output(Writer& out,
                  const Options& options,
                  const std::string& alphabet,
                  std::uint64_t total,
                  std::uint64_t count) {
    out << "# cppdb-configure-source v1\n";
    out << "# source_kind=configure_1\n";
    out << "# role=generated_symbol_candidate\n";
    out << "# record_mode=escaped_line\n";
    out << "# width=" << options.width << '\n';
    out << "# alphabet_name=" << options.alphabet_name << '\n';
    out << "# alphabet_size=" << alphabet.size() << '\n';
    out << "# expected_total=" << total << '\n';
    out << "# start=" << options.start << '\n';
    out << "# emitted_records=" << count << '\n';
    out << "# truncated=" << (options.start + count < total ? "true" : "false") << '\n';

    for (std::uint64_t i = 0; i < count; ++i) {
        const std::uint64_t ordinal = options.start + i;
        const std::string value = make_record_value(ordinal, options.width, alphabet);
        out << "record\t"
            << ordinal << '\t'
            << "generated_symbol_candidate\t"
            << options.width << '\t'
            << hex64(fnv1a_64(value)) << '\t'
            << escape_field(value) << '\n';
    }
}

void print_plan(const Options& options,
                const std::string& alphabet,
                std::uint64_t total,
                std::uint64_t count) {
    std::cout
        << "configure_1 dry run\n"
        << "  output: " << options.output_path << '\n'
        << "  width: " << options.width << '\n'
        << "  alphabet: " << options.alphabet_name << " (" << alphabet.size() << " characters)\n"
        << "  expected total: " << total << '\n'
        << "  start: " << options.start << '\n'
        << "  records selected: " << count << '\n'
        << "  execute: false\n\n"
        << "Add --execute to write the escaped cppdb source-record output.\n";
}

int run(int argc, char** argv) {
    const Options options = parse_options(argc, argv);
    const std::string alphabet = alphabet_for_name(options.alphabet_name);

    std::uint64_t total = 0;
    if (!safe_power(static_cast<std::uint64_t>(alphabet.size()), options.width, total)) {
        throw std::runtime_error("Combination count overflowed uint64_t.");
    }

    const std::uint64_t count = requested_count(options, total);
    validate_safety(options, count);

    if (!options.execute) {
        print_plan(options, alphabet, total, count);
        return 0;
    }

    if (options.output_path == "-") {
        StreamSink sink(std::cout);
        Writer writer(sink);
        write_output(writer, options, alphabet, total, count);
        return 0;
    }

    if (!options.overwrite && file_exists(options.output_path)) {
        throw std::runtime_error("Output file already exists. Use --overwrite to replace it: " +
                                 options.output_path);
    }

    FileSink sink(options.output_path);
    Writer writer(sink);
    write_output(writer, options, alphabet, total, count);
    std::cout << "Wrote " << count << " configure_1 records to " << options.output_path << '\n';
    return 0;
}

} // namespace

int main(int argc, char** argv) {
    try {
        return run(argc, argv);
    } catch (const std::exception& ex) {
        std::cerr << "configure_1 error: " << ex.what() << '\n';
        return 1;
    }
}

```

<a id="file-9"></a>
### [9] `5.cpp`

- **Bytes:** `11789`
- **Type:** `text`

```cpp
/*
 * configure_2.cpp
 *
 * Risk-free C++ replacement for the original numeric permutation generator.
 * It produces bounded, escaped cppdb source records suitable for use as stable
 * generated object handles in the planned CLI database.
 *
 * Build:
 *   c++ -std=c++17 -Wall -Wextra -pedantic -O2 configure_2.cpp -o configure_2
 *
 * Examples:
 *   ./configure_2 --help
 *   ./configure_2 --execute --limit 10000 --output x33.cppdb.txt
 *   ./configure_2 --execute --width 7 --all --allow-large
 */

#include <cctype>
#include <cstdio>
#include <cstdint>
#include <fstream>
#include <iomanip>
#include <iostream>
#include <limits>
#include <sstream>
#include <stdexcept>
#include <string>
#include <string_view>
#include <vector>

namespace {

constexpr std::uint64_t kDefaultWidth = 7;
constexpr std::uint64_t kDefaultLimit = 10000;
constexpr std::uint64_t kUnapprovedRecordLimit = 100000;
constexpr std::uint64_t kMaxWidth = 18;
constexpr std::string_view kAlphabet = "0123456789";

struct Options {
    std::uint64_t width = kDefaultWidth;
    std::uint64_t start = 0;
    std::uint64_t limit = kDefaultLimit;
    bool all = false;
    bool allow_large = false;
    bool execute = false;
    bool overwrite = false;
    std::string output_path = "x33.cppdb.txt";
};

class OutputSink {
public:
    virtual ~OutputSink() = default;
    virtual void write(std::string_view value) = 0;
};

class StreamSink final : public OutputSink {
public:
    explicit StreamSink(std::ostream& stream) : stream_(stream) {}

    void write(std::string_view value) override {
        stream_.write(value.data(), static_cast<std::streamsize>(value.size()));
        if (!stream_) {
            throw std::runtime_error("Failed while writing to stream.");
        }
    }

private:
    std::ostream& stream_;
};

class FileSink final : public OutputSink {
public:
    explicit FileSink(const std::string& path) : file_(std::fopen(path.c_str(), "wb")) {
        if (file_ == nullptr) {
            throw std::runtime_error("Unable to open output file: " + path);
        }
    }

    ~FileSink() override {
        if (file_ != nullptr) {
            std::fclose(file_);
        }
    }

    FileSink(const FileSink&) = delete;
    FileSink& operator=(const FileSink&) = delete;

    void write(std::string_view value) override {
        if (!value.empty() &&
            std::fwrite(value.data(), 1, value.size(), file_) != value.size()) {
            throw std::runtime_error("Failed while writing output file.");
        }
    }

private:
    std::FILE* file_;
};

class Writer {
public:
    explicit Writer(OutputSink& sink) : sink_(sink) {}

    Writer& operator<<(char value) {
        sink_.write(std::string_view(&value, 1));
        return *this;
    }

    Writer& operator<<(std::string_view value) {
        sink_.write(value);
        return *this;
    }

    Writer& operator<<(const std::string& value) {
        sink_.write(value);
        return *this;
    }

    Writer& operator<<(const char* value) {
        sink_.write(value == nullptr ? std::string_view{} : std::string_view(value));
        return *this;
    }

    template <typename T>
    Writer& operator<<(const T& value) {
        std::ostringstream out;
        out << value;
        sink_.write(out.str());
        return *this;
    }

private:
    OutputSink& sink_;
};

bool parse_u64(std::string_view text, std::uint64_t& out) {
    if (text.empty()) {
        return false;
    }

    std::uint64_t value = 0;
    for (char c : text) {
        if (!std::isdigit(static_cast<unsigned char>(c))) {
            return false;
        }
        const std::uint64_t digit = static_cast<std::uint64_t>(c - '0');
        if (value > (std::numeric_limits<std::uint64_t>::max() - digit) / 10) {
            return false;
        }
        value = value * 10 + digit;
    }

    out = value;
    return true;
}

std::uint64_t require_u64(const std::vector<std::string>& args, std::size_t& i) {
    if (i + 1 >= args.size()) {
        throw std::runtime_error("Missing value for " + args[i]);
    }

    std::uint64_t value = 0;
    if (!parse_u64(args[++i], value)) {
        throw std::runtime_error("Invalid unsigned integer for " + args[i - 1] + ": " + args[i]);
    }
    return value;
}

std::string require_string(const std::vector<std::string>& args, std::size_t& i) {
    if (i + 1 >= args.size()) {
        throw std::runtime_error("Missing value for " + args[i]);
    }
    return args[++i];
}

bool safe_power(std::uint64_t base, std::uint64_t exp, std::uint64_t& out) {
    if (base == 0) {
        return false;
    }

    std::uint64_t value = 1;
    for (std::uint64_t i = 0; i < exp; ++i) {
        if (value > std::numeric_limits<std::uint64_t>::max() / base) {
            return false;
        }
        value *= base;
    }

    out = value;
    return true;
}

std::uint64_t fnv1a_64(std::string_view value) {
    std::uint64_t hash = 14695981039346656037ull;
    for (unsigned char c : value) {
        hash ^= static_cast<std::uint64_t>(c);
        hash *= 1099511628211ull;
    }
    return hash;
}

std::string hex64(std::uint64_t value) {
    std::ostringstream out;
    out << std::hex << std::nouppercase << std::setfill('0') << std::setw(16) << value;
    return out.str();
}

std::string make_numeric_value(std::uint64_t ordinal, std::uint64_t width) {
    std::string value(static_cast<std::size_t>(width), '0');

    for (std::uint64_t pos = 0; pos < width; ++pos) {
        const std::uint64_t reversed_index = width - 1 - pos;
        value[static_cast<std::size_t>(reversed_index)] =
            static_cast<char>('0' + (ordinal % 10));
        ordinal /= 10;
    }

    return value;
}

bool file_exists(const std::string& path) {
    std::FILE* file = std::fopen(path.c_str(), "rb");
    if (file == nullptr) {
        return false;
    }
    std::fclose(file);
    return true;
}

void print_help(const char* executable) {
    std::cout
        << "Usage:\n"
        << "  " << executable << " --execute [options]\n"
        << "  " << executable << " [options]              # dry-run plan only\n\n"
        << "Options:\n"
        << "  --width <n>           numeric width, default 7, maximum 18\n"
        << "  --start <n>           first ordinal to emit, default 0\n"
        << "  --limit <n>           maximum records to emit, default 10000\n"
        << "  --all                 emit all records after --start\n"
        << "  --allow-large         allow more than 100000 records\n"
        << "  --output <path|- >    output file, or - for stdout\n"
        << "  --overwrite           allow replacing an existing output file\n"
        << "  --execute             actually write output\n"
        << "  --help                show this help\n\n"
        << "Output format:\n"
        << "  comment metadata lines beginning with '#'\n"
        << "  record<TAB>ordinal<TAB>role<TAB>width<TAB>fnv1a64<TAB>value\n";
}

Options parse_options(int argc, char** argv) {
    Options options;
    std::vector<std::string> args(argv, argv + argc);

    for (std::size_t i = 1; i < args.size(); ++i) {
        const std::string& arg = args[i];
        if (arg == "--help" || arg == "-h") {
            print_help(argv[0]);
            std::exit(0);
        } else if (arg == "--width" || arg == "-w") {
            options.width = require_u64(args, i);
        } else if (arg == "--start") {
            options.start = require_u64(args, i);
        } else if (arg == "--limit") {
            options.limit = require_u64(args, i);
        } else if (arg == "--all") {
            options.all = true;
        } else if (arg == "--allow-large") {
            options.allow_large = true;
        } else if (arg == "--output" || arg == "-o") {
            options.output_path = require_string(args, i);
        } else if (arg == "--overwrite") {
            options.overwrite = true;
        } else if (arg == "--execute") {
            options.execute = true;
        } else {
            throw std::runtime_error("Unknown option: " + arg);
        }
    }

    if (options.width == 0 || options.width > kMaxWidth) {
        throw std::runtime_error("Width must be between 1 and 18.");
    }

    return options;
}

std::uint64_t requested_count(const Options& options, std::uint64_t total) {
    if (options.start >= total) {
        return 0;
    }

    const std::uint64_t remaining = total - options.start;
    return options.all || options.limit > remaining ? remaining : options.limit;
}

void validate_safety(const Options& options, std::uint64_t count) {
    if (!options.allow_large && count > kUnapprovedRecordLimit) {
        throw std::runtime_error(
            "Refusing to emit more than 100000 records without --allow-large.");
    }
}

void write_output(Writer& out,
                  const Options& options,
                  std::uint64_t total,
                  std::uint64_t count) {
    out << "# cppdb-configure-source v1\n";
    out << "# source_kind=configure_2\n";
    out << "# role=numeric_object_handle\n";
    out << "# record_mode=line\n";
    out << "# width=" << options.width << '\n';
    out << "# alphabet_name=decimal_digits\n";
    out << "# alphabet_size=" << kAlphabet.size() << '\n';
    out << "# expected_total=" << total << '\n';
    out << "# start=" << options.start << '\n';
    out << "# emitted_records=" << count << '\n';
    out << "# truncated=" << (options.start + count < total ? "true" : "false") << '\n';

    for (std::uint64_t i = 0; i < count; ++i) {
        const std::uint64_t ordinal = options.start + i;
        const std::string value = make_numeric_value(ordinal, options.width);
        out << "record\t"
            << ordinal << '\t'
            << "numeric_object_handle\t"
            << options.width << '\t'
            << hex64(fnv1a_64(value)) << '\t'
            << value << '\n';
    }
}

void print_plan(const Options& options, std::uint64_t total, std::uint64_t count) {
    std::cout
        << "configure_2 dry run\n"
        << "  output: " << options.output_path << '\n'
        << "  width: " << options.width << '\n'
        << "  alphabet: decimal_digits (" << kAlphabet.size() << " characters)\n"
        << "  expected total: " << total << '\n'
        << "  start: " << options.start << '\n'
        << "  records selected: " << count << '\n'
        << "  execute: false\n\n"
        << "Add --execute to write the cppdb numeric source-record output.\n";
}

int run(int argc, char** argv) {
    const Options options = parse_options(argc, argv);

    std::uint64_t total = 0;
    if (!safe_power(static_cast<std::uint64_t>(kAlphabet.size()), options.width, total)) {
        throw std::runtime_error("Combination count overflowed uint64_t.");
    }

    const std::uint64_t count = requested_count(options, total);
    validate_safety(options, count);

    if (!options.execute) {
        print_plan(options, total, count);
        return 0;
    }

    if (options.output_path == "-") {
        StreamSink sink(std::cout);
        Writer writer(sink);
        write_output(writer, options, total, count);
        return 0;
    }

    if (!options.overwrite && file_exists(options.output_path)) {
        throw std::runtime_error("Output file already exists. Use --overwrite to replace it: " +
                                 options.output_path);
    }

    FileSink sink(options.output_path);
    Writer writer(sink);
    write_output(writer, options, total, count);
    std::cout << "Wrote " << count << " configure_2 records to " << options.output_path << '\n';
    return 0;
}

} // namespace

int main(int argc, char** argv) {
    try {
        return run(argc, argv);
    } catch (const std::exception& ex) {
        std::cerr << "configure_2 error: " << ex.what() << '\n';
        return 1;
    }
}

```

<a id="file-10"></a>
### [10] `6.cpp`

- **Bytes:** `31328`
- **Type:** `text`

```cpp
/*
 * configure_3.cpp
 *
 * Risk-free C++ generative fabric for cppdb. This replaces the C configurator
 * with explicit object specs, bounded execution, dry-run planning, escaped
 * source records, and config/direct/menu input paths.
 *
 * Build:
 *   c++ -std=c++17 -Wall -Wextra -pedantic -O2 configure_3.cpp -o configure_3
 *
 * Examples:
 *   ./configure_3 --sample-config x33.fabric
 *   ./configure_3 --config x33.fabric
 *   ./configure_3 --config x33.fabric --execute --overwrite
 *   ./configure_3 --execute --object-name x33 --input-value 01 --input-width 3
 */

#include <algorithm>
#include <cctype>
#include <cstdio>
#include <cstdint>
#include <fstream>
#include <iomanip>
#include <iostream>
#include <limits>
#include <map>
#include <sstream>
#include <stdexcept>
#include <string>
#include <string_view>
#include <vector>

namespace {

constexpr std::uint64_t kDefaultLimit = 10000;
constexpr std::uint64_t kUnapprovedRecordLimit = 100000;
constexpr std::uint64_t kMaxCartesianWidth = 64;
constexpr std::uint64_t kMaxRepeatCount = 1000000;
constexpr std::size_t kMaxTextLength = 4096;

enum class Flow {
    cartesian,
    literal,
    repeat,
    reverse
};

struct ObjectSpec {
    bool touched = false;
    std::string object_name = "x33";
    std::string input_name = "x1";
    std::string input_value = "0123456789";
    std::uint64_t input_width = 7;
    std::string flow_name = "x15";
    Flow flow = Flow::cartesian;
    std::string output_name = "x31";
    std::string output_target = "x33.cppdb.txt";
    std::string separator = "\n";
    std::string prefix;
    std::string suffix;
};

struct Options {
    bool execute = false;
    bool allow_large = false;
    bool all = false;
    bool overwrite = false;
    bool menu = false;
    std::uint64_t start = 0;
    std::uint64_t limit = kDefaultLimit;
    std::string config_path;
    std::string sample_config_path;
};

struct Program {
    Options options;
    ObjectSpec direct_spec;
    bool has_direct_spec = false;
};

class OutputSink {
public:
    virtual ~OutputSink() = default;
    virtual void write(std::string_view value) = 0;
};

class StreamSink final : public OutputSink {
public:
    explicit StreamSink(std::ostream& stream) : stream_(stream) {}

    void write(std::string_view value) override {
        stream_.write(value.data(), static_cast<std::streamsize>(value.size()));
        if (!stream_) {
            throw std::runtime_error("Failed while writing to stream.");
        }
    }

private:
    std::ostream& stream_;
};

class FileSink final : public OutputSink {
public:
    FileSink(const std::string& path, const char* mode) : file_(std::fopen(path.c_str(), mode)) {
        if (file_ == nullptr) {
            throw std::runtime_error("Unable to open output file: " + path);
        }
    }

    ~FileSink() override {
        if (file_ != nullptr) {
            std::fclose(file_);
        }
    }

    FileSink(const FileSink&) = delete;
    FileSink& operator=(const FileSink&) = delete;

    void write(std::string_view value) override {
        if (!value.empty() &&
            std::fwrite(value.data(), 1, value.size(), file_) != value.size()) {
            throw std::runtime_error("Failed while writing output file.");
        }
    }

private:
    std::FILE* file_;
};

class InputFile {
public:
    explicit InputFile(const std::string& path) : file_(std::fopen(path.c_str(), "rb")) {
        if (file_ == nullptr) {
            throw std::runtime_error("Unable to open config file: " + path);
        }
    }

    ~InputFile() {
        if (file_ != nullptr) {
            std::fclose(file_);
        }
    }

    InputFile(const InputFile&) = delete;
    InputFile& operator=(const InputFile&) = delete;

    bool read_line(std::string& line) {
        char buffer[8192];
        line.clear();

        for (;;) {
            if (std::fgets(buffer, sizeof(buffer), file_) == nullptr) {
                if (std::ferror(file_) != 0) {
                    throw std::runtime_error("Failed while reading config file.");
                }
                return !line.empty();
            }

            line += buffer;
            if (!line.empty() && line.back() == '\n') {
                break;
            }

            const std::size_t chunk_size = std::char_traits<char>::length(buffer);
            if (chunk_size + 1 < sizeof(buffer)) {
                break;
            }
        }

        while (!line.empty() && (line.back() == '\n' || line.back() == '\r')) {
            line.pop_back();
        }

        return true;
    }

private:
    std::FILE* file_;
};

class Writer {
public:
    explicit Writer(OutputSink& sink) : sink_(sink) {}

    Writer& operator<<(char value) {
        sink_.write(std::string_view(&value, 1));
        return *this;
    }

    Writer& operator<<(std::string_view value) {
        sink_.write(value);
        return *this;
    }

    Writer& operator<<(const std::string& value) {
        sink_.write(value);
        return *this;
    }

    Writer& operator<<(const char* value) {
        sink_.write(value == nullptr ? std::string_view{} : std::string_view(value));
        return *this;
    }

    template <typename T>
    Writer& operator<<(const T& value) {
        std::ostringstream out;
        out << value;
        sink_.write(out.str());
        return *this;
    }

private:
    OutputSink& sink_;
};

std::string to_lower(std::string value) {
    for (char& c : value) {
        c = static_cast<char>(std::tolower(static_cast<unsigned char>(c)));
    }
    return value;
}

std::string trim(std::string value) {
    while (!value.empty() && std::isspace(static_cast<unsigned char>(value.front()))) {
        value.erase(value.begin());
    }
    while (!value.empty() && std::isspace(static_cast<unsigned char>(value.back()))) {
        value.pop_back();
    }
    return value;
}

bool equals_ignore_case(std::string_view lhs, std::string_view rhs) {
    if (lhs.size() != rhs.size()) {
        return false;
    }

    for (std::size_t i = 0; i < lhs.size(); ++i) {
        if (std::tolower(static_cast<unsigned char>(lhs[i])) !=
            std::tolower(static_cast<unsigned char>(rhs[i]))) {
            return false;
        }
    }

    return true;
}

bool parse_u64(std::string_view text, std::uint64_t& out) {
    if (text.empty()) {
        return false;
    }

    std::uint64_t value = 0;
    for (char c : text) {
        if (!std::isdigit(static_cast<unsigned char>(c))) {
            return false;
        }
        const std::uint64_t digit = static_cast<std::uint64_t>(c - '0');
        if (value > (std::numeric_limits<std::uint64_t>::max() - digit) / 10) {
            return false;
        }
        value = value * 10 + digit;
    }

    out = value;
    return true;
}

std::uint64_t require_u64(const std::vector<std::string>& args, std::size_t& i) {
    if (i + 1 >= args.size()) {
        throw std::runtime_error("Missing value for " + args[i]);
    }

    std::uint64_t value = 0;
    if (!parse_u64(args[++i], value)) {
        throw std::runtime_error("Invalid unsigned integer for " + args[i - 1] + ": " + args[i]);
    }
    return value;
}

std::string require_string(const std::vector<std::string>& args, std::size_t& i) {
    if (i + 1 >= args.size()) {
        throw std::runtime_error("Missing value for " + args[i]);
    }
    return args[++i];
}

bool safe_power(std::uint64_t base, std::uint64_t exp, std::uint64_t& out) {
    if (base == 0) {
        return false;
    }

    std::uint64_t value = 1;
    for (std::uint64_t i = 0; i < exp; ++i) {
        if (value > std::numeric_limits<std::uint64_t>::max() / base) {
            return false;
        }
        value *= base;
    }

    out = value;
    return true;
}

std::string unescape_text(std::string_view value) {
    std::string out;
    out.reserve(value.size());

    for (std::size_t i = 0; i < value.size(); ++i) {
        char c = value[i];
        if (c != '\\' || i + 1 >= value.size()) {
            out.push_back(c);
            continue;
        }

        const char next = value[++i];
        if (next == 'n') {
            out.push_back('\n');
        } else if (next == 't') {
            out.push_back('\t');
        } else if (next == 'r') {
            out.push_back('\r');
        } else if (next == '\\') {
            out.push_back('\\');
        } else {
            out.push_back(next);
        }
    }

    return out;
}

std::string escape_field(std::string_view value) {
    std::ostringstream out;
    out << std::hex << std::uppercase << std::setfill('0');

    for (unsigned char c : value) {
        if (c == '\\') {
            out << "\\\\";
        } else if (c == '\n') {
            out << "\\n";
        } else if (c == '\r') {
            out << "\\r";
        } else if (c == '\t') {
            out << "\\t";
        } else if (c < 32 || c == 127) {
            out << "\\x" << std::setw(2) << static_cast<unsigned int>(c);
        } else {
            out << static_cast<char>(c);
        }
    }

    return out.str();
}

std::uint64_t fnv1a_64(std::string_view value) {
    std::uint64_t hash = 14695981039346656037ull;
    for (unsigned char c : value) {
        hash ^= static_cast<std::uint64_t>(c);
        hash *= 1099511628211ull;
    }
    return hash;
}

std::string hex64(std::uint64_t value) {
    std::ostringstream out;
    out << std::hex << std::nouppercase << std::setfill('0') << std::setw(16) << value;
    return out.str();
}

std::string flow_name(Flow flow) {
    if (flow == Flow::cartesian) {
        return "cartesian";
    }
    if (flow == Flow::literal) {
        return "literal";
    }
    if (flow == Flow::repeat) {
        return "repeat";
    }
    return "reverse";
}

Flow parse_flow(std::string_view value) {
    const std::string normalized = to_lower(std::string(value));
    if (normalized == "cartesian" || normalized == "product" || normalized == "combinations") {
        return Flow::cartesian;
    }
    if (normalized == "literal" || normalized == "echo") {
        return Flow::literal;
    }
    if (normalized == "repeat") {
        return Flow::repeat;
    }
    if (normalized == "reverse") {
        return Flow::reverse;
    }
    throw std::runtime_error("Unknown flow_type: " + std::string(value));
}

std::string make_cartesian_value(std::uint64_t ordinal,
                                 std::uint64_t width,
                                 std::string_view alphabet) {
    std::string value(static_cast<std::size_t>(width), '\0');
    const std::uint64_t base = static_cast<std::uint64_t>(alphabet.size());

    for (std::uint64_t pos = 0; pos < width; ++pos) {
        const std::uint64_t reversed_index = width - 1 - pos;
        const std::uint64_t alphabet_index = ordinal % base;
        value[static_cast<std::size_t>(reversed_index)] = alphabet[static_cast<std::size_t>(alphabet_index)];
        ordinal /= base;
    }

    return value;
}

std::string make_payload(const ObjectSpec& spec, std::uint64_t ordinal) {
    if (spec.flow == Flow::cartesian) {
        return make_cartesian_value(ordinal, spec.input_width, spec.input_value);
    }
    if (spec.flow == Flow::literal) {
        return spec.input_value;
    }
    if (spec.flow == Flow::repeat) {
        return spec.input_value;
    }

    std::string reversed = spec.input_value;
    std::reverse(reversed.begin(), reversed.end());
    return reversed;
}

std::uint64_t expected_total(const ObjectSpec& spec) {
    if (spec.flow == Flow::cartesian) {
        std::uint64_t total = 0;
        if (!safe_power(static_cast<std::uint64_t>(spec.input_value.size()),
                        spec.input_width,
                        total)) {
            throw std::runtime_error("Combination count overflowed uint64_t for " +
                                     spec.object_name);
        }
        return total;
    }

    if (spec.flow == Flow::repeat) {
        return spec.input_width;
    }

    return 1;
}

std::uint64_t requested_count(const Options& options, std::uint64_t total) {
    if (options.start >= total) {
        return 0;
    }

    const std::uint64_t remaining = total - options.start;
    return options.all || options.limit > remaining ? remaining : options.limit;
}

bool file_exists(const std::string& path) {
    std::FILE* file = std::fopen(path.c_str(), "rb");
    if (file == nullptr) {
        return false;
    }
    std::fclose(file);
    return true;
}

void validate_text_length(const std::string& label, const std::string& value) {
    if (value.size() > kMaxTextLength) {
        throw std::runtime_error(label + " is too large; maximum length is 4096 bytes.");
    }
}

void validate_spec(const ObjectSpec& spec) {
    if (spec.object_name.empty() || spec.input_name.empty() || spec.flow_name.empty() ||
        spec.output_name.empty() || spec.output_target.empty()) {
        throw std::runtime_error("Object spec has an empty required name or output target.");
    }

    validate_text_length("input_value", spec.input_value);
    validate_text_length("prefix", spec.prefix);
    validate_text_length("suffix", spec.suffix);
    validate_text_length("separator", spec.separator);

    if (spec.input_width == 0) {
        throw std::runtime_error("input_width must be greater than zero for " + spec.object_name);
    }

    if (spec.flow == Flow::cartesian) {
        if (spec.input_value.empty()) {
            throw std::runtime_error("cartesian input_value cannot be empty for " + spec.object_name);
        }
        if (spec.input_width > kMaxCartesianWidth) {
            throw std::runtime_error("cartesian input_width exceeds 64 for " + spec.object_name);
        }
        (void)expected_total(spec);
    }

    if (spec.flow == Flow::repeat && spec.input_width > kMaxRepeatCount) {
        throw std::runtime_error("repeat input_width exceeds 1000000 for " + spec.object_name);
    }
}

void validate_safety(const Options& options,
                     const ObjectSpec& spec,
                     std::uint64_t count) {
    if (!options.allow_large && count > kUnapprovedRecordLimit) {
        throw std::runtime_error("Refusing to emit more than 100000 records for " +
                                 spec.object_name + " without --allow-large.");
    }
}

void set_spec_value(ObjectSpec& spec, std::string key, std::string value) {
    key = to_lower(trim(key));
    value = trim(value);
    spec.touched = true;

    if (key == "object" || key == "object_name" || key == "name") {
        spec.object_name = value;
    } else if (key == "input_name") {
        spec.input_name = value;
    } else if (key == "input" || key == "input_value" || key == "alphabet") {
        spec.input_value = unescape_text(value);
    } else if (key == "input_width" || key == "width" || key == "length" || key == "depth") {
        if (!parse_u64(value, spec.input_width)) {
            throw std::runtime_error("Invalid input_width: " + value);
        }
    } else if (key == "flow_name") {
        spec.flow_name = value;
    } else if (key == "flow" || key == "flow_type" || key == "processing_flow") {
        spec.flow = parse_flow(value);
    } else if (key == "output_name") {
        spec.output_name = value;
    } else if (key == "output" || key == "output_target" || key == "file") {
        spec.output_target = value;
    } else if (key == "separator" || key == "sep") {
        spec.separator = unescape_text(value);
    } else if (key == "prefix") {
        spec.prefix = unescape_text(value);
    } else if (key == "suffix") {
        spec.suffix = unescape_text(value);
    } else {
        throw std::runtime_error("Unknown config key: " + key);
    }
}

bool parse_config_line(std::string line, std::string& key, std::string& value) {
    const std::size_t comment = line.find('#');
    if (comment != std::string::npos) {
        line.erase(comment);
    }

    line = trim(line);
    if (line.empty()) {
        return false;
    }

    const std::size_t equals = line.find('=');
    if (equals != std::string::npos) {
        key = trim(line.substr(0, equals));
        value = trim(line.substr(equals + 1));
        return !key.empty();
    }

    std::istringstream in(line);
    in >> key;
    std::getline(in, value);
    value = trim(value);
    return !key.empty();
}

std::vector<ObjectSpec> load_config(const std::string& path) {
    InputFile in(path);
    std::vector<ObjectSpec> specs;
    ObjectSpec current;
    std::string line;
    std::uint64_t line_number = 0;

    while (in.read_line(line)) {
        ++line_number;

        std::string key;
        std::string value;
        if (!parse_config_line(line, key, value)) {
            continue;
        }

        if (equals_ignore_case(key, "end")) {
            if (current.touched) {
                validate_spec(current);
                specs.push_back(current);
                current = ObjectSpec{};
            }
            continue;
        }

        if ((equals_ignore_case(key, "object") || equals_ignore_case(key, "object_name")) &&
            current.touched) {
            validate_spec(current);
            specs.push_back(current);
            current = ObjectSpec{};
        }

        try {
            set_spec_value(current, key, value);
        } catch (const std::exception& ex) {
            std::ostringstream message;
            message << "Config line " << line_number << ": " << ex.what();
            throw std::runtime_error(message.str());
        }
    }

    if (current.touched) {
        validate_spec(current);
        specs.push_back(current);
    }

    if (specs.empty()) {
        throw std::runtime_error("Config file contains no object specs: " + path);
    }

    return specs;
}

void write_sample_config(const std::string& path, bool overwrite) {
    if (!overwrite && file_exists(path)) {
        throw std::runtime_error("Sample config already exists. Use --overwrite to replace it: " + path);
    }

    FileSink sink(path, "wb");
    Writer out(sink);
    out
        << "# cppdb configure_3 fabric config\n"
        << "object_name=x33\n"
        << "input_name=x1\n"
        << "input_value=0123456789\n"
        << "input_width=3\n"
        << "flow_name=x15\n"
        << "flow_type=cartesian\n"
        << "output_name=x31\n"
        << "output_target=x33.cppdb.txt\n"
        << "separator=\\n\n"
        << "prefix=\n"
        << "suffix=\n"
        << "end\n"
        << "\n"
        << "object_name=asset_alias_seed\n"
        << "input_name=alias_text\n"
        << "input_value=PlayerController\n"
        << "input_width=1\n"
        << "flow_name=reverse_alias_demo\n"
        << "flow_type=reverse\n"
        << "output_name=stdout_preview\n"
        << "output_target=stdout\n"
        << "separator=\\n\n"
        << "end\n";
}

std::string prompt_string(const std::string& label, const std::string& default_value) {
    std::cout << label << " [" << escape_field(default_value) << "]: ";
    std::string value;
    std::getline(std::cin, value);
    return value.empty() ? default_value : value;
}

std::uint64_t prompt_u64(const std::string& label, std::uint64_t default_value) {
    for (;;) {
        std::cout << label << " [" << default_value << "]: ";
        std::string value;
        std::getline(std::cin, value);
        if (value.empty()) {
            return default_value;
        }

        std::uint64_t parsed = 0;
        if (parse_u64(value, parsed)) {
            return parsed;
        }

        std::cout << "Please enter a non-negative integer.\n";
    }
}

ObjectSpec prompt_spec() {
    ObjectSpec spec;
    spec.object_name = prompt_string("object_name", spec.object_name);
    spec.input_name = prompt_string("input_name", spec.input_name);
    spec.input_value = unescape_text(prompt_string("input_value", spec.input_value));
    spec.input_width = prompt_u64("input_width", spec.input_width);
    spec.flow_name = prompt_string("flow_name", spec.flow_name);
    spec.flow = parse_flow(prompt_string("flow_type cartesian|literal|repeat|reverse", "cartesian"));
    spec.output_name = prompt_string("output_name", spec.output_name);
    spec.output_target = prompt_string("output_target file|stdout|-", spec.output_target);
    spec.separator = unescape_text(prompt_string("separator", "\\n"));
    spec.prefix = unescape_text(prompt_string("prefix", ""));
    spec.suffix = unescape_text(prompt_string("suffix", ""));
    spec.touched = true;
    validate_spec(spec);
    return spec;
}

void print_help(const char* executable) {
    std::cout
        << "Usage:\n"
        << "  " << executable << " --menu\n"
        << "  " << executable << " --config <path> [--execute]\n"
        << "  " << executable << " --sample-config <path>\n"
        << "  " << executable << " [object flags] [--execute]\n\n"
        << "Object flags:\n"
        << "  --object-name <name>       runtime object name\n"
        << "  --input-name <name>        runtime input name\n"
        << "  --input-value <value>      payload/alphabet, supports \\\\n, \\\\t, \\\\r\n"
        << "  --input-width <n>          width/count/depth\n"
        << "  --flow-name <name>         runtime processing-flow name\n"
        << "  --flow-type <type>         cartesian|literal|repeat|reverse\n"
        << "  --output-name <name>       runtime output name\n"
        << "  --output-target <target>   file path, stdout, or -\n"
        << "  --separator <text>         stored as metadata, supports escapes\n"
        << "  --prefix <text>            prepended to rendered value\n"
        << "  --suffix <text>            appended to rendered value\n\n"
        << "Execution controls:\n"
        << "  --start <n>                first ordinal, default 0\n"
        << "  --limit <n>                maximum records, default 10000\n"
        << "  --all                      emit all records after --start\n"
        << "  --allow-large              allow more than 100000 records\n"
        << "  --overwrite                allow replacing output files\n"
        << "  --execute                  actually write output\n\n"
        << "Aliases:\n"
        << "  -n <name>  -i <value>  -w <n>  -f <type>  -o <target>\n";
}

Program parse_program(int argc, char** argv) {
    Program program;
    std::vector<std::string> args(argv, argv + argc);

    for (std::size_t i = 1; i < args.size(); ++i) {
        const std::string& arg = args[i];

        if (arg == "--help" || arg == "-h") {
            print_help(argv[0]);
            std::exit(0);
        } else if (arg == "--execute") {
            program.options.execute = true;
        } else if (arg == "--allow-large") {
            program.options.allow_large = true;
        } else if (arg == "--all") {
            program.options.all = true;
        } else if (arg == "--overwrite") {
            program.options.overwrite = true;
        } else if (arg == "--menu" || arg == "--ui") {
            program.options.menu = true;
        } else if (arg == "--start") {
            program.options.start = require_u64(args, i);
        } else if (arg == "--limit") {
            program.options.limit = require_u64(args, i);
        } else if (arg == "--config") {
            program.options.config_path = require_string(args, i);
        } else if (arg == "--sample-config") {
            program.options.sample_config_path = require_string(args, i);
        } else if (arg == "--object-name" || arg == "-n") {
            set_spec_value(program.direct_spec, "object_name", require_string(args, i));
            program.has_direct_spec = true;
        } else if (arg == "--input-name") {
            set_spec_value(program.direct_spec, "input_name", require_string(args, i));
            program.has_direct_spec = true;
        } else if (arg == "--input-value" || arg == "-i") {
            set_spec_value(program.direct_spec, "input_value", require_string(args, i));
            program.has_direct_spec = true;
        } else if (arg == "--input-width" || arg == "-w") {
            set_spec_value(program.direct_spec, "input_width", require_string(args, i));
            program.has_direct_spec = true;
        } else if (arg == "--flow-name") {
            set_spec_value(program.direct_spec, "flow_name", require_string(args, i));
            program.has_direct_spec = true;
        } else if (arg == "--flow-type" || arg == "-f") {
            set_spec_value(program.direct_spec, "flow_type", require_string(args, i));
            program.has_direct_spec = true;
        } else if (arg == "--output-name") {
            set_spec_value(program.direct_spec, "output_name", require_string(args, i));
            program.has_direct_spec = true;
        } else if (arg == "--output-target" || arg == "-o") {
            set_spec_value(program.direct_spec, "output_target", require_string(args, i));
            program.has_direct_spec = true;
        } else if (arg == "--separator") {
            set_spec_value(program.direct_spec, "separator", require_string(args, i));
            program.has_direct_spec = true;
        } else if (arg == "--prefix") {
            set_spec_value(program.direct_spec, "prefix", require_string(args, i));
            program.has_direct_spec = true;
        } else if (arg == "--suffix") {
            set_spec_value(program.direct_spec, "suffix", require_string(args, i));
            program.has_direct_spec = true;
        } else {
            throw std::runtime_error("Unknown option: " + arg);
        }
    }

    return program;
}

void write_spec_output(Writer& out,
                       const Options& options,
                       const ObjectSpec& spec,
                       std::uint64_t total,
                       std::uint64_t count) {
    out << "# cppdb-configure-source v1\n";
    out << "# source_kind=configure_3\n";
    out << "# role=configured_object_fabric\n";
    out << "# record_mode=escaped_line\n";
    out << "# object_name=" << escape_field(spec.object_name) << '\n';
    out << "# input_name=" << escape_field(spec.input_name) << '\n';
    out << "# input_width=" << spec.input_width << '\n';
    out << "# input_size=" << spec.input_value.size() << '\n';
    out << "# flow_name=" << escape_field(spec.flow_name) << '\n';
    out << "# flow_type=" << flow_name(spec.flow) << '\n';
    out << "# output_name=" << escape_field(spec.output_name) << '\n';
    out << "# separator=" << escape_field(spec.separator) << '\n';
    out << "# prefix=" << escape_field(spec.prefix) << '\n';
    out << "# suffix=" << escape_field(spec.suffix) << '\n';
    out << "# expected_total=" << total << '\n';
    out << "# start=" << options.start << '\n';
    out << "# emitted_records=" << count << '\n';
    out << "# truncated=" << (options.start + count < total ? "true" : "false") << '\n';

    for (std::uint64_t i = 0; i < count; ++i) {
        const std::uint64_t ordinal = options.start + i;
        const std::string payload = make_payload(spec, ordinal);
        const std::string rendered = spec.prefix + payload + spec.suffix;

        out << "record\t"
            << ordinal << '\t'
            << escape_field(spec.object_name) << '\t'
            << escape_field(spec.input_name) << '\t'
            << escape_field(spec.flow_name) << '\t'
            << escape_field(spec.output_name) << '\t'
            << flow_name(spec.flow) << '\t'
            << payload.size() << '\t'
            << hex64(fnv1a_64(rendered)) << '\t'
            << escape_field(payload) << '\t'
            << escape_field(rendered) << '\n';
    }
}

void print_spec_plan(const Options& options,
                     const ObjectSpec& spec,
                     std::uint64_t total,
                     std::uint64_t count) {
    std::cout
        << "configure_3 dry run\n"
        << "  object: " << spec.object_name << '\n'
        << "  output: " << spec.output_target << '\n'
        << "  flow: " << flow_name(spec.flow) << '\n'
        << "  input width: " << spec.input_width << '\n'
        << "  expected total: " << total << '\n'
        << "  start: " << options.start << '\n'
        << "  records selected: " << count << '\n'
        << "  execute: false\n\n";
}

void emit_specs(const Options& options, const std::vector<ObjectSpec>& specs) {
    std::map<std::string, bool> initialized_targets;

    for (const ObjectSpec& spec : specs) {
        validate_spec(spec);
        const std::uint64_t total = expected_total(spec);
        const std::uint64_t count = requested_count(options, total);
        validate_safety(options, spec, count);

        if (!options.execute) {
            print_spec_plan(options, spec, total, count);
            continue;
        }

        if (spec.output_target == "-" || equals_ignore_case(spec.output_target, "stdout")) {
            StreamSink sink(std::cout);
            Writer writer(sink);
            write_spec_output(writer, options, spec, total, count);
            continue;
        }

        const bool already_initialized = initialized_targets[spec.output_target];
        if (!already_initialized && !options.overwrite && file_exists(spec.output_target)) {
            throw std::runtime_error("Output file already exists. Use --overwrite to replace it: " +
                                     spec.output_target);
        }

        FileSink sink(spec.output_target, already_initialized ? "ab" : "wb");
        Writer writer(sink);
        write_spec_output(writer, options, spec, total, count);
        initialized_targets[spec.output_target] = true;
        std::cout << "Wrote " << count << " configure_3 records for "
                  << spec.object_name << " to " << spec.output_target << '\n';
    }

    if (!options.execute) {
        std::cout << "Add --execute to write the escaped cppdb source-record output.\n";
    }
}

std::vector<ObjectSpec> resolve_specs(const Program& program, const char* executable) {
    if (!program.options.sample_config_path.empty()) {
        return {};
    }

    const bool uses_config = !program.options.config_path.empty();
    const bool uses_direct = program.has_direct_spec;

    if ((uses_config ? 1 : 0) + (program.options.menu ? 1 : 0) + (uses_direct ? 1 : 0) > 1) {
        throw std::runtime_error("Use only one input mode: --config, --menu, or direct object flags.");
    }

    if (uses_config) {
        return load_config(program.options.config_path);
    }

    if (program.options.menu) {
        return {prompt_spec()};
    }

    if (uses_direct) {
        validate_spec(program.direct_spec);
        return {program.direct_spec};
    }

    print_help(executable);
    return {};
}

int run(int argc, char** argv) {
    Program program = parse_program(argc, argv);

    if (!program.options.sample_config_path.empty()) {
        write_sample_config(program.options.sample_config_path, program.options.overwrite);
        std::cout << "Wrote sample config to " << program.options.sample_config_path << '\n';
        return 0;
    }

    const std::vector<ObjectSpec> specs = resolve_specs(program, argv[0]);
    if (specs.empty()) {
        return 0;
    }

    emit_specs(program.options, specs);
    return 0;
}

} // namespace

int main(int argc, char** argv) {
    try {
        return run(argc, argv);
    } catch (const std::exception& ex) {
        std::cerr << "configure_3 error: " << ex.what() << '\n';
        return 1;
    }
}

```

<a id="file-11"></a>
### [11] `7.php`

- **Bytes:** `67375`
- **Type:** `text`

```php
<?php
declare(strict_types=1);

/**
 * nDCodex.php
 *
 * CLI REPL for n-dimensional hash-length fabric analysis.
 *
 * Main additions in this revision:
 * - multiline paste mode for code/text analysis
 * - ranked config matching based on:
 *   - observed unique characters
 *   - pasted character length
 *   - active dimension n
 * - match application back into the active session
 */

if (PHP_SAPI !== 'cli') {
    header('Content-Type: text/html; charset=utf-8');
    echo '<!doctype html><html><head><meta charset="utf-8"><title>nDCodex.php</title>';
    echo '<style>body{font-family:system-ui,Segoe UI,Arial,sans-serif;max-width:860px;margin:40px auto;padding:0 16px;line-height:1.5}code,pre{font-family:Consolas,monospace;background:#f4f4f4;padding:2px 4px}pre{padding:12px;overflow:auto}</style>';
    echo '</head><body>';
    echo '<h1>nDCodex.php</h1>';
    echo '<p>This file is a CLI REPL. Run it from a terminal:</p>';
    echo '<pre>php nDCodex.php</pre>';
    echo '<p>Then use:</p>';
    echo '<pre>paste\n.end</pre>';
    echo '</body></html>';
    exit;
}

final class BigDec
{
    /** @var array<string,string> */
    private static array $powCache = [];

    public static function norm(string $n): string
    {
        $n = ltrim($n, '0');
        return $n === '' ? '0' : $n;
    }

    public static function cmp(string $a, string $b): int
    {
        $a = self::norm($a);
        $b = self::norm($b);
        $la = strlen($a);
        $lb = strlen($b);
        if ($la !== $lb) {
            return $la <=> $lb;
        }
        return $a <=> $b;
    }

    public static function mulInt(string $a, int $m): string
    {
        if ($m < 0) {
            throw new InvalidArgumentException('Negative multiplier not supported.');
        }
        $a = self::norm($a);
        if ($a === '0' || $m === 0) {
            return '0';
        }
        if ($m === 1) {
            return $a;
        }
        $carry = 0;
        $out = '';
        for ($i = strlen($a) - 1; $i >= 0; --$i) {
            $digit = ord($a[$i]) - 48;
            $prod = $digit * $m + $carry;
            $out = (string)($prod % 10) . $out;
            $carry = intdiv($prod, 10);
        }
        while ($carry > 0) {
            $out = (string)($carry % 10) . $out;
            $carry = intdiv($carry, 10);
        }
        return self::norm($out);
    }

    public static function powInt(int $base, int $exp): string
    {
        if ($base < 0 || $exp < 0) {
            throw new InvalidArgumentException('Base and exponent must be non-negative.');
        }
        $key = $base . '^' . $exp;
        if (isset(self::$powCache[$key])) {
            return self::$powCache[$key];
        }
        $result = '1';
        for ($i = 0; $i < $exp; ++$i) {
            $result = self::mulInt($result, $base);
        }
        self::$powCache[$key] = $result;
        return $result;
    }
}

final class MathUtil
{
    public static function boundedPow(int $base, int $exp, int $limit): int
    {
        $result = 1;
        for ($i = 0; $i < $exp; ++$i) {
            if ($base !== 0 && $result > intdiv($limit, max(1, $base))) {
                return $limit + 1;
            }
            $result *= $base;
            if ($result > $limit) {
                return $limit + 1;
            }
        }
        return $result;
    }

    public static function exactNthRoot(int $x, int $n): ?int
    {
        if ($n <= 0 || $x < 0) {
            throw new InvalidArgumentException('Invalid nth-root arguments.');
        }
        if ($x === 0 || $x === 1) {
            return $x;
        }
        $lo = 1;
        $hi = $x;
        while ($lo <= $hi) {
            $mid = intdiv($lo + $hi, 2);
            $pow = self::boundedPow($mid, $n, $x);
            if ($pow === $x) {
                return $mid;
            }
            if ($pow < $x) {
                $lo = $mid + 1;
            } else {
                $hi = $mid - 1;
            }
        }
        return null;
    }

    public static function nthRootFloor(int $x, int $n): int
    {
        if ($n <= 0 || $x < 0) {
            throw new InvalidArgumentException('Invalid nth-root arguments.');
        }
        if ($x === 0 || $x === 1) {
            return $x;
        }
        $lo = 1;
        $hi = $x;
        $best = 1;
        while ($lo <= $hi) {
            $mid = intdiv($lo + $hi, 2);
            $pow = self::boundedPow($mid, $n, $x);
            if ($pow === $x) {
                return $mid;
            }
            if ($pow < $x) {
                $best = $mid;
                $lo = $mid + 1;
            } else {
                $hi = $mid - 1;
            }
        }
        return $best;
    }

    /** @return array{valid:bool,m:?int} */
    public static function validLengthInfo(int $L, int $n, int $minRoot): array
    {
        $m = self::exactNthRoot($L, $n);
        return [
            'valid' => $m !== null && $m >= $minRoot,
            'm' => $m,
        ];
    }

    public static function nearestValidLength(int $target, int $n, int $minRoot, int $Lmax): array
    {
        if ($target < 1 || $Lmax < 1) {
            return [
                'closest_L' => null,
                'm' => null,
                'gap' => null,
                'exact' => false,
                'below_L' => null,
                'above_L' => null,
            ];
        }

        $exact = self::validLengthInfo($target, $n, $minRoot);
        if ($exact['valid'] && $target <= $Lmax) {
            return [
                'closest_L' => $target,
                'm' => $exact['m'],
                'gap' => 0,
                'exact' => true,
                'below_L' => $target,
                'above_L' => $target,
            ];
        }

        $rootFloor = self::nthRootFloor($target, $n);
        $candidates = [];
        $start = max($minRoot, $rootFloor - 3);
        $end = $rootFloor + 4;
        for ($m = $start; $m <= $end; ++$m) {
            $L = self::boundedPow($m, $n, PHP_INT_MAX >> 4);
            if ($L >= 1 && $L <= $Lmax) {
                $candidates[$L] = $m;
            }
        }

        if ($candidates === []) {
            $maxRoot = self::nthRootFloor($Lmax, $n);
            if ($maxRoot >= $minRoot) {
                $L = self::boundedPow($maxRoot, $n, PHP_INT_MAX >> 4);
                return [
                    'closest_L' => $L,
                    'm' => $maxRoot,
                    'gap' => abs($L - $target),
                    'exact' => false,
                    'below_L' => $L,
                    'above_L' => null,
                ];
            }
            return [
                'closest_L' => null,
                'm' => null,
                'gap' => null,
                'exact' => false,
                'below_L' => null,
                'above_L' => null,
            ];
        }

        ksort($candidates);
        $bestL = null;
        $bestM = null;
        $bestGap = null;
        $below = null;
        $above = null;
        foreach ($candidates as $L => $m) {
            if ($L <= $target) {
                $below = $L;
            }
            if ($L >= $target && $above === null) {
                $above = $L;
            }
            $gap = abs($L - $target);
            if ($bestGap === null || $gap < $bestGap || ($gap === $bestGap && $L > (int)$bestL)) {
                $bestGap = $gap;
                $bestL = $L;
                $bestM = $m;
            }
        }

        return [
            'closest_L' => $bestL,
            'm' => $bestM,
            'gap' => $bestGap,
            'exact' => false,
            'below_L' => $below,
            'above_L' => $above,
        ];
    }

    public static function classifyBase(int $p): string
    {
        if ($p < 2) {
            return 'invalid';
        }
        if ($p === 2) {
            return 'prime';
        }
        if ($p % 2 === 0) {
            return 'other';
        }
        $r = (int)floor(sqrt((float)$p));
        for ($k = 3; $k <= $r; $k += 2) {
            if ($p % $k === 0) {
                return 'other';
            }
        }
        return 'prime';
    }

    /** @return list<int> */
    public static function centeredIntegers(int $center, int $min, int $max, int $limit): array
    {
        $out = [];
        $seen = [];
        $push = static function (int $v) use (&$out, &$seen, $min, $max, $limit): void {
            if ($v < $min || $v > $max || isset($seen[$v]) || count($out) >= $limit) {
                return;
            }
            $seen[$v] = true;
            $out[] = $v;
        };

        $push($center);
        for ($d = 1; count($out) < $limit && ($center - $d >= $min || $center + $d <= $max); ++$d) {
            $push($center - $d);
            $push($center + $d);
        }
        return $out;
    }
}

final class Fabric
{
    public static function maxPrimaryLength(int $h, int $s, int $p): int
    {
        if ($h < 2 || $s < 1 || $p < 2) {
            throw new InvalidArgumentException('Require h >= 2, s >= 1, p >= 2.');
        }
        $target = BigDec::powInt($h, $s);
        $lo = 1;
        $hi = 1;
        while (BigDec::cmp(BigDec::powInt($p, $hi), $target) < 0) {
            $hi *= 2;
        }
        while ($lo < $hi) {
            $mid = intdiv($lo + $hi, 2);
            if (BigDec::cmp(BigDec::powInt($p, $mid), $target) >= 0) {
                $hi = $mid;
            } else {
                $lo = $mid + 1;
            }
        }
        return $lo;
    }

    /** @return list<array{L:int,m:int}> */

    public static function maxPrimaryLengthApprox(int $h, int $s, int $p): int
    {
        if ($h < 2 || $s < 1 || $p < 2) {
            throw new InvalidArgumentException('Require h >= 2, s >= 1, p >= 2.');
        }
        $value = ((float)$s * log((float)$h)) / log((float)$p);
        $L = (int)ceil($value - 1e-12);
        return max(1, $L);
    }

    public static function validLengthsUpTo(int $Lmax, int $n, int $minRoot): array
    {
        $out = [];
        for ($L = 1; $L <= $Lmax; ++$L) {
            $info = MathUtil::validLengthInfo($L, $n, $minRoot);
            if ($info['valid']) {
                $out[] = ['L' => $L, 'm' => (int)$info['m']];
            }
        }
        return $out;
    }

    public static function minSecondaryLengthForPrimaryLength(int $h, int $p, int $L): int
    {
        if ($L < 1) {
            throw new InvalidArgumentException('L must be >= 1.');
        }
        $threshold = BigDec::powInt($p, $L - 1);
        $s = 1;
        while (BigDec::cmp(BigDec::powInt($h, $s), $threshold) <= 0) {
            ++$s;
        }
        return $s;
    }
}

final class TextAnalyzer
{
    /** @return array{chars:list<string>,char_length:int,unique_count:int,unique_chars:list<string>,line_count:int,preview:string} */
    public static function analyze(string $text): array
    {
        $chars = self::splitChars($text);
        $charLength = count($chars);
        $uniqueMap = [];
        foreach ($chars as $ch) {
            $uniqueMap[$ch] = true;
        }
        $uniqueChars = array_map(static fn ($x): string => (string)$x, array_keys($uniqueMap));
        usort($uniqueChars, static fn (string $a, string $b): int => $a <=> $b);

        $lineCount = $text === '' ? 0 : substr_count($text, "\n") + 1;
        return [
            'chars' => $chars,
            'char_length' => $charLength,
            'unique_count' => count($uniqueChars),
            'unique_chars' => $uniqueChars,
            'line_count' => $lineCount,
            'preview' => self::previewUniqueChars($uniqueChars),
        ];
    }

    /** @return list<string> */
    private static function splitChars(string $text): array
    {
        if ($text === '') {
            return [];
        }
        if (preg_match_all('/./us', $text, $m) !== false) {
            /** @var list<string> $chars */
            $chars = $m[0];
            return $chars;
        }
        return str_split($text);
    }

    /** @param list<string> $chars */
    private static function previewUniqueChars(array $chars): string
    {
        $parts = [];
        foreach ($chars as $ch) {
            $parts[] = self::escapeChar($ch);
            if (count($parts) >= 80) {
                break;
            }
        }
        return implode(' ', $parts);
    }

    public static function escapeChar(string $ch): string
    {
        return match ($ch) {
            "\n" => '\\n',
            "\r" => '\\r',
            "\t" => '\\t',
            ' ' => '<space>',
            default => $ch,
        };
    }
}

final class MatchEngine
{
    /**
     * @param array<string,mixed> $cfg
     * @param array{char_length:int,unique_count:int} $analysis
     * @return list<array<string,mixed>>
     */
    public static function rank(array $cfg, array $analysis, int $limit): array
    {
        $limit = max(1, min(12, $limit));
        $targetLength = max(1, (int)$analysis['char_length']);
        $observedUnique = max(1, (int)$analysis['unique_count']);

        $dimension = (int)$cfg['dimension'];
        $minRoot = (int)$cfg['min_root'];
        $primaryMin = max(2, (int)$cfg['match_primary_min']);
        $primaryMax = max($primaryMin, (int)$cfg['match_primary_max']);
        $hRadius = max(0, (int)$cfg['match_h_radius']);
        $pCandidates = MathUtil::centeredIntegers($observedUnique, $primaryMin, $primaryMax, max(24, $limit * 4));
        $hCandidates = MathUtil::centeredIntegers($observedUnique, max(2, $observedUnique - $hRadius), max(2, $observedUnique + $hRadius), max(1, 2 * $hRadius + 1));
        if ($hCandidates === []) {
            $hCandidates = [max(2, $observedUnique)];
        }

        return self::rankInternal($targetLength, $observedUnique, $dimension, $minRoot, $limit, $pCandidates, $hCandidates);
    }

    /**
     * @param array<string,mixed> $cfg
     * @param array{char_length:int,unique_count:int} $analysis
     * @param list<int> $pCandidates
     * @param list<int> $hCandidates
     * @return list<array<string,mixed>>
     */
    public static function rankWithCandidates(array $cfg, array $analysis, int $limit, array $pCandidates, array $hCandidates, ?int $dimension = null): array
    {
        $limit = max(1, min(12, $limit));
        $targetLength = max(1, (int)$analysis['char_length']);
        $observedUnique = max(1, (int)$analysis['unique_count']);
        $minRoot = (int)$cfg['min_root'];
        $dimension = $dimension === null ? (int)$cfg['dimension'] : max(1, $dimension);

        $pCandidates = self::sanitizeCandidates($pCandidates, 2);
        $hCandidates = self::sanitizeCandidates($hCandidates, 2);
        if ($pCandidates === [] || $hCandidates === []) {
            return [];
        }

        return self::rankInternal($targetLength, $observedUnique, $dimension, $minRoot, $limit, $pCandidates, $hCandidates);
    }

    /** @param list<int> $candidates */
    private static function sanitizeCandidates(array $candidates, int $min): array
    {
        $out = [];
        $seen = [];
        foreach ($candidates as $value) {
            $value = (int)$value;
            if ($value < $min || isset($seen[$value])) {
                continue;
            }
            $seen[$value] = true;
            $out[] = $value;
        }
        return $out;
    }

    /**
     * @param list<int> $pCandidates
     * @param list<int> $hCandidates
     * @return list<array<string,mixed>>
     */
    private static function rankInternal(int $targetLength, int $observedUnique, int $dimension, int $minRoot, int $limit, array $pCandidates, array $hCandidates): array
    {
        $rows = [];
        foreach ($hCandidates as $h) {
            foreach ($pCandidates as $p) {
                $s = $targetLength;
                $Lmax = Fabric::maxPrimaryLength($h, $s, $p);
                $nearest = MathUtil::nearestValidLength($targetLength, $dimension, $minRoot, $Lmax);
                if ($nearest['closest_L'] === null) {
                    continue;
                }

                $gap = (int)$nearest['gap'];
                $closestL = (int)$nearest['closest_L'];
                $m = (int)$nearest['m'];
                $exact = (bool)$nearest['exact'];
                $hDelta = abs($h - $observedUnique);
                $pDelta = abs($p - $observedUnique);
                $score = ($exact ? -1000000 : 0) + $gap * 1000 + $hDelta * 100 + $pDelta;

                $rows[] = [
                    'score' => $score,
                    'p' => $p,
                    'h' => $h,
                    's' => $s,
                    'n' => $dimension,
                    'Lmax' => $Lmax,
                    'closest_L' => $closestL,
                    'm' => $m,
                    'gap' => $gap,
                    'exact' => $exact,
                    'base_type' => MathUtil::classifyBase($p),
                ];
            }
        }

        usort(
            $rows,
            static function (array $a, array $b): int {
                foreach (['score', 'gap', 'h', 'p'] as $key) {
                    if ($a[$key] < $b[$key]) {
                        return -1;
                    }
                    if ($a[$key] > $b[$key]) {
                        return 1;
                    }
                }
                return 0;
            }
        );

        $unique = [];
        $seen = [];
        foreach ($rows as $row) {
            $key = $row['p'] . '|' . $row['h'] . '|' . $row['s'] . '|' . $row['n'];
            if (isset($seen[$key])) {
                continue;
            }
            $seen[$key] = true;
            $unique[] = $row;
            if (count($unique) >= $limit) {
                break;
            }
        }
        return $unique;
    }
}

final class NDCodex
{
    /** @var array<string,mixed> */
    private array $cfg;

    /** @var array<string,mixed> */
    private array $last = [];

    /** @var list<array<string,mixed>> */
    private array $lastMatches = [];

    /** @var array{line_count:int,char_length:int,unique_count:int,unique_chars:list<string>,preview:string}|null */
    private ?array $lastPasteAnalysis = null;

    public function __construct()
    {
        $this->cfg = [
            'primary_alphabet' => 7,
            'secondary_alphabet' => 10,
            'secondary_length' => 5,
            'dimension' => 2,
            'min_root' => 2,
            'range_s_min' => 1,
            'range_s_max' => 12,
            'match_limit' => 12,
            'match_primary_min' => 2,
            'match_primary_max' => 128,
            'match_h_radius' => 0,
        ];
    }

    public function run(): void
    {
        $this->banner();
        while (true) {
            $line = $this->prompt('nDCodex> ');
            if ($line === null) {
                echo PHP_EOL;
                return;
            }
            $line = trim($line);
            if ($line === '') {
                continue;
            }
            $parts = preg_split('/\s+/', $line) ?: [];
            $cmd = strtolower((string)array_shift($parts));
            try {
                switch ($cmd) {
                    case 'help':
                    case 'commands':
                        $this->help();
                        break;
                    case 'explain':
                        $this->explainCommand($parts);
                        break;
                    case 'show':
                        $this->show();
                        break;
                    case 'set':
                        $this->setCommand($parts);
                        break;
                    case 'reset':
                        $this->__construct();
                        echo "Configuration reset.\n";
                        break;
                    case 'classify':
                        $this->classify();
                        break;
                    case 'lengths':
                        $this->lengthsCommand($parts);
                        break;
                    case 'fabric':
                        $this->fabricCommand($parts);
                        break;
                    case 'witness':
                        $this->witnessCommand($parts);
                        break;
                    case 'paste':
                        $this->pasteCommand($parts);
                        break;
                    case 'match':
                        $this->matchCommand($parts);
                        break;
                    case 'save':
                        $this->saveCommand($parts);
                        break;
                    case 'load':
                        $this->loadCommand($parts);
                        break;
                    case 'export':
                        $this->exportCommand($parts);
                        break;
                    case 'quit':
                    case 'exit':
                        return;
                    default:
                        echo "Unknown command. Type 'help' or 'commands'.\n";
                        break;
                }
            } catch (Throwable $e) {
                echo '[error] ' . $e->getMessage() . PHP_EOL;
            }
        }
    }

    private function banner(): void
    {
        echo str_repeat('=', 108) . PHP_EOL;
        echo "nDCodex.php — n-Dimensional Hash-Length Fabric REPL\n";
        echo str_repeat('=', 108) . PHP_EOL;
        echo "Type 'help' or 'commands' for the command list.\n\n";
    }

    /**
     * @return array<string,array{
     *   usage:string,
     *   summary:string,
     *   group:string,
     *   role:string,
     *   mutates:bool,
     *   config_keys:list<string>,
     *   constraints:list<string>,
     *   result_type:?string,
     *   related:list<string>,
     *   aliases:list<string>,
     *   visible:bool,
     *   details:string
     * }>
     */
    private function commandCatalog(): array
    {
        $allConfigKeys = array_map(static fn ($key): string => (string)$key, array_keys($this->cfg));

        return [
            'help' => [
                'usage' => 'help | commands',
                'summary' => 'Show the grouped command list.',
                'group' => 'Core',
                'role' => 'introspection',
                'mutates' => false,
                'config_keys' => [],
                'constraints' => [],
                'result_type' => null,
                'related' => ['explain help', 'show'],
                'aliases' => ['commands'],
                'visible' => true,
                'details' => 'Lists the available REPL commands grouped by role and reminds you which configuration keys exist.',
            ],
            'explain' => [
                'usage' => 'explain <topic>',
                'summary' => 'Explain a command in the context of the active config.',
                'group' => 'Core',
                'role' => 'interpretation',
                'mutates' => false,
                'config_keys' => [],
                'constraints' => [],
                'result_type' => 'explain',
                'related' => ['help', 'show'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Resolves a command topic, shows what it computes or mutates, lists relevant config inputs, surfaces the formulas involved, and points to likely next commands.',
            ],
            'show' => [
                'usage' => 'show',
                'summary' => 'Display the current configuration and derived state.',
                'group' => 'Core',
                'role' => 'state inspection',
                'mutates' => false,
                'config_keys' => ['primary_alphabet', 'secondary_alphabet', 'secondary_length', 'dimension', 'min_root'],
                'constraints' => ['h^s > p^(L-1)', 'L is valid iff L = m^n and m >= min_root'],
                'result_type' => null,
                'related' => ['lengths', 'fabric traverse'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Prints the active bases and dimensional settings, then derives the current base classification, Lmax, and currently valid primary lengths.',
            ],
            'set' => [
                'usage' => 'set <key> <value>',
                'summary' => 'Update one configuration key.',
                'group' => 'Core',
                'role' => 'state mutation',
                'mutates' => true,
                'config_keys' => $allConfigKeys,
                'constraints' => ['Configured alphabet sizes must stay >= 2.', 'secondary_length, dimension, and min_root must stay >= 1.'],
                'result_type' => null,
                'related' => ['show', 'reset'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Parses one value, writes it into the active session config, and validates the whole configuration before accepting the change.',
            ],
            'reset' => [
                'usage' => 'reset',
                'summary' => 'Restore the default configuration.',
                'group' => 'Core',
                'role' => 'state reset',
                'mutates' => true,
                'config_keys' => $allConfigKeys,
                'constraints' => [],
                'result_type' => null,
                'related' => ['show', 'set'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Replaces the active configuration with the constructor defaults so the REPL returns to its baseline fabric settings.',
            ],
            'classify' => [
                'usage' => 'classify',
                'summary' => 'Classify the current primary alphabet base.',
                'group' => 'Analysis',
                'role' => 'base-type analysis',
                'mutates' => false,
                'config_keys' => ['primary_alphabet'],
                'constraints' => ['Classifies p as prime or other.'],
                'result_type' => null,
                'related' => ['show', 'lengths'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Inspects only the active primary alphabet length p and reports whether the base is prime or composite/other.',
            ],
            'lengths' => [
                'usage' => 'lengths [s]',
                'summary' => 'Compute valid primary lengths for one secondary length.',
                'group' => 'Analysis',
                'role' => 'dimensional validation',
                'mutates' => false,
                'config_keys' => ['primary_alphabet', 'secondary_alphabet', 'secondary_length', 'dimension', 'min_root'],
                'constraints' => ['h^s > p^(L-1)', 'L is valid iff L = m^n and m >= min_root'],
                'result_type' => 'lengths',
                'related' => ['show', 'fabric traverse', 'witness'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Uses the current bases and dimensional validity rule to compute Lmax and enumerate the valid primary lengths up to that bound for one chosen s.',
            ],
            'fabric traverse' => [
                'usage' => 'fabric traverse',
                'summary' => 'Sweep the configured s-range and visualize fabric growth.',
                'group' => 'Analysis',
                'role' => 'structural exploration',
                'mutates' => false,
                'config_keys' => ['primary_alphabet', 'secondary_alphabet', 'dimension', 'min_root', 'range_s_min', 'range_s_max'],
                'constraints' => ['h^s > p^(L-1)', 'L is valid iff L = m^n and m >= min_root'],
                'result_type' => 'fabric_traverse',
                'related' => ['lengths', 'witness', 'export'],
                'aliases' => ['fabric'],
                'visible' => true,
                'details' => 'Traverses the configured secondary-length range, computes Lmax and valid lengths for each row, then reports valid-count and density charts.',
            ],
            'witness' => [
                'usage' => 'witness <L>',
                'summary' => 'Find the minimal s that realizes a target primary length.',
                'group' => 'Analysis',
                'role' => 'inverse constraint solving',
                'mutates' => false,
                'config_keys' => ['primary_alphabet', 'secondary_alphabet', 'dimension', 'min_root'],
                'constraints' => ['h^s > p^(L-1)', 'L is valid iff L = m^n and m >= min_root'],
                'result_type' => 'witness',
                'related' => ['lengths', 'fabric traverse'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Checks whether a target L is dimension-valid and then solves for the minimal secondary length s needed to witness that primary length.',
            ],
            'paste' => [
                'usage' => 'paste [limit]',
                'summary' => 'Paste multiline text, analyze it, and rank matching configs.',
                'group' => 'Matching',
                'role' => 'input encoding',
                'mutates' => true,
                'config_keys' => ['dimension', 'min_root', 'match_limit', 'match_primary_min', 'match_primary_max', 'match_h_radius'],
                'constraints' => ['char_length becomes the target s.', 'unique_count anchors h and p candidate search.', 'Ranking prefers exact valid hits, then lower gap, then closer h/p proximity to observed unique characters.'],
                'result_type' => 'paste_match',
                'related' => ['match', 'match apply', 'match refine', 'export'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Captures multiline text until .end, analyzes character length and uniqueness, then ranks nearby mathematically consistent configurations for the active dimension.',
            ],
            'match' => [
                'usage' => 'match apply <rank> | match refine <rank>',
                'summary' => 'Operate on the cached ranked match list.',
                'group' => 'Matching',
                'role' => 'adaptive optimisation',
                'mutates' => true,
                'config_keys' => ['primary_alphabet', 'secondary_alphabet', 'secondary_length', 'dimension', 'min_root', 'match_limit', 'match_primary_min', 'match_primary_max', 'match_h_radius'],
                'constraints' => ['Ranking prefers exact valid hits, then lower gap, then closer h/p proximity to observed unique characters.'],
                'result_type' => null,
                'related' => ['paste', 'match apply', 'match refine'],
                'aliases' => [],
                'visible' => false,
                'details' => 'Acts on the current ranked results produced by paste or a prior refine step, either applying one row to config or reranking locally around it.',
            ],
            'match apply' => [
                'usage' => 'match apply <rank>',
                'summary' => 'Apply a ranked match to the active session config.',
                'group' => 'Matching',
                'role' => 'adaptive optimisation',
                'mutates' => true,
                'config_keys' => ['primary_alphabet', 'secondary_alphabet', 'secondary_length', 'dimension'],
                'constraints' => ['Requires ranked matches from paste or match refine.'],
                'result_type' => null,
                'related' => ['paste', 'match refine', 'show'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Takes one ranked row and writes p, h, s, and n back into the active configuration after validating the resulting session state.',
            ],
            'match refine' => [
                'usage' => 'match refine <rank>',
                'summary' => 'Locally rerank around one cached match.',
                'group' => 'Matching',
                'role' => 'local reranking',
                'mutates' => true,
                'config_keys' => ['dimension', 'min_root', 'match_limit', 'match_primary_min', 'match_primary_max', 'match_h_radius'],
                'constraints' => ['Requires a prior paste analysis.', 'Keeps s fixed to the pasted char_length and n fixed to the selected row.', 'Local window uses p centered on the selected row with limit 7 and h centered on the selected row with radius max(1, match_h_radius).', 'Ranking prefers exact valid hits, then lower gap, then closer h/p proximity to observed unique characters.'],
                'result_type' => 'match_refine',
                'related' => ['paste', 'match apply', 'export'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Builds a narrow candidate window around one ranked row, reruns the ranking logic against the same pasted analysis, and replaces the cached ranked result list without auto-applying config.',
            ],
            'save' => [
                'usage' => 'save [file.json]',
                'summary' => 'Persist the current configuration to JSON.',
                'group' => 'Persistence',
                'role' => 'state persistence',
                'mutates' => false,
                'config_keys' => $allConfigKeys,
                'constraints' => [],
                'result_type' => null,
                'related' => ['load', 'show'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Writes the active configuration array to disk so it can be restored in a later session.',
            ],
            'load' => [
                'usage' => 'load [file.json]',
                'summary' => 'Load configuration from JSON.',
                'group' => 'Persistence',
                'role' => 'state restoration',
                'mutates' => true,
                'config_keys' => $allConfigKeys,
                'constraints' => ['Only recognized configuration keys are applied.', 'Loaded config is validated before it becomes active.'],
                'result_type' => null,
                'related' => ['save', 'show'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Reads a JSON config file, applies recognized keys onto the active session, and validates the resulting configuration before accepting it.',
            ],
            'export' => [
                'usage' => 'export [file.json]',
                'summary' => 'Export the last cached result payload.',
                'group' => 'Persistence',
                'role' => 'data extraction',
                'mutates' => false,
                'config_keys' => [],
                'constraints' => ['Requires a cached result payload in $last.'],
                'result_type' => null,
                'related' => ['paste', 'fabric traverse', 'explain'],
                'aliases' => [],
                'visible' => true,
                'details' => 'Serializes the most recent structured result payload, including richer explain, fabric traversal, and match refinement metadata.',
            ],
            'quit' => [
                'usage' => 'quit | exit',
                'summary' => 'Terminate the REPL session.',
                'group' => 'Core',
                'role' => 'termination',
                'mutates' => false,
                'config_keys' => [],
                'constraints' => [],
                'result_type' => null,
                'related' => ['save'],
                'aliases' => ['exit'],
                'visible' => true,
                'details' => 'Ends the current session immediately without mutating config or writing any files.',
            ],
        ];
    }

    /** @return array<string,string> */
    private function configKeyNotes(): array
    {
        return [
            'primary_alphabet' => '>= 2    primary output alphabet length p',
            'secondary_alphabet' => '>= 2    secondary input alphabet length h',
            'secondary_length' => '>= 1    secondary string length s',
            'dimension' => '>= 1    validity dimension n',
            'min_root' => '>= 1    require L = m^n with m >= min_root',
            'range_s_min' => '>= 1    start of traversal range',
            'range_s_max' => '>= 1    end of traversal range',
            'match_limit' => '1..12   default ranked results count',
            'match_primary_min' => '>= 2    minimum candidate primary alphabet',
            'match_primary_max' => '>= 2    maximum candidate primary alphabet',
            'match_h_radius' => '>= 0    how far h may vary from observed unique chars',
        ];
    }

    private function help(): void
    {
        $catalog = $this->commandCatalog();
        $groups = ['Core', 'Analysis', 'Matching', 'Persistence'];

        echo "Commands\n";
        echo "--------\n";
        foreach ($groups as $group) {
            $printed = false;
            foreach ($catalog as $meta) {
                if (!$meta['visible'] || $meta['group'] !== $group) {
                    continue;
                }
                if (!$printed) {
                    echo $group . PHP_EOL;
                    echo str_repeat('-', strlen($group)) . PHP_EOL;
                    $printed = true;
                }
                echo str_pad($meta['usage'], 28) . $meta['summary'] . ' [' . $meta['role'] . ']' . PHP_EOL;
            }
            if ($printed) {
                echo PHP_EOL;
            }
        }

        echo "Keys\n";
        echo "----\n";
        foreach ($this->configKeyNotes() as $key => $note) {
            echo str_pad($key, 19) . $note . PHP_EOL;
        }
    }

    /** @return list<string> */
    private function explainTopics(): array
    {
        return [
            'help',
            'commands',
            'explain',
            'show',
            'set',
            'reset',
            'classify',
            'lengths',
            'fabric',
            'fabric traverse',
            'witness',
            'paste',
            'match',
            'match apply',
            'match refine',
            'save',
            'load',
            'export',
            'quit',
            'exit',
        ];
    }

    private function explainCommand(array $parts): void
    {
        $requested = strtolower(trim(implode(' ', $parts)));
        if ($requested === '') {
            throw new InvalidArgumentException('Usage: explain <topic>. Supported topics: ' . implode(', ', $this->explainTopics()));
        }

        $topicMap = [];
        foreach ($this->commandCatalog() as $canonical => $meta) {
            $topicMap[$canonical] = $canonical;
            foreach ($meta['aliases'] as $alias) {
                $topicMap[$alias] = $canonical;
            }
        }

        if (!isset($topicMap[$requested])) {
            throw new InvalidArgumentException('Unknown explain topic: ' . $requested . '. Supported topics: ' . implode(', ', $this->explainTopics()));
        }

        $payload = $this->buildExplainPayload($topicMap[$requested], $requested);
        $this->last = $payload;

        echo str_repeat('-', 108) . PHP_EOL;
        echo 'topic                  : ' . $payload['topic'] . PHP_EOL;
        echo 'usage                  : ' . $payload['usage'] . PHP_EOL;
        echo 'summary                : ' . $payload['summary'] . PHP_EOL;
        echo 'group / role           : ' . $payload['group'] . ' / ' . $payload['role'] . PHP_EOL;
        echo 'mutates session        : ' . ($payload['mutates'] ? 'true' : 'false') . PHP_EOL;
        echo 'what it does           : ' . $payload['details'] . PHP_EOL;
        echo 'active config inputs   : ' . $this->formatKeyValuePairs($payload['config_snapshot']) . PHP_EOL;
        echo 'constraints            : ' . $this->formatStringList($payload['constraints']) . PHP_EOL;
        foreach ($payload['context'] as $label => $value) {
            echo str_pad($label, 23) . ': ' . $value . PHP_EOL;
        }
        echo 'cached result type     : ' . ($payload['result_type'] ?? '(none)') . PHP_EOL;
        echo 'related commands       : ' . $this->formatStringList($payload['related']) . PHP_EOL;
        echo str_repeat('-', 108) . PHP_EOL;
    }

    /** @return array<string,mixed> */
    private function buildExplainPayload(string $canonical, string $requested): array
    {
        $meta = $this->commandCatalog()[$canonical];
        $configSnapshot = [];
        foreach ($meta['config_keys'] as $key) {
            if (array_key_exists($key, $this->cfg)) {
                $configSnapshot[$key] = $this->cfg[$key];
            }
        }

        return [
            'type' => 'explain',
            'requested_topic' => $requested,
            'topic' => $canonical,
            'usage' => $meta['usage'],
            'summary' => $meta['summary'],
            'group' => $meta['group'],
            'role' => $meta['role'],
            'mutates' => $meta['mutates'],
            'details' => $meta['details'],
            'config_snapshot' => $configSnapshot,
            'constraints' => $meta['constraints'],
            'context' => $this->buildExplainContext($canonical),
            'result_type' => $meta['result_type'],
            'related' => $meta['related'],
        ];
    }

    /** @return array<string,string> */
    private function buildExplainContext(string $canonical): array
    {
        $context = [];

        switch ($canonical) {
            case 'help':
                $context['supports aliases'] = 'commands is a direct alias for help.';
                break;
            case 'explain':
                $context['supported topics'] = implode(', ', $this->explainTopics());
                break;
            case 'show':
                $p = (int)$this->cfg['primary_alphabet'];
                $h = (int)$this->cfg['secondary_alphabet'];
                $s = (int)$this->cfg['secondary_length'];
                $n = (int)$this->cfg['dimension'];
                $minRoot = (int)$this->cfg['min_root'];
                $Lmax = Fabric::maxPrimaryLength($h, $s, $p);
                $valid = Fabric::validLengthsUpTo($Lmax, $n, $minRoot);
                $context['current derived state'] = 'base=' . MathUtil::classifyBase($p) . ', Lmax=' . $Lmax . ', valid=' . $this->formatValidList($valid);
                break;
            case 'set':
                $context['configurable keys'] = implode(', ', array_keys($this->configKeyNotes()));
                break;
            case 'reset':
                $context['reset target'] = 'constructor defaults for every config key';
                break;
            case 'classify':
                $context['current base'] = 'p=' . $this->cfg['primary_alphabet'];
                break;
            case 'lengths':
                $context['current secondary s'] = (string)$this->cfg['secondary_length'];
                break;
            case 'fabric traverse':
                $context['current range'] = 's=' . $this->cfg['range_s_min'] . '..' . $this->cfg['range_s_max'];
                $context['chart metrics'] = 'Each row reports Lmax, valid_count, and density = valid_count / Lmax.';
                break;
            case 'witness':
                $context['current bases'] = 'h=' . $this->cfg['secondary_alphabet'] . ', p=' . $this->cfg['primary_alphabet'];
                break;
            case 'paste':
                $context['current rank limit'] = (string)$this->cfg['match_limit'];
                $context['last paste cached'] = $this->lastPasteAnalysis === null ? 'false' : 'true';
                break;
            case 'match':
                $context['available ranked matches'] = $this->lastMatches === [] ? 'none (run paste first)' : (string)count($this->lastMatches);
                $context['ranking factors'] = 'exact valid hit, gap, |h - unique_count|, |p - unique_count|';
                break;
            case 'match apply':
                $context['available ranked matches'] = $this->lastMatches === [] ? 'none (run paste first)' : (string)count($this->lastMatches);
                $context['writes keys'] = 'primary_alphabet, secondary_alphabet, secondary_length, dimension';
                break;
            case 'match refine':
                $context['available ranked matches'] = $this->lastMatches === [] ? 'none (run paste first)' : (string)count($this->lastMatches);
                $context['last paste analysis'] = $this->lastPasteAnalysis === null
                    ? 'none (run paste first)'
                    : 'char_length=' . $this->lastPasteAnalysis['char_length'] . ', unique_count=' . $this->lastPasteAnalysis['unique_count'];
                $context['local window'] = 'p centered on selected row (limit 7); h centered on selected row with radius ' . max(1, (int)$this->cfg['match_h_radius']);
                break;
            case 'save':
                $context['writes file'] = 'current config only';
                break;
            case 'load':
                $context['recognized keys'] = implode(', ', array_keys($this->configKeyNotes()));
                break;
            case 'export':
                $context['current cached result'] = $this->last === [] ? 'none' : (string)$this->last['type'];
                break;
            case 'quit':
                $context['session effect'] = 'terminates immediately';
                break;
        }

        return $context;
    }

    private function show(): void
    {
        $p = (int)$this->cfg['primary_alphabet'];
        $h = (int)$this->cfg['secondary_alphabet'];
        $s = (int)$this->cfg['secondary_length'];
        $n = (int)$this->cfg['dimension'];
        $minRoot = (int)$this->cfg['min_root'];
        $Lmax = Fabric::maxPrimaryLength($h, $s, $p);
        $valid = Fabric::validLengthsUpTo($Lmax, $n, $minRoot);

        echo str_repeat('-', 108) . PHP_EOL;
        foreach ($this->cfg as $k => $v) {
            echo str_pad($k, 22) . ' : ' . (is_bool($v) ? ($v ? 'true' : 'false') : (string)$v) . PHP_EOL;
        }
        echo str_repeat('-', 108) . PHP_EOL;
        echo 'base classification     : ' . MathUtil::classifyBase($p) . PHP_EOL;
        echo 'max primary length      : ' . $Lmax . PHP_EOL;
        echo 'valid lengths @ s=' . $s . '    : ' . $this->formatValidList($valid) . PHP_EOL;
        echo 'fabric rule             : L is valid iff L = m^n and m >= min_root' . PHP_EOL;
        echo str_repeat('-', 108) . PHP_EOL;
    }

    private function setCommand(array $parts): void
    {
        if (count($parts) < 2) {
            throw new InvalidArgumentException('Usage: set <key> <value>');
        }
        $key = (string)$parts[0];
        $value = implode(' ', array_slice($parts, 1));
        if (!array_key_exists($key, $this->cfg)) {
            throw new InvalidArgumentException('Unknown key: ' . $key);
        }
        $this->cfg[$key] = $this->coerceValue($this->cfg[$key], $value);
        $this->validateConfig();
        echo 'Updated ' . $key . '.' . PHP_EOL;
    }

    private function classify(): void
    {
        $p = (int)$this->cfg['primary_alphabet'];
        echo 'primary_alphabet=' . $p . ' is ' . MathUtil::classifyBase($p) . PHP_EOL;
    }

    private function lengthsCommand(array $parts): void
    {
        $s = isset($parts[0]) ? max(1, (int)$parts[0]) : (int)$this->cfg['secondary_length'];
        $h = (int)$this->cfg['secondary_alphabet'];
        $p = (int)$this->cfg['primary_alphabet'];
        $n = (int)$this->cfg['dimension'];
        $minRoot = (int)$this->cfg['min_root'];

        $Lmax = Fabric::maxPrimaryLength($h, $s, $p);
        $valid = Fabric::validLengthsUpTo($Lmax, $n, $minRoot);

        echo 'secondary length s      : ' . $s . PHP_EOL;
        echo 'max primary length      : ' . $Lmax . PHP_EOL;
        echo 'valid primary lengths   : ' . $this->formatValidList($valid) . PHP_EOL;

        $this->last = [
            'type' => 'lengths',
            'secondary_length' => $s,
            'Lmax' => $Lmax,
            'valid' => $valid,
            'config' => $this->cfg,
        ];
    }

    private function fabricCommand(array $parts): void
    {
        $sub = strtolower((string)($parts[0] ?? 'traverse'));
        if ($sub !== 'traverse') {
            throw new InvalidArgumentException('Usage: fabric traverse');
        }

        $h = (int)$this->cfg['secondary_alphabet'];
        $p = (int)$this->cfg['primary_alphabet'];
        $n = (int)$this->cfg['dimension'];
        $minRoot = (int)$this->cfg['min_root'];
        $sMin = (int)$this->cfg['range_s_min'];
        $sMax = (int)$this->cfg['range_s_max'];

        if ($sMin > $sMax) {
            throw new InvalidArgumentException('range_s_min must be <= range_s_max.');
        }

        echo str_pad('s', 8) . str_pad('Lmax', 10) . str_pad('count', 8) . str_pad('density', 10) . "valid primary lengths\n";
        echo str_repeat('-', 108) . PHP_EOL;
        $rows = [];
        for ($s = $sMin; $s <= $sMax; ++$s) {
            $Lmax = Fabric::maxPrimaryLength($h, $s, $p);
            $valid = Fabric::validLengthsUpTo($Lmax, $n, $minRoot);
            $validCount = count($valid);
            $density = $Lmax > 0 ? $validCount / $Lmax : 0.0;
            echo str_pad((string)$s, 8)
                . str_pad((string)$Lmax, 10)
                . str_pad((string)$validCount, 8)
                . str_pad($this->formatPercent($density), 10)
                . $this->formatValidList($valid)
                . PHP_EOL;
            $rows[] = [
                's' => $s,
                'Lmax' => $Lmax,
                'valid' => $valid,
                'valid_count' => $validCount,
                'density' => $density,
            ];
        }

        $summary = $this->summarizeFabricRows($rows);
        $this->renderFabricCharts($rows, $summary);

        $this->last = [
            'type' => 'fabric_traverse',
            'rows' => $rows,
            'summary' => $summary,
            'config' => $this->cfg,
        ];
    }

    private function witnessCommand(array $parts): void
    {
        if (!isset($parts[0])) {
            throw new InvalidArgumentException('Usage: witness <L>');
        }
        $L = max(1, (int)$parts[0]);
        $h = (int)$this->cfg['secondary_alphabet'];
        $p = (int)$this->cfg['primary_alphabet'];
        $n = (int)$this->cfg['dimension'];
        $minRoot = (int)$this->cfg['min_root'];
        $valid = MathUtil::validLengthInfo($L, $n, $minRoot);
        $s = Fabric::minSecondaryLengthForPrimaryLength($h, $p, $L);

        echo 'target primary length   : ' . $L . PHP_EOL;
        echo 'dimension-valid         : ' . ($valid['valid'] ? 'true' : 'false') . PHP_EOL;
        echo 'root m                  : ' . ($valid['m'] === null ? 'n/a' : (string)$valid['m']) . PHP_EOL;
        echo 'minimal secondary s     : ' . $s . PHP_EOL;
        echo 'condition               : h^s > p^(L-1)' . PHP_EOL;

        $this->last = [
            'type' => 'witness',
            'L' => $L,
            'valid' => $valid,
            'minimal_secondary_length' => $s,
            'config' => $this->cfg,
        ];
    }

    private function pasteCommand(array $parts): void
    {
        $limit = isset($parts[0]) ? (int)$parts[0] : (int)$this->cfg['match_limit'];
        $limit = max(1, min(12, $limit));

        echo "Paste multiline text. End with a line containing only .end\n";
        echo "Enter .cancel on its own line to abort.\n";

        $lines = [];
        while (true) {
            $line = $this->prompt('... ');
            if ($line === null) {
                echo PHP_EOL;
                return;
            }
            if ($line === '.cancel') {
                echo "Paste cancelled.\n";
                return;
            }
            if ($line === '.end') {
                break;
            }
            $lines[] = $line;
        }

        $text = implode("\n", $lines);
        $analysis = $this->compactAnalysis(TextAnalyzer::analyze($text));
        $matches = MatchEngine::rank($this->cfg, $analysis, $limit);
        $this->lastPasteAnalysis = $analysis;
        $this->lastMatches = $matches;
        $this->renderMatchResults($analysis, $matches);

        $this->last = [
            'type' => 'paste_match',
            'analysis' => $analysis,
            'matches' => $matches,
            'config' => $this->cfg,
        ];
    }

    private function matchCommand(array $parts): void
    {
        $sub = strtolower((string)($parts[0] ?? ''));
        $rank = isset($parts[1]) ? (int)$parts[1] : 0;

        switch ($sub) {
            case 'apply':
                if (!isset($parts[1])) {
                    throw new InvalidArgumentException('Usage: match apply <rank>');
                }
                $row = $this->requireMatchRow($rank);
                $this->cfg['primary_alphabet'] = (int)$row['p'];
                $this->cfg['secondary_alphabet'] = (int)$row['h'];
                $this->cfg['secondary_length'] = (int)$row['s'];
                $this->cfg['dimension'] = (int)$row['n'];
                $this->validateConfig();
                echo 'Applied match #' . $rank . ' -> p=' . $row['p'] . ', h=' . $row['h'] . ', s=' . $row['s'] . ', n=' . $row['n'] . PHP_EOL;
                break;
            case 'refine':
                if (!isset($parts[1])) {
                    throw new InvalidArgumentException('Usage: match refine <rank>');
                }
                if ($this->lastPasteAnalysis === null) {
                    throw new RuntimeException('No paste analysis available yet. Run paste first.');
                }
                $row = $this->requireMatchRow($rank);
                $primaryMin = max(2, (int)$this->cfg['match_primary_min']);
                $primaryMax = max($primaryMin, (int)$this->cfg['match_primary_max']);
                $pCandidates = MathUtil::centeredIntegers((int)$row['p'], $primaryMin, $primaryMax, 7);
                $radius = max(1, (int)$this->cfg['match_h_radius']);
                $hCenter = max(2, (int)$row['h']);
                $hCandidates = MathUtil::centeredIntegers($hCenter, max(2, $hCenter - $radius), max(2, $hCenter + $radius), max(3, 2 * $radius + 1));
                $matches = MatchEngine::rankWithCandidates($this->cfg, $this->lastPasteAnalysis, (int)$this->cfg['match_limit'], $pCandidates, $hCandidates, (int)$row['n']);
                if ($matches === []) {
                    throw new RuntimeException('No refined matches found in the local search window.');
                }

                $this->lastMatches = $matches;
                $context = [
                    'Refined match candidates (local rerank)',
                    'refinement source      : rank #' . $rank . ' -> p=' . $row['p'] . ', h=' . $row['h'] . ', s=' . $row['s'] . ', n=' . $row['n'],
                    'candidate window       : p=' . implode(', ', $pCandidates) . ' | h=' . implode(', ', $hCandidates),
                ];
                $this->renderMatchResults($this->lastPasteAnalysis, $matches, $context, (int)$row['n']);
                $this->last = [
                    'type' => 'match_refine',
                    'source_rank' => $rank,
                    'source_row' => $row,
                    'analysis' => $this->lastPasteAnalysis,
                    'candidate_window' => [
                        'p' => $pCandidates,
                        'h' => $hCandidates,
                    ],
                    'matches' => $matches,
                    'config' => $this->cfg,
                ];
                break;
            default:
                throw new InvalidArgumentException('Usage: match apply <rank> | match refine <rank>');
        }
    }

    private function saveCommand(array $parts): void
    {
        $file = (string)($parts[0] ?? 'ndcodex_config.json');
        file_put_contents($file, json_encode($this->cfg, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES));
        echo 'Saved config to ' . $file . PHP_EOL;
    }

    private function loadCommand(array $parts): void
    {
        $file = (string)($parts[0] ?? 'ndcodex_config.json');
        if (!is_file($file)) {
            throw new RuntimeException('Config file not found: ' . $file);
        }
        $data = json_decode((string)file_get_contents($file), true, 512, JSON_THROW_ON_ERROR);
        if (!is_array($data)) {
            throw new RuntimeException('Invalid config payload.');
        }
        foreach ($data as $k => $v) {
            if (array_key_exists($k, $this->cfg)) {
                $this->cfg[$k] = $v;
            }
        }
        $this->validateConfig();
        echo 'Loaded config from ' . $file . PHP_EOL;
    }

    private function exportCommand(array $parts): void
    {
        if ($this->last === []) {
            throw new RuntimeException('Nothing to export yet.');
        }
        $file = (string)($parts[0] ?? 'ndcodex_export.json');
        file_put_contents($file, json_encode($this->last, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES));
        echo 'Exported last result to ' . $file . PHP_EOL;
    }

    private function validateConfig(): void
    {
        foreach (['primary_alphabet', 'secondary_alphabet', 'secondary_length', 'dimension', 'min_root', 'range_s_min', 'range_s_max', 'match_limit', 'match_primary_min', 'match_primary_max', 'match_h_radius'] as $key) {
            $this->cfg[$key] = (int)$this->cfg[$key];
        }
        if ($this->cfg['primary_alphabet'] < 2 || $this->cfg['secondary_alphabet'] < 2) {
            throw new InvalidArgumentException('Alphabet lengths must be >= 2.');
        }
        if ($this->cfg['secondary_length'] < 1 || $this->cfg['dimension'] < 1 || $this->cfg['min_root'] < 1) {
            throw new InvalidArgumentException('secondary_length, dimension, and min_root must be >= 1.');
        }
        if ($this->cfg['range_s_min'] < 1 || $this->cfg['range_s_max'] < 1) {
            throw new InvalidArgumentException('Traversal bounds must be >= 1.');
        }
        if ($this->cfg['match_limit'] < 1 || $this->cfg['match_limit'] > 12) {
            throw new InvalidArgumentException('match_limit must be between 1 and 12.');
        }
        if ($this->cfg['match_primary_min'] < 2 || $this->cfg['match_primary_max'] < $this->cfg['match_primary_min']) {
            throw new InvalidArgumentException('Invalid match_primary_min/max bounds.');
        }
        if ($this->cfg['match_h_radius'] < 0) {
            throw new InvalidArgumentException('match_h_radius must be >= 0.');
        }
    }

    private function coerceValue(mixed $current, string $raw): mixed
    {
        if (is_int($current)) {
            if (!preg_match('/^-?\d+$/', trim($raw))) {
                throw new InvalidArgumentException('Expected integer value.');
            }
            return (int)$raw;
        }
        if (is_bool($current)) {
            $x = strtolower(trim($raw));
            if (in_array($x, ['1', 'true', 'yes', 'on'], true)) {
                return true;
            }
            if (in_array($x, ['0', 'false', 'no', 'off'], true)) {
                return false;
            }
            throw new InvalidArgumentException('Expected boolean value.');
        }
        return $raw;
    }

    /** @param list<array{L:int,m:int}> $valid */
    private function formatValidList(array $valid): string
    {
        if ($valid === []) {
            return '(none)';
        }
        $parts = [];
        foreach ($valid as $row) {
            $parts[] = $row['L'] . '(m=' . $row['m'] . ')';
            if (count($parts) >= 18) {
                $parts[] = '...';
                break;
            }
        }
        return implode(', ', $parts);
    }

    /** @param array{chars:list<string>,char_length:int,unique_count:int,unique_chars:list<string>,line_count:int,preview:string} $analysis */
    private function compactAnalysis(array $analysis): array
    {
        return [
            'line_count' => (int)$analysis['line_count'],
            'char_length' => (int)$analysis['char_length'],
            'unique_count' => (int)$analysis['unique_count'],
            'unique_chars' => $analysis['unique_chars'],
            'preview' => (string)$analysis['preview'],
        ];
    }

    /** @param list<string> $contextLines */
    private function renderMatchResults(array $analysis, array $matches, array $contextLines = [], ?int $dimension = null): void
    {
        foreach ($contextLines as $line) {
            echo $line . PHP_EOL;
        }
        echo 'pasted text lines      : ' . $analysis['line_count'] . PHP_EOL;
        echo 'pasted char length     : ' . $analysis['char_length'] . PHP_EOL;
        echo 'observed unique chars  : ' . $analysis['unique_count'] . PHP_EOL;
        echo 'effective sec alphabet : ' . max(2, (int)$analysis['unique_count']) . PHP_EOL;
        echo 'active dimension       : ' . ($dimension ?? (int)$this->cfg['dimension']) . PHP_EOL;
        echo 'unique char preview    : ' . $analysis['preview'] . PHP_EOL;
        echo str_repeat('-', 108) . PHP_EOL;
        echo str_pad('#', 4)
            . str_pad('p', 8)
            . str_pad('h', 8)
            . str_pad('s', 12)
            . str_pad('n', 6)
            . str_pad('Lmax', 10)
            . str_pad('closest_L', 12)
            . str_pad('m', 8)
            . str_pad('gap', 8)
            . str_pad('exact', 8)
            . "type\n";
        echo str_repeat('-', 108) . PHP_EOL;

        foreach ($matches as $i => $row) {
            echo str_pad((string)($i + 1), 4)
                . str_pad((string)$row['p'], 8)
                . str_pad((string)$row['h'], 8)
                . str_pad((string)$row['s'], 12)
                . str_pad((string)$row['n'], 6)
                . str_pad((string)$row['Lmax'], 10)
                . str_pad((string)$row['closest_L'], 12)
                . str_pad((string)$row['m'], 8)
                . str_pad((string)$row['gap'], 8)
                . str_pad($row['exact'] ? 'true' : 'false', 8)
                . $row['base_type']
                . PHP_EOL;
        }
        echo str_repeat('-', 108) . PHP_EOL;
        echo "Use 'match apply <rank>' to load one of these configs into the active session.\n";
    }

    /** @return array<string,mixed> */
    private function requireMatchRow(int $rank): array
    {
        if ($this->lastMatches === []) {
            throw new RuntimeException('No ranked matches available yet. Run paste first.');
        }
        if ($rank < 1 || $rank > count($this->lastMatches)) {
            throw new InvalidArgumentException('Rank out of range.');
        }
        return $this->lastMatches[$rank - 1];
    }

    /** @param array<string,mixed> $summary */
    private function renderFabricCharts(array $rows, array $summary): void
    {
        echo str_repeat('-', 108) . PHP_EOL;
        echo 'range summary          : s=' . $summary['start_s'] . '..' . $summary['end_s']
            . ' | max Lmax=' . $summary['max_Lmax'] . ' @ s=' . $summary['max_Lmax_s']
            . ' | best density=' . $this->formatPercent((float)$summary['best_density']) . ' @ s=' . $summary['best_density_s']
            . PHP_EOL;
        echo 'max valid-count        : ' . $summary['max_valid_count'] . ' @ s=' . $summary['max_valid_count_s'] . PHP_EOL;
        echo str_repeat('-', 108) . PHP_EOL;
        echo "Lmax growth chart\n";
        echo "-----------------\n";
        $barWidth = 32;
        $scaleMax = max(1, (int)$summary['max_Lmax']);
        foreach ($rows as $row) {
            $filled = (int)round((((int)$row['Lmax']) / $scaleMax) * $barWidth);
            echo str_pad('s=' . $row['s'], 8) . '|' . $this->buildBar($filled, $barWidth) . '| ' . $row['Lmax'] . PHP_EOL;
        }
        echo str_repeat('-', 108) . PHP_EOL;
        echo "Valid-length density chart\n";
        echo "--------------------------\n";
        foreach ($rows as $row) {
            $filled = (int)round(((float)$row['density']) * $barWidth);
            echo str_pad('s=' . $row['s'], 8)
                . '|'
                . $this->buildBar($filled, $barWidth)
                . '| '
                . $this->formatPercent((float)$row['density'])
                . ' (' . $row['valid_count'] . '/' . $row['Lmax'] . ')'
                . PHP_EOL;
        }
        echo str_repeat('-', 108) . PHP_EOL;
    }

    /** @return array<string,int|float> */
    private function summarizeFabricRows(array $rows): array
    {
        $first = $rows[0];
        $summary = [
            'start_s' => (int)$first['s'],
            'end_s' => (int)$first['s'],
            'max_Lmax' => (int)$first['Lmax'],
            'max_Lmax_s' => (int)$first['s'],
            'best_density' => (float)$first['density'],
            'best_density_s' => (int)$first['s'],
            'max_valid_count' => (int)$first['valid_count'],
            'max_valid_count_s' => (int)$first['s'],
        ];

        foreach ($rows as $row) {
            $summary['end_s'] = (int)$row['s'];
            if ((int)$row['Lmax'] > $summary['max_Lmax']) {
                $summary['max_Lmax'] = (int)$row['Lmax'];
                $summary['max_Lmax_s'] = (int)$row['s'];
            }
            if ((float)$row['density'] > $summary['best_density']) {
                $summary['best_density'] = (float)$row['density'];
                $summary['best_density_s'] = (int)$row['s'];
            }
            if ((int)$row['valid_count'] > $summary['max_valid_count']) {
                $summary['max_valid_count'] = (int)$row['valid_count'];
                $summary['max_valid_count_s'] = (int)$row['s'];
            }
        }

        return $summary;
    }

    private function buildBar(int $filled, int $width): string
    {
        $filled = max(0, min($width, $filled));
        return str_repeat('#', $filled) . str_repeat('.', $width - $filled);
    }

    private function formatPercent(float $ratio): string
    {
        return number_format($ratio * 100, 2) . '%';
    }

    /** @param array<string,mixed> $pairs */
    private function formatKeyValuePairs(array $pairs): string
    {
        if ($pairs === []) {
            return '(none)';
        }
        $parts = [];
        foreach ($pairs as $key => $value) {
            if (is_bool($value)) {
                $value = $value ? 'true' : 'false';
            }
            $parts[] = $key . '=' . $value;
        }
        return implode(', ', $parts);
    }

    /** @param list<string> $items */
    private function formatStringList(array $items): string
    {
        return $items === [] ? '(none)' : implode('; ', $items);
    }

    private function prompt(string $text): ?string
    {
        if (function_exists('readline')) {
            $line = readline($text);
            if ($line === false) {
                return null;
            }
            if ($text === 'nDCodex> ' && trim($line) !== '') {
                readline_add_history($line);
            }
            return $line;
        }
        echo $text;
        $line = fgets(STDIN);
        return $line === false ? null : rtrim($line, "\r\n");
    }
}

$repl = new NDCodex();
$repl->run();

```

<a id="file-12"></a>
### [12] `8.py`

- **Bytes:** `287351`
- **Type:** `text`

```python
"""
Graphical Codex CLI-REPL OS - Tkinter Edition

A pure-stdlib Tkinter kernel shell where services are controlled only through
commands submitted to the blue CLI engine. The interface keeps the Codex data
model from the graphical editor but converts the application into a general
purpose, kernel-based, API-routable REPL operating surface.

Run:
    python graphical_codex_cli_repl_os.py

Core rule:
    Only the REPL Console and the blue CLI engine are visible. Kernel services
    can only activate or execute through commands submitted to the blue engine.
"""

from __future__ import annotations

import copy
import datetime as _dt
import hashlib
import itertools
import json
import os
import re
import shlex
import shutil
import subprocess
import tkinter as tk
from dataclasses import asdict, dataclass, field
from pathlib import Path
from tkinter import ttk
from tkinter.scrolledtext import ScrolledText


INITIAL_CODEX_NODES = [
    {
        "id": "project.manifest",
        "label": "ProjectManifest",
        "type": "root",
        "status": "accepted",
        "description": "Top-level project control object: language targets, package layout, build commands, release rules, and CI gates.",
    },
    {
        "id": "policy.security",
        "label": "SecurityPolicy",
        "type": "policy",
        "status": "locked",
        "description": "Controls filesystem boundaries, shell access, dependency permissions, and unsafe operation rejection.",
    },
    {
        "id": "schema.layer",
        "label": "Schema Layer",
        "type": "schema",
        "status": "accepted",
        "description": "JSON schemas for characters, lines, blocks, files, patches, diagnostics, and runtime reports.",
    },
    {
        "id": "module.registry",
        "label": "Module Registry",
        "type": "module",
        "status": "accepted",
        "description": "Reusable character rules, line templates, block modules, class modules, methods, tests, and emit-ready files.",
    },
    {
        "id": "emitter.backend.cpp",
        "label": "C++ Emitter",
        "type": "emitter",
        "status": "warning",
        "description": "Deterministic JSON-to-C++ source compiler with formatting, include ordering, and hash reporting.",
    },
    {
        "id": "diagnostics.runtime",
        "label": "Diagnostics Runtime",
        "type": "diagnostic",
        "status": "error",
        "description": "Build, lint, test, static analysis, and runtime failures returned as DiagnosticManifest objects.",
    },
]

INITIAL_FILE_TREE = [
    {"path": "codex_project/project_manifest.json", "kind": "ProjectManifest", "state": "source-of-truth"},
    {"path": "codex_project/policies/security_policy.json", "kind": "SecurityPolicy", "state": "locked"},
    {"path": "codex_project/schemas/block_module.schema.json", "kind": "Schema", "state": "valid"},
    {"path": "codex_project/modules/classes/hardware_controller.block.json", "kind": "BlockModule", "state": "editable"},
    {"path": "codex_project/modules/files/main.file.json", "kind": "FileManifest", "state": "editable"},
    {"path": "codex_project/emitted/src/main.cpp", "kind": "Emitted Source", "state": "generated-only"},
    {"path": "codex_project/reports/diagnostic_manifest.json", "kind": "DiagnosticManifest", "state": "runtime-feedback"},
]

INITIAL_DIAGNOSTICS = [
    {
        "severity": "error",
        "target": "hardware_controller.block.json",
        "message": "Method start_device references undefined dependency DeviceBus.",
    },
    {
        "severity": "warning",
        "target": "main.file.json",
        "message": "Generated include order differs from current policy ordering rule.",
    },
    {
        "severity": "accepted",
        "target": "security_policy.json",
        "message": "Filesystem output restricted to ./emitted and ./reports.",
    },
]

MANIFEST_PREVIEW = {
    "id": "hardware_editor.project.v1",
    "type": "ProjectManifest",
    "project_name": "Graphical Codex CLI-REPL OS",
    "targets": ["cpp", "python", "verilog", "api"],
    "source_of_truth": "json_graphical_codex",
    "direct_source_editing": False,
    "activation_boundary": "blue_cli_repl_command_only",
    "kernel_model": {
        "service_manager": True,
        "api_gateway": True,
        "repl_bus": True,
        "event_log": True,
        "capability_gates": True,
        "linux_terminal": True,
        "powershell_7": True,
    },
    "control_surfaces": {
        "character_rules": True,
        "line_templates": True,
        "block_modules": True,
        "file_manifests": True,
        "patch_manifests": True,
        "diagnostics": True,
        "emitter_plugins": True,
        "api_routes": True,
        "kernel_services": True,
        "linux_terminal": True,
        "powershell_7": True,
    },
    "validation_gates": [
        "schema",
        "policy",
        "reference_integrity",
        "dependency_policy",
        "emission_hash",
        "service_activation_command",
    ],
}

STATUS_LABELS = {
    "accepted": "Accepted",
    "locked": "Locked",
    "warning": "Warning",
    "error": "Repair",
    "valid": "Valid",
    "editable": "Editable",
    "source-of-truth": "Source",
    "generated-only": "Generated",
    "runtime-feedback": "Feedback",
    "active": "Active",
    "dormant": "Dormant",
    "faulted": "Faulted",
}

TYPE_SYMBOLS = {
    "root": "[ROOT]",
    "policy": "[POLICY]",
    "schema": "[SCHEMA]",
    "module": "[MODULE]",
    "emitter": "[EMITTER]",
    "diagnostic": "[DIAG]",
}


CONFIGURE4_SERVICE_ID = "configure_4.authoring.fabric"
CONFIGURE4_LABEL = "Configure4AuthoringFabric"
CONFIGURE4_ALLOWED_MODES = ("manual", "semi_auto", "auto")
CONFIGURE4_ALLOWED_FLOW_TYPES = ("literal", "template", "cartesian", "repeat", "reverse", "service_scaffold")
CONFIGURE4_ALLOWED_PERSISTENCE_POLICIES = ("memory_only", "manifest", "manifest_and_export", "disabled")
CONFIGURE4_ALLOWED_FILE_POLICIES = ("none", "read_explicit", "write_explicit", "read_write_explicit", "manifest_only")
CONFIGURE4_ALLOWED_SIDE_EFFECT_POLICIES = ("none", "memory_only", "manifest_only", "explicit_filesystem", "runtime_registration")
CONFIGURE4_GENERIC_HANDLER_NAME = "generic_configure4_service_handler"
CONFIGURE4_GENERIC_API_HANDLER_NAME = "generic_configure4_api_handler"
CONFIGURE4_MAX_VARIANTS = 256
CONFIGURE4_MAX_DRAFTS = 128
CONFIGURE4_MAX_HISTORY = 500

QUADTREE_DESKTOP_SERVICE_ID = "quadtree.desktop"
QUADTREE_DESKTOP_LABEL = "QuadtreeDesktop"
QUADTREE_DESKTOP_ID = "qdt.desktop.main"
QUADTREE_DESKTOP_VERSION = "0.1.0"
QUADTREE_LAYER_TYPES = ("input", "processing", "output")
QUADTREE_QUADRANTS = ("nw", "ne", "sw", "se")
QUADTREE_VALIDATION_MODES = ("schema_only", "policy_only", "full", "preview_only")
QUADTREE_APPLY_MODES = ("stage", "apply", "apply_and_snapshot")
QUADTREE_MAX_DEPTH_LIMIT = 8
QUADTREE_EVENT_LIMIT = 500
QUADTREE_DEFAULT_MODULE_IDS = (
    "qdt.system.services",
    "qdt.system.api",
    "qdt.codex.graph",
    "qdt.codex.files",
    "qdt.codex.diagnostics",
    "qdt.codex.emission",
    "qdt.runtime.terminal",
    "qdt.runtime.memory",
    "qdt.configure4.authoring",
    "qdt.desktop.output",
)

CONFIGURE4_MODE_ALIASES = {
    "manual": "manual",
    "semi": "semi_auto",
    "semi-auto": "semi_auto",
    "semi_auto": "semi_auto",
    "semi-automatic": "semi_auto",
    "semiautomatic": "semi_auto",
    "auto": "auto",
    "automatic": "auto",
}

CONFIGURE4_FIELD_ALIASES = {
    "id": "identity.service_id",
    "service": "identity.service_id",
    "service_id": "identity.service_id",
    "name": "identity.label",
    "label": "identity.label",
    "aliases": "identity.aliases",
    "layer": "identity.layer",
    "state": "identity.state",
    "description": "identity.description",
    "version": "identity.version",
    "commands": "command_surface.verbs",
    "command_verbs": "command_surface.verbs",
    "verbs": "command_surface.verbs",
    "handler": "command_surface.handler_name",
    "handler_name": "command_surface.handler_name",
    "help": "command_surface.help_text",
    "help_text": "command_surface.help_text",
    "default_payload": "command_surface.default_payload",
    "api_routes": "api_surface.routes",
    "routes": "api_surface.routes",
    "api_handler": "api_surface.handler_name",
    "api_handler_name": "api_surface.handler_name",
    "payload_schema": "api_surface.payload_schema",
    "output_schema": "api_surface.output_schema",
    "activation_requirements": "execution_behavior.activation_requirements",
    "dependency_requirements": "execution_behavior.dependency_requirements",
    "dependencies": "execution_behavior.dependency_requirements",
    "allowed_modes": "execution_behavior.allowed_modes",
    "timeout_policy": "execution_behavior.timeout_policy",
    "file_access_policy": "execution_behavior.file_access_policy",
    "side_effect_policy": "execution_behavior.side_effect_policy",
    "can_register_immediately": "execution_behavior.can_register_immediately",
    "register_immediately": "execution_behavior.can_register_immediately",
    "validation_gates": "validation_rules.gates",
    "allow_command_collisions": "validation_rules.allow_command_collisions",
    "persistence_policy": "persistence_behavior.policy",
    "import_path": "persistence_behavior.import_path",
    "export_path": "persistence_behavior.export_path",
    "versioning_policy": "persistence_behavior.versioning_policy",
    "flow": "generated_code_plan.flow",
    "flow_type": "generated_code_plan.flow.type",
    "input_value": "generated_code_plan.flow.input_value",
    "input_width": "generated_code_plan.flow.input_width",
    "template": "generated_code_plan.flow.template",
    "output_target": "generated_code_plan.flow.output_target",
    "prefix": "generated_code_plan.flow.prefix",
    "suffix": "generated_code_plan.flow.suffix",
    "separator": "generated_code_plan.flow.separator",
}

CONFIGURE4_MANUAL_REQUIRED_PATHS = (
    "identity.service_id",
    "identity.label",
    "identity.aliases",
    "identity.layer",
    "identity.state",
    "identity.description",
    "identity.version",
    "command_surface.verbs",
    "command_surface.handler_name",
    "command_surface.help_text",
    "command_surface.default_payload",
    "api_surface.routes",
    "api_surface.handler_name",
    "api_surface.payload_schema",
    "api_surface.output_schema",
    "execution_behavior.activation_requirements",
    "execution_behavior.dependency_requirements",
    "execution_behavior.allowed_modes",
    "execution_behavior.timeout_policy",
    "execution_behavior.file_access_policy",
    "execution_behavior.side_effect_policy",
    "execution_behavior.can_register_immediately",
    "validation_rules.gates",
    "validation_rules.allow_command_collisions",
    "persistence_behavior.policy",
    "persistence_behavior.import_path",
    "persistence_behavior.export_path",
    "persistence_behavior.versioning_policy",
    "examples",
    "generated_code_plan.flow.type",
    "generated_code_plan.handler_name",
    "generated_code_plan.api_handler_name",
    "generated_code_plan.runtime_strategy",
    "metadata.owner",
    "object_graph",
)


def configure4_normalize_mode(value: object) -> str:
    key = str(value).strip().lower().replace(" ", "-")
    return CONFIGURE4_MODE_ALIASES.get(key, key)


def configure4_slug_words(text: object) -> list[str]:
    raw = str(text or "").lower()
    words = re.findall(r"[a-z0-9]+", raw)
    stop_words = {
        "a",
        "an",
        "and",
        "as",
        "build",
        "create",
        "for",
        "from",
        "make",
        "new",
        "service",
        "that",
        "the",
        "to",
        "with",
    }
    return [word for word in words if word not in stop_words]


def configure4_pascal_label(value: object) -> str:
    words = configure4_slug_words(value)
    if not words:
        return "GeneratedService"
    return "".join(word[:1].upper() + word[1:] for word in words)


def configure4_dotted_id(value: object, *, prefix: str = "codex") -> str:
    words = configure4_slug_words(value)
    if not words:
        words = ["generated", "service"]
    parts = [prefix] + words[:3]
    if parts[-1] != "service" and len(parts) < 4:
        parts.append("service")
    return ".".join(parts)


@dataclass
class Configure4FabricRecord:
    kind: str
    name: str
    role: str = ""
    value: object = ""
    links: list[str] = field(default_factory=list)
    state: str = "staged"

    @classmethod
    def from_dict(cls, payload: object) -> "Configure4FabricRecord":
        if not isinstance(payload, dict):
            return cls(kind="object", name="unnamed")
        return cls(
            kind=str(payload.get("kind", "object")),
            name=str(payload.get("name", "unnamed")),
            role=str(payload.get("role", "")),
            value=copy.deepcopy(payload.get("value", "")),
            links=[str(item) for item in payload.get("links", [])] if isinstance(payload.get("links", []), list) else [],
            state=str(payload.get("state", "staged")),
        )

    def to_dict(self) -> dict[str, object]:
        return copy.deepcopy(asdict(self))


@dataclass
class Configure4Spec:
    draft_id: str
    mode: str
    identity: dict[str, object] = field(default_factory=dict)
    command_surface: dict[str, object] = field(default_factory=dict)
    api_surface: dict[str, object] = field(default_factory=dict)
    execution_behavior: dict[str, object] = field(default_factory=dict)
    validation_rules: dict[str, object] = field(default_factory=dict)
    persistence_behavior: dict[str, object] = field(default_factory=dict)
    examples: list[object] = field(default_factory=list)
    generated_code_plan: dict[str, object] = field(default_factory=dict)
    diagnostics: list[dict[str, object]] = field(default_factory=list)
    metadata: dict[str, object] = field(default_factory=dict)
    object_graph: list[dict[str, object]] = field(default_factory=list)
    custom_fields: dict[str, object] = field(default_factory=dict)
    inferred_fields: list[str] = field(default_factory=list)

    @classmethod
    def from_dict(cls, payload: object) -> "Configure4Spec":
        if not isinstance(payload, dict):
            payload = {}
        draft_id = str(payload.get("draft_id", "draft_000"))
        mode = configure4_normalize_mode(payload.get("mode", "manual"))
        object_graph = []
        for item in payload.get("object_graph", []):
            record = Configure4FabricRecord.from_dict(item)
            object_graph.append(record.to_dict())
        return cls(
            draft_id=draft_id,
            mode=mode if mode in CONFIGURE4_ALLOWED_MODES else "manual",
            identity=copy.deepcopy(payload.get("identity", {})) if isinstance(payload.get("identity", {}), dict) else {},
            command_surface=copy.deepcopy(payload.get("command_surface", {})) if isinstance(payload.get("command_surface", {}), dict) else {},
            api_surface=copy.deepcopy(payload.get("api_surface", {})) if isinstance(payload.get("api_surface", {}), dict) else {},
            execution_behavior=copy.deepcopy(payload.get("execution_behavior", {})) if isinstance(payload.get("execution_behavior", {}), dict) else {},
            validation_rules=copy.deepcopy(payload.get("validation_rules", {})) if isinstance(payload.get("validation_rules", {}), dict) else {},
            persistence_behavior=copy.deepcopy(payload.get("persistence_behavior", {})) if isinstance(payload.get("persistence_behavior", {}), dict) else {},
            examples=copy.deepcopy(payload.get("examples", [])) if isinstance(payload.get("examples", []), list) else [],
            generated_code_plan=copy.deepcopy(payload.get("generated_code_plan", {})) if isinstance(payload.get("generated_code_plan", {}), dict) else {},
            diagnostics=copy.deepcopy(payload.get("diagnostics", [])) if isinstance(payload.get("diagnostics", []), list) else [],
            metadata=copy.deepcopy(payload.get("metadata", {})) if isinstance(payload.get("metadata", {}), dict) else {},
            object_graph=object_graph,
            custom_fields=copy.deepcopy(payload.get("custom_fields", {})) if isinstance(payload.get("custom_fields", {}), dict) else {},
            inferred_fields=[str(item) for item in payload.get("inferred_fields", [])] if isinstance(payload.get("inferred_fields", []), list) else [],
        )

    def to_dict(self) -> dict[str, object]:
        return copy.deepcopy(asdict(self))


class Configure4Sink:
    name = "abstract"

    def write(self, text: str) -> bool:
        raise NotImplementedError

    def write_json(self, payload: object) -> bool:
        return self.write(json.dumps(payload, indent=2))

    def getvalue(self) -> str:
        return ""

    def close(self) -> None:
        return None


class Configure4MemorySink(Configure4Sink):
    name = "memory"

    def __init__(self) -> None:
        self.parts: list[str] = []

    def write(self, text: str) -> bool:
        self.parts.append(str(text))
        return True

    def getvalue(self) -> str:
        return "".join(self.parts)


class Configure4ConsoleSink(Configure4MemorySink):
    name = "console"

    def __init__(self, writer: object | None = None) -> None:
        super().__init__()
        self.writer = writer

    def write(self, text: str) -> bool:
        super().write(text)
        if callable(self.writer):
            self.writer(str(text))
        return True


class Configure4FilePreviewSink(Configure4MemorySink):
    name = "file-preview"

    def __init__(self, path: str) -> None:
        super().__init__()
        self.path = path

    def preview(self) -> dict[str, str]:
        return {"target": self.path, "content": self.getvalue()}


class Configure4JsonSink(Configure4MemorySink):
    name = "json"

    def __init__(self) -> None:
        super().__init__()
        self.payloads: list[object] = []

    def write_json(self, payload: object) -> bool:
        self.payloads.append(copy.deepcopy(payload))
        return super().write_json(payload)

    def latest(self) -> object:
        return copy.deepcopy(self.payloads[-1]) if self.payloads else None


class Theme:
    bg = "#07070a"
    panel = "#101014"
    panel_2 = "#15151b"
    card = "#0b0b10"
    card_2 = "#181820"
    field = "#030306"
    border = "#2b2b34"
    border_active = "#d8d8e0"
    text = "#f4f4f7"
    muted = "#a9a9b4"
    faint = "#6f6f7c"
    blue = "#07118f"
    blue_2 = "#0b1caf"
    blue_3 = "#1027d7"
    blue_text = "#ffffff"
    green = "#8de0a5"
    yellow = "#f5d071"
    red = "#ff9ca3"
    purple = "#c9c3ff"
    cyan = "#85d8f5"


STATUS_COLORS = {
    "accepted": ("#11291c", Theme.green, "#245f39"),
    "locked": ("#1a2436", "#9bb7f4", "#344866"),
    "warning": ("#322710", Theme.yellow, "#6e541d"),
    "error": ("#351719", Theme.red, "#7b3037"),
    "valid": ("#11291c", Theme.green, "#245f39"),
    "editable": ("#202035", Theme.purple, "#565083"),
    "source-of-truth": ("#11252c", Theme.cyan, "#28596a"),
    "generated-only": ("#2b2032", "#e9b7ff", "#61456f"),
    "runtime-feedback": ("#351719", Theme.red, "#7b3037"),
    "active": ("#102d1b", Theme.green, "#2e7a48"),
    "dormant": ("#17171d", Theme.muted, "#383844"),
    "faulted": ("#351719", Theme.red, "#7b3037"),
}


class ScrollFrame(tk.Frame):
    def __init__(self, parent: tk.Widget, *, background: str = Theme.panel, **kwargs) -> None:
        super().__init__(parent, bg=background, **kwargs)
        self.canvas = tk.Canvas(self, bg=background, highlightthickness=0, bd=0)
        self.scrollbar = ttk.Scrollbar(self, orient="vertical", command=self.canvas.yview)
        self.inner = tk.Frame(self.canvas, bg=background)
        self.window_id = self.canvas.create_window((0, 0), window=self.inner, anchor="nw")
        self.canvas.configure(yscrollcommand=self.scrollbar.set)
        self.canvas.pack(side="left", fill="both", expand=True)
        self.scrollbar.pack(side="right", fill="y")
        self.inner.bind("<Configure>", self._on_frame_configure)
        self.canvas.bind("<Configure>", self._on_canvas_configure)
        self.canvas.bind_all("<MouseWheel>", self._on_mousewheel_windows)
        self.canvas.bind_all("<Button-4>", self._on_mousewheel_linux)
        self.canvas.bind_all("<Button-5>", self._on_mousewheel_linux)

    def _on_frame_configure(self, _event: tk.Event) -> None:
        self.canvas.configure(scrollregion=self.canvas.bbox("all"))

    def _on_canvas_configure(self, event: tk.Event) -> None:
        self.canvas.itemconfigure(self.window_id, width=event.width)

    def _on_mousewheel_windows(self, event: tk.Event) -> None:
        if self.winfo_containing(event.x_root, event.y_root) is not None:
            self.canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")

    def _on_mousewheel_linux(self, event: tk.Event) -> None:
        if self.winfo_containing(event.x_root, event.y_root) is None:
            return
        direction = -1 if event.num == 4 else 1
        self.canvas.yview_scroll(direction, "units")


class GraphicalCodexCliReplOS(tk.Tk):
    def __init__(self) -> None:
        super().__init__()
        self.title("Graphical Codex CLI-REPL OS - Console Only")
        self.geometry("1180x760")
        self.minsize(760, 420)
        self.configure(bg=Theme.bg)

        self.nodes = copy.deepcopy(INITIAL_CODEX_NODES)
        self.files = copy.deepcopy(INITIAL_FILE_TREE)
        self.diagnostics = copy.deepcopy(INITIAL_DIAGNOSTICS)
        self.manifest = copy.deepcopy(MANIFEST_PREVIEW)
        self.versions: list[dict[str, str]] = []
        self.vars: dict[str, str] = {}
        self.history: list[str] = []
        self.history_index: int | None = None
        self.event_log: list[dict[str, str]] = []
        self.terminal_cwd = Path.cwd()
        self.terminal_timeout_seconds = 30
        self.linux_shell = shutil.which("bash") or shutil.which("sh") or ""
        self.pwsh_executable = shutil.which("pwsh") or shutil.which("pwsh.exe") or ""
        self.selected_service_id = "blue.cli.engine"
        self.selected_node_id = self.nodes[0]["id"]
        self.configure4_state = self._create_configure4_state()
        self.quadtreeDesktop_state = self._create_quadtree_desktop_state()
        self.shell_frame: tk.Frame | None = None
        self.console_card: tk.Frame | None = None
        self.quadtree_desktop_frame: tk.Frame | None = None
        self.quadtree_canvas: tk.Canvas | None = None
        self.quadtree_inspector_text: ScrolledText | None = None
        self.quadtree_module_text: ScrolledText | None = None
        self.quadtree_batch_text: ScrolledText | None = None
        self.quadtree_preview_text: ScrolledText | None = None
        self.quadtree_layer_var: tk.StringVar | None = None
        self.command_var = tk.StringVar(value="")
        self.status_var = tk.StringVar(value="Booting blue CLI engine...")
        self.clock_var = tk.StringVar(value="")

        self._setup_fonts()
        self._setup_styles()
        self.services = self._create_services()
        self.api_routes = self._create_api_routes()
        self._build_ui()
        self._render_all()
        self._boot_console()
        self._tick_clock()

    def _setup_fonts(self) -> None:
        self.font_title = ("Segoe UI", 24, "bold")
        self.font_h2 = ("Segoe UI", 14, "bold")
        self.font_h3 = ("Segoe UI", 10, "bold")
        self.font_body = ("Segoe UI", 10)
        self.font_small = ("Segoe UI", 9)
        self.font_micro = ("Segoe UI", 8, "bold")
        self.font_mono = ("Consolas", 10)
        self.font_prompt = ("Consolas", 12, "bold")

    def _setup_styles(self) -> None:
        self.style = ttk.Style(self)
        self.style.theme_use("clam")
        self.style.configure(
            "Vertical.TScrollbar",
            background=Theme.card_2,
            troughcolor=Theme.panel,
            bordercolor=Theme.panel,
            arrowcolor=Theme.muted,
            relief="flat",
        )
        self.style.map("Vertical.TScrollbar", background=[("active", Theme.blue_2)])
        self.style.configure(
            "Kernel.TCombobox",
            fieldbackground=Theme.field,
            background=Theme.field,
            foreground=Theme.text,
            arrowcolor=Theme.text,
            bordercolor=Theme.border,
        )

    def _create_services(self) -> dict[str, dict[str, object]]:
        specs = [
            (
                "blue.cli.engine",
                "repl",
                "active",
                "The command gate. It is the only execution path that may activate or execute kernel services.",
                ["help", "service.activate api.gateway", "service.list"],
            ),
            (
                "kernel.clock",
                "kernel",
                "dormant",
                "Provides monotonic timestamps and wall-clock metadata to API and diagnostic reports.",
                ["service.activate kernel.clock", "service.exec kernel.clock now"],
            ),
            (
                "kernel.memory",
                "kernel",
                "dormant",
                "Stores REPL variables, staged values, and small JSON payloads for later API calls.",
                ["service.activate kernel.memory", "set build_mode debug", "get build_mode"],
            ),
            (
                "kernel.fs",
                "kernel",
                "dormant",
                "Persists and loads manifest snapshots through explicit REPL commands.",
                ["service.activate kernel.fs", "manifest.save", "manifest.load"],
            ),
            (
                "kernel.terminal.linux",
                "kernel",
                "dormant",
                "Inherited Linux terminal service. Executes host shell commands only after REPL activation.",
                ["service.activate kernel.terminal.linux", "linux pwd", "linux ls -la", "terminal.cwd"],
            ),
            (
                "kernel.terminal.powershell7",
                "kernel",
                "dormant",
                "Inherited PowerShell 7 service. Executes pwsh commands only after REPL activation, when pwsh is installed.",
                ["service.activate kernel.terminal.powershell7", "pwsh $PSVersionTable.PSVersion", "pwsh Get-ChildItem", "terminal.cwd"],
            ),
            (
                "api.gateway",
                "api",
                "dormant",
                "Routes REPL-submitted API calls to activated services through local endpoints.",
                ["service.activate api.gateway", "api.list", "api.call /kernel/services"],
            ),
            (
                "codex.manifest",
                "codex",
                "dormant",
                "Owns the source-of-truth project manifest and exported JSON graph state.",
                ["service.activate codex.manifest", "manifest.show", "api.call /codex/manifest"],
            ),
            (
                "codex.graph",
                "codex",
                "dormant",
                "Manages graphical codex nodes, file objects, selections, and object creation.",
                ["service.activate codex.graph", "node.list", "node.select project.manifest"],
            ),
            (
                "codex.validator",
                "codex",
                "dormant",
                "Runs schema, policy, reference, dependency, and emission-hash gates.",
                ["service.activate codex.validator", "node.validate all", "api.call /codex/validate"],
            ),
            (
                "codex.emitter",
                "codex",
                "dormant",
                "Emits deterministic source previews from accepted JSON graphical objects.",
                ["service.activate codex.emitter", "emit.preview", "api.call /codex/emit"],
            ),
            (
                "codex.builder",
                "codex",
                "dormant",
                "Builds from emitted state and returns build reports as REPL text and API JSON.",
                ["service.activate codex.builder", "build.run", "api.call /codex/build"],
            ),
            (
                "diagnostics.runtime",
                "runtime",
                "dormant",
                "Converts build, lint, static analysis, and runtime failures into repair diagnostics.",
                ["service.activate diagnostics.runtime", "diagnostics.list", "diagnostics.patch"],
            ),
            (
                "version.ledger",
                "runtime",
                "dormant",
                "Captures immutable snapshots of the current kernel, API, and codex state.",
                ["service.activate version.ledger", "version.snapshot", "version.list"],
            ),
            (
                CONFIGURE4_SERVICE_ID,
                "kernel/codex-authoring",
                "dormant",
                "Authors, validates, previews, registers, exports, imports, and persists new services for start.py through blue CLI commands only.",
                [
                    f"service.activate {CONFIGURE4_SERVICE_ID}",
                    "configure4.help",
                    "configure4.sample",
                    "configure4.status",
                ],
            ),
            (
                QUADTREE_DESKTOP_SERVICE_ID,
                "kernel/codex-graphical",
                "dormant",
                "Maps start.py services, routes, manifest objects, diagnostics, Configure4 drafts, batches, and outputs onto a programmable quadtree desktop.",
                [
                    f"service.activate {QUADTREE_DESKTOP_SERVICE_ID}",
                    "quadtree.desktop.status",
                    "quadtree.desktop.show",
                    "quadtree.desktop.module.list",
                ],
            ),
        ]
        services: dict[str, dict[str, object]] = {}
        for service_id, layer, state, description, examples in specs:
            services[service_id] = {
                "id": service_id,
                "layer": layer,
                "state": state,
                "description": description,
                "examples": examples,
                "activated_at": "boot" if state == "active" else "",
                "last_result": "",
                "calls": 0,
            }
        if CONFIGURE4_SERVICE_ID in services:
            services[CONFIGURE4_SERVICE_ID]["label"] = CONFIGURE4_LABEL
        if QUADTREE_DESKTOP_SERVICE_ID in services:
            services[QUADTREE_DESKTOP_SERVICE_ID]["label"] = QUADTREE_DESKTOP_LABEL
        if "kernel.terminal.linux" in services:
            services["kernel.terminal.linux"]["executable"] = self.linux_shell or "<missing bash/sh>"
            services["kernel.terminal.linux"]["available"] = bool(self.linux_shell)
        if "kernel.terminal.powershell7" in services:
            services["kernel.terminal.powershell7"]["executable"] = self.pwsh_executable or "<missing pwsh>"
            services["kernel.terminal.powershell7"]["available"] = bool(self.pwsh_executable)
        return services

    def _create_api_routes(self) -> dict[str, dict[str, object]]:
        return {
            "/kernel/status": {"service": "blue.cli.engine", "description": "Return high-level REPL OS status.", "handler": self.api_kernel_status},
            "/kernel/services": {"service": "blue.cli.engine", "description": "Return every service and activation state.", "handler": self.api_kernel_services},
            "/kernel/events": {"service": "blue.cli.engine", "description": "Return recent REPL event log entries.", "handler": self.api_kernel_events},
            "/terminal/linux": {"service": "kernel.terminal.linux", "description": "Execute a Linux shell command through the inherited terminal service.", "handler": self.api_terminal_linux},
            "/terminal/powershell7": {"service": "kernel.terminal.powershell7", "description": "Execute a PowerShell 7 command through the inherited pwsh service.", "handler": self.api_terminal_powershell7},
            "/codex/manifest": {"service": "codex.manifest", "description": "Return the manifest and selected mode.", "handler": self.api_codex_manifest},
            "/codex/nodes": {"service": "codex.graph", "description": "Return graphical codex nodes.", "handler": self.api_codex_nodes},
            "/codex/files": {"service": "codex.graph", "description": "Return file configuration records.", "handler": self.api_codex_files},
            "/codex/diagnostics": {"service": "diagnostics.runtime", "description": "Return diagnostic manifest items.", "handler": self.api_diagnostics},
            "/codex/validate": {"service": "codex.validator", "description": "Run project validation and return a report.", "handler": self.api_validate},
            "/codex/emit": {"service": "codex.emitter", "description": "Return deterministic emitted source preview.", "handler": self.api_emit},
            "/codex/build": {"service": "codex.builder", "description": "Run build report generation.", "handler": self.api_build},
            "/version/snapshot": {"service": "version.ledger", "description": "Create and return a version snapshot.", "handler": self.api_version_snapshot},
            "/configure4/status": {"service": CONFIGURE4_SERVICE_ID, "description": "Return configure_4 authoring fabric status.", "handler": self.api_configure4_status},
            "/configure4/specs": {"service": CONFIGURE4_SERVICE_ID, "description": "Return configure_4 staged service specifications.", "handler": self.api_configure4_specs},
            "/configure4/validate": {"service": CONFIGURE4_SERVICE_ID, "description": "Validate a configure_4 draft or all drafts.", "handler": self.api_configure4_validate},
            "/configure4/preview": {"service": CONFIGURE4_SERVICE_ID, "description": "Preview a configure_4 integration plan.", "handler": self.api_configure4_preview},
            "/configure4/register": {"service": CONFIGURE4_SERVICE_ID, "description": "Register a configure_4 draft when runtime-safe.", "handler": self.api_configure4_register},
            "/quadtreeDesktop/status": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Return quadtree desktop status.", "handler": self.api_quadtree_desktop_status},
            "/quadtreeDesktop/modules": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Return quadtree desktop modules.", "handler": self.api_quadtree_desktop_modules},
            "/quadtreeDesktop/modules/<module-id>": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Return one quadtree module by payload module_id.", "handler": self.api_quadtree_desktop_module},
            "/quadtreeDesktop/layers/<module-id>/<layer-type>": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Return one quadtree layer by payload module_id and layer_type.", "handler": self.api_quadtree_desktop_layer},
            "/quadtreeDesktop/cells/<cell-address>": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Return one quadtree cell by payload address.", "handler": self.api_quadtree_desktop_cell},
            "/quadtreeDesktop/selection": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Return or set quadtree selection.", "handler": self.api_quadtree_desktop_selection},
            "/quadtreeDesktop/batches": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Return quadtree batch registry.", "handler": self.api_quadtree_desktop_batches},
            "/quadtreeDesktop/validate": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Validate quadtree desktop state or patch payload.", "handler": self.api_quadtree_desktop_validate},
            "/quadtreeDesktop/preview": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Preview a quadtree desktop operation.", "handler": self.api_quadtree_desktop_preview},
            "/quadtreeDesktop/apply": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Apply a validated quadtree desktop operation.", "handler": self.api_quadtree_desktop_apply},
            "/quadtreeDesktop/export": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Export quadtree desktop manifest data.", "handler": self.api_quadtree_desktop_export},
            "/quadtreeDesktop/import": {"service": QUADTREE_DESKTOP_SERVICE_ID, "description": "Import quadtree desktop manifest data.", "handler": self.api_quadtree_desktop_import},
        }

    def _build_ui(self) -> None:
        """Build the locked console-only surface.

        The shell intentionally exposes only two visual surfaces:
        1. the REPL Console, where all inspection/results appear, and
        2. the blue CLI engine, the only command submission/activation gate.
        No service panels, route panels, quick-command palettes, dialogs, or
        status bars are mounted in the visible interface.
        """
        shell = tk.Frame(self, bg=Theme.bg, padx=14, pady=14)
        shell.pack(fill="both", expand=True)
        shell.rowconfigure(0, weight=1)
        shell.columnconfigure(0, weight=1)
        self.shell_frame = shell

        console_card = self.card(shell, padx=12, pady=12)
        console_card.grid(row=0, column=0, sticky="nsew")
        console_card.rowconfigure(1, weight=1)
        console_card.columnconfigure(0, weight=1)
        self.console_card = console_card

        console_head = tk.Frame(console_card, bg=Theme.panel)
        console_head.grid(row=0, column=0, sticky="ew")
        console_head.columnconfigure(0, weight=1)
        tk.Label(
            console_head,
            text="REPL CONSOLE",
            bg=Theme.panel,
            fg=Theme.muted,
            font=self.font_micro,
            anchor="w",
        ).grid(row=0, column=0, sticky="ew")
        self.command_gate_label = tk.Label(
            console_head,
            text="blue.cli.engine: ACTIVE",
            bg=Theme.panel,
            fg=Theme.green,
            font=self.font_micro,
            anchor="e",
        )
        self.command_gate_label.grid(row=0, column=1, sticky="e")

        self.console = ScrolledText(
            console_card,
            bg="#000000",
            fg=Theme.text,
            insertbackground=Theme.text,
            relief="flat",
            font=self.font_mono,
            wrap="word",
            padx=12,
            pady=12,
            height=28,
        )
        self.console.grid(row=1, column=0, sticky="nsew", pady=(10, 10))
        self.console.tag_configure("cmd", foreground=Theme.cyan)
        self.console.tag_configure("ok", foreground=Theme.green)
        self.console.tag_configure("warn", foreground=Theme.yellow)
        self.console.tag_configure("err", foreground=Theme.red)
        self.console.tag_configure("json", foreground=Theme.purple)
        self.console.tag_configure("muted", foreground=Theme.muted)
        self.console.configure(state="disabled")

        self._build_blue_cli_engine(console_card)

    def _build_header(self, parent: tk.Frame) -> None:
        parent.columnconfigure(0, weight=1)
        parent.columnconfigure(1, weight=0)
        title_block = tk.Frame(parent, bg=Theme.panel_2)
        title_block.grid(row=0, column=0, sticky="ew")

        tk.Label(
            title_block,
            text="KERNEL + API CONTROL SURFACE  |  all services activate by blue CLI REPL command only",
            bg=Theme.panel_2,
            fg=Theme.cyan,
            font=self.font_micro,
            anchor="w",
        ).pack(anchor="w")
        tk.Label(
            title_block,
            text="Graphical Codex CLI-REPL OS",
            bg=Theme.panel_2,
            fg=Theme.text,
            font=self.font_title,
            anchor="w",
        ).pack(anchor="w", pady=(4, 3))
        tk.Label(
            title_block,
            text=(
                "A general purpose local REPL operating shell with a kernel service registry, local API gateway, "
                "Codex object graph, manifest persistence, diagnostics, emission, build reports, and version snapshots. "
                "The surrounding UI can inspect and stage commands, but the blue CLI engine is the only execution path."
            ),
            bg=Theme.panel_2,
            fg=Theme.muted,
            font=self.font_small,
            wraplength=880,
            justify="left",
            anchor="w",
        ).pack(anchor="w")

        indicators = self.card(parent, bg=Theme.card, padx=12, pady=10)
        indicators.grid(row=0, column=1, sticky="ne", padx=(18, 0))
        tk.Label(indicators, text="BOOT STATE", bg=Theme.card, fg=Theme.faint, font=self.font_micro).pack(anchor="w")
        self.active_count_label = tk.Label(indicators, bg=Theme.card, fg=Theme.green, font=self.font_h2, anchor="w")
        self.active_count_label.pack(fill="x", pady=(5, 0))
        tk.Label(indicators, textvariable=self.clock_var, bg=Theme.card, fg=Theme.muted, font=self.font_small, anchor="w").pack(fill="x")

    def _build_left_panel(self, parent: tk.Frame) -> None:
        top = tk.Frame(parent, bg=Theme.panel)
        top.pack(fill="x")
        tk.Label(top, text="KERNEL SERVICES", bg=Theme.panel, fg=Theme.muted, font=self.font_micro).pack(side="left")
        tk.Label(top, text="inspect only", bg=Theme.panel, fg=Theme.faint, font=self.font_small).pack(side="right")

        self.service_filter_var = tk.StringVar(value="")
        filter_box = tk.Frame(parent, bg=Theme.field, bd=1, relief="solid")
        filter_box.pack(fill="x", pady=(10, 8))
        tk.Label(filter_box, text="filter", bg=Theme.field, fg=Theme.faint, font=self.font_small, padx=8).pack(side="left")
        tk.Entry(
            filter_box,
            textvariable=self.service_filter_var,
            bg=Theme.field,
            fg=Theme.text,
            insertbackground=Theme.text,
            relief="flat",
            font=self.font_small,
        ).pack(side="left", fill="x", expand=True, padx=(0, 8), pady=8)
        self.service_filter_var.trace_add("write", lambda *_: self.render_service_list())

        self.service_list = ScrollFrame(parent, background=Theme.panel)
        self.service_list.pack(fill="both", expand=True)

        note = self.card(parent, bg=Theme.card, padx=10, pady=10)
        note.pack(fill="x", pady=(10, 0))
        tk.Label(note, text="COMMAND BOUNDARY", bg=Theme.card, fg=Theme.cyan, font=self.font_micro).pack(anchor="w")
        tk.Label(
            note,
            text="Selecting a service only inspects it. Activation requires a command such as service.activate codex.graph submitted in the blue CLI engine.",
            bg=Theme.card,
            fg=Theme.muted,
            font=self.font_small,
            wraplength=265,
            justify="left",
        ).pack(anchor="w", pady=(5, 0))

    def _build_center_panel(self, parent: tk.Frame) -> None:
        console_card = self.card(parent, padx=12, pady=12)
        console_card.grid(row=0, column=0, sticky="nsew")
        console_card.rowconfigure(1, weight=1)
        console_card.columnconfigure(0, weight=1)

        console_head = tk.Frame(console_card, bg=Theme.panel)
        console_head.grid(row=0, column=0, sticky="ew")
        console_head.columnconfigure(0, weight=1)
        tk.Label(
            console_head,
            text="REPL CONSOLE",
            bg=Theme.panel,
            fg=Theme.muted,
            font=self.font_micro,
            anchor="w",
        ).grid(row=0, column=0, sticky="ew")
        self.command_gate_label = tk.Label(
            console_head,
            text="blue.cli.engine: ACTIVE",
            bg=Theme.panel,
            fg=Theme.green,
            font=self.font_micro,
            anchor="e",
        )
        self.command_gate_label.grid(row=0, column=1, sticky="e")

        self.console = ScrolledText(
            console_card,
            bg="#000000",
            fg=Theme.text,
            insertbackground=Theme.text,
            relief="flat",
            font=self.font_mono,
            wrap="word",
            padx=12,
            pady=12,
            height=24,
        )
        self.console.grid(row=1, column=0, sticky="nsew", pady=(10, 10))
        self.console.tag_configure("cmd", foreground=Theme.cyan)
        self.console.tag_configure("ok", foreground=Theme.green)
        self.console.tag_configure("warn", foreground=Theme.yellow)
        self.console.tag_configure("err", foreground=Theme.red)
        self.console.tag_configure("json", foreground=Theme.purple)
        self.console.tag_configure("muted", foreground=Theme.muted)
        self.console.configure(state="disabled")

        self._build_blue_cli_engine(console_card)
        self._build_command_palette(console_card)

    def _build_blue_cli_engine(self, parent: tk.Frame) -> None:
        engine = tk.Frame(parent, bg=Theme.blue, height=74, padx=12, pady=12, bd=0)
        engine.grid(row=2, column=0, sticky="ew")
        engine.grid_propagate(False)
        engine.columnconfigure(1, weight=1)

        prompt = tk.Label(
            engine,
            text="BLUE CLI ENGINE >",
            bg=Theme.blue,
            fg=Theme.blue_text,
            font=self.font_prompt,
            padx=5,
        )
        prompt.grid(row=0, column=0, sticky="w")
        self.command_entry = tk.Entry(
            engine,
            textvariable=self.command_var,
            bg=Theme.blue_2,
            fg=Theme.blue_text,
            insertbackground=Theme.blue_text,
            relief="flat",
            font=self.font_prompt,
            bd=0,
        )
        self.command_entry.grid(row=0, column=1, sticky="ew", ipady=9, padx=(10, 10))
        self.command_entry.bind("<Return>", lambda _event: self.execute_current_command())
        self.command_entry.bind("<Up>", self.history_up)
        self.command_entry.bind("<Down>", self.history_down)
        self.command_entry.bind("<Control-l>", lambda _event: self.run_command("clear"))
        self.command_entry.focus_set()
        run_button = tk.Button(
            engine,
            text="RUN",
            command=self.execute_current_command,
            bg=Theme.blue_text,
            fg=Theme.blue,
            activebackground="#dfe4ff",
            activeforeground=Theme.blue,
            relief="flat",
            bd=0,
            padx=18,
            pady=8,
            font=self.font_prompt,
            cursor="hand2",
        )
        run_button.grid(row=0, column=2, sticky="e")

    def _build_command_palette(self, parent: tk.Frame) -> None:
        palette = self.card(parent, bg=Theme.card, padx=10, pady=10)
        palette.grid(row=3, column=0, sticky="ew", pady=(10, 0))
        palette.columnconfigure(0, weight=1)
        tk.Label(
            palette,
            text="QUICK COMMANDS - buttons stage text only; press Enter in the blue CLI engine to execute",
            bg=Theme.card,
            fg=Theme.faint,
            font=self.font_micro,
            anchor="w",
        ).grid(row=0, column=0, sticky="ew", columnspan=6, pady=(0, 7))
        commands = [
            "help",
            "service.list",
            "service.activate api.gateway",
            "api.list",
            "service.activate all",
            "node.list",
            "diagnostics.list",
            "diagnostics.patch",
            "node.validate all",
            "emit.preview",
            "build.run",
            "version.snapshot",
        ]
        for index, command in enumerate(commands):
            btn = tk.Button(
                palette,
                text=command,
                command=lambda value=command: self.stage_command(value),
                bg=Theme.card_2,
                fg=Theme.text,
                activebackground=Theme.blue_2,
                activeforeground=Theme.blue_text,
                relief="flat",
                bd=0,
                padx=8,
                pady=6,
                font=self.font_small,
                cursor="hand2",
            )
            btn.grid(row=1 + index // 3, column=index % 3, sticky="ew", padx=4, pady=4)
        for col in range(3):
            palette.columnconfigure(col, weight=1)

    def _build_right_panel(self, parent: tk.Frame) -> None:
        self.service_detail_card = self.card(parent, padx=12, pady=12)
        self.service_detail_card.pack(fill="x")
        tk.Label(
            self.service_detail_card,
            text="SELECTED SERVICE",
            bg=Theme.panel,
            fg=Theme.muted,
            font=self.font_micro,
        ).pack(anchor="w")
        self.service_detail = tk.Frame(self.service_detail_card, bg=Theme.panel)
        self.service_detail.pack(fill="x", pady=(8, 0))

        self.api_card = self.card(parent, padx=12, pady=12)
        self.api_card.pack(fill="x", pady=(12, 0))
        tk.Label(self.api_card, text="API ROUTES", bg=Theme.panel, fg=Theme.muted, font=self.font_micro).pack(anchor="w")
        self.api_list_frame = tk.Frame(self.api_card, bg=Theme.panel)
        self.api_list_frame.pack(fill="x", pady=(8, 0))

        self.state_card = self.card(parent, padx=12, pady=12)
        self.state_card.pack(fill="x", pady=(12, 0))
        tk.Label(self.state_card, text="KERNEL STATE", bg=Theme.panel, fg=Theme.muted, font=self.font_micro).pack(anchor="w")
        self.state_text = ScrolledText(
            self.state_card,
            height=13,
            bg="#000000",
            fg=Theme.muted,
            insertbackground=Theme.text,
            relief="flat",
            font=("Consolas", 9),
            wrap="word",
            padx=10,
            pady=10,
        )
        self.state_text.pack(fill="x", pady=(8, 0))
        self.state_text.configure(state="disabled")

        self.cheat_card = self.card(parent, padx=12, pady=12)
        self.cheat_card.pack(fill="x", pady=(12, 0))
        tk.Label(self.cheat_card, text="COMMAND CHEATSHEET", bg=Theme.panel, fg=Theme.muted, font=self.font_micro).pack(anchor="w")
        cheat = [
            "service.activate <name|all>",
            "service.deactivate <name|all>",
            "service.exec <name> [payload]",
            "api.call <route> [json]",
            "node.add {json}",
            "node.select <id>",
            "manifest.save [path]",
            "set <key> <value>",
            "get <key>",
        ]
        for line in cheat:
            label = tk.Label(
                self.cheat_card,
                text=line,
                bg=Theme.card,
                fg=Theme.cyan,
                font=self.font_small,
                anchor="w",
                padx=8,
                pady=5,
                cursor="hand2",
            )
            label.pack(fill="x", pady=(6, 0))
            label.bind("<Button-1>", lambda _event, value=line: self.stage_command(value))

    def _render_all(self) -> None:
        """Console-only mode has no side panels to refresh."""
        if hasattr(self, "command_gate_label"):
            self.command_gate_label.configure(text="blue.cli.engine: ACTIVE")

    def render_header_counts(self) -> None:
        active = sum(1 for svc in self.services.values() if svc["state"] == "active")
        total = len(self.services)
        self.active_count_label.configure(text=f"{active}/{total} services active")

    def render_service_list(self) -> None:
        for child in self.service_list.inner.winfo_children():
            child.destroy()
        query = self.service_filter_var.get().strip().lower() if hasattr(self, "service_filter_var") else ""
        for service_id, svc in self.services.items():
            haystack = f"{service_id} {svc['layer']} {svc['state']} {svc['description']}".lower()
            if query and query not in haystack:
                continue
            selected = service_id == self.selected_service_id
            bg = Theme.card_2 if selected else Theme.card
            border = Theme.border_active if selected else Theme.border
            row = self.card(self.service_list.inner, bg=bg, border=border, padx=10, pady=9)
            row.pack(fill="x", pady=4)
            row.bind("<Button-1>", lambda _event, sid=service_id: self.select_service(sid))
            for child in row.winfo_children():
                child.bind("<Button-1>", lambda _event, sid=service_id: self.select_service(sid))
            top = tk.Frame(row, bg=bg)
            top.pack(fill="x")
            tk.Label(top, text=service_id, bg=bg, fg=Theme.text, font=self.font_h3, anchor="w").pack(side="left", fill="x", expand=True)
            self.badge(top, str(svc["state"])).pack(side="right")
            tk.Label(row, text=str(svc["layer"]).upper(), bg=bg, fg=Theme.faint, font=self.font_micro, anchor="w").pack(fill="x", pady=(4, 0))

    def render_service_detail(self) -> None:
        for child in self.service_detail.winfo_children():
            child.destroy()
        svc = self.services[self.selected_service_id]
        body = self.card(self.service_detail, bg=Theme.card, padx=10, pady=10)
        body.pack(fill="x")
        top = tk.Frame(body, bg=Theme.card)
        top.pack(fill="x")
        tk.Label(top, text=str(svc["id"]), bg=Theme.card, fg=Theme.text, font=self.font_h3).pack(side="left", fill="x", expand=True)
        self.badge(top, str(svc["state"])).pack(side="right")
        tk.Label(
            body,
            text=str(svc["description"]),
            bg=Theme.card,
            fg=Theme.muted,
            font=self.font_small,
            wraplength=330,
            justify="left",
        ).pack(anchor="w", pady=(8, 8))
        facts = [
            f"layer: {svc['layer']}",
            f"calls: {svc['calls']}",
            f"activated_at: {svc['activated_at'] or 'not active'}",
        ]
        for fact in facts:
            tk.Label(body, text=fact, bg=Theme.card, fg=Theme.faint, font=self.font_small).pack(anchor="w")
        tk.Label(body, text="stage examples", bg=Theme.card, fg=Theme.cyan, font=self.font_micro).pack(anchor="w", pady=(10, 2))
        for command in svc["examples"]:
            btn = tk.Button(
                body,
                text=str(command),
                command=lambda value=str(command): self.stage_command(value),
                bg=Theme.card_2,
                fg=Theme.text,
                activebackground=Theme.blue_2,
                activeforeground=Theme.blue_text,
                relief="flat",
                bd=0,
                font=self.font_small,
                anchor="w",
                padx=8,
                pady=5,
                cursor="hand2",
            )
            btn.pack(fill="x", pady=3)
        if svc["last_result"]:
            tk.Label(body, text="last result", bg=Theme.card, fg=Theme.cyan, font=self.font_micro).pack(anchor="w", pady=(10, 2))
            tk.Label(
                body,
                text=str(svc["last_result"]),
                bg=Theme.card,
                fg=Theme.muted,
                font=self.font_small,
                wraplength=330,
                justify="left",
            ).pack(anchor="w")

    def render_api_routes(self) -> None:
        for child in self.api_list_frame.winfo_children():
            child.destroy()
        for route, spec in self.api_routes.items():
            service_id = str(spec["service"])
            state = self.services.get(service_id, {}).get("state", "dormant")
            row = self.card(self.api_list_frame, bg=Theme.card, padx=8, pady=7)
            row.pack(fill="x", pady=4)
            row.bind("<Button-1>", lambda _event, value=f"api.call {route}": self.stage_command(value))
            top = tk.Frame(row, bg=Theme.card)
            top.pack(fill="x")
            tk.Label(top, text=route, bg=Theme.card, fg=Theme.text, font=self.font_small).pack(side="left", fill="x", expand=True)
            self.badge(top, str(state)).pack(side="right")
            tk.Label(row, text=f"service: {service_id}", bg=Theme.card, fg=Theme.faint, font=self.font_small).pack(anchor="w", pady=(3, 0))

    def render_kernel_state(self) -> None:
        payload = {
            "selected_service": self.selected_service_id,
            "selected_node": self.selected_node_id,
            "vars": self.vars,
            "history_size": len(self.history),
            "event_count": len(self.event_log),
            "active_services": [sid for sid, svc in self.services.items() if svc["state"] == "active"],
            "diagnostic_counts": self._diagnostic_counts(),
        }
        text = json.dumps(payload, indent=2)
        self.state_text.configure(state="normal")
        self.state_text.delete("1.0", "end")
        self.state_text.insert("1.0", text)
        self.state_text.configure(state="disabled")

    def _boot_console(self) -> None:
        self.write_console("BOOT", "Blue CLI engine online. Kernel services are dormant until activated by REPL command.", "ok")
        self.write_console("BOOT", "Type help, then activate services explicitly. Terminal examples: service.activate kernel.terminal.linux; linux pwd; service.activate kernel.terminal.powershell7; pwsh $PSVersionTable.PSVersion", "muted")
        self.status_var.set("Ready. Submit commands through the blue CLI engine.")

    def _tick_clock(self) -> None:
        self.clock_var.set(_dt.datetime.now().strftime("%Y-%m-%d %H:%M:%S"))
        self.after(1000, self._tick_clock)

    def select_service(self, service_id: str) -> None:
        self.selected_service_id = service_id
        self.render_service_list()
        self.render_service_detail()
        self.render_kernel_state()
        self.status_var.set(f"Selected {service_id}. Selection does not activate it.")

    def stage_command(self, command: str) -> None:
        self.command_var.set(command)
        self.command_entry.focus_set()
        self.command_entry.icursor("end")
        self.status_var.set("Command staged. Press Enter in the blue CLI engine to execute.")

    def execute_current_command(self) -> None:
        command = self.command_var.get().strip()
        if not command:
            return
        self.command_var.set("")
        self.run_command(command, from_entry=True)

    def history_up(self, _event: tk.Event) -> str:
        if not self.history:
            return "break"
        if self.history_index is None:
            self.history_index = len(self.history) - 1
        else:
            self.history_index = max(0, self.history_index - 1)
        self.command_var.set(self.history[self.history_index])
        self.command_entry.icursor("end")
        return "break"

    def history_down(self, _event: tk.Event) -> str:
        if not self.history:
            return "break"
        if self.history_index is None:
            return "break"
        self.history_index += 1
        if self.history_index >= len(self.history):
            self.history_index = None
            self.command_var.set("")
        else:
            self.command_var.set(self.history[self.history_index])
        self.command_entry.icursor("end")
        return "break"

    def run_command(self, command: str, *, from_entry: bool = False) -> None:
        command = command.strip()
        if not command:
            return
        if from_entry:
            self.history.append(command)
            self.history_index = None
        self.write_console("BLUE>", command, "cmd")
        self._record_event("command", command)
        try:
            output, tag = self.dispatch_command(command)
        except Exception as exc:  # Defensive UI shell; command errors should not crash the OS.
            output, tag = f"Unhandled command fault: {exc}", "err"
        if output:
            self.write_console("OS", output, tag)
        self._render_all()
        self.status_var.set("Command completed." if tag != "err" else "Command failed. See console output.")

    def dispatch_command(self, command: str) -> tuple[str, str]:
        try:
            tokens = shlex.split(command)
        except ValueError as exc:
            return f"Parse error: {exc}", "err"
        if not tokens:
            return "", "muted"
        cmd = tokens[0].lower()
        args = tokens[1:]
        raw_after_cmd = command.split(maxsplit=1)[1] if len(command.split(maxsplit=1)) > 1 else ""

        if cmd in {"help", "?"}:
            return self.help_text(), "ok"
        if cmd == "clear":
            self.clear_console()
            return "Console cleared.", "ok"
        if cmd == "history":
            return "\n".join(f"{i + 1}: {item}" for i, item in enumerate(self.history[-40:])) or "No command history yet.", "muted"
        if cmd == "blue.engine":
            return "blue.cli.engine is the command-only activation boundary. UI buttons can stage commands, but service activation occurs only when the blue engine executes a REPL command.", "ok"
        if cmd == "terminal.cwd":
            return str(self.terminal_cwd), "muted"
        if cmd == "terminal.cd":
            target = raw_after_cmd.strip() or "~"
            return self.command_terminal_cd(target)
        if cmd == "terminal.timeout":
            if not args:
                return f"terminal timeout = {self.terminal_timeout_seconds}s", "muted"
            return self.command_terminal_timeout(args[0])
        if cmd in {"linux", "bash", "sh"}:
            return self.execute_linux_terminal(raw_after_cmd)
        if cmd in {"pwsh", "powershell", "powershell7"}:
            return self.execute_powershell7_terminal(raw_after_cmd)

        if cmd == "service.list":
            return self.command_service_list(), "json"
        if cmd == "service.status":
            target = args[0] if args else "all"
            return self.command_service_status(target), "json"
        if cmd == "service.activate":
            if not args:
                return "Usage: service.activate <name|all>", "err"
            return self.command_service_activate(args[0]), "ok"
        if cmd == "service.deactivate":
            if not args:
                return "Usage: service.deactivate <name|all>", "err"
            return self.command_service_deactivate(args[0]), "warn"
        if cmd == "service.restart":
            if not args:
                return "Usage: service.restart <name|all>", "err"
            self.command_service_deactivate(args[0])
            return self.command_service_activate(args[0]), "ok"
        if cmd == "service.exec":
            if not args:
                return "Usage: service.exec <name> [payload]", "err"
            service_id = args[0]
            payload_raw = raw_after_cmd.split(maxsplit=1)[1] if len(raw_after_cmd.split(maxsplit=1)) > 1 else ""
            return self.execute_service(service_id, payload_raw)

        if cmd == "api.list":
            return self.command_api_list(), "json"
        if cmd == "api.call":
            if not args:
                return "Usage: api.call <route> [json-payload]", "err"
            route = args[0]
            payload_raw = raw_after_cmd.split(maxsplit=1)[1] if len(raw_after_cmd.split(maxsplit=1)) > 1 else ""
            payload_raw = payload_raw.split(maxsplit=1)[1] if payload_raw.startswith(route) and len(payload_raw.split(maxsplit=1)) > 1 else ""
            return self.command_api_call(route, payload_raw)
        if cmd == "api.register":
            return self.command_api_register(args, raw_after_cmd)

        if cmd == "kernel.info":
            return self.format_json(self.api_kernel_status({})), "json"
        if cmd == "set":
            if len(args) < 2:
                return "Usage: set <key> <value>", "err"
            return self.command_set(args[0], " ".join(args[1:])), "ok"
        if cmd == "get":
            if not args:
                return self.format_json(self.vars), "json"
            return self.vars.get(args[0], "<unset>"), "muted"
        if cmd == "vars":
            return self.format_json(self.vars), "json"

        if cmd.startswith("quadtree.desktop."):
            return self.dispatch_quadtree_desktop_command(cmd, args, raw_after_cmd)

        if cmd.startswith("configure4."):
            return self.dispatch_configure4_command(cmd, args, raw_after_cmd)

        if cmd == "manifest.show":
            return self.command_manifest_show(), "json"
        if cmd == "manifest.save":
            path = args[0] if args else ""
            return self.command_manifest_save(path), "ok"
        if cmd == "manifest.load":
            path = args[0] if args else ""
            return self.command_manifest_load(path)

        if cmd == "node.list":
            return self.command_node_list(), "json"
        if cmd == "node.select":
            if not args:
                return "Usage: node.select <node-id>", "err"
            return self.command_node_select(args[0])
        if cmd == "node.add":
            payload_raw = raw_after_cmd
            return self.command_node_add(payload_raw)
        if cmd == "node.patch":
            target = args[0] if args else self.selected_node_id
            return self.command_node_patch(target)
        if cmd == "node.validate":
            target = args[0] if args else self.selected_node_id
            return self.command_node_validate(target)

        if cmd == "file.search":
            query = " ".join(args)
            return self.command_file_search(query), "json"
        if cmd == "diagnostics.list":
            return self.command_diagnostics_list(), "json"
        if cmd == "diagnostics.patch":
            return self.command_diagnostics_patch()
        if cmd == "emit.preview":
            return self.command_emit_preview(), "muted"
        if cmd == "build.run":
            return self.command_build_run()
        if cmd == "version.snapshot":
            return self.command_version_snapshot(), "json"
        if cmd == "version.list":
            return self.format_json(self.versions), "json"
        if cmd == "reboot":
            return self.command_reboot(), "warn"

        dynamic_result = self.dispatch_configure4_runtime_command(cmd, raw_after_cmd)
        if dynamic_result is not None:
            return dynamic_result

        return f"Unknown command: {cmd}. Type help for available commands.", "err"

    def help_text(self) -> str:
        return """
Blue CLI REPL OS commands

Service control:
  service.list
  service.status <name|all>
  service.activate <name|all>
  service.deactivate <name|all>
  service.restart <name|all>
  service.exec <name> [payload]

API gateway:
  service.activate api.gateway
  api.list
  api.call <route> [json]
  api.register <route> <service-id>

Codex services:
  service.activate codex.graph
  node.list
  node.select <id>
  node.add {"id":"custom.object","label":"Custom Object","type":"module","description":"..."}
  node.patch [id]
  node.validate <id|all>
  file.search <query>

Manifest, diagnostics, emission, build, versioning:
  service.activate codex.manifest
  manifest.show
  manifest.save [path]
  manifest.load [path]
  service.activate diagnostics.runtime
  diagnostics.list
  diagnostics.patch
  service.activate codex.emitter
  emit.preview
  service.activate codex.builder
  build.run
  service.activate version.ledger
  version.snapshot
  version.list

Configure 4 authoring fabric:
  service.activate configure_4.authoring.fabric
  configure4.help
  configure4.help includes in-depth usage for text generation, pointer generation, and language-agnostic codebase planning.
  configure4.status
  configure4.mode <manual|semi_auto|auto>
  configure4.new {json}
  configure4.set <path> <value>
  configure4.get [path]
  configure4.validate [draft-id|all]
  configure4.preview [draft-id]
  configure4.register [draft-id]
  configure4.export <path>
  configure4.import <path>
  configure4.sample
  configure4.reset [draft-id|all]
  configure4.history
  configure4.list
  configure_4.authoring.fabric is dormant until activated with service.activate configure_4.authoring.fabric.

Quadtree desktop:
  service.activate quadtree.desktop
  quadtree.desktop.status
  quadtree.desktop.show
  quadtree.desktop.hide
  quadtree.desktop.module.list
  quadtree.desktop.module.create {json}
  quadtree.desktop.module.get {json}
  quadtree.desktop.module.patch {json}
  quadtree.desktop.layer.get {json}
  quadtree.desktop.layer.patch {json}
  quadtree.desktop.cell.get {json}
  quadtree.desktop.cell.set {json}
  quadtree.desktop.cell.patch {json}
  quadtree.desktop.cell.reset {json}
  quadtree.desktop.cell.subdivide {json}
  quadtree.desktop.cell.bind {json}
  quadtree.desktop.selection.set {json}
  quadtree.desktop.batch.create {json}
  quadtree.desktop.batch.select {json}
  quadtree.desktop.batch.patch {json}
  quadtree.desktop.batch.validate {json}
  quadtree.desktop.batch.preview {json}
  quadtree.desktop.batch.apply {json}
  quadtree.desktop.batch.rollback {json}
  quadtree.desktop.validate [json]
  quadtree.desktop.preview [json]
  quadtree.desktop.apply {json}
  quadtree.desktop.export.manifest [path]
  quadtree.desktop.export.image [path]
  quadtree.desktop.import.manifest <path|json>
  quadtree.desktop.snapshot
  quadtree.desktop remains dormant until activated; graphical actions stage or preview blue-CLI-compatible work.

Terminal kernel services:
  service.activate kernel.terminal.linux
  service.activate kernel.terminal.powershell7
  linux <command>
  pwsh <command>
  service.exec kernel.terminal.linux <command>
  service.exec kernel.terminal.powershell7 <command>
  api.call /terminal/linux {"command":"pwd"}
  api.call /terminal/powershell7 {"command":"Get-ChildItem"}
  terminal.cwd
  terminal.cd <path>
  terminal.timeout <seconds>

Kernel memory:
  service.activate kernel.memory
  set <key> <value>
  get <key>
  vars

Notes:
  Only the REPL Console and blue CLI engine are visible.
  Services activate or execute only after a command is submitted through the blue engine.
  API calls require api.gateway to be active. Route handlers also require their target service to be active.
""".strip()

    def resolve_service_id(self, service_id: str) -> str:
        aliases = {
            "linux": "kernel.terminal.linux",
            "bash": "kernel.terminal.linux",
            "sh": "kernel.terminal.linux",
            "linux.terminal": "kernel.terminal.linux",
            "terminal.linux": "kernel.terminal.linux",
            "pwsh": "kernel.terminal.powershell7",
            "powershell": "kernel.terminal.powershell7",
            "powershell7": "kernel.terminal.powershell7",
            "powershell.7": "kernel.terminal.powershell7",
            "terminal.pwsh": "kernel.terminal.powershell7",
            "terminal.powershell7": "kernel.terminal.powershell7",
            "configure4": CONFIGURE4_SERVICE_ID,
            "configure_4": CONFIGURE4_SERVICE_ID,
            "configure4.authoring": CONFIGURE4_SERVICE_ID,
            "authoring.fabric": CONFIGURE4_SERVICE_ID,
            "quadtree": QUADTREE_DESKTOP_SERVICE_ID,
            "quadtreeDesktop": QUADTREE_DESKTOP_SERVICE_ID,
            "quadtree.desktop": QUADTREE_DESKTOP_SERVICE_ID,
            "qdt": QUADTREE_DESKTOP_SERVICE_ID,
        }
        if service_id in aliases:
            return aliases[service_id]
        if hasattr(self, "configure4_state") and hasattr(self, "services"):
            for sid, svc in self.services.items():
                raw_aliases = svc.get("aliases", []) if isinstance(svc, dict) else []
                if isinstance(raw_aliases, list) and service_id in raw_aliases:
                    return sid
        return service_id

    def command_service_list(self) -> str:
        payload = []
        for sid, svc in self.services.items():
            item = {
                "id": sid,
                "layer": svc["layer"],
                "state": svc["state"],
                "calls": svc["calls"],
            }
            if sid.startswith("kernel.terminal."):
                item["available"] = svc.get("available", False)
                item["executable"] = svc.get("executable", "")
                item["cwd"] = str(self.terminal_cwd)
                item["timeout_seconds"] = self.terminal_timeout_seconds
            payload.append(item)
        return self.format_json(payload)

    def command_service_status(self, target: str) -> str:
        if target == "all":
            return self.command_service_list()
        resolved = self.resolve_service_id(target)
        svc = self.services.get(resolved)
        if not svc:
            return self.format_json({"error": f"Unknown service {target}"})
        payload = dict(svc)
        if resolved.startswith("kernel.terminal."):
            payload["cwd"] = str(self.terminal_cwd)
            payload["timeout_seconds"] = self.terminal_timeout_seconds
        return self.format_json(payload)

    def command_service_activate(self, target: str) -> str:
        targets = list(self.services) if target == "all" else [self.resolve_service_id(target)]
        activated = []
        warnings = []
        for service_id in targets:
            svc = self.services.get(service_id)
            if not svc:
                return f"Unknown service: {service_id}"
            if service_id == "blue.cli.engine":
                svc["state"] = "active"
            else:
                svc["state"] = "active"
                svc["activated_at"] = _dt.datetime.now().isoformat(timespec="seconds")
            if service_id == "kernel.terminal.linux" and not self.linux_shell:
                warnings.append("linux terminal executable missing: expected bash or sh")
            if service_id == "kernel.terminal.powershell7" and not self.pwsh_executable:
                warnings.append("PowerShell 7 executable missing: expected pwsh")
            if service_id == QUADTREE_DESKTOP_SERVICE_ID:
                self.quadtreeDesktop_state["activation_state"] = "active"
                self._qdt_record_event("service.activate", service_id)
            activated.append(service_id)
            self._record_event("service.activate", service_id)
        suffix = "" if not warnings else "\nWarnings: " + "; ".join(warnings)
        return "Activated by REPL command: " + ", ".join(activated) + suffix

    def command_service_deactivate(self, target: str) -> str:
        targets = [sid for sid in self.services if sid != "blue.cli.engine"] if target == "all" else [self.resolve_service_id(target)]
        deactivated = []
        for service_id in targets:
            if service_id == "blue.cli.engine":
                return "blue.cli.engine cannot be deactivated; it is the execution boundary."
            svc = self.services.get(service_id)
            if not svc:
                return f"Unknown service: {service_id}"
            svc["state"] = "dormant"
            svc["activated_at"] = ""
            if service_id == QUADTREE_DESKTOP_SERVICE_ID:
                self.quadtreeDesktop_state["activation_state"] = "dormant"
                self.quadtreeDesktop_state["visible"] = False
                self._hide_quadtree_desktop()
                self._qdt_record_event("service.deactivate", service_id)
            deactivated.append(service_id)
            self._record_event("service.deactivate", service_id)
        return "Deactivated by REPL command: " + ", ".join(deactivated)

    def execute_service(self, service_id: str, payload_raw: str) -> tuple[str, str]:
        service_id = self.resolve_service_id(service_id)
        if service_id not in self.services:
            return f"Unknown service: {service_id}", "err"
        if not self.require_service(service_id):
            return f"Service {service_id} is dormant. Activate it first: service.activate {service_id}", "err"
        svc = self.services[service_id]
        svc["calls"] = int(svc["calls"]) + 1
        payload = self.parse_payload(payload_raw)
        if service_id == "blue.cli.engine":
            result = self.help_text()
            tag = "ok"
        elif service_id == "kernel.clock":
            result = self.format_json({"now": _dt.datetime.now().isoformat(timespec="seconds"), "payload": payload})
            tag = "json"
        elif service_id == "kernel.memory":
            result = self.format_json({"vars": self.vars, "payload": payload})
            tag = "json"
        elif service_id == "kernel.fs":
            result = "kernel.fs ready. Use manifest.save [path] or manifest.load [path]."
            tag = "ok"
        elif service_id == "kernel.terminal.linux":
            result, tag = self.execute_linux_terminal(payload_raw)
        elif service_id == "kernel.terminal.powershell7":
            result, tag = self.execute_powershell7_terminal(payload_raw)
        elif service_id == "api.gateway":
            result = self.command_api_list()
            tag = "json"
        elif service_id == "codex.manifest":
            result = self.command_manifest_show()
            tag = "json"
        elif service_id == "codex.graph":
            result = self.command_node_list()
            tag = "json"
        elif service_id == "codex.validator":
            result, tag = self.command_node_validate("all")
        elif service_id == "codex.emitter":
            result = self.command_emit_preview()
            tag = "muted"
        elif service_id == "codex.builder":
            result, tag = self.command_build_run()
        elif service_id == "diagnostics.runtime":
            result = self.command_diagnostics_list()
            tag = "json"
        elif service_id == "version.ledger":
            result = self.command_version_snapshot()
            tag = "json"
        elif service_id == CONFIGURE4_SERVICE_ID:
            result = self.command_configure4_status()
            tag = "json"
        elif service_id == QUADTREE_DESKTOP_SERVICE_ID:
            result = self.command_quadtree_desktop_status()
            tag = "json"
        else:
            result = self.format_json({"service": service_id, "payload": payload})
            tag = "json"
        svc["last_result"] = result[:260]
        self._record_event("service.exec", service_id)
        return result, tag

    def command_api_list(self) -> str:
        if not self.require_service("api.gateway"):
            return "api.gateway is dormant. Activate it first: service.activate api.gateway"
        payload = []
        for route, spec in self.api_routes.items():
            payload.append({"route": route, "service": spec["service"], "description": spec["description"]})
        return self.format_json(payload)

    def command_api_call(self, route: str, payload_raw: str) -> tuple[str, str]:
        if not self.require_service("api.gateway"):
            return "api.gateway is dormant. Activate it first: service.activate api.gateway", "err"
        spec = self.api_routes.get(route)
        if not spec:
            spec = self.resolve_dynamic_api_route(route)
        if not spec:
            return f"Unknown API route: {route}", "err"
        service_id = str(spec["service"])
        if not self.require_service(service_id):
            return f"Route {route} requires dormant service {service_id}. Activate it first: service.activate {service_id}", "err"
        payload = self.parse_payload(payload_raw)
        handler = spec["handler"]
        result = handler(payload)
        self.services["api.gateway"]["calls"] = int(self.services["api.gateway"]["calls"]) + 1
        self.services[service_id]["calls"] = int(self.services[service_id]["calls"]) + 1
        self.services["api.gateway"]["last_result"] = f"{route} -> {service_id}"
        self.services[service_id]["last_result"] = self.format_json(result)[:260]
        self._record_event("api.call", route)
        return self.format_json(result), "json"

    def command_api_register(self, args: list[str], raw_after_cmd: str) -> tuple[str, str]:
        if not self.require_service("api.gateway"):
            return "api.gateway is dormant. Activate it first: service.activate api.gateway", "err"
        if len(args) < 2:
            return "Usage: api.register <route> <service-id>", "err"
        route, service_id = args[0], self.resolve_service_id(args[1])
        if not route.startswith("/"):
            return "API routes must start with /", "err"
        if service_id not in self.services:
            return f"Unknown service: {service_id}", "err"
        self.api_routes[route] = {
            "service": service_id,
            "description": f"Custom route bound to {service_id}.",
            "handler": lambda payload, sid=service_id, rt=route: {
                "route": rt,
                "service": sid,
                "payload": payload,
                "note": "Custom API route reached. Bind real logic by extending _create_api_routes.",
            },
        }
        self._record_event("api.register", f"{route} -> {service_id}")
        return f"Registered {route} -> {service_id}", "ok"

    def resolve_dynamic_api_route(self, route: str) -> dict[str, object] | None:
        if not route.startswith("/quadtreeDesktop/"):
            return None
        dynamic_patterns = (
            ("/quadtreeDesktop/modules/", self.api_quadtree_desktop_module),
            ("/quadtreeDesktop/layers/", self.api_quadtree_desktop_layer),
            ("/quadtreeDesktop/cells/", self.api_quadtree_desktop_cell),
        )
        for prefix, handler in dynamic_patterns:
            if route.startswith(prefix):
                return {
                    "service": QUADTREE_DESKTOP_SERVICE_ID,
                    "description": f"Dynamic quadtreeDesktop route for {route}.",
                    "handler": lambda payload, rt=route, fn=handler: fn(self._qdt_payload_with_route(payload, rt)),
                }
        return None

    def _qdt_payload_with_route(self, payload: object, route: str) -> dict[str, object]:
        result = copy.deepcopy(payload) if isinstance(payload, dict) else {}
        result["_route"] = route
        return result

    def _create_quadtree_desktop_state(self) -> dict[str, object]:
        state: dict[str, object] = {
            "desktop_id": QUADTREE_DESKTOP_ID,
            "version": QUADTREE_DESKTOP_VERSION,
            "activation_state": "dormant",
            "visible": False,
            "selected_module_ids": ["qdt.system.services"],
            "selected_cell_paths": [],
            "selected_layer_type": "input",
            "root_bounds": {"x": 0, "y": 0, "width": 560, "height": 560},
            "maximum_depth": 4,
            "global_style_tokens": {
                "background": Theme.bg,
                "panel": Theme.panel,
                "cell_border": Theme.border,
                "selection": Theme.cyan,
                "active": Theme.green,
                "warning": Theme.yellow,
                "error": Theme.red,
            },
            "layer_registry": {},
            "module_registry": {},
            "batch_registry": {},
            "rollback_registry": {},
            "pending_operations": {},
            "event_log": [],
            "validation_reports": {},
            "render_cache": {},
            "persistence_metadata": {"policy": "manifest", "last_saved_at": "", "last_loaded_at": ""},
            "export_metadata": {"last_export_path": "", "last_export_hash": "", "last_snapshot_id": ""},
        }
        for module_id in QUADTREE_DEFAULT_MODULE_IDS:
            module = self._qdt_create_module_record(module_id, self._qdt_default_module_label(module_id), self._qdt_default_module_description(module_id))
            state["module_registry"][module_id] = module
            for layer_type in QUADTREE_LAYER_TYPES:
                state["layer_registry"][f"{module_id}:{layer_type}"] = module[f"{layer_type}_layer"]
        return state

    def _qdt_default_module_label(self, module_id: str) -> str:
        parts = [part for part in module_id.replace("qdt.", "").split(".") if part]
        return "Qdt" + "".join(part[:1].upper() + part[1:] for part in parts)

    def _qdt_default_module_description(self, module_id: str) -> str:
        descriptions = {
            "qdt.system.services": "Quadtree view over the kernel service registry.",
            "qdt.system.api": "Quadtree view over API gateway routes.",
            "qdt.codex.graph": "Quadtree view over Codex graph nodes.",
            "qdt.codex.files": "Quadtree view over file objects.",
            "qdt.codex.diagnostics": "Quadtree view over diagnostics.",
            "qdt.codex.emission": "Quadtree view over emitted previews and build-facing outputs.",
            "qdt.runtime.terminal": "Quadtree view over terminal session metadata.",
            "qdt.runtime.memory": "Quadtree view over kernel memory variables and command history.",
            "qdt.configure4.authoring": "Quadtree view over Configure4 authoring drafts and diagnostics.",
            "qdt.desktop.output": "Quadtree output module for desktop render, validation, and export reports.",
        }
        return descriptions.get(module_id, "Quadtree module.")

    def _qdt_default_binding(self, module_id: str) -> dict[str, object]:
        bindings = {
            "qdt.system.services": {"type": "service_registry", "target": "self.services"},
            "qdt.system.api": {"type": "api_route_registry", "target": "self.api_routes"},
            "qdt.codex.graph": {"type": "codex_nodes", "target": "self.nodes"},
            "qdt.codex.files": {"type": "file_objects", "target": "self.files"},
            "qdt.codex.diagnostics": {"type": "diagnostics", "target": "self.diagnostics"},
            "qdt.codex.emission": {"type": "emission", "target": "emit.preview"},
            "qdt.runtime.terminal": {"type": "terminal_session", "target": "self.terminal_cwd"},
            "qdt.runtime.memory": {"type": "kernel_memory", "target": "self.vars"},
            "qdt.configure4.authoring": {"type": "configure4_state", "target": "self.configure4_state"},
            "qdt.desktop.output": {"type": "quadtree_desktop_state", "target": "self.quadtreeDesktop_state"},
        }
        return copy.deepcopy(bindings.get(module_id, {"type": "custom", "target": module_id}))

    def _qdt_create_module_record(self, module_id: str, label: str, description: str) -> dict[str, object]:
        module = {
            "schema": "quadtree-module",
            "module_id": module_id,
            "label": label,
            "description": description,
            "status": "dormant",
            "bindings": {"root": self._qdt_default_binding(module_id)},
            "batch_profiles": {},
            "validation_gates": ["schema", "policy", "reference_integrity", "bounds", "batch_conflicts", "command_boundary"],
            "permissions": {
                "can_stage_commands": True,
                "can_apply_mutations": True,
                "can_execute_kernel_actions": False,
                "filesystem_writes": "explicit_only",
            },
            "persistence_policy": "manifest",
            "render_policy": {"visible": True, "canvas": "tkinter", "show_badges": True, "show_text": True},
        }
        for layer_type in QUADTREE_LAYER_TYPES:
            module[f"{layer_type}_layer"] = self._qdt_create_layer(module_id, layer_type)
        return module

    def _qdt_create_layer(self, module_id: str, layer_type: str) -> dict[str, object]:
        existing_state = self.__dict__.get("quadtreeDesktop_state", {})
        bounds = copy.deepcopy(existing_state.get("root_bounds", {"x": 0, "y": 0, "width": 560, "height": 560})) if isinstance(existing_state, dict) else {"x": 0, "y": 0, "width": 560, "height": 560}
        root_cell = self._qdt_create_cell(module_id, layer_type, "root", bounds=bounds, object_binding=self._qdt_default_binding(module_id))
        layer: dict[str, object] = {
            "layer_type": layer_type,
            "root_cell": root_cell["address"],
            "max_depth": 4,
            "root_size": bounds["width"],
            "cell_defaults": {
                "status": "dormant",
                "visible": True,
                "locked": False,
                "z_order": 0,
                "style_binding": {"token": "default"},
                "content_binding": {"text": ""},
            },
            "subdivision_policy": {"enabled": True, "max_depth": 4, "quadrants": list(QUADTREE_QUADRANTS)},
            "selection_policy": {"mode": "single_or_batch", "aliases": ["selected", "root"]},
            "mutation_policy": {"default_apply_mode": "stage", "requires_validation": True, "transactional_batches": True},
            "event_policy": {"record_input_events": True, "route_to_processing": True, "no_direct_kernel_execution": True},
            "command_policy": {"stage_only_by_default": True, "blue_cli_boundary": True},
            "render_policy": {"canvas": "tkinter", "rectangle": True, "text": True, "badge": True, "selection_outline": True},
            "persistence_policy": {"policy": "manifest", "include_cells": True},
            "style_policy": {"allow_color": True, "allow_badge": True, "allow_image_reference": True},
            "content_policy": {"allow_text": True, "allow_json": True, "allow_image_reference": True},
            "allowed_bindings": [
                "service",
                "api_route",
                "codex_node",
                "file_object",
                "diagnostic",
                "manifest_subtree",
                "configure4_draft",
                "terminal_cwd",
                "command_history",
                "event_log",
                "version_snapshot",
                "output_artifact",
            ],
            "cells": {"root": root_cell},
        }
        if layer_type == "input":
            layer.update(
                {
                    "max_depth": 4,
                    "root_size": bounds["width"],
                    "subdivision_enabled": True,
                    "context_menu_enabled": True,
                    "allowed_input_events": ["click", "right_click", "keyboard", "paste", "drag", "batch_edit", "api_payload", "terminal_text"],
                    "accepted_payload_types": ["text", "json", "image_reference", "file_reference", "clipboard_payload", "manifest_object"],
                    "clipboard_policy": "record_payload_then_route",
                    "file_input_policy": "explicit_reference_only",
                    "cell_customization_policy": "stage_patch_then_preview",
                    "export_intent_policy": "manifest_first_explicit_filesystem",
                }
            )
        return layer

    def _qdt_create_cell(
        self,
        module_id: str,
        layer_type: str,
        path: str,
        *,
        parent_id: str = "",
        quadrant: str = "root",
        depth: int = 0,
        bounds: dict[str, object] | None = None,
        object_binding: dict[str, object] | None = None,
    ) -> dict[str, object]:
        bounds = copy.deepcopy(bounds or {"x": 0, "y": 0, "width": 560, "height": 560})
        address = self._qdt_address(module_id, layer_type, path)
        return {
            "id": address,
            "address": address,
            "path": path,
            "module_id": module_id,
            "layer_type": layer_type,
            "parent_id": parent_id,
            "child_ids": [],
            "quadrant": quadrant,
            "depth": depth,
            "x": int(bounds.get("x", 0)),
            "y": int(bounds.get("y", 0)),
            "width": int(bounds.get("width", 0)),
            "height": int(bounds.get("height", 0)),
            "z_order": 0,
            "visible": True,
            "locked": False,
            "selected": False,
            "status": "dormant",
            "object_binding": copy.deepcopy(object_binding or {}),
            "command_binding": {},
            "api_route_binding": {},
            "service_binding": {},
            "style_binding": {"token": "default", "fill": "", "outline": ""},
            "content_binding": {"text": path, "image_reference": "", "json": {}},
            "validators": ["bounds", "depth", "permissions"],
            "event_handlers": {"click": "stage_selection", "right_click": "preview_context_actions"},
            "permissions": {"mutate": True, "bind": True, "subdivide": True, "execute": False},
            "tags": [],
            "last_mutation_record": {},
        }

    def _qdt_address(self, module_id: str, layer_type: str, path: str) -> str:
        return f"qdt://{QUADTREE_DESKTOP_ID}/{module_id}/{layer_type}/{path}"

    def _qdt_parse_address(self, address: str) -> dict[str, str] | None:
        prefix = f"qdt://{QUADTREE_DESKTOP_ID}/"
        if not isinstance(address, str) or not address.startswith(prefix):
            return None
        rest = address[len(prefix):]
        parts = rest.split("/", 2)
        if len(parts) != 3:
            return None
        return {"module_id": parts[0], "layer_type": parts[1], "path": parts[2]}

    def _qdt_require_service(self) -> tuple[bool, str]:
        if not self.require_service(QUADTREE_DESKTOP_SERVICE_ID):
            return False, f"{QUADTREE_DESKTOP_SERVICE_ID} is dormant. Activate it first: service.activate {QUADTREE_DESKTOP_SERVICE_ID}"
        return True, ""

    def dispatch_quadtree_desktop_command(self, cmd: str, args: list[str], raw_after_cmd: str) -> tuple[str, str]:
        ok, message = self._qdt_require_service()
        if not ok:
            return message, "err"
        action = cmd.removeprefix("quadtree.desktop.")
        payload = self._qdt_command_payload(args, raw_after_cmd)
        simple_json_actions = {
            "status": self.command_quadtree_desktop_status,
            "module.list": self.command_quadtree_desktop_module_list,
            "snapshot": self.command_quadtree_desktop_snapshot,
        }
        if action in simple_json_actions:
            return simple_json_actions[action](), "json"
        if action == "show":
            return self.command_quadtree_desktop_show(), "ok"
        if action == "hide":
            return self.command_quadtree_desktop_hide(), "ok"
        handler_map = {
            "module.create": self.command_quadtree_desktop_module_create,
            "module.get": self.command_quadtree_desktop_module_get,
            "module.patch": self.command_quadtree_desktop_module_patch,
            "layer.get": self.command_quadtree_desktop_layer_get,
            "layer.patch": self.command_quadtree_desktop_layer_patch,
            "cell.get": self.command_quadtree_desktop_cell_get,
            "cell.set": self.command_quadtree_desktop_cell_set,
            "cell.patch": self.command_quadtree_desktop_cell_patch,
            "cell.reset": self.command_quadtree_desktop_cell_reset,
            "cell.subdivide": self.command_quadtree_desktop_cell_subdivide,
            "cell.bind": self.command_quadtree_desktop_cell_bind,
            "selection.set": self.command_quadtree_desktop_selection_set,
            "batch.create": self.command_quadtree_desktop_batch_create,
            "batch.select": self.command_quadtree_desktop_batch_select,
            "batch.patch": self.command_quadtree_desktop_batch_patch,
            "batch.validate": self.command_quadtree_desktop_batch_validate,
            "batch.preview": self.command_quadtree_desktop_batch_preview,
            "batch.apply": self.command_quadtree_desktop_batch_apply,
            "batch.rollback": self.command_quadtree_desktop_batch_rollback,
            "validate": self.command_quadtree_desktop_validate,
            "preview": self.command_quadtree_desktop_preview,
            "apply": self.command_quadtree_desktop_apply,
            "export.manifest": self.command_quadtree_desktop_export_manifest,
            "export.image": self.command_quadtree_desktop_export_image,
            "import.manifest": self.command_quadtree_desktop_import_manifest,
        }
        handler = handler_map.get(action)
        if not handler:
            return f"Unknown quadtree desktop command: {cmd}", "err"
        result, tag = handler(payload)
        return self.format_json(result) if isinstance(result, (dict, list)) else str(result), tag

    def _qdt_command_payload(self, args: list[str], raw_after_cmd: str) -> object:
        raw = raw_after_cmd.strip()
        if raw.startswith("{") or raw.startswith("["):
            return self.parse_payload(raw)
        if raw:
            parsed = self.parse_payload(raw)
            if isinstance(parsed, dict) and parsed.get("raw") == raw and args:
                return {"path": args[0], "raw": raw, "args": args}
            return parsed
        return {}

    def command_quadtree_desktop_status(self) -> str:
        return self.format_json(self._qdt_status_payload())

    def _qdt_status_payload(self) -> dict[str, object]:
        modules = self.quadtreeDesktop_state.get("module_registry", {})
        batches = self.quadtreeDesktop_state.get("batch_registry", {})
        events = self.quadtreeDesktop_state.get("event_log", [])
        return {
            "service_id": QUADTREE_DESKTOP_SERVICE_ID,
            "label": QUADTREE_DESKTOP_LABEL,
            "desktop_id": self.quadtreeDesktop_state.get("desktop_id", QUADTREE_DESKTOP_ID),
            "version": self.quadtreeDesktop_state.get("version", QUADTREE_DESKTOP_VERSION),
            "activation_state": self.quadtreeDesktop_state.get("activation_state", "dormant"),
            "visible": self.quadtreeDesktop_state.get("visible", False),
            "module_count": len(modules) if isinstance(modules, dict) else 0,
            "batch_count": len(batches) if isinstance(batches, dict) else 0,
            "selected_module_ids": copy.deepcopy(self.quadtreeDesktop_state.get("selected_module_ids", [])),
            "selected_layer_type": self.quadtreeDesktop_state.get("selected_layer_type", "input"),
            "selected_cell_paths": copy.deepcopy(self.quadtreeDesktop_state.get("selected_cell_paths", [])),
            "latest_events": copy.deepcopy(events[-10:]) if isinstance(events, list) else [],
            "state_hash": self._qdt_state_hash(),
        }

    def command_quadtree_desktop_show(self) -> str:
        self.quadtreeDesktop_state["visible"] = True
        self.quadtreeDesktop_state["activation_state"] = "active"
        self._qdt_record_event("desktop.show", "Quadtree desktop requested through blue CLI.")
        self._show_quadtree_desktop()
        return "QuadtreeDesktop visible. Blue CLI remains the authoritative execution boundary."

    def command_quadtree_desktop_hide(self) -> str:
        self.quadtreeDesktop_state["visible"] = False
        self._qdt_record_event("desktop.hide", "Quadtree desktop hidden through blue CLI.")
        self._hide_quadtree_desktop()
        return "QuadtreeDesktop hidden. Service remains active until service.deactivate quadtree.desktop."

    def command_quadtree_desktop_module_list(self) -> str:
        modules = self.quadtreeDesktop_state.get("module_registry", {})
        rows = []
        if isinstance(modules, dict):
            for module_id, module in modules.items():
                rows.append(
                    {
                        "module_id": module_id,
                        "label": module.get("label", module_id),
                        "status": module.get("status", ""),
                        "layers": [layer for layer in QUADTREE_LAYER_TYPES if f"{layer}_layer" in module],
                    }
                )
        return self.format_json(rows)

    def command_quadtree_desktop_module_get(self, payload: object) -> tuple[dict[str, object], str]:
        module_id = self._qdt_payload_value(payload, "module_id", "")
        if not module_id and isinstance(payload, dict):
            module_id = self._qdt_module_from_target(payload)
        module = self._qdt_get_module(module_id)
        if not module:
            return {"accepted": False, "error": f"Unknown module: {module_id}"}, "err"
        return {"accepted": True, "module": copy.deepcopy(module)}, "json"

    def command_quadtree_desktop_module_create(self, payload: object) -> tuple[dict[str, object], str]:
        if not isinstance(payload, dict) or payload.get("parse_error"):
            return {"accepted": False, "error": "module.create requires a JSON object."}, "err"
        module_id = str(payload.get("module_id", "")).strip()
        if not self._qdt_valid_module_id(module_id):
            return {"accepted": False, "error": "module_id must be a stable dot-delimited id."}, "err"
        if module_id in self.quadtreeDesktop_state["module_registry"]:
            return {"accepted": False, "error": f"Module already exists: {module_id}"}, "err"
        module = self._qdt_create_module_record(module_id, str(payload.get("label", self._qdt_default_module_label(module_id))), str(payload.get("description", "Custom quadtree module.")))
        module = self._qdt_deep_merge(module, {key: copy.deepcopy(value) for key, value in payload.items() if key in {"status", "bindings", "batch_profiles", "validation_gates", "permissions", "persistence_policy", "render_policy"}})
        validation = self._qdt_validate_module(module)
        if not validation["accepted"]:
            return {"accepted": False, "validation": validation}, "err"
        self.quadtreeDesktop_state["module_registry"][module_id] = module
        for layer_type in QUADTREE_LAYER_TYPES:
            self.quadtreeDesktop_state["layer_registry"][f"{module_id}:{layer_type}"] = module[f"{layer_type}_layer"]
        self._qdt_record_event("module.create", module_id)
        self._render_quadtree_desktop()
        return {"accepted": True, "module_id": module_id, "module": module}, "json"

    def command_quadtree_desktop_module_patch(self, payload: object) -> tuple[dict[str, object], str]:
        return self._qdt_apply_target_patch(payload, target_type="module")

    def command_quadtree_desktop_layer_get(self, payload: object) -> tuple[dict[str, object], str]:
        module_id, layer_type = self._qdt_layer_ref(payload)
        layer = self._qdt_get_layer(module_id, layer_type)
        if not layer:
            return {"accepted": False, "error": f"Unknown layer: {module_id}/{layer_type}"}, "err"
        return {"accepted": True, "layer": copy.deepcopy(layer)}, "json"

    def command_quadtree_desktop_layer_patch(self, payload: object) -> tuple[dict[str, object], str]:
        return self._qdt_apply_target_patch(payload, target_type="layer")

    def command_quadtree_desktop_cell_get(self, payload: object) -> tuple[dict[str, object], str]:
        cell_ref = self._qdt_cell_ref(payload)
        cell = self._qdt_get_cell_by_ref(cell_ref)
        if not cell:
            return {"accepted": False, "error": f"Unknown cell: {cell_ref}"}, "err"
        return {"accepted": True, "cell": copy.deepcopy(cell)}, "json"

    def command_quadtree_desktop_cell_set(self, payload: object) -> tuple[dict[str, object], str]:
        if not isinstance(payload, dict):
            return {"accepted": False, "error": "cell.set requires JSON."}, "err"
        prop = str(payload.get("property", payload.get("path", ""))).strip()
        if not prop:
            return {"accepted": False, "error": "cell.set requires property or path."}, "err"
        patch: dict[str, object] = {}
        self._qdt_set_path(patch, prop.split("."), copy.deepcopy(payload.get("value", "")))
        rewritten = copy.deepcopy(payload)
        rewritten["patch"] = patch
        return self._qdt_apply_target_patch(rewritten, target_type="cell")

    def command_quadtree_desktop_cell_patch(self, payload: object) -> tuple[dict[str, object], str]:
        return self._qdt_apply_target_patch(payload, target_type="cell")

    def command_quadtree_desktop_cell_reset(self, payload: object) -> tuple[dict[str, object], str]:
        cell_ref = self._qdt_cell_ref(payload)
        parsed = self._qdt_resolve_cell_ref(cell_ref)
        if not parsed:
            return {"accepted": False, "error": f"Unknown cell: {cell_ref}"}, "err"
        module_id, layer_type, path = parsed
        cell = self._qdt_get_cell(module_id, layer_type, path)
        if not cell:
            return {"accepted": False, "error": f"Unknown cell: {cell_ref}"}, "err"
        replacement = self._qdt_create_cell(
            module_id,
            layer_type,
            path,
            parent_id=str(cell.get("parent_id", "")),
            quadrant=str(cell.get("quadrant", "root")),
            depth=int(cell.get("depth", 0)),
            bounds={"x": cell.get("x", 0), "y": cell.get("y", 0), "width": cell.get("width", 0), "height": cell.get("height", 0)},
            object_binding=copy.deepcopy(cell.get("object_binding", {})),
        )
        replacement["last_mutation_record"] = self._qdt_mutation_record("cell.reset")
        self._qdt_set_cell(module_id, layer_type, path, replacement)
        self._qdt_record_event("cell.reset", replacement["address"])
        self._render_quadtree_desktop()
        return {"accepted": True, "cell": replacement}, "json"

    def command_quadtree_desktop_cell_subdivide(self, payload: object) -> tuple[dict[str, object], str]:
        cell_ref = self._qdt_cell_ref(payload)
        parsed = self._qdt_resolve_cell_ref(cell_ref)
        if not parsed:
            return {"accepted": False, "error": f"Unknown cell: {cell_ref}"}, "err"
        result = self._qdt_subdivide_cell(parsed[0], parsed[1], parsed[2], mutate=True)
        self._render_quadtree_desktop()
        return result, "json" if result.get("accepted") else "err"

    def command_quadtree_desktop_cell_bind(self, payload: object) -> tuple[dict[str, object], str]:
        if not isinstance(payload, dict):
            return {"accepted": False, "error": "cell.bind requires JSON."}, "err"
        binding_type = str(payload.get("binding_type", payload.get("type", "object_binding")))
        binding = copy.deepcopy(payload.get("binding", {}))
        if not binding:
            binding = {key: copy.deepcopy(value) for key, value in payload.items() if key in {"service_id", "api_route", "object_type", "target", "command"}}
        target_field = {
            "object": "object_binding",
            "object_binding": "object_binding",
            "command": "command_binding",
            "command_binding": "command_binding",
            "api": "api_route_binding",
            "api_route": "api_route_binding",
            "api_route_binding": "api_route_binding",
            "service": "service_binding",
            "service_binding": "service_binding",
            "style": "style_binding",
            "content": "content_binding",
        }.get(binding_type, "object_binding")
        rewritten = copy.deepcopy(payload)
        rewritten["patch"] = {target_field: binding}
        return self._qdt_apply_target_patch(rewritten, target_type="cell")

    def command_quadtree_desktop_selection_set(self, payload: object) -> tuple[dict[str, object], str]:
        refs = self._qdt_selection_refs(payload)
        if not refs:
            return {"accepted": False, "error": "selection.set requires address, target, or addresses."}, "err"
        selected = []
        for ref in refs:
            parsed = self._qdt_resolve_cell_ref(ref)
            if not parsed:
                continue
            cell = self._qdt_get_cell(*parsed)
            if not cell:
                continue
            selected.append(cell["address"])
        self._qdt_set_selection(selected)
        self._qdt_record_event("selection.set", ", ".join(selected))
        self._render_quadtree_desktop()
        return {"accepted": True, "selected_cell_paths": selected}, "json"

    def command_quadtree_desktop_batch_create(self, payload: object) -> tuple[dict[str, object], str]:
        if not isinstance(payload, dict):
            return {"accepted": False, "error": "batch.create requires JSON."}, "err"
        batch_id = str(payload.get("batch_id", f"batch_{len(self.quadtreeDesktop_state['batch_registry']) + 1:03d}"))
        if batch_id in self.quadtreeDesktop_state["batch_registry"]:
            return {"accepted": False, "error": f"Batch already exists: {batch_id}"}, "err"
        batch = {
            "batch_id": batch_id,
            "label": str(payload.get("label", batch_id)),
            "selectors": copy.deepcopy(payload.get("selectors", payload.get("selection", {}))),
            "patch": copy.deepcopy(payload.get("patch", {})),
            "status": "staged",
            "created_at": _dt.datetime.now().isoformat(timespec="seconds"),
            "last_preview": {},
            "last_apply": {},
        }
        self.quadtreeDesktop_state["batch_registry"][batch_id] = batch
        self.quadtreeDesktop_state["selected_batch_id"] = batch_id
        self._qdt_record_event("batch.create", batch_id)
        return {"accepted": True, "batch": batch}, "json"

    def command_quadtree_desktop_batch_select(self, payload: object) -> tuple[dict[str, object], str]:
        batch_id = self._qdt_payload_value(payload, "batch_id", "")
        if not batch_id or batch_id not in self.quadtreeDesktop_state["batch_registry"]:
            return {"accepted": False, "error": f"Unknown batch: {batch_id}"}, "err"
        self.quadtreeDesktop_state["selected_batch_id"] = batch_id
        self._qdt_record_event("batch.select", batch_id)
        self._render_quadtree_desktop()
        return {"accepted": True, "batch_id": batch_id}, "json"

    def command_quadtree_desktop_batch_patch(self, payload: object) -> tuple[dict[str, object], str]:
        batch_id = self._qdt_payload_value(payload, "batch_id", str(self.quadtreeDesktop_state.get("selected_batch_id", "")))
        batch = self.quadtreeDesktop_state["batch_registry"].get(batch_id)
        if not batch:
            return {"accepted": False, "error": f"Unknown batch: {batch_id}"}, "err"
        batch_patch = copy.deepcopy(payload.get("patch", payload)) if isinstance(payload, dict) else {}
        for key in ("batch_id", "raw", "args"):
            batch_patch.pop(key, None)
        batch.update(self._qdt_deep_merge(batch, batch_patch))
        self._qdt_record_event("batch.patch", batch_id)
        return {"accepted": True, "batch": batch}, "json"

    def command_quadtree_desktop_batch_validate(self, payload: object) -> tuple[dict[str, object], str]:
        batch = self._qdt_batch_from_payload(payload)
        if not batch:
            return {"accepted": False, "error": "Unknown batch."}, "err"
        report = self._qdt_preview_batch(batch, mutate=False)
        return report, "json" if report.get("accepted") else "err"

    def command_quadtree_desktop_batch_preview(self, payload: object) -> tuple[dict[str, object], str]:
        batch = self._qdt_batch_from_payload(payload)
        if not batch:
            return {"accepted": False, "error": "Unknown batch."}, "err"
        report = self._qdt_preview_batch(batch, mutate=False)
        batch["last_preview"] = report
        self._qdt_record_event("batch.preview", str(batch.get("batch_id", "")))
        return report, "json" if report.get("accepted") else "err"

    def command_quadtree_desktop_batch_apply(self, payload: object) -> tuple[dict[str, object], str]:
        batch = self._qdt_batch_from_payload(payload)
        if not batch:
            return {"accepted": False, "error": "Unknown batch."}, "err"
        report = self._qdt_preview_batch(batch, mutate=True)
        batch["last_apply"] = report
        self._qdt_record_event("batch.apply", str(batch.get("batch_id", "")))
        self._render_quadtree_desktop()
        return report, "json" if report.get("accepted") else "err"

    def command_quadtree_desktop_batch_rollback(self, payload: object) -> tuple[dict[str, object], str]:
        token = self._qdt_payload_value(payload, "rollback_token", self._qdt_payload_value(payload, "token", ""))
        rollback = self.quadtreeDesktop_state.get("rollback_registry", {}).get(token)
        if not rollback:
            return {"accepted": False, "error": f"Unknown rollback token: {token}"}, "err"
        self._restore_quadtree_desktop_state(rollback["before_state"])
        self._qdt_record_event("batch.rollback", token)
        self._render_quadtree_desktop()
        return {"accepted": True, "rollback_token": token}, "json"

    def command_quadtree_desktop_validate(self, payload: object) -> tuple[dict[str, object], str]:
        report = self._qdt_validate_state(self.quadtreeDesktop_state)
        self.quadtreeDesktop_state.setdefault("validation_reports", {})["latest"] = report
        self._qdt_record_event("desktop.validate", "accepted" if report["accepted"] else "failed")
        return report, "json" if report["accepted"] else "err"

    def command_quadtree_desktop_preview(self, payload: object) -> tuple[dict[str, object], str]:
        if isinstance(payload, dict) and "batch_id" in payload:
            return self.command_quadtree_desktop_batch_preview(payload)
        if isinstance(payload, dict) and "patch" in payload:
            return self._qdt_apply_target_patch(payload, target_type=str(payload.get("target_type", "cell")), force_stage=True)
        report = {
            "accepted": True,
            "state_hash": self._qdt_state_hash(),
            "selected": copy.deepcopy(self.quadtreeDesktop_state.get("selected_cell_paths", [])),
            "modules": list(self.quadtreeDesktop_state.get("module_registry", {})),
            "note": "Preview is manifest-safe and does not mutate desktop state.",
        }
        self._qdt_record_event("desktop.preview", "state")
        return report, "json"

    def command_quadtree_desktop_apply(self, payload: object) -> tuple[dict[str, object], str]:
        if isinstance(payload, dict) and "operation_token" in payload:
            op = self.quadtreeDesktop_state.get("pending_operations", {}).get(str(payload["operation_token"]))
            if not op:
                return {"accepted": False, "error": f"Unknown operation token: {payload['operation_token']}"}, "err"
            return self._qdt_apply_target_patch(op["payload"], target_type=str(op.get("target_type", "cell")), force_apply=True)
        if isinstance(payload, dict) and "batch_id" in payload:
            return self.command_quadtree_desktop_batch_apply(payload)
        return self._qdt_apply_target_patch(payload, target_type=str(payload.get("target_type", "cell")) if isinstance(payload, dict) else "cell", force_apply=True)

    def command_quadtree_desktop_export_manifest(self, payload: object) -> tuple[dict[str, object] | str, str]:
        path = ""
        if isinstance(payload, dict):
            path = str(payload.get("path", payload.get("raw", ""))).strip()
        export_payload = {"quadtreeDesktop_state": copy.deepcopy(self.quadtreeDesktop_state)}
        export_hash = self._qdt_hash(export_payload)
        self.quadtreeDesktop_state.setdefault("export_metadata", {})["last_export_hash"] = export_hash
        if path:
            if not self.require_service("kernel.fs"):
                return {"accepted": False, "error": "kernel.fs is dormant. Activate it first: service.activate kernel.fs"}, "err"
            output_path = Path(path).expanduser()
            if output_path.parent and str(output_path.parent) not in {"", "."}:
                output_path.parent.mkdir(parents=True, exist_ok=True)
            with open(output_path, "w", encoding="utf-8") as handle:
                json.dump(export_payload, handle, indent=2)
            self.quadtreeDesktop_state["export_metadata"]["last_export_path"] = str(output_path)
            self._qdt_record_event("export.manifest", str(output_path))
            return {"accepted": True, "path": str(output_path), "hash": export_hash}, "json"
        self._qdt_record_event("export.manifest", "console")
        return {"accepted": True, "hash": export_hash, "manifest": export_payload}, "json"

    def command_quadtree_desktop_export_image(self, payload: object) -> tuple[dict[str, object], str]:
        path = ""
        if isinstance(payload, dict):
            path = str(payload.get("path", payload.get("raw", ""))).strip()
        preview = {
            "accepted": True,
            "artifact_type": "tk_canvas_postscript",
            "note": "Pure-stdlib Tkinter can export the visible canvas as PostScript. PNG conversion is intentionally not performed without an explicit external tool.",
            "visible": bool(self.quadtreeDesktop_state.get("visible", False)),
            "path": path,
        }
        if not path:
            self._qdt_record_event("export.image.preview", "console")
            return preview, "json"
        if not self.require_service("kernel.fs"):
            return {"accepted": False, "error": "kernel.fs is dormant. Activate it first: service.activate kernel.fs"}, "err"
        if self.quadtree_canvas is None:
            return {"accepted": False, "error": "Quadtree canvas is not visible. Run quadtree.desktop.show first."}, "err"
        output_path = Path(path).expanduser()
        if output_path.suffix.lower() == ".png":
            return {"accepted": False, "error": "PNG export is not available in pure stdlib Tkinter. Use .ps or .eps, or export the manifest."}, "err"
        if output_path.parent and str(output_path.parent) not in {"", "."}:
            output_path.parent.mkdir(parents=True, exist_ok=True)
        self.quadtree_canvas.postscript(file=str(output_path), colormode="color")
        preview["path"] = str(output_path)
        preview["hash"] = self._qdt_hash({"path": str(output_path), "state": self._qdt_state_hash()})[:16]
        self.quadtreeDesktop_state.setdefault("export_metadata", {})["last_image_export_path"] = str(output_path)
        self._qdt_record_event("export.image", str(output_path))
        return preview, "json"

    def command_quadtree_desktop_import_manifest(self, payload: object) -> tuple[dict[str, object], str]:
        source = ""
        if isinstance(payload, dict):
            source = str(payload.get("path", payload.get("raw", ""))).strip()
        if source and not source.startswith("{"):
            if not self.require_service("kernel.fs"):
                return {"accepted": False, "error": "kernel.fs is dormant. Activate it first: service.activate kernel.fs"}, "err"
            with open(Path(source).expanduser(), "r", encoding="utf-8") as handle:
                imported = json.load(handle)
        else:
            imported = payload
        state_payload = imported.get("quadtreeDesktop_state", imported) if isinstance(imported, dict) else {}
        validation = self._qdt_validate_state(state_payload)
        if not validation["accepted"]:
            return {"accepted": False, "validation": validation}, "err"
        self._restore_quadtree_desktop_state(state_payload)
        self.quadtreeDesktop_state.setdefault("persistence_metadata", {})["last_loaded_at"] = _dt.datetime.now().isoformat(timespec="seconds")
        self._qdt_record_event("import.manifest", source or "payload")
        self._render_quadtree_desktop()
        return {"accepted": True, "validation": validation, "state_hash": self._qdt_state_hash()}, "json"

    def command_quadtree_desktop_snapshot(self) -> str:
        snapshot = {
            "id": f"qdt-snap-{_dt.datetime.now().strftime('%Y%m%d.%H%M%S')}",
            "created_at": _dt.datetime.now().isoformat(timespec="seconds"),
            "state_hash": self._qdt_state_hash(),
            "status": self._qdt_status_payload(),
        }
        self.quadtreeDesktop_state.setdefault("render_cache", {})["latest_snapshot"] = snapshot
        self.quadtreeDesktop_state.setdefault("export_metadata", {})["last_snapshot_id"] = snapshot["id"]
        self._qdt_record_event("desktop.snapshot", snapshot["id"])
        return self.format_json(snapshot)

    def _qdt_payload_value(self, payload: object, key: str, default: object = "") -> str:
        if isinstance(payload, dict):
            value = payload.get(key, default)
            if value == "" and key == "module_id":
                value = payload.get("id", default)
            return str(value)
        return str(default)

    def _qdt_valid_module_id(self, module_id: str) -> bool:
        return bool(re.fullmatch(r"[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)+", module_id or ""))

    def _qdt_module_from_target(self, payload: object) -> str:
        if not isinstance(payload, dict):
            return ""
        target = str(payload.get("target", payload.get("address", "")))
        parsed = self._qdt_parse_address(target)
        if parsed:
            return parsed["module_id"]
        return target if target in self.quadtreeDesktop_state.get("module_registry", {}) else ""

    def _qdt_layer_ref(self, payload: object) -> tuple[str, str]:
        if not isinstance(payload, dict):
            selected_modules = self.quadtreeDesktop_state.get("selected_module_ids", ["qdt.system.services"])
            return str(selected_modules[0]), str(self.quadtreeDesktop_state.get("selected_layer_type", "input"))
        address = str(payload.get("address", payload.get("target", "")))
        parsed = self._qdt_parse_address(address)
        if parsed:
            return parsed["module_id"], parsed["layer_type"]
        module_id = str(payload.get("module_id", self._qdt_module_from_target(payload) or ""))
        if not module_id:
            selected_modules = self.quadtreeDesktop_state.get("selected_module_ids", ["qdt.system.services"])
            module_id = str(selected_modules[0]) if selected_modules else "qdt.system.services"
        layer_type = str(payload.get("layer_type", payload.get("layer", self.quadtreeDesktop_state.get("selected_layer_type", "input"))))
        return module_id, layer_type

    def _qdt_cell_ref(self, payload: object) -> str:
        if isinstance(payload, str):
            return payload
        if not isinstance(payload, dict):
            selected = self.quadtreeDesktop_state.get("selected_cell_paths", [])
            return str(selected[0]) if selected else self._qdt_address("qdt.system.services", "input", "root")
        for key in ("address", "target", "id"):
            if payload.get(key):
                return str(payload[key])
        if payload.get("selection") == "selected" or payload.get("alias") == "selected":
            selected = self.quadtreeDesktop_state.get("selected_cell_paths", [])
            return str(selected[0]) if selected else ""
        module_id, layer_type = self._qdt_layer_ref(payload)
        path = str(payload.get("path", "root"))
        return self._qdt_address(module_id, layer_type, path)

    def _qdt_selection_refs(self, payload: object) -> list[str]:
        if isinstance(payload, dict):
            addresses = payload.get("addresses", payload.get("selection", []))
            if isinstance(addresses, list):
                return [str(item) for item in addresses]
            ref = self._qdt_cell_ref(payload)
            return [ref] if ref else []
        if isinstance(payload, list):
            return [str(item) for item in payload]
        return []

    def _qdt_resolve_cell_ref(self, ref: str) -> tuple[str, str, str] | None:
        if not ref:
            return None
        parsed = self._qdt_parse_address(ref)
        if parsed:
            return parsed["module_id"], parsed["layer_type"], parsed["path"]
        if ref == "selected":
            selected = self.quadtreeDesktop_state.get("selected_cell_paths", [])
            if selected:
                return self._qdt_resolve_cell_ref(str(selected[0]))
        parts = ref.split(":")
        if len(parts) == 3:
            return parts[0], parts[1], parts[2]
        selected_modules = self.quadtreeDesktop_state.get("selected_module_ids", ["qdt.system.services"])
        module_id = str(selected_modules[0]) if selected_modules else "qdt.system.services"
        layer_type = str(self.quadtreeDesktop_state.get("selected_layer_type", "input"))
        return module_id, layer_type, ref

    def _qdt_get_module(self, module_id: str) -> dict[str, object] | None:
        modules = self.quadtreeDesktop_state.get("module_registry", {})
        return modules.get(module_id) if isinstance(modules, dict) else None

    def _qdt_get_layer(self, module_id: str, layer_type: str) -> dict[str, object] | None:
        module = self._qdt_get_module(module_id)
        if not module or layer_type not in QUADTREE_LAYER_TYPES:
            return None
        return module.get(f"{layer_type}_layer") if isinstance(module.get(f"{layer_type}_layer"), dict) else None

    def _qdt_get_cell(self, module_id: str, layer_type: str, path: str) -> dict[str, object] | None:
        layer = self._qdt_get_layer(module_id, layer_type)
        if not layer:
            return None
        cells = layer.get("cells", {})
        return cells.get(path) if isinstance(cells, dict) else None

    def _qdt_get_cell_by_ref(self, ref: str) -> dict[str, object] | None:
        parsed = self._qdt_resolve_cell_ref(ref)
        return self._qdt_get_cell(*parsed) if parsed else None

    def _qdt_set_cell(self, module_id: str, layer_type: str, path: str, cell: dict[str, object]) -> None:
        layer = self._qdt_get_layer(module_id, layer_type)
        if layer is None:
            return
        layer.setdefault("cells", {})[path] = cell
        self.quadtreeDesktop_state.setdefault("layer_registry", {})[f"{module_id}:{layer_type}"] = layer

    def _qdt_set_selection(self, addresses: list[str]) -> None:
        for module in self.quadtreeDesktop_state.get("module_registry", {}).values():
            if not isinstance(module, dict):
                continue
            for layer_type in QUADTREE_LAYER_TYPES:
                layer = module.get(f"{layer_type}_layer", {})
                for cell in layer.get("cells", {}).values() if isinstance(layer, dict) else []:
                    if isinstance(cell, dict):
                        cell["selected"] = False
        selected_modules: list[str] = []
        selected_layers: list[str] = []
        for address in addresses:
            parsed = self._qdt_resolve_cell_ref(address)
            if not parsed:
                continue
            cell = self._qdt_get_cell(*parsed)
            if not cell:
                continue
            cell["selected"] = True
            if parsed[0] not in selected_modules:
                selected_modules.append(parsed[0])
            if parsed[1] not in selected_layers:
                selected_layers.append(parsed[1])
        self.quadtreeDesktop_state["selected_cell_paths"] = addresses
        if selected_modules:
            self.quadtreeDesktop_state["selected_module_ids"] = selected_modules
        if selected_layers:
            self.quadtreeDesktop_state["selected_layer_type"] = selected_layers[0]

    def _qdt_apply_target_patch(self, payload: object, *, target_type: str, force_stage: bool = False, force_apply: bool = False) -> tuple[dict[str, object], str]:
        if not isinstance(payload, dict) or payload.get("parse_error"):
            return {"accepted": False, "error": f"{target_type}.patch requires a JSON object."}, "err"
        validation_mode = str(payload.get("validation_mode", "full"))
        apply_mode = str(payload.get("apply_mode", "stage"))
        if force_stage:
            apply_mode = "stage"
        if force_apply:
            apply_mode = "apply"
        if validation_mode not in QUADTREE_VALIDATION_MODES:
            return {"accepted": False, "error": f"Unsupported validation_mode: {validation_mode}"}, "err"
        if apply_mode not in QUADTREE_APPLY_MODES:
            return {"accepted": False, "error": f"Unsupported apply_mode: {apply_mode}"}, "err"
        patch = copy.deepcopy(payload.get("patch", {}))
        if not isinstance(patch, dict):
            return {"accepted": False, "error": "patch must be an object."}, "err"
        target = self._qdt_resolve_target(payload, target_type)
        if not target:
            return {"accepted": False, "error": f"Could not resolve {target_type} target."}, "err"
        before_hash = self._qdt_state_hash()
        working_state = copy.deepcopy(self.quadtreeDesktop_state)
        result = self._qdt_apply_patch_to_state(working_state, target_type, target, patch)
        if not result["accepted"]:
            return {**result, "before_hash": before_hash, "after_hash": before_hash, "mutated": False}, "err"
        validation = self._qdt_validate_state(working_state) if validation_mode in {"full", "schema_only", "policy_only"} else {"accepted": True, "mode": validation_mode}
        after_hash = self._qdt_hash(working_state)
        preview = {
            "accepted": bool(validation.get("accepted", False)),
            "target_type": target_type,
            "target": target,
            "validation_mode": validation_mode,
            "apply_mode": apply_mode,
            "validation": validation,
            "before_hash": before_hash,
            "after_hash": after_hash,
            "mutated": False,
        }
        if not preview["accepted"] or apply_mode == "stage":
            token = self._qdt_operation_token(target_type, target, patch)
            self.quadtreeDesktop_state.setdefault("pending_operations", {})[token] = {
                "target_type": target_type,
                "target": target,
                "payload": copy.deepcopy(payload),
                "preview": preview,
                "created_at": _dt.datetime.now().isoformat(timespec="seconds"),
            }
            preview["operation_token"] = token
            self._qdt_record_event(f"{target_type}.patch.stage", token)
            return preview, "json" if preview["accepted"] else "err"
        self.quadtreeDesktop_state = working_state
        self._qdt_refresh_layer_registry()
        if apply_mode == "apply_and_snapshot":
            self.command_quadtree_desktop_snapshot()
        self._qdt_record_event(f"{target_type}.patch.apply", str(target))
        self._render_quadtree_desktop()
        return {**preview, "mutated": True}, "json"

    def _qdt_resolve_target(self, payload: dict[str, object], target_type: str) -> dict[str, str] | None:
        if target_type == "cell":
            parsed = self._qdt_resolve_cell_ref(self._qdt_cell_ref(payload))
            if not parsed:
                return None
            return {"module_id": parsed[0], "layer_type": parsed[1], "path": parsed[2]}
        if target_type == "layer":
            module_id, layer_type = self._qdt_layer_ref(payload)
            return {"module_id": module_id, "layer_type": layer_type}
        if target_type == "module":
            module_id = str(payload.get("module_id", self._qdt_module_from_target(payload)))
            return {"module_id": module_id} if module_id else None
        return None

    def _qdt_apply_patch_to_state(self, state: dict[str, object], target_type: str, target: dict[str, str], patch: dict[str, object]) -> dict[str, object]:
        try:
            if target_type == "module":
                module = state["module_registry"][target["module_id"]]
                protected = {"module_id", "input_layer", "processing_layer", "output_layer"}
                safe_patch = {key: copy.deepcopy(value) for key, value in patch.items() if key not in protected}
                module.update(self._qdt_deep_merge(module, safe_patch))
            elif target_type == "layer":
                module = state["module_registry"][target["module_id"]]
                layer_key = f"{target['layer_type']}_layer"
                layer = module[layer_key]
                safe_patch = {key: copy.deepcopy(value) for key, value in patch.items() if key not in {"layer_type", "root_cell", "cells"}}
                layer.update(self._qdt_deep_merge(layer, safe_patch))
            elif target_type == "cell":
                module = state["module_registry"][target["module_id"]]
                layer = module[f"{target['layer_type']}_layer"]
                cell = layer["cells"][target["path"]]
                if cell.get("locked") and not patch.get("permissions", {}).get("mutate_locked"):
                    return {"accepted": False, "error": "Cell is locked."}
                protected = {"id", "address", "module_id", "layer_type", "path", "parent_id", "child_ids", "depth", "x", "y", "width", "height"}
                safe_patch = {key: copy.deepcopy(value) for key, value in patch.items() if key not in protected}
                merged = self._qdt_deep_merge(cell, safe_patch)
                merged["last_mutation_record"] = self._qdt_mutation_record("cell.patch")
                layer["cells"][target["path"]] = merged
            else:
                return {"accepted": False, "error": f"Unsupported target_type: {target_type}"}
        except KeyError as exc:
            return {"accepted": False, "error": f"Patch target not found: {exc}"}
        return {"accepted": True}

    def _qdt_subdivide_cell(self, module_id: str, layer_type: str, path: str, *, mutate: bool) -> dict[str, object]:
        state = self.quadtreeDesktop_state if mutate else copy.deepcopy(self.quadtreeDesktop_state)
        module = state.get("module_registry", {}).get(module_id)
        if not isinstance(module, dict):
            return {"accepted": False, "error": f"Unknown module: {module_id}"}
        layer = module.get(f"{layer_type}_layer")
        if not isinstance(layer, dict):
            return {"accepted": False, "error": f"Unknown layer: {layer_type}"}
        cell = layer.get("cells", {}).get(path)
        if not isinstance(cell, dict):
            return {"accepted": False, "error": f"Unknown cell path: {path}"}
        if cell.get("child_ids"):
            return {"accepted": False, "error": "Cell is already subdivided."}
        max_depth = int(layer.get("max_depth", layer.get("subdivision_policy", {}).get("max_depth", 4)))
        depth = int(cell.get("depth", 0))
        if depth >= max_depth or depth >= QUADTREE_MAX_DEPTH_LIMIT:
            return {"accepted": False, "error": "Max depth reached."}
        if not cell.get("permissions", {}).get("subdivide", True):
            return {"accepted": False, "error": "Cell does not allow subdivision."}
        x = int(cell.get("x", 0))
        y = int(cell.get("y", 0))
        width = max(int(cell.get("width", 0)) // 2, 1)
        height = max(int(cell.get("height", 0)) // 2, 1)
        child_specs = {
            "nw": {"x": x, "y": y, "width": width, "height": height},
            "ne": {"x": x + width, "y": y, "width": width, "height": height},
            "sw": {"x": x, "y": y + height, "width": width, "height": height},
            "se": {"x": x + width, "y": y + height, "width": width, "height": height},
        }
        child_ids = []
        for quadrant, bounds in child_specs.items():
            child_path = f"{path}.{quadrant}"
            child = self._qdt_create_cell(module_id, layer_type, child_path, parent_id=cell["address"], quadrant=quadrant, depth=depth + 1, bounds=bounds, object_binding=copy.deepcopy(cell.get("object_binding", {})))
            child["status"] = "editable"
            layer["cells"][child_path] = child
            child_ids.append(child["address"])
        cell["child_ids"] = child_ids
        cell["last_mutation_record"] = self._qdt_mutation_record("cell.subdivide")
        if mutate:
            self._qdt_refresh_layer_registry()
            self._qdt_record_event("cell.subdivide", cell["address"])
        return {"accepted": True, "cell": copy.deepcopy(cell), "children": child_ids}

    def _qdt_batch_from_payload(self, payload: object) -> dict[str, object] | None:
        if isinstance(payload, dict) and ("selectors" in payload or "patch" in payload) and "batch_id" not in payload:
            return {
                "batch_id": "ad_hoc",
                "selectors": copy.deepcopy(payload.get("selectors", payload.get("selection", {}))),
                "patch": copy.deepcopy(payload.get("patch", {})),
            }
        batch_id = self._qdt_payload_value(payload, "batch_id", str(self.quadtreeDesktop_state.get("selected_batch_id", "")))
        batch = self.quadtreeDesktop_state.get("batch_registry", {}).get(batch_id)
        return batch if isinstance(batch, dict) else None

    def _qdt_preview_batch(self, batch: dict[str, object], *, mutate: bool) -> dict[str, object]:
        selectors = batch.get("selectors", {})
        patch = batch.get("patch", {})
        if not isinstance(patch, dict):
            return {"accepted": False, "error": "Batch patch must be an object."}
        targets = self._qdt_select_cells(selectors)
        before_state = copy.deepcopy(self.quadtreeDesktop_state)
        before_hash = self._qdt_hash(before_state)
        working_state = copy.deepcopy(self.quadtreeDesktop_state)
        failures = []
        skipped = 0
        for address in targets:
            parsed = self._qdt_parse_address(address)
            if not parsed:
                skipped += 1
                continue
            result = self._qdt_apply_patch_to_state(working_state, "cell", parsed, patch)
            if not result["accepted"]:
                failures.append({"address": address, "error": result.get("error", "patch failed")})
        validation = self._qdt_validate_state(working_state)
        accepted = not failures and bool(validation.get("accepted", False))
        after_hash = self._qdt_hash(working_state) if accepted else before_hash
        token = self._qdt_hash({"batch": batch.get("batch_id", "ad_hoc"), "before": before_hash, "after": after_hash, "time": _dt.datetime.now().isoformat(timespec="seconds")})[:16]
        report = {
            "accepted": accepted,
            "batch_id": batch.get("batch_id", "ad_hoc"),
            "affected_cell_count": len(targets) if accepted else 0,
            "candidate_cell_count": len(targets),
            "skipped_cell_count": skipped,
            "validation_failures": failures + ([] if validation.get("accepted") else validation.get("errors", [])),
            "before_hash": before_hash,
            "after_hash": after_hash,
            "rollback_token": token if accepted and mutate else "",
            "mutated": False,
        }
        if accepted and mutate:
            self.quadtreeDesktop_state = working_state
            self.quadtreeDesktop_state.setdefault("rollback_registry", {})[token] = {"before_state": before_state, "after_hash": after_hash}
            self._qdt_refresh_layer_registry()
            report["mutated"] = True
        return report

    def _qdt_select_cells(self, selectors: object) -> list[str]:
        if isinstance(selectors, list):
            return [str(item) for item in selectors if self._qdt_get_cell_by_ref(str(item))]
        if not isinstance(selectors, dict):
            return copy.deepcopy(self.quadtreeDesktop_state.get("selected_cell_paths", []))
        explicit = selectors.get("addresses", selectors.get("cells", []))
        if isinstance(explicit, list) and explicit:
            return [str(item) for item in explicit if self._qdt_get_cell_by_ref(str(item))]
        results = []
        for module_id, module in self.quadtreeDesktop_state.get("module_registry", {}).items():
            if selectors.get("module_id") and selectors.get("module_id") != module_id:
                continue
            for layer_type in QUADTREE_LAYER_TYPES:
                if selectors.get("layer_type") and selectors.get("layer_type") != layer_type:
                    continue
                layer = module.get(f"{layer_type}_layer", {})
                for cell in layer.get("cells", {}).values() if isinstance(layer, dict) else []:
                    if not isinstance(cell, dict):
                        continue
                    if "depth" in selectors and int(cell.get("depth", -1)) != int(selectors["depth"]):
                        continue
                    if selectors.get("status") and selectors["status"] != cell.get("status"):
                        continue
                    if selectors.get("service_binding") and selectors["service_binding"] != cell.get("service_binding", {}).get("service_id"):
                        continue
                    if selectors.get("api_route_binding") and selectors["api_route_binding"] != cell.get("api_route_binding", {}).get("route"):
                        continue
                    if selectors.get("object_type") and selectors["object_type"] != cell.get("object_binding", {}).get("type"):
                        continue
                    if selectors.get("tag") and selectors["tag"] not in cell.get("tags", []):
                        continue
                    query = str(selectors.get("query", "")).lower()
                    if query and query not in json.dumps(cell, sort_keys=True).lower():
                        continue
                    results.append(str(cell["address"]))
        return results

    def _qdt_validate_state(self, state: object) -> dict[str, object]:
        errors: list[dict[str, object]] = []
        warnings: list[dict[str, object]] = []
        if not isinstance(state, dict):
            return {"accepted": False, "errors": [{"target": "state", "message": "State must be an object."}], "warnings": []}
        modules = state.get("module_registry", {})
        if not isinstance(modules, dict) or not modules:
            errors.append({"target": "module_registry", "message": "At least one module is required."})
        else:
            for module_id, module in modules.items():
                module_report = self._qdt_validate_module(module)
                for item in module_report.get("errors", []):
                    errors.append({"target": f"{module_id}:{item.get('target', '')}", "message": item.get("message", "")})
                warnings.extend(module_report.get("warnings", []))
        report = {
            "accepted": len(errors) == 0,
            "checked_at": _dt.datetime.now().isoformat(timespec="seconds"),
            "errors": errors,
            "warnings": warnings,
            "state_hash": self._qdt_hash(state),
        }
        return report

    def _qdt_validate_module(self, module: object) -> dict[str, object]:
        errors: list[dict[str, object]] = []
        warnings: list[dict[str, object]] = []
        if not isinstance(module, dict):
            return {"accepted": False, "errors": [{"target": "module", "message": "Module must be an object."}], "warnings": []}
        module_id = str(module.get("module_id", ""))
        if not self._qdt_valid_module_id(module_id):
            errors.append({"target": "module_id", "message": "Invalid module_id."})
        if module.get("schema") != "quadtree-module":
            errors.append({"target": "schema", "message": "Module schema must be quadtree-module."})
        for field_name in ("label", "description", "status", "bindings", "batch_profiles", "validation_gates", "permissions", "persistence_policy", "render_policy"):
            if field_name not in module:
                errors.append({"target": field_name, "message": "Missing required module field."})
        for layer_type in QUADTREE_LAYER_TYPES:
            layer = module.get(f"{layer_type}_layer")
            if not isinstance(layer, dict):
                errors.append({"target": f"{layer_type}_layer", "message": "Missing required layer."})
                continue
            errors.extend(self._qdt_validate_layer(module_id, layer_type, layer))
        return {"accepted": len(errors) == 0, "errors": errors, "warnings": warnings}

    def _qdt_validate_layer(self, module_id: str, layer_type: str, layer: dict[str, object]) -> list[dict[str, object]]:
        errors: list[dict[str, object]] = []
        required = ("layer_type", "root_cell", "max_depth", "cell_defaults", "subdivision_policy", "selection_policy", "mutation_policy", "event_policy", "style_policy", "content_policy", "allowed_bindings", "cells")
        for field_name in required:
            if field_name not in layer:
                errors.append({"target": f"{layer_type}.{field_name}", "message": "Missing required layer field."})
        if layer.get("layer_type") != layer_type:
            errors.append({"target": f"{layer_type}.layer_type", "message": "Layer type mismatch."})
        try:
            max_depth = int(layer.get("max_depth", 0))
        except (TypeError, ValueError):
            max_depth = -1
        if max_depth < 0 or max_depth > QUADTREE_MAX_DEPTH_LIMIT:
            errors.append({"target": f"{layer_type}.max_depth", "message": f"max_depth must be 0..{QUADTREE_MAX_DEPTH_LIMIT}."})
        cells = layer.get("cells", {})
        if not isinstance(cells, dict) or "root" not in cells:
            errors.append({"target": f"{layer_type}.cells", "message": "Layer must contain root cell."})
            return errors
        for path, cell in cells.items():
            if not isinstance(cell, dict):
                errors.append({"target": f"{layer_type}.{path}", "message": "Cell must be an object."})
                continue
            expected = self._qdt_address(module_id, layer_type, path)
            if cell.get("address") != expected:
                errors.append({"target": expected, "message": "Cell address does not match module/layer/path."})
            if int(cell.get("depth", 0)) > max_depth:
                errors.append({"target": expected, "message": "Cell depth exceeds layer max_depth."})
            for field_name in ("parent_id", "child_ids", "quadrant", "x", "y", "width", "height", "z_order", "visible", "locked", "selected", "status", "object_binding", "command_binding", "api_route_binding", "service_binding", "style_binding", "content_binding", "validators", "event_handlers", "permissions", "last_mutation_record"):
                if field_name not in cell:
                    errors.append({"target": expected, "message": f"Missing cell field: {field_name}."})
            service_id = str(cell.get("service_binding", {}).get("service_id", ""))
            if service_id and service_id not in self.services:
                errors.append({"target": expected, "message": f"Unknown service binding: {service_id}."})
            route = str(cell.get("api_route_binding", {}).get("route", ""))
            if route and route not in self.api_routes:
                errors.append({"target": expected, "message": f"Unknown API route binding: {route}."})
            command_name = str(cell.get("command_binding", {}).get("command", ""))
            if command_name and command_name not in self._configure4_known_commands() and command_name not in self.configure4_state.get("registered_runtime_commands", {}):
                errors.append({"target": expected, "message": f"Unknown command binding: {command_name}."})
            if bool(cell.get("permissions", {}).get("execute", False)):
                errors.append({"target": expected, "message": "Cells may not directly execute kernel actions; they must stage commands."})
        return errors

    def _qdt_deep_merge(self, base: object, patch: object) -> object:
        if isinstance(base, dict) and isinstance(patch, dict):
            result = copy.deepcopy(base)
            for key, value in patch.items():
                result[key] = self._qdt_deep_merge(result.get(key), value) if key in result else copy.deepcopy(value)
            return result
        return copy.deepcopy(patch)

    def _qdt_get_path(self, root: object, parts: list[str]) -> tuple[bool, object]:
        current = root
        for part in parts:
            if isinstance(current, dict) and part in current:
                current = current[part]
            elif isinstance(current, list) and part.isdigit() and int(part) < len(current):
                current = current[int(part)]
            else:
                return False, None
        return True, copy.deepcopy(current)

    def _qdt_set_path(self, root: dict[str, object], parts: list[str], value: object) -> None:
        current = root
        for part in parts[:-1]:
            if part not in current or not isinstance(current[part], dict):
                current[part] = {}
            current = current[part]
        if parts:
            current[parts[-1]] = copy.deepcopy(value)

    def _qdt_refresh_layer_registry(self) -> None:
        registry = {}
        for module_id, module in self.quadtreeDesktop_state.get("module_registry", {}).items():
            if not isinstance(module, dict):
                continue
            for layer_type in QUADTREE_LAYER_TYPES:
                layer = module.get(f"{layer_type}_layer")
                if isinstance(layer, dict):
                    registry[f"{module_id}:{layer_type}"] = layer
        self.quadtreeDesktop_state["layer_registry"] = registry

    def _qdt_mutation_record(self, kind: str) -> dict[str, str]:
        return {"kind": kind, "time": _dt.datetime.now().isoformat(timespec="seconds"), "boundary": "blue.cli.engine"}

    def _qdt_operation_token(self, target_type: str, target: dict[str, str], patch: dict[str, object]) -> str:
        return self._qdt_hash({"target_type": target_type, "target": target, "patch": patch, "time": _dt.datetime.now().isoformat(timespec="seconds")})[:16]

    def _qdt_hash(self, payload: object) -> str:
        return hashlib.sha256(json.dumps(payload, sort_keys=True, default=str, separators=(",", ":")).encode("utf-8")).hexdigest()

    def _qdt_state_hash(self) -> str:
        payload = copy.deepcopy(self.quadtreeDesktop_state)
        payload.pop("render_cache", None)
        return self._qdt_hash(payload)[:16]

    def _qdt_record_event(self, kind: str, detail: str) -> None:
        entry = {"time": _dt.datetime.now().isoformat(timespec="seconds"), "kind": kind, "detail": detail}
        self.quadtreeDesktop_state.setdefault("event_log", []).append(entry)
        if len(self.quadtreeDesktop_state["event_log"]) > QUADTREE_EVENT_LIMIT:
            self.quadtreeDesktop_state["event_log"] = self.quadtreeDesktop_state["event_log"][-QUADTREE_EVENT_LIMIT:]
        self._record_event(f"quadtree.{kind}", detail)

    def _restore_quadtree_desktop_state(self, payload: object) -> None:
        if not isinstance(payload, dict):
            self.quadtreeDesktop_state = self._create_quadtree_desktop_state()
            self._qdt_record_event("restore", "Malformed state reset to defaults.")
            return
        existing_state = self.__dict__.get("quadtreeDesktop_state", {})
        previous_visible = bool(existing_state.get("visible", False)) if isinstance(existing_state, dict) else False
        self.quadtreeDesktop_state = copy.deepcopy(payload)
        self.quadtreeDesktop_state.setdefault("desktop_id", QUADTREE_DESKTOP_ID)
        self.quadtreeDesktop_state.setdefault("version", QUADTREE_DESKTOP_VERSION)
        self.quadtreeDesktop_state.setdefault("activation_state", "dormant")
        self.quadtreeDesktop_state.setdefault("visible", previous_visible)
        self.quadtreeDesktop_state.setdefault("selected_module_ids", ["qdt.system.services"])
        self.quadtreeDesktop_state.setdefault("selected_layer_type", "input")
        self.quadtreeDesktop_state.setdefault("selected_cell_paths", [])
        self.quadtreeDesktop_state.setdefault("batch_registry", {})
        self.quadtreeDesktop_state.setdefault("rollback_registry", {})
        self.quadtreeDesktop_state.setdefault("pending_operations", {})
        self.quadtreeDesktop_state.setdefault("event_log", [])
        self.quadtreeDesktop_state.setdefault("validation_reports", {})
        self.quadtreeDesktop_state.setdefault("render_cache", {})
        self.quadtreeDesktop_state.setdefault("persistence_metadata", {"policy": "manifest"})
        self.quadtreeDesktop_state.setdefault("export_metadata", {})
        self._qdt_refresh_layer_registry()

    def _show_quadtree_desktop(self) -> None:
        if self.shell_frame is None or self.console_card is None:
            return
        shell = self.shell_frame
        shell.rowconfigure(0, weight=3)
        shell.rowconfigure(1, weight=2)
        self.console_card.grid_configure(row=1, column=0, sticky="nsew", pady=(10, 0))
        if self.quadtree_desktop_frame is None:
            self.quadtree_desktop_frame = self._build_quadtree_desktop_surface(shell)
        self.quadtree_desktop_frame.grid(row=0, column=0, sticky="nsew")
        self._render_quadtree_desktop()

    def _hide_quadtree_desktop(self) -> None:
        if self.quadtree_desktop_frame is not None:
            self.quadtree_desktop_frame.grid_remove()
        if self.shell_frame is not None and self.console_card is not None:
            self.shell_frame.rowconfigure(0, weight=1)
            self.shell_frame.rowconfigure(1, weight=0)
            self.console_card.grid_configure(row=0, column=0, sticky="nsew", pady=0)

    def _build_quadtree_desktop_surface(self, parent: tk.Widget) -> tk.Frame:
        frame = self.card(parent, padx=10, pady=10)
        frame.rowconfigure(1, weight=1)
        frame.columnconfigure(0, weight=3)
        frame.columnconfigure(1, weight=2)
        header = tk.Frame(frame, bg=Theme.panel)
        header.grid(row=0, column=0, columnspan=2, sticky="ew", pady=(0, 8))
        header.columnconfigure(1, weight=1)
        tk.Label(header, text="QUADTREE DESKTOP", bg=Theme.panel, fg=Theme.cyan, font=self.font_micro).grid(row=0, column=0, sticky="w")
        tk.Label(
            header,
            text="graphical actions stage/preview only; blue CLI remains authoritative",
            bg=Theme.panel,
            fg=Theme.muted,
            font=self.font_small,
        ).grid(row=0, column=1, sticky="w", padx=(12, 0))
        tk.Button(
            header,
            text="HIDE",
            command=lambda: self.stage_command("quadtree.desktop.hide"),
            bg=Theme.card_2,
            fg=Theme.text,
            activebackground=Theme.blue_2,
            activeforeground=Theme.blue_text,
            relief="flat",
            bd=0,
            padx=8,
            pady=4,
            font=self.font_small,
        ).grid(row=0, column=2, sticky="e")

        left = tk.Frame(frame, bg=Theme.panel)
        left.grid(row=1, column=0, sticky="nsew", padx=(0, 8))
        left.rowconfigure(1, weight=1)
        left.columnconfigure(0, weight=1)
        layer_bar = tk.Frame(left, bg=Theme.panel)
        layer_bar.grid(row=0, column=0, sticky="ew", pady=(0, 8))
        self.quadtree_layer_var = tk.StringVar(value=str(self.quadtreeDesktop_state.get("selected_layer_type", "input")))
        for index, layer_type in enumerate(QUADTREE_LAYER_TYPES):
            tk.Button(
                layer_bar,
                text=layer_type.upper(),
                command=lambda value=layer_type: self._qdt_stage_layer_switch(value),
                bg=Theme.card_2,
                fg=Theme.text,
                activebackground=Theme.blue_2,
                activeforeground=Theme.blue_text,
                relief="flat",
                bd=0,
                padx=8,
                pady=5,
                font=self.font_small,
            ).grid(row=0, column=index, sticky="ew", padx=(0 if index == 0 else 6, 0))
            layer_bar.columnconfigure(index, weight=1)
        self.quadtree_canvas = tk.Canvas(left, bg="#000000", highlightthickness=1, highlightbackground=Theme.border)
        self.quadtree_canvas.grid(row=1, column=0, sticky="nsew")
        self.quadtree_canvas.bind("<Button-1>", self._qdt_canvas_click)
        self.quadtree_canvas.bind("<Button-3>", self._qdt_canvas_right_click)

        right = tk.Frame(frame, bg=Theme.panel)
        right.grid(row=1, column=1, sticky="nsew")
        right.rowconfigure(1, weight=1)
        right.rowconfigure(3, weight=1)
        right.columnconfigure(0, weight=1)
        tk.Label(right, text="SELECTED CELL INSPECTOR", bg=Theme.panel, fg=Theme.muted, font=self.font_micro).grid(row=0, column=0, sticky="w")
        self.quadtree_inspector_text = ScrolledText(right, height=7, bg="#000000", fg=Theme.text, insertbackground=Theme.text, relief="flat", font=("Consolas", 9), wrap="word")
        self.quadtree_inspector_text.grid(row=1, column=0, sticky="nsew", pady=(4, 8))
        tk.Label(right, text="MODULE + BATCH INSPECTORS", bg=Theme.panel, fg=Theme.muted, font=self.font_micro).grid(row=2, column=0, sticky="w")
        inspectors = tk.Frame(right, bg=Theme.panel)
        inspectors.grid(row=3, column=0, sticky="nsew", pady=(4, 8))
        inspectors.columnconfigure(0, weight=1)
        inspectors.columnconfigure(1, weight=1)
        inspectors.rowconfigure(0, weight=1)
        self.quadtree_module_text = ScrolledText(inspectors, height=6, bg="#000000", fg=Theme.muted, insertbackground=Theme.text, relief="flat", font=("Consolas", 9), wrap="word")
        self.quadtree_module_text.grid(row=0, column=0, sticky="nsew", padx=(0, 4))
        self.quadtree_batch_text = ScrolledText(inspectors, height=6, bg="#000000", fg=Theme.muted, insertbackground=Theme.text, relief="flat", font=("Consolas", 9), wrap="word")
        self.quadtree_batch_text.grid(row=0, column=1, sticky="nsew", padx=(4, 0))
        tk.Label(right, text="COMMAND PREVIEW", bg=Theme.panel, fg=Theme.muted, font=self.font_micro).grid(row=4, column=0, sticky="w")
        self.quadtree_preview_text = ScrolledText(right, height=5, bg="#000000", fg=Theme.cyan, insertbackground=Theme.text, relief="flat", font=("Consolas", 9), wrap="word")
        self.quadtree_preview_text.grid(row=5, column=0, sticky="ew", pady=(4, 0))
        for widget in (self.quadtree_inspector_text, self.quadtree_module_text, self.quadtree_batch_text, self.quadtree_preview_text):
            widget.configure(state="disabled")
        return frame

    def _qdt_stage_layer_switch(self, layer_type: str) -> None:
        self.quadtreeDesktop_state["selected_layer_type"] = layer_type
        self._qdt_record_event("layer.switch", layer_type)
        self.stage_command(f'quadtree.desktop.layer.get {{"module_id":"{self._qdt_current_module_id()}","layer_type":"{layer_type}"}}')
        self._render_quadtree_desktop()

    def _qdt_current_module_id(self) -> str:
        selected = self.quadtreeDesktop_state.get("selected_module_ids", ["qdt.system.services"])
        if isinstance(selected, list) and selected:
            return str(selected[0])
        return "qdt.system.services"

    def _render_quadtree_desktop(self) -> None:
        if self.quadtree_canvas is None or self.quadtree_desktop_frame is None or not bool(self.quadtreeDesktop_state.get("visible", False)):
            return
        module_id = self._qdt_current_module_id()
        layer_type = str(self.quadtreeDesktop_state.get("selected_layer_type", "input"))
        layer = self._qdt_get_layer(module_id, layer_type)
        if not layer:
            return
        self.quadtree_canvas.delete("all")
        width = max(int(self.quadtree_canvas.winfo_width() or 560), 200)
        height = max(int(self.quadtree_canvas.winfo_height() or 360), 200)
        root = layer.get("cells", {}).get("root", {})
        scale_x = width / max(int(root.get("width", 560)), 1)
        scale_y = height / max(int(root.get("height", 560)), 1)
        cells = sorted(layer.get("cells", {}).values(), key=lambda cell: (int(cell.get("depth", 0)), int(cell.get("z_order", 0))))
        for cell in cells:
            self._qdt_draw_cell(cell, scale_x, scale_y)
        self._qdt_refresh_inspectors()

    def _qdt_draw_cell(self, cell: dict[str, object], scale_x: float, scale_y: float) -> None:
        if self.quadtree_canvas is None or not cell.get("visible", True):
            return
        x0 = int(cell.get("x", 0) * scale_x)
        y0 = int(cell.get("y", 0) * scale_y)
        x1 = int((cell.get("x", 0) + cell.get("width", 0)) * scale_x)
        y1 = int((cell.get("y", 0) + cell.get("height", 0)) * scale_y)
        fill = str(cell.get("style_binding", {}).get("fill", ""))
        outline = str(cell.get("style_binding", {}).get("outline", ""))
        status = str(cell.get("status", "dormant"))
        status_bg, status_fg, status_border = STATUS_COLORS.get(status, (Theme.card_2, Theme.muted, Theme.border))
        fill = fill or status_bg
        outline = outline or (Theme.cyan if cell.get("selected") else status_border)
        width = 3 if cell.get("selected") else 1
        address = str(cell.get("address", ""))
        self.quadtree_canvas.create_rectangle(x0, y0, x1, y1, fill=fill, outline=outline, width=width, tags=("qdt-cell", address))
        text = str(cell.get("content_binding", {}).get("text", cell.get("path", "")))
        if int(cell.get("width", 0)) * scale_x > 42 and int(cell.get("height", 0)) * scale_y > 26:
            self.quadtree_canvas.create_text((x0 + x1) // 2, (y0 + y1) // 2, text=text[:28], fill=status_fg, font=("Consolas", 8), tags=("qdt-cell", address))

    def _qdt_refresh_inspectors(self) -> None:
        selected_addresses = self.quadtreeDesktop_state.get("selected_cell_paths", [])
        selected_cell = self._qdt_get_cell_by_ref(str(selected_addresses[0])) if selected_addresses else None
        module = self._qdt_get_module(self._qdt_current_module_id())
        batch_id = str(self.quadtreeDesktop_state.get("selected_batch_id", ""))
        batch = self.quadtreeDesktop_state.get("batch_registry", {}).get(batch_id, {}) if batch_id else {}
        preview = {
            "staged_command": f'quadtree.desktop.cell.get {{"address":"{selected_addresses[0]}"}}' if selected_addresses else "quadtree.desktop.module.list",
            "selected_layer": self.quadtreeDesktop_state.get("selected_layer_type", "input"),
            "note": "Canvas gestures record input events and stage blue CLI commands only.",
        }
        self._qdt_write_text_widget(self.quadtree_inspector_text, json.dumps(selected_cell or {"selected": None}, indent=2))
        self._qdt_write_text_widget(self.quadtree_module_text, json.dumps({"module_id": self._qdt_current_module_id(), "label": module.get("label", "") if module else ""}, indent=2))
        self._qdt_write_text_widget(self.quadtree_batch_text, json.dumps(batch or {"selected_batch_id": ""}, indent=2))
        self._qdt_write_text_widget(self.quadtree_preview_text, json.dumps(preview, indent=2))

    def _qdt_write_text_widget(self, widget: ScrolledText | None, text: str) -> None:
        if widget is None:
            return
        widget.configure(state="normal")
        widget.delete("1.0", "end")
        widget.insert("1.0", text)
        widget.configure(state="disabled")

    def _qdt_canvas_click(self, event: tk.Event) -> None:
        address = self._qdt_canvas_address_at(event)
        if not address:
            return
        self._qdt_set_selection([address])
        self._qdt_record_event("input.click", address)
        self.stage_command(f'quadtree.desktop.cell.get {{"address":"{address}"}}')
        self._render_quadtree_desktop()

    def _qdt_canvas_right_click(self, event: tk.Event) -> None:
        address = self._qdt_canvas_address_at(event)
        if not address:
            return
        self._qdt_set_selection([address])
        self._qdt_record_event("input.right_click", address)
        command = f'quadtree.desktop.preview {{"target":"{address}","context_actions":["inspect","subdivide","bind","patch","batch_select"]}}'
        self.stage_command(command)
        self._render_quadtree_desktop()

    def _qdt_canvas_address_at(self, event: tk.Event) -> str:
        if self.quadtree_canvas is None:
            return ""
        item = self.quadtree_canvas.find_withtag("current")
        if not item:
            item = self.quadtree_canvas.find_closest(event.x, event.y)
        if not item:
            return ""
        tags = self.quadtree_canvas.gettags(item[0])
        for tag in tags:
            if tag.startswith("qdt://"):
                return tag
        return ""

    def _create_configure4_state(self) -> dict[str, object]:
        return {
            "service_id": CONFIGURE4_SERVICE_ID,
            "label": CONFIGURE4_LABEL,
            "mode": "manual",
            "settings": {
                "default_sink": "memory",
                "max_variants": CONFIGURE4_MAX_VARIANTS,
                "max_drafts": CONFIGURE4_MAX_DRAFTS,
                "default_persistence_policy": "manifest",
                "default_file_access_policy": "none",
                "default_side_effect_policy": "memory_only",
                "registration_policy": "staged_by_default",
                "allow_custom_fields": True,
            },
            "drafts": {},
            "generated_service_drafts": {},
            "current_draft_id": "",
            "validation_reports": {},
            "registration_history": [],
            "diagnostics": [],
            "registered_runtime_services": {},
            "registered_runtime_commands": {},
            "registered_runtime_routes": {},
            "next_draft_index": 1,
        }

    def dispatch_configure4_command(self, cmd: str, args: list[str], raw_after_cmd: str) -> tuple[str, str]:
        if not self.require_service(CONFIGURE4_SERVICE_ID):
            return f"{CONFIGURE4_SERVICE_ID} is dormant. Activate it first: service.activate {CONFIGURE4_SERVICE_ID}", "err"
        action = cmd.split(".", 1)[1]
        if action == "help":
            return self.command_configure4_help(), "ok"
        if action == "status":
            return self.command_configure4_status(), "json"
        if action == "mode":
            return self.command_configure4_mode(args[0] if args else ""), "json"
        if action == "new":
            return self.command_configure4_new(raw_after_cmd)
        if action == "set":
            if not args:
                return "Usage: configure4.set <path> <value>", "err"
            value_raw = raw_after_cmd.split(maxsplit=1)[1] if len(raw_after_cmd.split(maxsplit=1)) > 1 else ""
            return self.command_configure4_set(args[0], value_raw)
        if action == "get":
            return self.command_configure4_get(args[0] if args else ""), "json"
        if action == "validate":
            return self.command_configure4_validate(args[0] if args else "all"), "json"
        if action == "preview":
            return self.command_configure4_preview(args[0] if args else ""), "json"
        if action == "register":
            return self.command_configure4_register(args[0] if args else ""), "json"
        if action == "export":
            if not args:
                return "Usage: configure4.export <path>", "err"
            return self.command_configure4_export(args[0])
        if action == "import":
            if not args:
                return "Usage: configure4.import <path>", "err"
            return self.command_configure4_import(args[0])
        if action == "sample":
            return self.command_configure4_sample(), "json"
        if action == "reset":
            return self.command_configure4_reset(args[0] if args else ""), "json"
        if action == "history":
            return self.command_configure4_history(), "json"
        if action == "list":
            return self.command_configure4_list(), "json"
        return f"Unknown configure4 command: {cmd}. Try configure4.help.", "err"

    def dispatch_configure4_runtime_command(self, cmd: str, raw_after_cmd: str) -> tuple[str, str] | None:
        commands = self.configure4_state.get("registered_runtime_commands", {})
        if not isinstance(commands, dict) or cmd not in commands:
            return None
        record = commands.get(cmd, {})
        if not isinstance(record, dict):
            return None
        service_id = str(record.get("service_id", ""))
        if not self.require_service(service_id):
            return f"Service {service_id} is dormant. Activate it first: service.activate {service_id}", "err"
        payload = self.parse_payload(raw_after_cmd)
        result = self.generic_configure4_service_handler(service_id, cmd, payload)
        return self.format_json(result), "json"

    def command_configure4_help(self) -> str:
        return self._configure4_help_text()

    def command_configure4_status(self) -> str:
        return self.format_json(self._configure4_status_payload())

    def command_configure4_mode(self, mode_text: str) -> str:
        if not mode_text:
            return self.format_json({"mode": self.configure4_state.get("mode", "manual"), "allowed": list(CONFIGURE4_ALLOWED_MODES)})
        mode = configure4_normalize_mode(mode_text)
        if mode not in CONFIGURE4_ALLOWED_MODES:
            diag = self._configure4_append_diagnostic(
                "error",
                "mode",
                f"Unsupported configure4 mode: {mode_text}",
                "Use manual, semi_auto, or auto.",
            )
            return self.format_json({"accepted": False, "diagnostic": diag, "allowed": list(CONFIGURE4_ALLOWED_MODES)})
        self.configure4_state["mode"] = mode
        self._record_event("configure4.mode", mode)
        return self.format_json({"accepted": True, "mode": mode})

    def command_configure4_new(self, payload_raw: str) -> tuple[str, str]:
        payload, error = self._configure4_parse_json_object(payload_raw)
        if error:
            diag = self._configure4_append_diagnostic("error", "configure4.new", error, "Submit a JSON object.")
            return self.format_json({"accepted": False, "diagnostics": [diag]}), "err"
        mode = configure4_normalize_mode(payload.get("mode", self.configure4_state.get("mode", "manual")))
        if mode not in CONFIGURE4_ALLOWED_MODES:
            diag = self._configure4_append_diagnostic("error", "configure4.new", f"Unsupported mode: {mode}", "Use configure4.mode first or include a valid mode.")
            return self.format_json({"accepted": False, "diagnostics": [diag]}), "err"
        if mode == "manual":
            spec, missing = self._configure4_manual_spec(payload)
            if missing:
                diag = self._configure4_append_diagnostic(
                    "error",
                    "configure4.new",
                    "Manual mode rejected an incomplete service specification.",
                    "Provide every required field or switch to semi_auto.",
                )
                return self.format_json({"accepted": False, "missing_required_fields": missing, "diagnostics": [diag]}), "err"
        elif mode == "semi_auto":
            spec = self._configure4_semi_auto_spec(payload)
        else:
            spec = self._configure4_auto_spec(payload)
            if spec is None:
                diag = self._configure4_append_diagnostic(
                    "error",
                    "configure4.new",
                    "Automatic mode requires an intent string.",
                    "Example: configure4.new {\"intent\":\"create a diagnostics summarizer service\"}",
                )
                return self.format_json({"accepted": False, "diagnostics": [diag]}), "err"

        max_drafts = int(self.configure4_state.get("settings", {}).get("max_drafts", CONFIGURE4_MAX_DRAFTS))
        drafts = self.configure4_state.setdefault("drafts", {})
        if isinstance(drafts, dict) and len(drafts) >= max_drafts:
            diag = self._configure4_append_diagnostic("error", "configure4.new", "Draft limit reached.", "Reset old drafts or raise settings.max_drafts.")
            return self.format_json({"accepted": False, "diagnostics": [diag]}), "err"
        if spec.draft_id in drafts:
            diag = self._configure4_append_diagnostic("error", spec.draft_id, "Draft id already exists.", "Use configure4.reset for the old draft or choose a new draft_id.")
            return self.format_json({"accepted": False, "diagnostics": [diag]}), "err"

        drafts[spec.draft_id] = spec.to_dict()
        self.configure4_state["current_draft_id"] = spec.draft_id
        if mode == "auto":
            self.configure4_state.setdefault("generated_service_drafts", {})[spec.draft_id] = spec.to_dict()
        report = self._configure4_validate_spec(spec, append=False)
        self.configure4_state.setdefault("validation_reports", {})[spec.draft_id] = report
        self._record_event("configure4.new", spec.draft_id)
        tag = "json" if report["accepted"] else "warn"
        return self.format_json({"accepted": True, "draft_id": spec.draft_id, "mode": mode, "spec_hash": report["spec_hash"], "validation": report}), tag

    def command_configure4_set(self, path: str, value_raw: str) -> tuple[str, str]:
        if not path or not value_raw:
            return "Usage: configure4.set <path> <value>", "err"
        value = self._configure4_parse_value(value_raw)
        target, parts, error = self._configure4_resolve_state_path(path, for_write=True)
        if error:
            diag = self._configure4_append_diagnostic("error", path, error)
            return self.format_json({"accepted": False, "diagnostics": [diag]}), "err"
        self._configure4_set_path(target, parts, value)
        draft_id = self._configure4_path_draft_id(path)
        if draft_id:
            draft = self.configure4_state["drafts"].get(draft_id)
            if isinstance(draft, dict):
                draft.setdefault("metadata", {})["updated_at"] = _dt.datetime.now().isoformat(timespec="seconds")
                inferred = draft.get("inferred_fields", [])
                relative = ".".join(parts)
                if isinstance(inferred, list) and relative in inferred:
                    inferred.remove(relative)
        diag = self._configure4_append_diagnostic("accepted", path, "Field updated through blue CLI command.")
        self._record_event("configure4.set", path)
        return self.format_json({"accepted": True, "path": path, "value": value, "diagnostic": diag}), "json"

    def command_configure4_get(self, path: str) -> str:
        if not path:
            return self.format_json(copy.deepcopy(self.configure4_state))
        target, parts, error = self._configure4_resolve_state_path(path, for_write=False)
        if error:
            return self.format_json({"error": error, "path": path})
        found, value = self._configure4_get_path(target, parts)
        if not found:
            return self.format_json({"error": "path not found", "path": path})
        return self.format_json(value)

    def command_configure4_validate(self, target: str) -> str:
        return self.format_json(self._configure4_validate_target(target or "all", append=True))

    def command_configure4_preview(self, target: str) -> str:
        draft_id = self._configure4_default_draft_id(target)
        spec = self._configure4_get_spec(draft_id)
        if spec is None:
            return self.format_json({"accepted": False, "error": f"Unknown draft: {draft_id or '<none>'}"})
        preview = self._configure4_build_preview(spec, append_validation=True)
        self._record_event("configure4.preview", spec.draft_id)
        return self.format_json(preview)

    def command_configure4_register(self, target: str) -> str:
        draft_id = self._configure4_default_draft_id(target)
        result = self._configure4_register_draft(draft_id)
        return self.format_json(result)

    def command_configure4_export(self, path: str) -> tuple[str, str]:
        if not self.require_service("kernel.fs"):
            return "kernel.fs is dormant. Activate it first: service.activate kernel.fs", "err"
        output_path = Path(path).expanduser()
        if output_path.parent and str(output_path.parent) not in {"", "."}:
            output_path.parent.mkdir(parents=True, exist_ok=True)
        payload = {"configure4_state": copy.deepcopy(self.configure4_state)}
        with open(output_path, "w", encoding="utf-8") as handle:
            json.dump(payload, handle, indent=2)
        self.configure4_state.setdefault("settings", {})["last_export_path"] = str(output_path)
        self._configure4_append_diagnostic("accepted", "configure4.export", f"Exported configure4 state to {output_path}.")
        self._record_event("configure4.export", str(output_path))
        return f"Exported configure4 state to {output_path}", "ok"

    def command_configure4_import(self, path: str) -> tuple[str, str]:
        if not self.require_service("kernel.fs"):
            return "kernel.fs is dormant. Activate it first: service.activate kernel.fs", "err"
        input_path = Path(path).expanduser()
        with open(input_path, "r", encoding="utf-8") as handle:
            text = handle.read()
        try:
            payload = json.loads(text)
        except json.JSONDecodeError:
            payload = self._configure4_parse_key_value_config(text)
        if isinstance(payload, dict) and "configure4_state" in payload:
            self._restore_configure4_state(payload["configure4_state"])
            imported = {"kind": "state", "drafts": len(self.configure4_state.get("drafts", {}))}
        elif isinstance(payload, dict):
            spec = self._configure4_semi_auto_spec(payload)
            self.configure4_state.setdefault("drafts", {})[spec.draft_id] = spec.to_dict()
            self.configure4_state["current_draft_id"] = spec.draft_id
            imported = {"kind": "draft", "draft_id": spec.draft_id}
        else:
            diag = self._configure4_append_diagnostic("error", "configure4.import", "Import payload is not a JSON object or key-value fabric config.")
            return self.format_json({"accepted": False, "diagnostics": [diag]}), "err"
        self.configure4_state.setdefault("settings", {})["last_import_path"] = str(input_path)
        self._configure4_append_diagnostic("accepted", "configure4.import", f"Imported configure4 payload from {input_path}.")
        self._record_event("configure4.import", str(input_path))
        return self.format_json({"accepted": True, "source": str(input_path), "imported": imported}), "json"

    def command_configure4_sample(self) -> str:
        return self.format_json(self._configure4_sample_payload())

    def command_configure4_reset(self, target: str) -> str:
        if target in {"", "all", "*"}:
            self._remove_configure4_runtime_registrations()
            self.configure4_state = self._create_configure4_state()
            self._record_event("configure4.reset", "all")
            return self.format_json({"accepted": True, "reset": "all"})
        drafts = self.configure4_state.setdefault("drafts", {})
        if not isinstance(drafts, dict) or target not in drafts:
            return self.format_json({"accepted": False, "error": f"Unknown draft: {target}"})
        self._remove_configure4_runtime_registrations(draft_id=target)
        drafts.pop(target, None)
        self.configure4_state.setdefault("generated_service_drafts", {}).pop(target, None)
        self.configure4_state.setdefault("validation_reports", {}).pop(target, None)
        if self.configure4_state.get("current_draft_id") == target:
            self.configure4_state["current_draft_id"] = next(iter(drafts), "")
        self._record_event("configure4.reset", target)
        return self.format_json({"accepted": True, "reset": target})

    def command_configure4_history(self) -> str:
        return self.format_json(
            {
                "diagnostics": copy.deepcopy(self.configure4_state.get("diagnostics", [])),
                "validation_reports": copy.deepcopy(self.configure4_state.get("validation_reports", {})),
                "registration_history": copy.deepcopy(self.configure4_state.get("registration_history", [])),
            }
        )

    def command_configure4_list(self) -> str:
        drafts = self.configure4_state.get("drafts", {})
        rows = []
        if isinstance(drafts, dict):
            for draft_id, payload in drafts.items():
                spec = Configure4Spec.from_dict(payload)
                rows.append(
                    {
                        "draft_id": draft_id,
                        "mode": spec.mode,
                        "service_id": spec.identity.get("service_id", ""),
                        "label": spec.identity.get("label", ""),
                        "hash": self._configure4_spec_hash(spec),
                        "current": draft_id == self.configure4_state.get("current_draft_id"),
                    }
                )
        return self.format_json(rows)

    def _configure4_help_text(self) -> str:
        return f"""
{CONFIGURE4_LABEL} ({CONFIGURE4_SERVICE_ID})

Purpose
  configure_4 is the internal authoring fabric for start.py services. It stages a
  service specification, validates it, previews the exact integration plan, and
  registers only runtime-safe service records. It never opens a visual editor and
  never executes generated code with eval or exec. All use happens through blue
  CLI commands and all results are printed to the REPL Console.

Activation
  service.activate {CONFIGURE4_SERVICE_ID}
  configure4.status
  configure4.sample

Core workflow
  1. Choose a mode:
       configure4.mode manual
       configure4.mode semi_auto
       configure4.mode auto
  2. Stage a draft:
       configure4.new {{json}}
  3. Inspect and patch fields:
       configure4.get
       configure4.get identity.service_id
       configure4.set execution_behavior.can_register_immediately true
  4. Validate:
       configure4.validate all
  5. Preview the exact start.py integration plan:
       configure4.preview <draft-id>
  6. Register only when the preview says the draft is runtime-safe:
       configure4.register <draft-id>
  7. Persist:
       service.activate kernel.fs
       service.activate codex.manifest
       manifest.save codex_project/configure4_manifest.json

Operating modes
  manual
    Every required field must be supplied. Missing service identity, command
    surface, API surface, execution behavior, validation, persistence, examples,
    object graph, and metadata fields are rejected with diagnostics. Use this
    when you want full deterministic control.

  semi_auto
    A partial service specification is accepted and safe defaults are inferred.
    Inferred fields are recorded in inferred_fields and can be reviewed or patched
    with configure4.set before validation or registration.

  auto
    A high-level intent string is expanded into a complete staged service spec.
    Automatic mode never silently registers a generated service. The draft must
    be reviewed with configure4.preview, patched if needed, validated, and then
    explicitly registered.

Service specification map
  identity.service_id
    Stable lowercase dot-delimited id, such as codex.text.generator.
  identity.label
    User-facing label.
  identity.aliases
    Optional aliases for service.activate resolution.
  command_surface.verbs
    Blue CLI command tokens to expose after runtime registration.
  command_surface.handler_name
    Use generic_configure4_service_handler for runtime-safe registration.
  api_surface.routes
    Optional routes. Each must start with /. Routes are callable only through
    api.gateway and only when the generated service is active.
  api_surface.handler_name
    Use generic_configure4_api_handler for runtime-safe registration.
  execution_behavior.activation_requirements
    Must include blue.cli.engine. Existing activation checks are not weakened.
  execution_behavior.can_register_immediately
    false means the draft can be staged and previewed but registration returns a
    patch plan. true allows runtime registration when all other gates pass.
  persistence_behavior.policy
    memory_only, manifest, manifest_and_export, or disabled.
  generated_code_plan.flow
    The deterministic generation flow used by preview output.
  custom_fields
    Free-form developer metadata. Validation preserves custom_fields and does not
    reject unknown keys.

Generation flows
  literal
    Writes input_value once with optional prefix, suffix, and separator.

  template
    Replaces {{name}} placeholders in template using generated_code_plan.flow.values.
    This is the best flow for reusable text, file headers, CLI messages, source
    snippets, and pointer strings.

  cartesian
    Generates bounded combinations from input_value with input_width. This is
    useful for enumerating short symbolic names, test ids, route suffixes, enum
    variants, or pointer-like reference keys. The product is guarded by
    settings.max_variants.

  repeat
    Writes input_value input_width times, bounded by settings.max_variants.

  reverse
    Writes input_value reversed once.

  service_scaffold
    Generates a complete start.py service integration plan: service registry
    entry, command branches, API routes, handler skeletons, persistence notes,
    validation diagnostics, and a stable spec hash.

Text generation
  configure_4 can stage text generators as services. Use literal for one-off
  content, template for structured text, repeat for fixed lists, reverse for
  transforms, and cartesian for bounded variants. Text output defaults to memory
  or console preview. Filesystem writes require explicit intent through kernel.fs
  style commands or an export path.

  Minimal text generator:
    configure4.mode semi_auto
    configure4.new {{"service_id":"codex.text.banner","label":"CodexTextBanner","description":"Generates release banner text.","verbs":["text.banner.run"],"flow_type":"template","template":"Project {{name}} targets {{language}}.","generated_code_plan":{{"flow":{{"type":"template","template":"Project {{name}} targets {{language}}.","values":{{"name":"demo","language":"Python"}},"output_target":"memory","separator":"\\n"}}}}}}
    configure4.preview <draft-id>

Pointer generation
  A pointer in configure_4 documentation means a stable textual reference: a file
  path, API route, service id, symbol id, object graph id, test id, or source
  location string. It is not a raw memory pointer and configure_4 does not
  dereference memory. Use pointer generation to create consistent names that can
  be consumed by another service or by a future code emitter.

  Pointer-style template:
    configure4.new {{"mode":"semi_auto","service_id":"codex.pointer.index","label":"CodexPointerIndex","description":"Generates stable symbol pointers.","verbs":["pointer.index.run"],"flow_type":"template","generated_code_plan":{{"flow":{{"type":"template","template":"{{root}}/{{module}}.py::{{symbol}}","values":{{"root":"src","module":"diagnostics","symbol":"summarize"}},"output_target":"memory","separator":"\\n"}}}}}}

  Bounded pointer variants:
    configure4.new {{"mode":"semi_auto","service_id":"codex.pointer.variants","label":"CodexPointerVariants","description":"Generates bounded pointer suffixes.","verbs":["pointer.variants.run"],"flow_type":"cartesian","input_value":"ab","input_width":3,"prefix":"route.","suffix":".handler","can_register_immediately":false}}

Creating codebases in any programming language
  configure_4 does not assume one language. Put language-specific decisions in
  custom_fields.codebase and keep generated_code_plan.flow.type as
  service_scaffold or template. This creates a deterministic service draft and
  previewable plan that can describe files, directories, package metadata, build
  commands, tests, linters, docs, and release steps for any language.

  Recommended codebase fields:
    custom_fields.codebase.language
    custom_fields.codebase.project_type
    custom_fields.codebase.package_manager
    custom_fields.codebase.files
    custom_fields.codebase.entrypoints
    custom_fields.codebase.build_commands
    custom_fields.codebase.test_commands
    custom_fields.codebase.quality_gates
    custom_fields.codebase.output_policy

  Language-agnostic codebase draft:
    configure4.mode auto
    configure4.new {{"mode":"auto","intent":"create a Rust command line codebase that validates JSON manifests","custom_fields":{{"codebase":{{"language":"Rust","project_type":"cli","package_manager":"cargo","files":[{{"path":"Cargo.toml","role":"package manifest"}},{{"path":"src/main.rs","role":"entrypoint"}},{{"path":"tests/manifest_validation.rs","role":"integration tests"}}],"build_commands":["cargo build"],"test_commands":["cargo test"],"quality_gates":["format","lint","tests"],"output_policy":"preview first, write files only after explicit filesystem command"}}}}}}
    configure4.validate all
    configure4.preview <draft-id>

  For another language, change only custom_fields.codebase.language, files,
  commands, and package metadata. Examples: Python with pyproject.toml, Go with
  go.mod, TypeScript with package.json, C with Makefile, Java with pom.xml or
  build.gradle, C# with .csproj, Zig with build.zig, or any custom toolchain.

Registration rules
  Runtime registration succeeds only when validation passes, the draft allows
  immediate registration, commands/routes do not collide, activation dependencies
  are valid, file behavior is safe, and the handlers can be represented by the
  existing generic handlers. If a draft names Python methods that do not already
  exist, configure4.register refuses to fake success and returns a patch plan.

API routes
  service.activate api.gateway
  service.activate {CONFIGURE4_SERVICE_ID}
  api.call /configure4/status
  api.call /configure4/specs
  api.call /configure4/validate {{"target":"all"}}
  api.call /configure4/preview {{"draft_id":"draft_001"}}
  api.call /configure4/register {{"draft_id":"draft_001"}}

Persistence and history
  configure4_state is included in current_project_payload, manifest.save,
  manifest.load, and version.snapshot. The saved state includes staged specs,
  generated service drafts, validation reports, registration history,
  diagnostics, settings, and runtime registration metadata.

Safety notes
  Only the REPL Console and blue CLI engine are visible.
  configure_4 does not create panels, dialogs, pop-ups, palettes, toolbars,
  secondary windows, or graphical editors.
  Defaults use memory/console sinks. File writes require explicit commands.
  eval and exec are never used.
  blue.cli.engine cannot be deactivated or bypassed.
""".strip()

    def _configure4_parse_json_object(self, payload_raw: str) -> tuple[dict[str, object], str]:
        raw = payload_raw.strip()
        if not raw:
            return {}, "Missing JSON object."
        try:
            payload = json.loads(raw)
        except json.JSONDecodeError as exc:
            return {}, f"Malformed JSON: {exc}"
        if not isinstance(payload, dict):
            return {}, "configure4.new requires a JSON object."
        return payload, ""

    def _configure4_parse_value(self, value_raw: str) -> object:
        raw = value_raw.strip()
        if not raw:
            return ""
        if raw[0] in "[{\"" or raw in {"true", "false", "null"} or re.fullmatch(r"-?\d+(\.\d+)?", raw):
            try:
                return json.loads(raw)
            except json.JSONDecodeError:
                return raw
        return raw

    def _configure4_next_draft_id(self) -> str:
        index = int(self.configure4_state.get("next_draft_index", 1))
        while True:
            draft_id = f"draft_{index:03d}"
            index += 1
            if draft_id not in self.configure4_state.get("drafts", {}):
                self.configure4_state["next_draft_index"] = index
                return draft_id

    def _configure4_manual_spec(self, payload: dict[str, object]) -> tuple[Configure4Spec, list[str]]:
        expanded = self._configure4_expand_aliases(payload)
        missing = [path for path in CONFIGURE4_MANUAL_REQUIRED_PATHS if not self._configure4_path_exists(expanded, path)]
        draft_id = str(expanded.get("draft_id") or payload.get("draft_id") or self._configure4_next_draft_id())
        expanded["draft_id"] = draft_id
        expanded["mode"] = "manual"
        expanded.setdefault("diagnostics", [])
        expanded.setdefault("inferred_fields", [])
        expanded.setdefault("custom_fields", {})
        metadata = expanded.setdefault("metadata", {})
        if isinstance(metadata, dict):
            metadata.setdefault("created_at", _dt.datetime.now().isoformat(timespec="seconds"))
            metadata.setdefault("updated_at", metadata["created_at"])
        return Configure4Spec.from_dict(expanded), missing

    def _configure4_semi_auto_spec(self, payload: dict[str, object]) -> Configure4Spec:
        expanded = self._configure4_expand_aliases(payload)
        draft_id = str(expanded.get("draft_id") or payload.get("draft_id") or self._configure4_next_draft_id())
        defaults = self._configure4_default_spec_payload(expanded, "semi_auto", intent=str(expanded.get("intent", "")), draft_id=draft_id)
        provided_paths = set(self._configure4_flat_paths(expanded))
        merged = self._configure4_deep_merge(defaults, expanded)
        merged["draft_id"] = draft_id
        merged["mode"] = "semi_auto"
        merged["inferred_fields"] = sorted(path for path in self._configure4_flat_paths(defaults) if path not in provided_paths and not path.startswith("metadata."))
        return Configure4Spec.from_dict(merged)

    def _configure4_auto_spec(self, payload: dict[str, object]) -> Configure4Spec | None:
        intent = str(payload.get("intent", "")).strip()
        if not intent:
            return None
        expanded = self._configure4_expand_aliases(payload)
        draft_id = str(expanded.get("draft_id") or payload.get("draft_id") or self._configure4_next_draft_id())
        defaults = self._configure4_default_spec_payload(expanded, "auto", intent=intent, draft_id=draft_id)
        generated_service_id = configure4_dotted_id(intent)
        service_slug = generated_service_id.replace(".", "-")
        command_prefix = ".".join(generated_service_id.split(".")[-2:])
        auto_payload = {
            "identity": {
                "service_id": generated_service_id,
                "label": configure4_pascal_label(intent),
                "aliases": [service_slug, generated_service_id.split(".")[-1]],
                "layer": "codex-authoring",
                "state": "dormant",
                "description": f"Synthesized from intent: {intent}",
                "version": "0.1.0",
            },
            "command_surface": {
                "verbs": [f"{command_prefix}.status", f"{command_prefix}.run", f"{command_prefix}.preview"],
                "handler_name": CONFIGURE4_GENERIC_HANDLER_NAME,
                "help_text": f"Generated service for intent: {intent}",
                "default_payload": {"intent": intent},
            },
            "api_surface": {
                "routes": [f"/configure4/generated/{service_slug}"],
                "handler_name": CONFIGURE4_GENERIC_API_HANDLER_NAME,
                "payload_schema": {"type": "object", "additionalProperties": True},
                "output_schema": {"type": "object", "required": ["service", "payload"]},
            },
            "execution_behavior": {
                "activation_requirements": ["blue.cli.engine"],
                "dependency_requirements": [],
                "allowed_modes": list(CONFIGURE4_ALLOWED_MODES),
                "timeout_policy": {"seconds": 30},
                "file_access_policy": "none",
                "side_effect_policy": "memory_only",
                "can_register_immediately": False,
            },
            "generated_code_plan": {
                "handler_name": CONFIGURE4_GENERIC_HANDLER_NAME,
                "api_handler_name": CONFIGURE4_GENERIC_API_HANDLER_NAME,
                "runtime_strategy": "generic_service_record",
                "flow": {
                    "type": "service_scaffold",
                    "input_value": intent,
                    "input_width": 1,
                    "output_target": "memory",
                    "prefix": "",
                    "suffix": "",
                    "separator": "\n",
                },
            },
            "metadata": {"intent": intent, "owner": "configure4.auto", "review_required": True},
        }
        provided_paths = set(self._configure4_flat_paths(expanded))
        merged = self._configure4_deep_merge(defaults, auto_payload)
        merged = self._configure4_deep_merge(merged, expanded)
        merged["draft_id"] = draft_id
        merged["mode"] = "auto"
        merged["inferred_fields"] = sorted(path for path in self._configure4_flat_paths(merged) if path not in provided_paths and path not in {"draft_id", "mode"})
        return Configure4Spec.from_dict(merged)

    def _configure4_default_spec_payload(self, seed: dict[str, object], mode: str, *, intent: str, draft_id: str) -> dict[str, object]:
        identity = seed.get("identity", {}) if isinstance(seed.get("identity", {}), dict) else {}
        service_id = str(identity.get("service_id", "") or configure4_dotted_id(intent or identity.get("label", draft_id)))
        label = str(identity.get("label", "") or configure4_pascal_label(service_id))
        short = ".".join(service_id.split(".")[-2:]) if "." in service_id else service_id
        now = _dt.datetime.now().isoformat(timespec="seconds")
        flow = {
            "type": "service_scaffold",
            "input_value": service_id,
            "input_width": 1,
            "output_target": "memory",
            "prefix": "",
            "suffix": "",
            "separator": "\n",
        }
        return {
            "draft_id": draft_id,
            "mode": mode,
            "identity": {
                "service_id": service_id,
                "label": label,
                "aliases": [service_id.split(".")[-1]],
                "layer": "codex-authoring",
                "state": "dormant",
                "description": str(identity.get("description", "Generated by configure_4.authoring.fabric.")),
                "version": "0.1.0",
            },
            "command_surface": {
                "verbs": [f"{short}.status", f"{short}.run"],
                "handler_name": CONFIGURE4_GENERIC_HANDLER_NAME,
                "help_text": f"{label} generated by configure_4.authoring.fabric.",
                "default_payload": {},
            },
            "api_surface": {
                "routes": [],
                "handler_name": CONFIGURE4_GENERIC_API_HANDLER_NAME,
                "payload_schema": {"type": "object", "additionalProperties": True},
                "output_schema": {"type": "object", "additionalProperties": True},
            },
            "execution_behavior": {
                "activation_requirements": ["blue.cli.engine"],
                "dependency_requirements": [],
                "allowed_modes": list(CONFIGURE4_ALLOWED_MODES),
                "timeout_policy": {"seconds": 30},
                "file_access_policy": "none",
                "side_effect_policy": "memory_only",
                "can_register_immediately": False,
            },
            "validation_rules": {
                "gates": [
                    "service_id",
                    "command_surface",
                    "api_surface",
                    "activation_dependencies",
                    "filesystem_policy",
                    "flow_bounds",
                    "handler_presence",
                ],
                "allow_command_collisions": False,
            },
            "persistence_behavior": {
                "policy": "manifest",
                "import_path": "",
                "export_path": "",
                "versioning_policy": "snapshot_in_manifest",
            },
            "examples": [f"service.activate {service_id}", f"{short}.status"],
            "generated_code_plan": {
                "handler_name": CONFIGURE4_GENERIC_HANDLER_NAME,
                "api_handler_name": CONFIGURE4_GENERIC_API_HANDLER_NAME,
                "runtime_strategy": "generic_service_record",
                "flow": flow,
            },
            "diagnostics": [],
            "metadata": {"created_at": now, "updated_at": now, "owner": "configure4", "review_required": True},
            "object_graph": self._configure4_default_object_graph(service_id, flow),
            "custom_fields": {},
        }

    def _configure4_default_object_graph(self, service_id: str, flow: dict[str, object]) -> list[dict[str, object]]:
        return [
            Configure4FabricRecord(kind="object", name=service_id, role="service", state="staged").to_dict(),
            Configure4FabricRecord(kind="input", name=f"{service_id}.payload", role="default_payload", state="staged").to_dict(),
            Configure4FabricRecord(kind="flow", name=str(flow.get("type", "service_scaffold")), role="generation", value=copy.deepcopy(flow), links=[service_id], state="staged").to_dict(),
            Configure4FabricRecord(kind="output", name=f"{service_id}.preview", role="integration_plan", links=[service_id], state="staged").to_dict(),
            Configure4FabricRecord(kind="lifecycle", name=f"{service_id}.lifecycle", role="stage_validate_preview_register", links=[service_id], state="staged").to_dict(),
        ]

    def _configure4_expand_aliases(self, payload: dict[str, object]) -> dict[str, object]:
        expanded: dict[str, object] = {}
        custom: dict[str, object] = {}
        section_names = {
            "draft_id",
            "mode",
            "intent",
            "identity",
            "command_surface",
            "api_surface",
            "execution_behavior",
            "validation_rules",
            "persistence_behavior",
            "examples",
            "generated_code_plan",
            "diagnostics",
            "metadata",
            "object_graph",
            "custom_fields",
            "inferred_fields",
        }
        for key, value in payload.items():
            if key in CONFIGURE4_FIELD_ALIASES:
                self._configure4_set_path(expanded, CONFIGURE4_FIELD_ALIASES[key].split("."), copy.deepcopy(value))
            elif key in section_names:
                expanded[key] = copy.deepcopy(value)
            else:
                custom[key] = copy.deepcopy(value)
        if custom:
            existing = expanded.get("custom_fields", {})
            if not isinstance(existing, dict):
                existing = {}
            existing.update(custom)
            expanded["custom_fields"] = existing
        return expanded

    def _configure4_deep_merge(self, base: object, overlay: object) -> object:
        if isinstance(base, dict) and isinstance(overlay, dict):
            result = copy.deepcopy(base)
            for key, value in overlay.items():
                if key in result:
                    result[key] = self._configure4_deep_merge(result[key], value)
                else:
                    result[key] = copy.deepcopy(value)
            return result
        return copy.deepcopy(overlay)

    def _configure4_flat_paths(self, payload: object, prefix: str = "") -> list[str]:
        if isinstance(payload, dict):
            paths: list[str] = []
            for key, value in payload.items():
                child = f"{prefix}.{key}" if prefix else str(key)
                paths.extend(self._configure4_flat_paths(value, child))
            return paths
        return [prefix] if prefix else []

    def _configure4_path_exists(self, payload: object, path: str) -> bool:
        found, _value = self._configure4_get_path(payload, path.split("."))
        return found

    def _configure4_get_path(self, root: object, parts: list[str]) -> tuple[bool, object]:
        current = root
        for part in parts:
            if isinstance(current, dict) and part in current:
                current = current[part]
            elif isinstance(current, list) and part.isdigit() and int(part) < len(current):
                current = current[int(part)]
            else:
                return False, None
        return True, copy.deepcopy(current)

    def _configure4_set_path(self, root: object, parts: list[str], value: object) -> None:
        if not isinstance(root, dict) or not parts:
            return
        current = root
        for part in parts[:-1]:
            if part not in current or not isinstance(current[part], dict):
                current[part] = {}
            current = current[part]
        current[parts[-1]] = copy.deepcopy(value)

    def _configure4_resolve_state_path(self, path: str, *, for_write: bool) -> tuple[object, list[str], str]:
        parts = [part for part in path.split(".") if part]
        if not parts:
            return self.configure4_state, [], "Empty path."
        if parts[0] == "settings":
            return self.configure4_state, parts, ""
        if parts[0] == "drafts":
            if len(parts) < 2:
                return self.configure4_state, parts, ""
            if parts[1] not in self.configure4_state.get("drafts", {}):
                return self.configure4_state, parts, f"Unknown draft: {parts[1]}"
            return self.configure4_state["drafts"][parts[1]], parts[2:], ""
        drafts = self.configure4_state.get("drafts", {})
        if isinstance(drafts, dict) and parts[0] in drafts:
            return drafts[parts[0]], parts[1:], ""
        current = str(self.configure4_state.get("current_draft_id", ""))
        if current and isinstance(drafts, dict) and current in drafts:
            return drafts[current], parts, ""
        if for_write:
            return self.configure4_state, parts, "No current draft. Create a draft first or address settings.*."
        return self.configure4_state, parts, ""

    def _configure4_path_draft_id(self, path: str) -> str:
        parts = [part for part in path.split(".") if part]
        drafts = self.configure4_state.get("drafts", {})
        if parts[:1] == ["drafts"] and len(parts) > 1:
            return parts[1]
        if isinstance(drafts, dict) and parts and parts[0] in drafts:
            return parts[0]
        current = str(self.configure4_state.get("current_draft_id", ""))
        return current if current and parts and parts[0] != "settings" else ""

    def _configure4_default_draft_id(self, target: str) -> str:
        if target and target not in {"current", "."}:
            return target
        return str(self.configure4_state.get("current_draft_id", ""))

    def _configure4_get_spec(self, draft_id: str) -> Configure4Spec | None:
        drafts = self.configure4_state.get("drafts", {})
        if not draft_id or not isinstance(drafts, dict) or draft_id not in drafts:
            return None
        return Configure4Spec.from_dict(drafts[draft_id])

    def _configure4_validate_target(self, target: str, *, append: bool) -> dict[str, object]:
        drafts = self.configure4_state.get("drafts", {})
        if not isinstance(drafts, dict):
            return {"accepted": False, "error": "Draft store is unavailable."}
        if target in {"all", "*", ""}:
            reports = {}
            accepted = True
            for draft_id in sorted(drafts):
                spec = Configure4Spec.from_dict(drafts[draft_id])
                report = self._configure4_validate_spec(spec, append=append)
                reports[draft_id] = report
                accepted = accepted and bool(report.get("accepted"))
            return {"accepted": accepted, "reports": reports}
        spec = self._configure4_get_spec(target)
        if spec is None:
            return {"accepted": False, "error": f"Unknown draft: {target}"}
        return self._configure4_validate_spec(spec, append=append)

    def _configure4_validate_spec(self, spec: Configure4Spec, *, append: bool) -> dict[str, object]:
        errors: list[dict[str, object]] = []
        warnings: list[dict[str, object]] = []
        info: list[dict[str, object]] = []

        def issue(bucket: list[dict[str, object]], severity: str, target: str, message: str, recommendation: str = "") -> None:
            entry = {
                "severity": severity,
                "target": target,
                "message": message,
                "timestamp": _dt.datetime.now().isoformat(timespec="seconds"),
                "recommendation": recommendation,
            }
            bucket.append(entry)
            if append:
                self._configure4_append_diagnostic(severity, target, message, recommendation)

        service_id = str(spec.identity.get("service_id", ""))
        if not re.fullmatch(r"[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)+", service_id):
            issue(errors, "error", "identity.service_id", "Service id must be stable, lowercase, and dot-delimited.")
        if service_id in {"", "blue.cli.engine", CONFIGURE4_SERVICE_ID}:
            issue(errors, "error", "identity.service_id", "Reserved service id cannot be generated.")
        existing = self.services.get(service_id)
        registered = self.configure4_state.get("registered_runtime_services", {})
        if existing and service_id not in registered:
            issue(errors, "error", "identity.service_id", f"Service id already exists: {service_id}", "Choose another id.")

        state = str(spec.identity.get("state", "dormant"))
        if state == "active":
            issue(errors, "error", "identity.state", "Generated services may not start active.", "Use dormant; activate later through service.activate.")
        elif state not in {"dormant", "staged", "faulted"}:
            issue(warnings, "warning", "identity.state", f"Unusual service state: {state}", "Use dormant for services that can be registered.")

        aliases = spec.identity.get("aliases", [])
        if not isinstance(aliases, list):
            issue(errors, "error", "identity.aliases", "Aliases must be a list.")

        verbs = spec.command_surface.get("verbs", [])
        allow_collisions = bool(spec.validation_rules.get("allow_command_collisions", False))
        if not isinstance(verbs, list) or not verbs:
            issue(errors, "error", "command_surface.verbs", "At least one command verb is required.")
            verbs = []
        known_commands = self._configure4_known_commands()
        dynamic_commands = self.configure4_state.get("registered_runtime_commands", {})
        for verb_obj in verbs:
            verb = str(verb_obj).lower()
            if not re.fullmatch(r"[a-z][a-z0-9_]*(\.[a-z][a-z0-9_-]*)*", verb):
                issue(errors, "error", f"command:{verb}", "Command names must be lowercase CLI tokens.")
            if verb.startswith("configure4."):
                issue(errors, "error", f"command:{verb}", "configure4.* is reserved for the authoring fabric.")
            if not allow_collisions and (verb in known_commands or verb in dynamic_commands):
                issue(errors, "error", f"command:{verb}", "Command collides with an existing blue CLI command.", "Choose a unique command or explicitly allow collisions for review-only drafts.")

        routes = spec.api_surface.get("routes", [])
        if not isinstance(routes, list):
            issue(errors, "error", "api_surface.routes", "API routes must be a list.")
            routes = []
        registered_routes = self.configure4_state.get("registered_runtime_routes", {})
        for route_obj in routes:
            route = str(route_obj)
            if not route.startswith("/"):
                issue(errors, "error", f"api_route:{route}", "API routes must start with /.")
            if any(ch.isspace() for ch in route):
                issue(errors, "error", f"api_route:{route}", "API routes cannot contain whitespace.")
            if not allow_collisions and route in self.api_routes and route not in registered_routes:
                issue(errors, "error", f"api_route:{route}", "API route collides with an existing route.")

        handler_name = str(spec.generated_code_plan.get("handler_name", spec.command_surface.get("handler_name", "")))
        api_handler_name = str(spec.generated_code_plan.get("api_handler_name", spec.api_surface.get("handler_name", "")))
        if handler_name != CONFIGURE4_GENERIC_HANDLER_NAME and not hasattr(self, handler_name):
            issue(errors, "error", "generated_code_plan.handler_name", f"Missing command handler method: {handler_name}", "Use the generic handler or apply the preview patch plan.")
        if routes and api_handler_name != CONFIGURE4_GENERIC_API_HANDLER_NAME and not hasattr(self, api_handler_name):
            issue(errors, "error", "generated_code_plan.api_handler_name", f"Missing API handler method: {api_handler_name}", "Use the generic API handler or apply the preview patch plan.")

        flow = spec.generated_code_plan.get("flow", {})
        if not isinstance(flow, dict):
            issue(errors, "error", "generated_code_plan.flow", "Flow must be an object.")
            flow = {}
        flow_type = str(flow.get("type", spec.execution_behavior.get("flow_type", "service_scaffold")))
        if flow_type not in CONFIGURE4_ALLOWED_FLOW_TYPES:
            issue(errors, "error", "generated_code_plan.flow.type", f"Unsupported flow type: {flow_type}")
        else:
            bounded = self._configure4_check_flow_bounds(flow)
            if not bounded["ok"]:
                issue(errors, "error", "generated_code_plan.flow", str(bounded["message"]), "Reduce input_width, input values, or repeat count.")

        persistence_policy = str(spec.persistence_behavior.get("policy", "manifest"))
        if persistence_policy not in CONFIGURE4_ALLOWED_PERSISTENCE_POLICIES:
            issue(errors, "error", "persistence_behavior.policy", f"Unsupported persistence policy: {persistence_policy}")

        file_policy = str(spec.execution_behavior.get("file_access_policy", "none"))
        if file_policy not in CONFIGURE4_ALLOWED_FILE_POLICIES:
            issue(errors, "error", "execution_behavior.file_access_policy", f"Unsupported file access policy: {file_policy}")
        side_effect_policy = str(spec.execution_behavior.get("side_effect_policy", "memory_only"))
        if side_effect_policy not in CONFIGURE4_ALLOWED_SIDE_EFFECT_POLICIES:
            issue(errors, "error", "execution_behavior.side_effect_policy", f"Unsupported side-effect policy: {side_effect_policy}")
        output_target = str(flow.get("output_target", "memory"))
        filesystem_targets = {"stdout", "console", "memory", "json", "file-preview", ""}
        if output_target not in filesystem_targets and file_policy not in {"write_explicit", "read_write_explicit", "manifest_only"}:
            issue(errors, "error", "generated_code_plan.flow.output_target", "Filesystem output target requires an explicit file access policy.")

        allowed_modes = spec.execution_behavior.get("allowed_modes", list(CONFIGURE4_ALLOWED_MODES))
        if not isinstance(allowed_modes, list) or not all(configure4_normalize_mode(item) in CONFIGURE4_ALLOWED_MODES for item in allowed_modes):
            issue(errors, "error", "execution_behavior.allowed_modes", "allowed_modes must contain only manual, semi_auto, or auto.")

        timeout_policy = spec.execution_behavior.get("timeout_policy", {})
        if isinstance(timeout_policy, dict):
            seconds = timeout_policy.get("seconds", 30)
            if not isinstance(seconds, int) or seconds < 1 or seconds > 600:
                issue(errors, "error", "execution_behavior.timeout_policy", "Timeout seconds must be an integer from 1 to 600.")
        else:
            issue(errors, "error", "execution_behavior.timeout_policy", "timeout_policy must be an object.")

        activation_requirements = spec.execution_behavior.get("activation_requirements", [])
        dependency_requirements = spec.execution_behavior.get("dependency_requirements", [])
        if not isinstance(activation_requirements, list):
            issue(errors, "error", "execution_behavior.activation_requirements", "activation_requirements must be a list.")
            activation_requirements = []
        if "blue.cli.engine" not in activation_requirements:
            issue(errors, "error", "execution_behavior.activation_requirements", "blue.cli.engine must remain an activation requirement.")
        for dependency in list(activation_requirements) + (dependency_requirements if isinstance(dependency_requirements, list) else []):
            dep_id = self.resolve_service_id(str(dependency))
            if dep_id != service_id and dep_id not in self.services:
                issue(errors, "error", f"dependency:{dependency}", "Unknown activation or dependency requirement.")
        if not isinstance(dependency_requirements, list):
            issue(errors, "error", "execution_behavior.dependency_requirements", "dependency_requirements must be a list.")

        if spec.mode == "auto" and bool(spec.execution_behavior.get("can_register_immediately", False)):
            issue(warnings, "warning", "execution_behavior.can_register_immediately", "Automatic drafts should be reviewed before registration.")

        if not errors:
            issue(info, "accepted", spec.draft_id, "Validation accepted.")

        report = {
            "draft_id": spec.draft_id,
            "service_id": service_id,
            "accepted": len(errors) == 0,
            "spec_hash": self._configure4_spec_hash(spec),
            "checked_at": _dt.datetime.now().isoformat(timespec="seconds"),
            "errors": errors,
            "warnings": warnings,
            "info": info,
        }
        self.configure4_state.setdefault("validation_reports", {})[spec.draft_id] = copy.deepcopy(report)
        drafts = self.configure4_state.get("drafts", {})
        if isinstance(drafts, dict) and spec.draft_id in drafts:
            drafts[spec.draft_id]["diagnostics"] = errors + warnings + info
        return report

    def _configure4_known_commands(self) -> set[str]:
        return {
            "help",
            "?",
            "clear",
            "history",
            "blue.engine",
            "terminal.cwd",
            "terminal.cd",
            "terminal.timeout",
            "linux",
            "bash",
            "sh",
            "pwsh",
            "powershell",
            "powershell7",
            "service.list",
            "service.status",
            "service.activate",
            "service.deactivate",
            "service.restart",
            "service.exec",
            "api.list",
            "api.call",
            "api.register",
            "kernel.info",
            "set",
            "get",
            "vars",
            "manifest.show",
            "manifest.save",
            "manifest.load",
            "node.list",
            "node.select",
            "node.add",
            "node.patch",
            "node.validate",
            "file.search",
            "diagnostics.list",
            "diagnostics.patch",
            "emit.preview",
            "build.run",
            "version.snapshot",
            "version.list",
            "reboot",
            "configure4.help",
            "configure4.status",
            "configure4.mode",
            "configure4.new",
            "configure4.set",
            "configure4.get",
            "configure4.validate",
            "configure4.preview",
            "configure4.register",
            "configure4.export",
            "configure4.import",
            "configure4.sample",
            "configure4.reset",
            "configure4.history",
            "configure4.list",
            "quadtree.desktop.status",
            "quadtree.desktop.show",
            "quadtree.desktop.hide",
            "quadtree.desktop.module.create",
            "quadtree.desktop.module.list",
            "quadtree.desktop.module.get",
            "quadtree.desktop.module.patch",
            "quadtree.desktop.layer.get",
            "quadtree.desktop.layer.patch",
            "quadtree.desktop.cell.get",
            "quadtree.desktop.cell.set",
            "quadtree.desktop.cell.patch",
            "quadtree.desktop.cell.reset",
            "quadtree.desktop.cell.subdivide",
            "quadtree.desktop.cell.bind",
            "quadtree.desktop.selection.set",
            "quadtree.desktop.batch.create",
            "quadtree.desktop.batch.select",
            "quadtree.desktop.batch.patch",
            "quadtree.desktop.batch.validate",
            "quadtree.desktop.batch.preview",
            "quadtree.desktop.batch.apply",
            "quadtree.desktop.batch.rollback",
            "quadtree.desktop.validate",
            "quadtree.desktop.preview",
            "quadtree.desktop.apply",
            "quadtree.desktop.export.manifest",
            "quadtree.desktop.export.image",
            "quadtree.desktop.import.manifest",
            "quadtree.desktop.snapshot",
        }

    def _configure4_check_flow_bounds(self, flow: dict[str, object]) -> dict[str, object]:
        flow_type = str(flow.get("type", "service_scaffold"))
        max_variants = int(self.configure4_state.get("settings", {}).get("max_variants", CONFIGURE4_MAX_VARIANTS))
        if flow_type == "cartesian":
            alphabet = str(flow.get("input_value", ""))
            try:
                width = int(flow.get("input_width", 1))
            except (TypeError, ValueError):
                return {"ok": False, "message": "input_width must be an integer."}
            count, ok = self._configure4_safe_power(len(alphabet), width, max_variants)
            if not ok:
                return {"ok": False, "message": f"Cartesian flow would exceed {max_variants} variants."}
            return {"ok": True, "count": count}
        if flow_type == "repeat":
            try:
                count = int(flow.get("input_width", 1))
            except (TypeError, ValueError):
                return {"ok": False, "message": "repeat input_width must be an integer."}
            if count < 0 or count > max_variants:
                return {"ok": False, "message": f"Repeat flow must be between 0 and {max_variants}."}
            return {"ok": True, "count": count}
        return {"ok": True, "count": 1}

    def _configure4_safe_power(self, base: int, exp: int, limit: int) -> tuple[int, bool]:
        if base <= 0 or exp < 0:
            return 0, False
        result = 1
        for _index in range(exp):
            if result > limit // max(base, 1):
                return result, False
            result *= base
        return result, result <= limit

    def _configure4_safe_product(self, values: list[int], limit: int) -> tuple[int, bool]:
        result = 1
        for value in values:
            if value < 0 or result > limit // max(value, 1):
                return result, False
            result *= value
        return result, result <= limit

    def _configure4_build_preview(self, spec: Configure4Spec, *, append_validation: bool) -> dict[str, object]:
        report = self._configure4_validate_spec(spec, append=append_validation)
        service_entry = self._configure4_service_entry_from_spec(spec)
        routes = self._configure4_api_route_entries_from_spec(spec)
        flow_sink = Configure4MemorySink()
        self._configure4_generate_flow(spec, flow_sink)
        preview = {
            "draft_id": spec.draft_id,
            "service_id": spec.identity.get("service_id", ""),
            "spec_hash": report["spec_hash"],
            "accepted": report["accepted"],
            "service_registry_entry": service_entry,
            "service_registry_tuple_for_create_services": self._configure4_registry_tuple_snippet(spec),
            "dispatch_command_branches": self._configure4_dispatch_branch_snippets(spec),
            "api_routes_for_create_api_routes": routes,
            "api_route_snippets": self._configure4_api_route_snippets(spec),
            "handler_method_skeletons": self._configure4_handler_skeletons(spec),
            "persistence_additions": [
                "current_project_payload includes configure4_state.",
                "manifest.save persists configure4_state through current_project_payload.",
                "manifest.load restores configure4_state through _restore_configure4_state.",
                "version.snapshot records configure4 draft and registration counts.",
            ],
            "help_text_additions": [str(spec.command_surface.get("help_text", ""))],
            "generated_flow_preview": flow_sink.getvalue(),
            "validation": report,
            "patch_plan_required": not report["accepted"] or self._configure4_requires_python_patch(spec),
        }
        return preview

    def _configure4_service_entry_from_spec(self, spec: Configure4Spec) -> dict[str, object]:
        service_id = str(spec.identity.get("service_id", ""))
        return {
            "id": service_id,
            "label": str(spec.identity.get("label", service_id)),
            "layer": str(spec.identity.get("layer", "codex-authoring")),
            "state": "dormant",
            "description": str(spec.identity.get("description", "Generated by configure_4.authoring.fabric.")),
            "examples": [str(item) for item in spec.examples],
            "aliases": [str(item) for item in spec.identity.get("aliases", [])] if isinstance(spec.identity.get("aliases", []), list) else [],
            "activated_at": "",
            "last_result": "",
            "calls": 0,
            "configure4_generated": True,
            "configure4_draft_id": spec.draft_id,
            "configure4_spec_hash": self._configure4_spec_hash(spec),
        }

    def _configure4_api_route_entries_from_spec(self, spec: Configure4Spec) -> dict[str, dict[str, object]]:
        routes = {}
        route_list = spec.api_surface.get("routes", [])
        if not isinstance(route_list, list):
            return routes
        service_id = str(spec.identity.get("service_id", ""))
        for route_obj in route_list:
            route = str(route_obj)
            routes[route] = {
                "service": service_id,
                "description": f"Generated route for {service_id}.",
                "handler": str(spec.generated_code_plan.get("api_handler_name", spec.api_surface.get("handler_name", CONFIGURE4_GENERIC_API_HANDLER_NAME))),
            }
        return routes

    def _configure4_registry_tuple_snippet(self, spec: Configure4Spec) -> str:
        return "\n".join(
            [
                "(",
                f"    {str(spec.identity.get('service_id', ''))!r},",
                f"    {str(spec.identity.get('layer', 'codex-authoring'))!r},",
                "    'dormant',",
                f"    {str(spec.identity.get('description', ''))!r},",
                f"    {[str(item) for item in spec.examples]!r},",
                "),",
            ]
        )

    def _configure4_dispatch_branch_snippets(self, spec: Configure4Spec) -> list[str]:
        snippets = []
        service_id = str(spec.identity.get("service_id", ""))
        verbs = spec.command_surface.get("verbs", [])
        if not isinstance(verbs, list):
            return snippets
        for verb in verbs:
            snippets.append(
                "\n".join(
                    [
                        f"if cmd == {str(verb)!r}:",
                        f"    if not self.require_service({service_id!r}):",
                        f"        return 'Service {service_id} is dormant. Activate it first: service.activate {service_id}', 'err'",
                        f"    return self.{CONFIGURE4_GENERIC_HANDLER_NAME}({service_id!r}, cmd, self.parse_payload(raw_after_cmd)), 'json'",
                    ]
                )
            )
        return snippets

    def _configure4_api_route_snippets(self, spec: Configure4Spec) -> list[str]:
        snippets = []
        service_id = str(spec.identity.get("service_id", ""))
        routes = spec.api_surface.get("routes", [])
        if not isinstance(routes, list):
            return snippets
        for route in routes:
            snippets.append(
                f"{str(route)!r}: {{'service': {service_id!r}, 'description': 'Generated route for {service_id}.', 'handler': self.{CONFIGURE4_GENERIC_API_HANDLER_NAME}}},"
            )
        return snippets

    def _configure4_handler_skeletons(self, spec: Configure4Spec) -> dict[str, str]:
        service_id = str(spec.identity.get("service_id", ""))
        handler_name = str(spec.generated_code_plan.get("handler_name", spec.command_surface.get("handler_name", CONFIGURE4_GENERIC_HANDLER_NAME)))
        api_handler_name = str(spec.generated_code_plan.get("api_handler_name", spec.api_surface.get("handler_name", CONFIGURE4_GENERIC_API_HANDLER_NAME)))
        return {
            handler_name: "\n".join(
                [
                    f"def {handler_name}(self, service_id: str, command: str, payload: object) -> dict[str, object]:",
                    f"    # Generated command handler skeleton for {service_id}.",
                    "    return {'service': service_id, 'command': command, 'payload': payload}",
                ]
            ),
            api_handler_name: "\n".join(
                [
                    f"def {api_handler_name}(self, service_id: str, route: str, payload: object) -> dict[str, object]:",
                    f"    # Generated API handler skeleton for {service_id}.",
                    "    return {'service': service_id, 'route': route, 'payload': payload}",
                ]
            ),
        }

    def _configure4_register_draft(self, draft_id: str) -> dict[str, object]:
        spec = self._configure4_get_spec(draft_id)
        if spec is None:
            return {"accepted": False, "error": f"Unknown draft: {draft_id or '<none>'}"}
        report = self._configure4_validate_spec(spec, append=True)
        if not report["accepted"]:
            self._configure4_append_diagnostic("error", spec.draft_id, "Registration refused because validation failed.", "Run configure4.preview for the patch plan.")
            return {"accepted": False, "reason": "validation_failed", "validation": report, "patch_plan": self._configure4_build_preview(spec, append_validation=False)}
        if not bool(spec.execution_behavior.get("can_register_immediately", False)):
            self._configure4_append_diagnostic(
                "warning",
                spec.draft_id,
                "Registration refused because the draft is staged for review only.",
                "Patch execution_behavior.can_register_immediately to true after review.",
            )
            return {"accepted": False, "reason": "staged_for_review", "validation": report, "patch_plan": self._configure4_build_preview(spec, append_validation=False)}
        if self._configure4_requires_python_patch(spec):
            self._configure4_append_diagnostic("error", spec.draft_id, "Registration requires Python methods that cannot be injected safely.", "Apply the preview patch plan to start.py.")
            return {"accepted": False, "reason": "python_patch_required", "validation": report, "patch_plan": self._configure4_build_preview(spec, append_validation=False)}

        service_id = str(spec.identity.get("service_id", ""))
        service_entry = self._configure4_service_entry_from_spec(spec)
        self.services[service_id] = service_entry
        self._configure4_bind_runtime_commands(spec)
        self._configure4_bind_runtime_routes(spec)
        event = {
            "time": _dt.datetime.now().isoformat(timespec="seconds"),
            "draft_id": spec.draft_id,
            "service_id": service_id,
            "spec_hash": report["spec_hash"],
            "commands": copy.deepcopy(spec.command_surface.get("verbs", [])),
            "routes": copy.deepcopy(spec.api_surface.get("routes", [])),
            "status": "registered",
        }
        self.configure4_state.setdefault("registration_history", []).append(event)
        self.configure4_state.setdefault("registered_runtime_services", {})[service_id] = {
            "draft_id": spec.draft_id,
            "service_id": service_id,
            "spec_hash": report["spec_hash"],
            "service": service_entry,
        }
        self._configure4_trim_history()
        self._configure4_append_diagnostic("accepted", spec.draft_id, f"Registered runtime service {service_id}.")
        self._record_event("configure4.register", service_id)
        return {"accepted": True, "registered": event, "service": service_entry}

    def _configure4_requires_python_patch(self, spec: Configure4Spec) -> bool:
        handler_name = str(spec.generated_code_plan.get("handler_name", spec.command_surface.get("handler_name", CONFIGURE4_GENERIC_HANDLER_NAME)))
        api_handler_name = str(spec.generated_code_plan.get("api_handler_name", spec.api_surface.get("handler_name", CONFIGURE4_GENERIC_API_HANDLER_NAME)))
        if handler_name != CONFIGURE4_GENERIC_HANDLER_NAME and not hasattr(self, handler_name):
            return True
        routes = spec.api_surface.get("routes", [])
        if routes and api_handler_name != CONFIGURE4_GENERIC_API_HANDLER_NAME and not hasattr(self, api_handler_name):
            return True
        return False

    def _configure4_bind_runtime_commands(self, spec: Configure4Spec) -> None:
        service_id = str(spec.identity.get("service_id", ""))
        commands = self.configure4_state.setdefault("registered_runtime_commands", {})
        verbs = spec.command_surface.get("verbs", [])
        if not isinstance(verbs, list):
            return
        for verb_obj in verbs:
            commands[str(verb_obj).lower()] = {
                "service_id": service_id,
                "draft_id": spec.draft_id,
                "spec_hash": self._configure4_spec_hash(spec),
            }

    def _configure4_bind_runtime_routes(self, spec: Configure4Spec) -> None:
        service_id = str(spec.identity.get("service_id", ""))
        routes = spec.api_surface.get("routes", [])
        if not isinstance(routes, list):
            return
        for route_obj in routes:
            route = str(route_obj)
            self.api_routes[route] = {
                "service": service_id,
                "description": f"Generated route for {service_id}.",
                "handler": lambda payload, sid=service_id, rt=route: self.generic_configure4_api_handler(sid, rt, payload),
            }
            self.configure4_state.setdefault("registered_runtime_routes", {})[route] = {
                "service_id": service_id,
                "draft_id": spec.draft_id,
                "spec_hash": self._configure4_spec_hash(spec),
            }

    def _remove_configure4_runtime_registrations(self, *, draft_id: str = "") -> None:
        services = self.configure4_state.get("registered_runtime_services", {})
        commands = self.configure4_state.get("registered_runtime_commands", {})
        routes = self.configure4_state.get("registered_runtime_routes", {})
        service_ids: set[str] = set()
        if isinstance(services, dict):
            for service_id, record in list(services.items()):
                if not draft_id or (isinstance(record, dict) and record.get("draft_id") == draft_id):
                    service_ids.add(service_id)
                    services.pop(service_id, None)
        if isinstance(commands, dict):
            for command, record in list(commands.items()):
                if not draft_id or (isinstance(record, dict) and record.get("draft_id") == draft_id):
                    commands.pop(command, None)
        if isinstance(routes, dict):
            for route, record in list(routes.items()):
                if not draft_id or (isinstance(record, dict) and record.get("draft_id") == draft_id):
                    self.api_routes.pop(route, None)
                    routes.pop(route, None)
        for service_id in service_ids:
            self.services.pop(service_id, None)

    def _configure4_rebind_registered_services(self) -> None:
        services = self.configure4_state.get("registered_runtime_services", {})
        drafts = self.configure4_state.get("drafts", {})
        if not isinstance(services, dict) or not isinstance(drafts, dict):
            return
        for service_id, record in list(services.items()):
            if not isinstance(record, dict):
                continue
            draft_id = str(record.get("draft_id", ""))
            if draft_id not in drafts:
                continue
            spec = Configure4Spec.from_dict(drafts[draft_id])
            service_entry = self._configure4_service_entry_from_spec(spec)
            service_entry["state"] = "dormant"
            self.services[service_id] = service_entry
            self._configure4_bind_runtime_commands(spec)
            self._configure4_bind_runtime_routes(spec)

    def generic_configure4_service_handler(self, service_id: str, command: str, payload: object) -> dict[str, object]:
        svc = self.services.get(service_id)
        if svc:
            svc["calls"] = int(svc.get("calls", 0)) + 1
            svc["last_result"] = f"{command} handled by {CONFIGURE4_GENERIC_HANDLER_NAME}"
        result = {
            "service": service_id,
            "command": command,
            "payload": payload,
            "handler": CONFIGURE4_GENERIC_HANDLER_NAME,
            "note": "Runtime-safe generated service command handled without executing generated code.",
        }
        self._record_event("configure4.generated.command", f"{service_id}:{command}")
        return result

    def generic_configure4_api_handler(self, service_id: str, route: str, payload: object) -> dict[str, object]:
        return {
            "service": service_id,
            "route": route,
            "payload": payload,
            "handler": CONFIGURE4_GENERIC_API_HANDLER_NAME,
            "note": "Runtime-safe generated API route handled without executing generated code.",
        }

    def _configure4_generate_flow(self, spec: Configure4Spec, sink: Configure4Sink) -> bool:
        flow = spec.generated_code_plan.get("flow", {})
        if not isinstance(flow, dict):
            return False
        flow_type = str(flow.get("type", "service_scaffold"))
        input_value = str(flow.get("input_value", ""))
        separator = str(flow.get("separator", "\n"))
        prefix = str(flow.get("prefix", ""))
        suffix = str(flow.get("suffix", ""))
        if flow_type == "literal":
            return sink.write(f"{prefix}{input_value}{suffix}{separator}")
        if flow_type == "template":
            template = str(flow.get("template", input_value))
            values = flow.get("values", {})
            if isinstance(values, dict):
                rendered = template
                for key, value in values.items():
                    rendered = rendered.replace("{{" + str(key) + "}}", str(value))
            else:
                rendered = template
            return sink.write(f"{prefix}{rendered}{suffix}{separator}")
        if flow_type == "repeat":
            count = int(flow.get("input_width", 1))
            for _index in range(count):
                if not sink.write(f"{prefix}{input_value}{suffix}{separator}"):
                    return False
            return True
        if flow_type == "reverse":
            return sink.write(f"{prefix}{input_value[::-1]}{suffix}{separator}")
        if flow_type == "cartesian":
            width = int(flow.get("input_width", 1))
            count, ok = self._configure4_safe_power(len(input_value), width, int(self.configure4_state.get("settings", {}).get("max_variants", CONFIGURE4_MAX_VARIANTS)))
            if not ok:
                return False
            for combo in itertools.product(input_value, repeat=width):
                if not sink.write(f"{prefix}{''.join(combo)}{suffix}{separator}"):
                    return False
            return count >= 0
        if flow_type == "service_scaffold":
            return sink.write_json(
                {
                    "service_id": spec.identity.get("service_id", ""),
                    "commands": spec.command_surface.get("verbs", []),
                    "routes": spec.api_surface.get("routes", []),
                    "handler": spec.generated_code_plan.get("handler_name", CONFIGURE4_GENERIC_HANDLER_NAME),
                    "api_handler": spec.generated_code_plan.get("api_handler_name", CONFIGURE4_GENERIC_API_HANDLER_NAME),
                    "persistence": spec.persistence_behavior,
                }
            )
        return False

    def _configure4_spec_hash(self, spec: Configure4Spec) -> str:
        payload = spec.to_dict()
        payload.pop("diagnostics", None)
        payload.pop("inferred_fields", None)
        metadata = payload.get("metadata", {})
        if isinstance(metadata, dict):
            metadata.pop("created_at", None)
            metadata.pop("updated_at", None)
        payload.pop("draft_id", None)
        encoded = json.dumps(payload, sort_keys=True, separators=(",", ":"))
        return hashlib.sha256(encoded.encode("utf-8")).hexdigest()[:16]

    def _configure4_append_diagnostic(self, severity: str, target: str, message: str, recommendation: str = "", code: str = "") -> dict[str, object]:
        entry = {
            "severity": severity,
            "target": target,
            "message": message,
            "timestamp": _dt.datetime.now().isoformat(timespec="seconds"),
            "recommendation": recommendation,
        }
        if code:
            entry["code"] = code
        self.configure4_state.setdefault("diagnostics", []).append(entry)
        self._configure4_trim_history()
        return entry

    def _configure4_trim_history(self) -> None:
        for key in ("diagnostics", "registration_history"):
            items = self.configure4_state.get(key, [])
            if isinstance(items, list) and len(items) > CONFIGURE4_MAX_HISTORY:
                self.configure4_state[key] = items[-CONFIGURE4_MAX_HISTORY:]

    def _configure4_status_payload(self) -> dict[str, object]:
        drafts = self.configure4_state.get("drafts", {})
        registered = self.configure4_state.get("registered_runtime_services", {})
        diagnostics = self.configure4_state.get("diagnostics", [])
        return {
            "service_id": CONFIGURE4_SERVICE_ID,
            "label": CONFIGURE4_LABEL,
            "mode": self.configure4_state.get("mode", "manual"),
            "allowed_modes": list(CONFIGURE4_ALLOWED_MODES),
            "flow_types": list(CONFIGURE4_ALLOWED_FLOW_TYPES),
            "draft_count": len(drafts) if isinstance(drafts, dict) else 0,
            "current_draft_id": self.configure4_state.get("current_draft_id", ""),
            "registered_runtime_services": list(registered) if isinstance(registered, dict) else [],
            "settings": copy.deepcopy(self.configure4_state.get("settings", {})),
            "latest_diagnostics": copy.deepcopy(diagnostics[-10:]) if isinstance(diagnostics, list) else [],
        }

    def _configure4_sample_payload(self) -> dict[str, object]:
        manual = {
            "mode": "manual",
            "draft_id": "manual_diagnostics_echo",
            "identity": {
                "service_id": "codex.diagnostics.echo",
                "label": "CodexDiagnosticsEcho",
                "aliases": ["diagnostics.echo"],
                "layer": "codex-authoring",
                "state": "dormant",
                "description": "Echoes diagnostic payloads through a runtime-safe generated service.",
                "version": "1.0.0",
            },
            "command_surface": {
                "verbs": ["diagnostics.echo.status", "diagnostics.echo.run"],
                "handler_name": CONFIGURE4_GENERIC_HANDLER_NAME,
                "help_text": "Runtime-safe diagnostics echo service.",
                "default_payload": {"message": "hello"},
            },
            "api_surface": {
                "routes": ["/configure4/generated/codex-diagnostics-echo"],
                "handler_name": CONFIGURE4_GENERIC_API_HANDLER_NAME,
                "payload_schema": {"type": "object", "additionalProperties": True},
                "output_schema": {"type": "object", "required": ["service", "payload"]},
            },
            "execution_behavior": {
                "activation_requirements": ["blue.cli.engine"],
                "dependency_requirements": [],
                "allowed_modes": ["manual", "semi_auto", "auto"],
                "timeout_policy": {"seconds": 30},
                "file_access_policy": "none",
                "side_effect_policy": "memory_only",
                "can_register_immediately": True,
            },
            "validation_rules": {
                "gates": ["service_id", "command_surface", "api_surface", "activation_dependencies", "filesystem_policy", "flow_bounds", "handler_presence"],
                "allow_command_collisions": False,
            },
            "persistence_behavior": {
                "policy": "manifest",
                "import_path": "",
                "export_path": "",
                "versioning_policy": "snapshot_in_manifest",
            },
            "examples": ["service.activate codex.diagnostics.echo", "diagnostics.echo.run {\"message\":\"hello\"}"],
            "generated_code_plan": {
                "handler_name": CONFIGURE4_GENERIC_HANDLER_NAME,
                "api_handler_name": CONFIGURE4_GENERIC_API_HANDLER_NAME,
                "runtime_strategy": "generic_service_record",
                "flow": {
                    "type": "service_scaffold",
                    "input_value": "codex.diagnostics.echo",
                    "input_width": 1,
                    "output_target": "memory",
                    "prefix": "",
                    "suffix": "",
                    "separator": "\n",
                },
            },
            "metadata": {"owner": "developer", "review_required": False},
            "object_graph": self._configure4_default_object_graph("codex.diagnostics.echo", {"type": "service_scaffold"}),
            "custom_fields": {"domain": "diagnostics"},
        }
        semi = {
            "mode": "semi_auto",
            "service_id": "codex.summary.quick",
            "label": "CodexSummaryQuick",
            "description": "Summarizes short text payloads.",
            "verbs": ["summary.quick.run"],
            "flow_type": "literal",
            "input_value": "summary-ready",
            "can_register_immediately": False,
        }
        auto = {"mode": "auto", "intent": "create a diagnostics summarizer service"}
        text_generation = {
            "mode": "semi_auto",
            "service_id": "codex.text.banner",
            "label": "CodexTextBanner",
            "description": "Generates reusable release banner text from a template.",
            "verbs": ["text.banner.run"],
            "generated_code_plan": {
                "flow": {
                    "type": "template",
                    "template": "Project {{name}} targets {{language}}.",
                    "values": {"name": "demo", "language": "Python"},
                    "output_target": "memory",
                    "separator": "\n",
                }
            },
            "custom_fields": {
                "use_case": "text_generation",
                "notes": "Preview returns generated text without filesystem writes.",
            },
        }
        pointer_generation = {
            "mode": "semi_auto",
            "service_id": "codex.pointer.index",
            "label": "CodexPointerIndex",
            "description": "Generates stable textual pointers for files, symbols, routes, and object ids.",
            "verbs": ["pointer.index.run"],
            "generated_code_plan": {
                "flow": {
                    "type": "template",
                    "template": "{{root}}/{{module}}.py::{{symbol}}",
                    "values": {"root": "src", "module": "diagnostics", "symbol": "summarize"},
                    "output_target": "memory",
                    "separator": "\n",
                }
            },
            "custom_fields": {
                "use_case": "pointer_generation",
                "pointer_kind": "source_location_string",
                "memory_pointer": False,
            },
        }
        codebase_creation = {
            "mode": "auto",
            "intent": "create a Rust command line codebase that validates JSON manifests",
            "custom_fields": {
                "use_case": "codebase_creation",
                "codebase": {
                    "language": "Rust",
                    "project_type": "cli",
                    "package_manager": "cargo",
                    "files": [
                        {"path": "Cargo.toml", "role": "package manifest"},
                        {"path": "src/main.rs", "role": "entrypoint"},
                        {"path": "tests/manifest_validation.rs", "role": "integration tests"},
                    ],
                    "entrypoints": ["src/main.rs"],
                    "build_commands": ["cargo build"],
                    "test_commands": ["cargo test"],
                    "quality_gates": ["format", "lint", "tests"],
                    "output_policy": "preview first; write files only after explicit filesystem command",
                }
            },
        }
        return {
            "manual": manual,
            "semi_auto": semi,
            "auto": auto,
            "text_generation": text_generation,
            "pointer_generation": pointer_generation,
            "codebase_creation": codebase_creation,
            "safe_service_scaffold_flow": manual["generated_code_plan"]["flow"],
        }

    def _configure4_parse_key_value_config(self, text: str) -> dict[str, object]:
        payload: dict[str, object] = {}
        objects: list[dict[str, object]] = []
        current: dict[str, object] = {}
        for raw_line in text.splitlines():
            line = raw_line.strip()
            if not line or line.startswith("#"):
                continue
            if line.lower() == "end":
                if current:
                    objects.append(current)
                    current = {}
                continue
            if "=" in line:
                key, value = line.split("=", 1)
            else:
                parts = line.split(None, 1)
                key, value = parts[0], parts[1] if len(parts) > 1 else ""
            key = key.strip()
            value = value.strip().encode("utf-8").decode("unicode_escape")
            current[key] = value
        if current:
            objects.append(current)
        if objects:
            first = objects[0]
            payload.update(first)
            payload["object_graph"] = [
                Configure4FabricRecord(kind="object", name=str(item.get("object_name", item.get("name", f"object_{index}"))), value=copy.deepcopy(item)).to_dict()
                for index, item in enumerate(objects, start=1)
            ]
            payload["custom_fields"] = {"kv_objects": objects}
        return self._configure4_expand_aliases(payload)

    def _restore_configure4_state(self, payload: object) -> None:
        self._remove_configure4_runtime_registrations()
        fresh = self._create_configure4_state()
        if not isinstance(payload, dict):
            self.configure4_state = fresh
            self._configure4_append_diagnostic("error", "manifest.load", "configure4_state was malformed and has been reset.")
            return
        if isinstance(payload.get("settings", {}), dict):
            fresh["settings"].update(copy.deepcopy(payload["settings"]))
        mode = configure4_normalize_mode(payload.get("mode", fresh["mode"]))
        fresh["mode"] = mode if mode in CONFIGURE4_ALLOWED_MODES else "manual"
        for key in ("validation_reports", "registered_runtime_commands", "registered_runtime_routes"):
            if isinstance(payload.get(key), dict):
                fresh[key] = copy.deepcopy(payload[key])
        for key in ("registration_history", "diagnostics"):
            if isinstance(payload.get(key), list):
                fresh[key] = copy.deepcopy(payload[key])[-CONFIGURE4_MAX_HISTORY:]
        drafts = payload.get("drafts", {})
        if isinstance(drafts, dict):
            fresh["drafts"] = {str(draft_id): Configure4Spec.from_dict(draft).to_dict() for draft_id, draft in drafts.items()}
        generated = payload.get("generated_service_drafts", {})
        if isinstance(generated, dict):
            fresh["generated_service_drafts"] = {str(draft_id): Configure4Spec.from_dict(draft).to_dict() for draft_id, draft in generated.items()}
        registered_services = payload.get("registered_runtime_services", {})
        if isinstance(registered_services, dict):
            fresh["registered_runtime_services"] = copy.deepcopy(registered_services)
        current = str(payload.get("current_draft_id", ""))
        fresh["current_draft_id"] = current if current in fresh["drafts"] else next(iter(fresh["drafts"]), "")
        try:
            fresh["next_draft_index"] = max(int(payload.get("next_draft_index", 1)), 1)
        except (TypeError, ValueError):
            fresh["next_draft_index"] = 1
        self.configure4_state = fresh
        self._configure4_rebind_registered_services()

    def command_set(self, key: str, value: str) -> str:
        if not self.require_service("kernel.memory"):
            return "kernel.memory is dormant. Activate it first: service.activate kernel.memory"
        self.vars[key] = value
        self._record_event("set", key)
        return f"{key} = {value}"

    def command_terminal_timeout(self, seconds_text: str) -> tuple[str, str]:
        if not self.require_any_terminal_service():
            return "Activate a terminal service first: service.activate kernel.terminal.linux or service.activate kernel.terminal.powershell7", "err"
        try:
            seconds = int(seconds_text)
        except ValueError:
            return "Usage: terminal.timeout <seconds>", "err"
        if seconds < 1 or seconds > 600:
            return "Terminal timeout must be between 1 and 600 seconds.", "err"
        self.terminal_timeout_seconds = seconds
        self._record_event("terminal.timeout", str(seconds))
        return f"terminal timeout = {seconds}s", "ok"

    def command_terminal_cd(self, target: str) -> tuple[str, str]:
        if not self.require_any_terminal_service():
            return "Activate a terminal service first: service.activate kernel.terminal.linux or service.activate kernel.terminal.powershell7", "err"
        raw = target.strip() or "~"
        try:
            parts = shlex.split(raw)
        except ValueError:
            parts = [raw]
        path_text = parts[0] if parts else "~"
        new_path = Path(os.path.expanduser(os.path.expandvars(path_text)))
        if not new_path.is_absolute():
            new_path = self.terminal_cwd / new_path
        try:
            new_path = new_path.resolve()
        except OSError as exc:
            return f"Could not resolve path: {exc}", "err"
        if not new_path.exists():
            return f"Path does not exist: {new_path}", "err"
        if not new_path.is_dir():
            return f"Path is not a directory: {new_path}", "err"
        self.terminal_cwd = new_path
        self._record_event("terminal.cd", str(new_path))
        return f"terminal cwd = {new_path}", "ok"

    def require_any_terminal_service(self) -> bool:
        return self.require_service("kernel.terminal.linux") or self.require_service("kernel.terminal.powershell7")

    def execute_linux_terminal(self, command: str) -> tuple[str, str]:
        service_id = "kernel.terminal.linux"
        if not self.require_service(service_id):
            return f"Service {service_id} is dormant. Activate it first: service.activate {service_id}", "err"
        if not self.linux_shell:
            return "Linux terminal service is active, but no bash/sh executable was found on this host.", "err"
        command = command.strip()
        if not command:
            return self.format_json({
                "service": service_id,
                "available": True,
                "executable": self.linux_shell,
                "cwd": str(self.terminal_cwd),
                "usage": "linux <command> or service.exec kernel.terminal.linux <command>",
            }), "json"
        cd_result = self._maybe_handle_terminal_cd(command)
        if cd_result is not None:
            return cd_result
        return self._run_external_terminal(
            service_id=service_id,
            display_name="Linux terminal",
            executable=[self.linux_shell, "-lc", command],
            command=command,
        )

    def execute_powershell7_terminal(self, command: str) -> tuple[str, str]:
        service_id = "kernel.terminal.powershell7"
        if not self.require_service(service_id):
            return f"Service {service_id} is dormant. Activate it first: service.activate {service_id}", "err"
        if not self.pwsh_executable:
            return "PowerShell 7 service is active, but no pwsh executable was found on this host.", "err"
        command = command.strip()
        if not command:
            return self.format_json({
                "service": service_id,
                "available": True,
                "executable": self.pwsh_executable,
                "cwd": str(self.terminal_cwd),
                "usage": "pwsh <command> or service.exec kernel.terminal.powershell7 <command>",
            }), "json"
        cd_result = self._maybe_handle_terminal_cd(command)
        if cd_result is not None:
            return cd_result
        return self._run_external_terminal(
            service_id=service_id,
            display_name="PowerShell 7",
            executable=[self.pwsh_executable, "-NoLogo", "-NoProfile", "-NonInteractive", "-Command", command],
            command=command,
        )

    def _maybe_handle_terminal_cd(self, command: str) -> tuple[str, str] | None:
        stripped = command.strip()
        lowered = stripped.lower()
        cd_prefixes = ("cd", "chdir", "set-location")
        for prefix in cd_prefixes:
            if lowered == prefix:
                return self.command_terminal_cd("~")
            if lowered.startswith(prefix + " "):
                return self.command_terminal_cd(stripped[len(prefix):].strip())
        return None

    def _run_external_terminal(self, *, service_id: str, display_name: str, executable: list[str], command: str) -> tuple[str, str]:
        svc = self.services[service_id]
        try:
            completed = subprocess.run(
                executable,
                cwd=str(self.terminal_cwd),
                env=os.environ.copy(),
                text=True,
                capture_output=True,
                timeout=self.terminal_timeout_seconds,
            )
        except subprocess.TimeoutExpired as exc:
            self._record_event("terminal.timeout", f"{service_id}: {command}")
            partial_stdout = exc.stdout or ""
            partial_stderr = exc.stderr or ""
            output = [
                f"{display_name} timed out after {self.terminal_timeout_seconds}s",
                f"cwd: {self.terminal_cwd}",
                f"command: {command}",
            ]
            if partial_stdout:
                output.extend(["", "stdout:", partial_stdout.rstrip()])
            if partial_stderr:
                output.extend(["", "stderr:", partial_stderr.rstrip()])
            return "\n".join(output), "err"
        except OSError as exc:
            return f"{display_name} failed to start: {exc}", "err"

        svc["calls"] = int(svc["calls"]) + 1
        self._record_event("terminal.exec", f"{service_id}: {command}")
        stdout = completed.stdout.rstrip()
        stderr = completed.stderr.rstrip()
        lines = [
            f"{display_name} returncode={completed.returncode}",
            f"cwd: {self.terminal_cwd}",
            f"command: {command}",
        ]
        if stdout:
            lines.extend(["", "stdout:", stdout])
        if stderr:
            lines.extend(["", "stderr:", stderr])
        if not stdout and not stderr:
            lines.extend(["", "<no output>"])
        result = "\n".join(lines)
        svc["last_result"] = result[:260]
        return result, "ok" if completed.returncode == 0 else "err"

    def command_manifest_show(self) -> str:
        if not self.require_service("codex.manifest"):
            return "codex.manifest is dormant. Activate it first: service.activate codex.manifest"
        return self.format_json(self.current_project_payload())

    def command_manifest_save(self, path: str) -> str:
        if not self.require_service("kernel.fs"):
            return "kernel.fs is dormant. Activate it first: service.activate kernel.fs"
        if not self.require_service("codex.manifest"):
            return "codex.manifest is dormant. Activate it first: service.activate codex.manifest"
        if not path:
            return "Usage: manifest.save <path>. Console-only mode does not open file dialogs."
        output_path = Path(path).expanduser()
        if output_path.parent and str(output_path.parent) not in {"", "."}:
            output_path.parent.mkdir(parents=True, exist_ok=True)
        with open(output_path, "w", encoding="utf-8") as handle:
            json.dump(self.current_project_payload(), handle, indent=2)
        self._record_event("manifest.save", str(output_path))
        return f"Saved manifest to {output_path}"

    def command_manifest_load(self, path: str) -> tuple[str, str]:
        if not self.require_service("kernel.fs"):
            return "kernel.fs is dormant. Activate it first: service.activate kernel.fs", "err"
        if not self.require_service("codex.manifest"):
            return "codex.manifest is dormant. Activate it first: service.activate codex.manifest", "err"
        if not path:
            return "Usage: manifest.load <path>. Console-only mode does not open file dialogs.", "err"
        input_path = Path(path).expanduser()
        with open(input_path, "r", encoding="utf-8") as handle:
            payload = json.load(handle)
        self.nodes = payload.get("nodes", self.nodes)
        self.files = payload.get("files", self.files)
        self.diagnostics = payload.get("diagnostics", self.diagnostics)
        self.versions = payload.get("version_ledger", self.versions)
        if "manifest" in payload:
            self.manifest = payload["manifest"]
        if "configure4_state" in payload:
            self._restore_configure4_state(payload["configure4_state"])
        elif "configure_4_state" in payload:
            self._restore_configure4_state(payload["configure_4_state"])
        if "quadtreeDesktop_state" in payload:
            self._restore_quadtree_desktop_state(payload["quadtreeDesktop_state"])
        elif "quadtree_desktop_state" in payload:
            self._restore_quadtree_desktop_state(payload["quadtree_desktop_state"])
        self._record_event("manifest.load", str(input_path))
        return f"Loaded manifest from {input_path}", "ok"

    def command_node_list(self) -> str:
        if not self.require_service("codex.graph"):
            return "codex.graph is dormant. Activate it first: service.activate codex.graph"
        return self.format_json(self.nodes)

    def command_node_select(self, node_id: str) -> tuple[str, str]:
        if not self.require_service("codex.graph"):
            return "codex.graph is dormant. Activate it first: service.activate codex.graph", "err"
        if not any(node["id"] == node_id for node in self.nodes):
            return f"Unknown node: {node_id}", "err"
        self.selected_node_id = node_id
        self._record_event("node.select", node_id)
        return f"Selected node {node_id}", "ok"

    def command_node_add(self, payload_raw: str) -> tuple[str, str]:
        if not self.require_service("codex.graph"):
            return "codex.graph is dormant. Activate it first: service.activate codex.graph", "err"
        payload = self.parse_payload(payload_raw)
        if not isinstance(payload, dict) or "id" not in payload:
            return "Usage: node.add {\"id\":\"custom.object\",\"label\":\"Custom Object\",\"type\":\"module\",\"description\":\"...\"}", "err"
        node_id = str(payload["id"])
        if any(node["id"] == node_id for node in self.nodes):
            return f"Node already exists: {node_id}", "err"
        node = {
            "id": node_id,
            "label": str(payload.get("label", node_id)),
            "type": str(payload.get("type", "module")),
            "status": str(payload.get("status", "editable")),
            "description": str(payload.get("description", "Created by blue CLI REPL command.")),
        }
        self.nodes.append(node)
        self.selected_node_id = node_id
        self.diagnostics.append({"severity": "accepted", "target": node_id, "message": "Created through blue CLI node.add command."})
        self._record_event("node.add", node_id)
        return self.format_json(node), "json"

    def command_node_patch(self, target: str) -> tuple[str, str]:
        if not self.require_service("diagnostics.runtime"):
            return "diagnostics.runtime is dormant. Activate it first: service.activate diagnostics.runtime", "err"
        if not self.require_service("codex.graph"):
            return "codex.graph is dormant. Activate it first: service.activate codex.graph", "err"
        patched_any = False
        for node in self.nodes:
            if node["id"] == target or target in {"all", "*"}:
                if node["status"] == "error":
                    node["status"] = "accepted"
                    patched_any = True
        for diag in self.diagnostics:
            if diag["severity"] == "error" and (target in {"all", "*", self.selected_node_id} or target in diag["target"]):
                diag["severity"] = "accepted"
                diag["message"] = "Patched by blue CLI diagnostics runtime. Missing dependency placeholder resolved."
                patched_any = True
        if not patched_any:
            self.diagnostics.append({"severity": "accepted", "target": target, "message": "Patch command ran; no repair item matched."})
        self._record_event("node.patch", target)
        return f"Patch flow completed for {target}.", "ok"

    def command_node_validate(self, target: str) -> tuple[str, str]:
        if not self.require_service("codex.validator"):
            return "codex.validator is dormant. Activate it first: service.activate codex.validator", "err"
        errors = [diag for diag in self.diagnostics if diag["severity"] == "error"]
        warnings = [diag for diag in self.diagnostics if diag["severity"] == "warning"]
        report = {
            "target": target,
            "errors": len(errors),
            "warnings": len(warnings),
            "accepted": len(errors) == 0,
            "gates": self.manifest.get("validation_gates", []),
        }
        if len(errors) == 0:
            for node in self.nodes:
                if target in {"all", "*"} or node["id"] == target:
                    if node["status"] not in {"locked", "accepted"}:
                        node["status"] = "accepted"
            self.diagnostics.append({"severity": "accepted", "target": target, "message": "Validation gates accepted through blue CLI command."})
        self._record_event("node.validate", target)
        return self.format_json(report), "json" if len(errors) == 0 else "warn"

    def command_file_search(self, query: str) -> str:
        if not self.require_service("codex.graph"):
            return "codex.graph is dormant. Activate it first: service.activate codex.graph"
        q = query.strip().lower()
        results = [item for item in self.files if not q or q in f"{item['path']} {item['kind']} {item['state']}".lower()]
        self._record_event("file.search", query)
        return self.format_json(results)

    def command_diagnostics_list(self) -> str:
        if not self.require_service("diagnostics.runtime"):
            return "diagnostics.runtime is dormant. Activate it first: service.activate diagnostics.runtime"
        return self.format_json(self.diagnostics)

    def command_diagnostics_patch(self) -> tuple[str, str]:
        return self.command_node_patch("all")

    def command_emit_preview(self) -> str:
        if not self.require_service("codex.emitter"):
            return "codex.emitter is dormant. Activate it first: service.activate codex.emitter"
        source = self.generate_emitted_source()
        self._record_event("emit.preview", "generated")
        return source

    def command_build_run(self) -> tuple[str, str]:
        if not self.require_service("codex.builder"):
            return "codex.builder is dormant. Activate it first: service.activate codex.builder", "err"
        errors = [diag for diag in self.diagnostics if diag["severity"] == "error"]
        warnings = [diag for diag in self.diagnostics if diag["severity"] == "warning"]
        accepted = len(errors) == 0
        report_lines = [
            "BUILD PASSED" if accepted else "BUILD FAILED",
            "",
            f"objects: {len(self.nodes)}",
            f"warnings: {len(warnings)}",
            f"errors: {len(errors)}",
            f"accepted: {accepted}",
            "",
            "diagnostics:",
        ]
        for diag in self.diagnostics:
            report_lines.append(f"- [{diag['severity'].upper()}] {diag['target']}: {diag['message']}")
        report = "\n".join(report_lines)
        self._record_event("build.run", "passed" if accepted else "failed")
        return report, "ok" if accepted else "warn"

    def command_version_snapshot(self) -> str:
        if not self.require_service("version.ledger"):
            return "version.ledger is dormant. Activate it first: service.activate version.ledger"
        entry = self.create_version_entry()
        return self.format_json(entry)

    def command_reboot(self) -> str:
        self.nodes = copy.deepcopy(INITIAL_CODEX_NODES)
        self.files = copy.deepcopy(INITIAL_FILE_TREE)
        self.diagnostics = copy.deepcopy(INITIAL_DIAGNOSTICS)
        self.manifest = copy.deepcopy(MANIFEST_PREVIEW)
        self.versions = []
        self.vars = {}
        self.event_log = []
        self.configure4_state = self._create_configure4_state()
        self.quadtreeDesktop_state = self._create_quadtree_desktop_state()
        self._hide_quadtree_desktop()
        self.services = self._create_services()
        self.api_routes = self._create_api_routes()
        self._record_event("reboot", "soft reset")
        return "Soft reboot complete. Blue CLI engine remains active; all other services are dormant."

    def api_kernel_status(self, _payload: object) -> dict[str, object]:
        return {
            "os": "Graphical Codex CLI-REPL OS",
            "rule": "services_activate_only_by_blue_cli_repl_command",
            "time": _dt.datetime.now().isoformat(timespec="seconds"),
            "active_services": [sid for sid, svc in self.services.items() if svc["state"] == "active"],
            "selected_service": self.selected_service_id,
            "selected_node": self.selected_node_id,
        }

    def api_kernel_services(self, _payload: object) -> list[dict[str, object]]:
        return [
            {
                "id": sid,
                "layer": svc["layer"],
                "state": svc["state"],
                "description": svc["description"],
                "calls": svc["calls"],
            }
            for sid, svc in self.services.items()
        ]

    def api_kernel_events(self, _payload: object) -> list[dict[str, str]]:
        return self.event_log[-50:]

    def api_terminal_linux(self, payload: object) -> dict[str, object]:
        command = ""
        if isinstance(payload, dict):
            command = str(payload.get("command", payload.get("raw", "")))
        result, tag = self.execute_linux_terminal(command)
        return {
            "ok": tag != "err",
            "service": "kernel.terminal.linux",
            "cwd": str(self.terminal_cwd),
            "result": result,
        }

    def api_terminal_powershell7(self, payload: object) -> dict[str, object]:
        command = ""
        if isinstance(payload, dict):
            command = str(payload.get("command", payload.get("raw", "")))
        result, tag = self.execute_powershell7_terminal(command)
        return {
            "ok": tag != "err",
            "service": "kernel.terminal.powershell7",
            "cwd": str(self.terminal_cwd),
            "result": result,
        }

    def api_codex_manifest(self, _payload: object) -> dict[str, object]:
        return self.current_project_payload()

    def api_codex_nodes(self, _payload: object) -> list[dict[str, str]]:
        return copy.deepcopy(self.nodes)

    def api_codex_files(self, _payload: object) -> list[dict[str, str]]:
        return copy.deepcopy(self.files)

    def api_diagnostics(self, _payload: object) -> list[dict[str, str]]:
        return copy.deepcopy(self.diagnostics)

    def api_validate(self, payload: object) -> dict[str, object]:
        target = "all"
        if isinstance(payload, dict):
            target = str(payload.get("target", "all"))
        result, _tag = self.command_node_validate(target)
        try:
            return json.loads(result)
        except json.JSONDecodeError:
            return {"message": result}

    def api_emit(self, _payload: object) -> dict[str, object]:
        return {"language": "cpp", "source": self.generate_emitted_source()}

    def api_build(self, _payload: object) -> dict[str, object]:
        errors = [diag for diag in self.diagnostics if diag["severity"] == "error"]
        warnings = [diag for diag in self.diagnostics if diag["severity"] == "warning"]
        return {
            "accepted": len(errors) == 0,
            "objects": len(self.nodes),
            "warnings": len(warnings),
            "errors": len(errors),
            "diagnostics": copy.deepcopy(self.diagnostics),
        }

    def api_version_snapshot(self, _payload: object) -> dict[str, str]:
        return self.create_version_entry()

    def api_configure4_status(self, _payload: object) -> dict[str, object]:
        return self._configure4_status_payload()

    def api_configure4_specs(self, payload: object) -> dict[str, object]:
        draft_id = ""
        if isinstance(payload, dict):
            draft_id = str(payload.get("draft_id", payload.get("id", "")))
        if draft_id:
            spec = self._configure4_get_spec(draft_id)
            return {"accepted": spec is not None, "draft": spec.to_dict() if spec else None}
        return {"accepted": True, "drafts": copy.deepcopy(self.configure4_state.get("drafts", {}))}

    def api_configure4_validate(self, payload: object) -> dict[str, object]:
        target = "all"
        if isinstance(payload, dict):
            target = str(payload.get("draft_id", payload.get("target", "all")))
        return self._configure4_validate_target(target, append=True)

    def api_configure4_preview(self, payload: object) -> dict[str, object]:
        target = ""
        if isinstance(payload, dict):
            target = str(payload.get("draft_id", payload.get("target", "")))
        draft_id = self._configure4_default_draft_id(target)
        spec = self._configure4_get_spec(draft_id)
        if spec is None:
            return {"accepted": False, "error": f"Unknown draft: {draft_id or '<none>'}"}
        return self._configure4_build_preview(spec, append_validation=True)

    def api_configure4_register(self, payload: object) -> dict[str, object]:
        target = ""
        if isinstance(payload, dict):
            target = str(payload.get("draft_id", payload.get("target", "")))
        return self._configure4_register_draft(self._configure4_default_draft_id(target))

    def api_quadtree_desktop_status(self, _payload: object) -> dict[str, object]:
        return self._qdt_status_payload()

    def api_quadtree_desktop_modules(self, _payload: object) -> dict[str, object]:
        return {"accepted": True, "modules": copy.deepcopy(self.quadtreeDesktop_state.get("module_registry", {}))}

    def api_quadtree_desktop_module(self, payload: object) -> dict[str, object]:
        module_id = ""
        if isinstance(payload, dict):
            module_id = str(payload.get("module_id", ""))
            route = str(payload.get("_route", ""))
            if not module_id and route.startswith("/quadtreeDesktop/modules/"):
                module_id = route.removeprefix("/quadtreeDesktop/modules/")
        module = self._qdt_get_module(module_id)
        return {"accepted": module is not None, "module": copy.deepcopy(module) if module else None, "module_id": module_id}

    def api_quadtree_desktop_layer(self, payload: object) -> dict[str, object]:
        module_id = ""
        layer_type = ""
        if isinstance(payload, dict):
            module_id = str(payload.get("module_id", ""))
            layer_type = str(payload.get("layer_type", payload.get("layer", "")))
            route = str(payload.get("_route", ""))
            if route.startswith("/quadtreeDesktop/layers/"):
                parts = route.removeprefix("/quadtreeDesktop/layers/").split("/", 1)
                if len(parts) == 2:
                    module_id = module_id or parts[0]
                    layer_type = layer_type or parts[1]
        layer = self._qdt_get_layer(module_id, layer_type)
        return {"accepted": layer is not None, "layer": copy.deepcopy(layer) if layer else None, "module_id": module_id, "layer_type": layer_type}

    def api_quadtree_desktop_cell(self, payload: object) -> dict[str, object]:
        address = ""
        if isinstance(payload, dict):
            address = str(payload.get("address", payload.get("target", "")))
            route = str(payload.get("_route", ""))
            if not address and route.startswith("/quadtreeDesktop/cells/"):
                address = route.removeprefix("/quadtreeDesktop/cells/")
        cell = self._qdt_get_cell_by_ref(address)
        return {"accepted": cell is not None, "cell": copy.deepcopy(cell) if cell else None, "address": address}

    def api_quadtree_desktop_selection(self, payload: object) -> dict[str, object]:
        if isinstance(payload, dict) and any(key in payload for key in ("address", "addresses", "target", "selection")):
            result, _tag = self.command_quadtree_desktop_selection_set(payload)
            return result
        return {"accepted": True, "selected_cell_paths": copy.deepcopy(self.quadtreeDesktop_state.get("selected_cell_paths", []))}

    def api_quadtree_desktop_batches(self, _payload: object) -> dict[str, object]:
        return {"accepted": True, "batches": copy.deepcopy(self.quadtreeDesktop_state.get("batch_registry", {}))}

    def api_quadtree_desktop_validate(self, payload: object) -> dict[str, object]:
        if isinstance(payload, dict) and payload.get("batch_id"):
            result, _tag = self.command_quadtree_desktop_batch_validate(payload)
            return result
        result, _tag = self.command_quadtree_desktop_validate(payload)
        return result

    def api_quadtree_desktop_preview(self, payload: object) -> dict[str, object]:
        result, _tag = self.command_quadtree_desktop_preview(payload)
        return result

    def api_quadtree_desktop_apply(self, payload: object) -> dict[str, object]:
        result, _tag = self.command_quadtree_desktop_apply(payload)
        return result

    def api_quadtree_desktop_export(self, payload: object) -> dict[str, object]:
        result, _tag = self.command_quadtree_desktop_export_manifest(payload)
        return result if isinstance(result, dict) else {"accepted": True, "result": result}

    def api_quadtree_desktop_import(self, payload: object) -> dict[str, object]:
        result, _tag = self.command_quadtree_desktop_import_manifest(payload)
        return result

    def require_service(self, service_id: str) -> bool:
        service_id = self.resolve_service_id(service_id)
        svc = self.services.get(service_id)
        return bool(svc and svc["state"] == "active")

    def parse_payload(self, payload_raw: str) -> object:
        raw = payload_raw.strip()
        if not raw:
            return {}
        if raw.startswith("{") or raw.startswith("["):
            try:
                return json.loads(raw)
            except json.JSONDecodeError:
                return {"raw": raw, "parse_error": "invalid json"}
        return {"raw": raw, "args": shlex.split(raw) if raw else []}

    def current_project_payload(self) -> dict[str, object]:
        return {
            "manifest": copy.deepcopy(self.manifest),
            "nodes": copy.deepcopy(self.nodes),
            "files": copy.deepcopy(self.files),
            "diagnostics": copy.deepcopy(self.diagnostics),
            "version_ledger": copy.deepcopy(self.versions),
            "services": {sid: {key: value for key, value in svc.items() if key != "examples"} for sid, svc in self.services.items()},
            "api_routes": {route: {"service": spec["service"], "description": spec["description"]} for route, spec in self.api_routes.items()},
            "vars": copy.deepcopy(self.vars),
            "configure4_state": copy.deepcopy(self.configure4_state),
            "quadtreeDesktop_state": copy.deepcopy(self.quadtreeDesktop_state),
        }

    def create_version_entry(self) -> dict[str, str]:
        now = _dt.datetime.now()
        accepted = sum(1 for node in self.nodes if node["status"] in {"accepted", "locked"})
        errors = sum(1 for diag in self.diagnostics if diag["severity"] == "error")
        active = sum(1 for svc in self.services.values() if svc["state"] == "active")
        configure4_drafts = len(self.configure4_state.get("drafts", {})) if isinstance(self.configure4_state.get("drafts", {}), dict) else 0
        configure4_registered = len(self.configure4_state.get("registered_runtime_services", {})) if isinstance(self.configure4_state.get("registered_runtime_services", {}), dict) else 0
        qdt_modules = len(self.quadtreeDesktop_state.get("module_registry", {})) if isinstance(self.quadtreeDesktop_state.get("module_registry", {}), dict) else 0
        qdt_batches = len(self.quadtreeDesktop_state.get("batch_registry", {})) if isinstance(self.quadtreeDesktop_state.get("batch_registry", {}), dict) else 0
        entry = {
            "id": f"v{len(self.versions) + 1}.{now.strftime('%Y%m%d.%H%M%S')}",
            "created_at": now.isoformat(timespec="seconds"),
            "summary": f"{accepted}/{len(self.nodes)} codex nodes accepted, {errors} repair item(s), {active}/{len(self.services)} services active, configure4 drafts={configure4_drafts}, registered={configure4_registered}, quadtree modules={qdt_modules}, batches={qdt_batches}",
            "configure4_drafts": str(configure4_drafts),
            "configure4_registered": str(configure4_registered),
            "quadtree_modules": str(qdt_modules),
            "quadtree_batches": str(qdt_batches),
        }
        self.versions.append(entry)
        self._record_event("version.snapshot", entry["id"])
        return entry

    def generate_emitted_source(self) -> str:
        accepted_count = sum(1 for node in self.nodes if node["status"] in {"accepted", "locked"})
        error_count = sum(1 for diag in self.diagnostics if diag["severity"] == "error")
        active_count = sum(1 for svc in self.services.values() if svc["state"] == "active")
        return "\n".join(
            [
                "// Emitted by Graphical Codex CLI-REPL OS",
                "// Kernel services were activated only through the blue CLI engine.",
                "",
                "#include <iostream>",
                "#include <string>",
                "",
                "struct KernelReport {",
                "    int activeServices;",
                "    int acceptedObjects;",
                "    int repairItems;",
                "};",
                "",
                "int main() {",
                f"    KernelReport report{{{active_count}, {accepted_count}, {error_count}}};",
                "    std::cout << \"Graphical Codex CLI-REPL OS emission complete\\n\";",
                "    std::cout << \"Active services: \" << report.activeServices << \"\\n\";",
                "    std::cout << \"Accepted objects: \" << report.acceptedObjects << \"\\n\";",
                "    std::cout << \"Repair items: \" << report.repairItems << \"\\n\";",
                "    return report.repairItems == 0 ? 0 : 1;",
                "}",
            ]
        )

    def _diagnostic_counts(self) -> dict[str, int]:
        counts = {"accepted": 0, "warning": 0, "error": 0}
        for diag in self.diagnostics:
            counts[diag["severity"]] = counts.get(diag["severity"], 0) + 1
        return counts

    def _record_event(self, kind: str, detail: str) -> None:
        self.event_log.append(
            {
                "time": _dt.datetime.now().isoformat(timespec="seconds"),
                "kind": kind,
                "detail": detail,
            }
        )
        if len(self.event_log) > 500:
            self.event_log = self.event_log[-500:]

    def write_console(self, prefix: str, text: str, tag: str = "muted") -> None:
        self.console.configure(state="normal")
        self.console.insert("end", f"{prefix} ", "cmd" if prefix == "BLUE>" else "muted")
        self.console.insert("end", text.rstrip() + "\n\n", tag)
        self.console.see("end")
        self.console.configure(state="disabled")

    def clear_console(self) -> None:
        self.console.configure(state="normal")
        self.console.delete("1.0", "end")
        self.console.configure(state="disabled")

    def show_text_window(self, title: str, text: str, language: str) -> None:
        """No pop-up windows are allowed in console-only mode."""
        self.write_console(title.upper(), text, "muted")

    def copy_text(self, text: str) -> None:
        self.clipboard_clear()
        self.clipboard_append(text)
        self.status_var.set("Copied output to clipboard.")

    def format_json(self, payload: object) -> str:
        return json.dumps(payload, indent=2)

    def card(
        self,
        parent: tk.Widget,
        *,
        bg: str = Theme.panel,
        border: str = Theme.border,
        padx: int = 0,
        pady: int = 0,
    ) -> tk.Frame:
        return tk.Frame(
            parent,
            bg=bg,
            bd=1,
            relief="solid",
            highlightthickness=1,
            highlightbackground=border,
            padx=padx,
            pady=pady,
        )

    def badge(self, parent: tk.Widget, status: str) -> tk.Label:
        bg, fg, border = STATUS_COLORS.get(status, (Theme.card_2, Theme.muted, Theme.border))
        return tk.Label(
            parent,
            text=STATUS_LABELS.get(status, status).upper(),
            bg=bg,
            fg=fg,
            padx=6,
            pady=2,
            font=self.font_micro,
            bd=1,
            relief="solid",
            highlightthickness=1,
            highlightbackground=border,
        )


def main() -> None:
    app = GraphicalCodexCliReplOS()
    app.mainloop()


if __name__ == "__main__":
    main()

```

<a id="file-13"></a>
### [13] `9.py`

- **Bytes:** `51015`
- **Type:** `text`

```python
#!/usr/bin/env python3
"""
9.py — General Dimensional Circuit Composition and Rendering Utility

Purpose
-------
This file is a general-dimensional version of C700h_updated.py.

It keeps the C700h-style pipeline:

    accepted state -> colour-index tensor -> deterministic circuit/category/rendering outputs

and generalizes the dimensional logic using the same kind of fabric reasoning found in
nDCodex.php / 7.php:

    L = m^n, m >= min_root
    h^s compared with p^(L-1)
    nearest valid n-dimensional length
    ranked matching of observed text/code length and alphabet size

The result is a standalone Python utility for composing and rendering n-dimensional
logic fabrics. Qiskit, matplotlib, and Pillow are optional: the script always emits
JSON/text outputs, and emits PNG/Qiskit outputs when the optional dependencies exist.

Typical use
-----------
    python 9.py accept --state 0000001000000200000030000004 --dimension 2
    python 9.py derive --in state.txt --dimension 3 --outdir out9
    python 9.py category --in state.txt --dimension 4 --outdir out9 --assembly
    python 9.py projection --in state.txt --dimension 3 --axis-a 0 --axis-b 2
    python 9.py fabric --p 7 --h 10 --s 5 --dimension 2
    python 9.py match-text --text-file source.txt --dimension 3
"""

from __future__ import annotations

import argparse
import hashlib
import json
import math
import os
import re
import sys
from dataclasses import dataclass
from typing import Any, Iterable, Iterator, Optional, Sequence


# -----------------------------------------------------------------------------
# Constants
# -----------------------------------------------------------------------------

SEGMENT_LEN: int = 7
MAX_COLOR_INDEX: int = 16 ** 6  # 16777216, with 1 -> #000000 and 16^6 -> #ffffff.

DEFAULT_GATES: list[str] = [
    "x",
    "y",
    "z",
    "h",
    "s",
    "sdg",
    "t",
    "tdg",
    "rx",
    "ry",
    "rz",
    "cx",
]

_DEC_RE = re.compile(r"^[0-9]+$")


# -----------------------------------------------------------------------------
# Generic arithmetic / dimensional fabric helpers
# -----------------------------------------------------------------------------


def bounded_pow(base: int, exp: int, limit: int) -> int:
    """Return base**exp, or limit + 1 as soon as the value exceeds limit."""
    if base < 0 or exp < 0:
        raise ValueError("base and exponent must be non-negative")
    result = 1
    for _ in range(exp):
        if base != 0 and result > limit // max(1, base):
            return limit + 1
        result *= base
        if result > limit:
            return limit + 1
    return result


def exact_nth_root(x: int, n: int) -> Optional[int]:
    """Return m when x == m**n, otherwise None."""
    if n <= 0 or x < 0:
        raise ValueError("invalid nth-root arguments")
    if x in (0, 1):
        return x
    lo, hi = 1, x
    while lo <= hi:
        mid = (lo + hi) // 2
        p = bounded_pow(mid, n, x)
        if p == x:
            return mid
        if p < x:
            lo = mid + 1
        else:
            hi = mid - 1
    return None


def nth_root_floor(x: int, n: int) -> int:
    """Return floor(x ** (1/n)) using integer arithmetic."""
    if n <= 0 or x < 0:
        raise ValueError("invalid nth-root arguments")
    if x in (0, 1):
        return x
    lo, hi, best = 1, x, 1
    while lo <= hi:
        mid = (lo + hi) // 2
        p = bounded_pow(mid, n, x)
        if p == x:
            return mid
        if p < x:
            best = mid
            lo = mid + 1
        else:
            hi = mid - 1
    return best


def valid_length_info(length: int, dimension: int, min_root: int) -> dict[str, Any]:
    """Return validity information for L = m^n with m >= min_root."""
    root = exact_nth_root(length, dimension)
    return {"valid": root is not None and root >= min_root, "m": root}


def nearest_valid_length(target: int, dimension: int, min_root: int, lmax: int) -> dict[str, Any]:
    """Find the nearest L = m^n length to target, bounded by lmax."""
    if target < 1 or lmax < 1:
        return {
            "closest_L": None,
            "m": None,
            "gap": None,
            "exact": False,
            "below_L": None,
            "above_L": None,
        }

    exact = valid_length_info(target, dimension, min_root)
    if exact["valid"] and target <= lmax:
        return {
            "closest_L": target,
            "m": exact["m"],
            "gap": 0,
            "exact": True,
            "below_L": target,
            "above_L": target,
        }

    root_floor = nth_root_floor(target, dimension)
    candidates: dict[int, int] = {}
    start = max(min_root, root_floor - 4)
    end = root_floor + 5
    for m in range(start, end + 1):
        L = bounded_pow(m, dimension, max(1, lmax))
        if 1 <= L <= lmax:
            candidates[L] = m

    if not candidates:
        max_root = nth_root_floor(lmax, dimension)
        if max_root >= min_root:
            L = bounded_pow(max_root, dimension, max(1, lmax))
            return {
                "closest_L": L,
                "m": max_root,
                "gap": abs(L - target),
                "exact": False,
                "below_L": L,
                "above_L": None,
            }
        return {
            "closest_L": None,
            "m": None,
            "gap": None,
            "exact": False,
            "below_L": None,
            "above_L": None,
        }

    below = None
    above = None
    bestL = None
    bestM = None
    bestGap = None
    for L in sorted(candidates):
        m = candidates[L]
        if L <= target:
            below = L
        if L >= target and above is None:
            above = L
        gap = abs(L - target)
        if bestGap is None or gap < bestGap or (gap == bestGap and L > int(bestL or 0)):
            bestGap = gap
            bestL = L
            bestM = m

    return {
        "closest_L": bestL,
        "m": bestM,
        "gap": bestGap,
        "exact": False,
        "below_L": below,
        "above_L": above,
    }


def classify_base(p: int) -> str:
    if p < 2:
        return "invalid"
    if p == 2:
        return "prime"
    if p % 2 == 0:
        return "other"
    r = int(math.sqrt(p))
    for k in range(3, r + 1, 2):
        if p % k == 0:
            return "other"
    return "prime"


def centered_integers(center: int, min_value: int, max_value: int, limit: int) -> list[int]:
    out: list[int] = []
    seen: set[int] = set()

    def push(v: int) -> None:
        if v < min_value or v > max_value or v in seen or len(out) >= limit:
            return
        seen.add(v)
        out.append(v)

    push(center)
    d = 1
    while len(out) < limit and (center - d >= min_value or center + d <= max_value):
        push(center - d)
        push(center + d)
        d += 1
    return out


def max_primary_length(h: int, s: int, p: int) -> int:
    """
    Return the smallest L such that p^L >= h^s.

    This mirrors the 7.php fabric comparison style and supplies a finite bound for
    valid primary lengths whose predecessor capacity satisfies p^(L-1) < h^s.
    """
    if h < 2 or s < 1 or p < 2:
        raise ValueError("require h >= 2, s >= 1, p >= 2")
    target = pow(h, s)
    lo, hi = 1, 1
    while pow(p, hi) < target:
        hi *= 2
    while lo < hi:
        mid = (lo + hi) // 2
        if pow(p, mid) >= target:
            hi = mid
        else:
            lo = mid + 1
    return lo


def min_secondary_length_for_primary_length(h: int, p: int, L: int) -> int:
    """Return the minimum s such that h^s > p^(L-1)."""
    if h < 2 or p < 2 or L < 1:
        raise ValueError("require h >= 2, p >= 2, L >= 1")
    threshold = pow(p, L - 1)
    s = 1
    while pow(h, s) <= threshold:
        s += 1
    return s


def valid_lengths_up_to(lmax: int, dimension: int, min_root: int) -> list[dict[str, int]]:
    out: list[dict[str, int]] = []
    if lmax < 1:
        return out
    max_root = nth_root_floor(lmax, dimension)
    for m in range(max(1, min_root), max_root + 1):
        L = m ** dimension
        if L <= lmax:
            out.append({"L": L, "m": m})
    return out


# -----------------------------------------------------------------------------
# Tensor / colour acceptance
# -----------------------------------------------------------------------------


@dataclass(frozen=True)
class DimensionSpec:
    dimension: int
    root: int
    length: int
    shape: tuple[int, ...]

    def to_dict(self) -> dict[str, Any]:
        return {
            "dimension": self.dimension,
            "root": self.root,
            "length": self.length,
            "shape": list(self.shape),
        }


@dataclass(frozen=True)
class AcceptanceReport:
    ok: bool
    reason: str
    spec: Optional[DimensionSpec]
    indexes: list[int]
    hex_colors: list[str]
    segment_length: int
    wrap_adjacency: bool

    def to_dict(self) -> dict[str, Any]:
        return {
            "ok": self.ok,
            "reason": self.reason,
            "dimension": None if self.spec is None else self.spec.dimension,
            "root": None if self.spec is None else self.spec.root,
            "length": len(self.indexes),
            "shape": None if self.spec is None else list(self.spec.shape),
            "segment_length": self.segment_length,
            "wrap_adjacency": self.wrap_adjacency,
            "indexes": self.indexes,
            "hex_colors": self.hex_colors,
        }


def decode_id_to_color_indexes(id_string: str, segment_length: int = SEGMENT_LEN) -> list[int]:
    """Split a decimal string left-to-right into fixed-length integer chunks."""
    out: list[int] = []
    for i in range(0, len(id_string), segment_length):
        seg = id_string[i : i + segment_length]
        try:
            out.append(int(seg, 10))
        except ValueError:
            pass
    return out


def color_index_to_hex(index: int) -> str:
    if not (1 <= index <= MAX_COLOR_INDEX):
        raise ValueError(f"colour index outside [1..{MAX_COLOR_INDEX}]: {index}")
    return "#" + format(index - 1, "06x")


def hex_luma(hex_color: str) -> float:
    hc = hex_color.lstrip("#")
    r = int(hc[0:2], 16) / 255.0
    g = int(hc[2:4], 16) / 255.0
    b = int(hc[4:6], 16) / 255.0
    return 0.2126 * r + 0.7152 * g + 0.0722 * b


def hex_to_rgb01(hex_color: str) -> tuple[float, float, float]:
    hc = hex_color.lstrip("#")
    return (int(hc[0:2], 16) / 255.0, int(hc[2:4], 16) / 255.0, int(hc[4:6], 16) / 255.0)


def index_to_coord(index: int, shape: Sequence[int]) -> tuple[int, ...]:
    """Convert a row-major flat index to an n-dimensional coordinate."""
    coord = [0] * len(shape)
    x = index
    for axis in range(len(shape) - 1, -1, -1):
        size = shape[axis]
        coord[axis] = x % size
        x //= size
    return tuple(coord)


def coord_to_index(coord: Sequence[int], shape: Sequence[int]) -> int:
    """Convert an n-dimensional coordinate to a row-major flat index."""
    if len(coord) != len(shape):
        raise ValueError("coordinate and shape rank mismatch")
    idx = 0
    for c, size in zip(coord, shape):
        if c < 0 or c >= size:
            raise ValueError(f"coordinate {tuple(coord)} outside shape {tuple(shape)}")
        idx = idx * size + c
    return idx


def iter_coords(shape: Sequence[int]) -> Iterator[tuple[int, ...]]:
    if not shape:
        return
    total = math.prod(shape)
    for i in range(total):
        yield index_to_coord(i, shape)


def infer_dimension_spec(token_count: int, dimension: int, min_root: int) -> Optional[DimensionSpec]:
    root = exact_nth_root(token_count, dimension)
    if root is None or root < min_root:
        return None
    return DimensionSpec(dimension=dimension, root=root, length=token_count, shape=tuple([root] * dimension))


def has_nd_adjacent_conflict(indexes: Sequence[int], shape: Sequence[int], wrap: bool = False) -> bool:
    """True when any orthogonal neighbour pair in any axis has equal token value."""
    total = math.prod(shape)
    if len(indexes) != total:
        return True
    for flat, coord in enumerate(iter_coords(shape)):
        cur = indexes[flat]
        for axis, size in enumerate(shape):
            nxt = list(coord)
            if coord[axis] + 1 < size:
                nxt[axis] += 1
            elif wrap and size > 1:
                nxt[axis] = 0
            else:
                continue
            if cur == indexes[coord_to_index(nxt, shape)]:
                return True
    return False


def verify_acceptance(
    id_decimal: str,
    dimension: int,
    min_root: int = 2,
    segment_length: int = SEGMENT_LEN,
    wrap_adjacency: bool = False,
) -> AcceptanceReport:
    indexes = decode_id_to_color_indexes(id_decimal, segment_length=segment_length)
    spec = infer_dimension_spec(len(indexes), dimension, min_root)
    if spec is None:
        return AcceptanceReport(
            ok=False,
            reason=f"Token count {len(indexes)} is not a valid L=m^n with n={dimension} and m>={min_root}.",
            spec=None,
            indexes=list(indexes),
            hex_colors=[],
            segment_length=segment_length,
            wrap_adjacency=wrap_adjacency,
        )

    if not all(1 <= x <= MAX_COLOR_INDEX for x in indexes):
        return AcceptanceReport(
            ok=False,
            reason=f"One or more tokens are outside [1..{MAX_COLOR_INDEX}].",
            spec=spec,
            indexes=list(indexes),
            hex_colors=[],
            segment_length=segment_length,
            wrap_adjacency=wrap_adjacency,
        )

    if has_nd_adjacent_conflict(indexes, spec.shape, wrap=wrap_adjacency):
        return AcceptanceReport(
            ok=False,
            reason="Adjacency conflict: at least one orthogonal n-dimensional neighbour pair is equal.",
            spec=spec,
            indexes=list(indexes),
            hex_colors=[],
            segment_length=segment_length,
            wrap_adjacency=wrap_adjacency,
        )

    hex_colors = [color_index_to_hex(x) for x in indexes]
    return AcceptanceReport(
        ok=True,
        reason="Accepted.",
        spec=spec,
        indexes=list(indexes),
        hex_colors=hex_colors,
        segment_length=segment_length,
        wrap_adjacency=wrap_adjacency,
    )


# -----------------------------------------------------------------------------
# Parsing / text analysis / matching
# -----------------------------------------------------------------------------


def parse_state_to_decimal_string(raw: str) -> str:
    """Accept decimal or 0b-prefixed binary and return canonical decimal text."""
    s = raw.strip()
    if s.lower().startswith("0b"):
        return str(int(s, 2))
    if not _DEC_RE.match(s):
        raise ValueError("State must be a decimal integer string or a 0b-prefixed binary string.")
    return str(int(s)) if s else "0"


def read_text_file(path: str) -> str:
    with open(path, "r", encoding="utf-8") as f:
        return f.read().strip()


def ensure_dir(path: str) -> None:
    os.makedirs(path, exist_ok=True)


def escape_char(ch: str) -> str:
    if ch == "\n":
        return "\\n"
    if ch == "\r":
        return "\\r"
    if ch == "\t":
        return "\\t"
    if ch == " ":
        return "<space>"
    return ch


def analyze_text(text: str) -> dict[str, Any]:
    chars = list(text)
    unique_chars = sorted(set(chars))
    preview = " ".join(escape_char(ch) for ch in unique_chars[:80])
    return {
        "char_length": len(chars),
        "unique_count": len(unique_chars),
        "line_count": 0 if text == "" else text.count("\n") + 1,
        "unique_chars": unique_chars,
        "preview": preview,
    }


def rank_text_matches(
    analysis: dict[str, Any],
    dimension: int,
    min_root: int = 2,
    limit: int = 12,
    primary_min: int = 2,
    primary_max: int = 128,
    h_radius: int = 0,
) -> list[dict[str, Any]]:
    limit = max(1, min(24, int(limit)))
    target_length = max(1, int(analysis["char_length"]))
    observed_unique = max(1, int(analysis["unique_count"]))
    p_candidates = centered_integers(observed_unique, max(2, primary_min), max(primary_min, primary_max), max(24, limit * 4))
    h_candidates = centered_integers(
        observed_unique,
        max(2, observed_unique - h_radius),
        max(2, observed_unique + h_radius),
        max(1, 2 * h_radius + 1),
    ) or [max(2, observed_unique)]

    rows: list[dict[str, Any]] = []
    for h in h_candidates:
        for p in p_candidates:
            s = target_length
            try:
                Lmax = max_primary_length(h, s, p)
            except ValueError:
                continue
            nearest = nearest_valid_length(target_length, dimension, min_root, Lmax)
            if nearest["closest_L"] is None:
                continue
            gap = int(nearest["gap"])
            exact = bool(nearest["exact"])
            h_delta = abs(h - observed_unique)
            p_delta = abs(p - observed_unique)
            score = (-1_000_000 if exact else 0) + gap * 1000 + h_delta * 100 + p_delta
            rows.append(
                {
                    "score": score,
                    "p": p,
                    "h": h,
                    "s": s,
                    "n": dimension,
                    "Lmax": Lmax,
                    "closest_L": int(nearest["closest_L"]),
                    "m": int(nearest["m"]),
                    "gap": gap,
                    "exact": exact,
                    "base_type": classify_base(p),
                }
            )

    rows.sort(key=lambda r: (r["score"], r["gap"], r["h"], r["p"]))
    unique: list[dict[str, Any]] = []
    seen: set[tuple[int, int, int, int]] = set()
    for row in rows:
        key = (row["p"], row["h"], row["s"], row["n"])
        if key in seen:
            continue
        seen.add(key)
        unique.append(row)
        if len(unique) >= limit:
            break
    return unique


# -----------------------------------------------------------------------------
# Dimensional circuit composition
# -----------------------------------------------------------------------------


@dataclass(frozen=True)
class GateEvent:
    layer: int
    coord: tuple[int, ...]
    token: int
    axis: int
    gate: str
    qubits: tuple[int, ...]
    angle: Optional[float] = None

    def to_dict(self) -> dict[str, Any]:
        return {
            "layer": self.layer,
            "coord": list(self.coord),
            "token": self.token,
            "axis": self.axis,
            "gate": self.gate,
            "qubits": list(self.qubits),
            "angle": self.angle,
        }


def token_to_angle(token: int) -> float:
    k = (token % 32) + 1
    return k * (math.pi / 16.0)


def qubit_from_coord(coord: Sequence[int], token: int, q: int) -> int:
    """Deterministic coordinate-sensitive qubit assignment."""
    if q <= 1:
        return 0
    weighted = sum((axis + 1) * (value + 1) for axis, value in enumerate(coord))
    return (weighted + token) % q


def target_from_axis(control: int, axis: int, token: int, q: int) -> int:
    if q <= 1:
        return 0
    shift = 1 + ((token + axis) % (q - 1))
    return (control + shift) % q


def token_to_semantic_symbol(token: int, q: int, axis: int = 0, reversible_only: bool = False) -> str:
    gate = DEFAULT_GATES[token % len(DEFAULT_GATES)]
    if reversible_only:
        gate = "cx" if (token % 2 == 1 and q > 1) else "x"
    if gate in ("rx", "ry", "rz"):
        return f"{gate}(k={(token % 32) + 1})"
    if gate == "cx":
        if q <= 1:
            return "x"
        return f"cx(axis={axis},shift={1 + ((token + axis) % (q - 1))})"
    return gate


def derive_gate_events(
    indexes: Sequence[int],
    spec: DimensionSpec,
    max_qubits: int = 8,
    max_layers: int = 16,
    reversible_only: bool = False,
) -> list[GateEvent]:
    """
    Convert an n-dimensional token tensor into a bounded deterministic gate stream.

    Composition rule:
      - traversal is row-major over the n-dimensional tensor;
      - layer = ordinal // q;
      - source qubit is coordinate-sensitive;
      - cx target depends on the active axis and token value;
      - reversible_only coerces all events into x/cx.
    """
    q = max(1, min(max_qubits, spec.root, len(indexes)))
    max_events = max(1, max_layers) * q
    events: list[GateEvent] = []

    for ordinal, token in enumerate(indexes[:max_events]):
        coord = index_to_coord(ordinal, spec.shape)
        layer = ordinal // q
        axis = ordinal % spec.dimension
        gate = DEFAULT_GATES[token % len(DEFAULT_GATES)]
        if reversible_only:
            gate = "cx" if (token % 2 == 1 and q > 1) else "x"

        control = qubit_from_coord(coord, token, q)
        if gate == "cx" and q > 1:
            target = target_from_axis(control, axis, token, q)
            events.append(GateEvent(layer, coord, int(token), axis, "cx", (control, target), None))
        elif gate == "cx" and q == 1:
            events.append(GateEvent(layer, coord, int(token), axis, "x", (0,), None))
        elif gate in ("rx", "ry", "rz"):
            events.append(GateEvent(layer, coord, int(token), axis, gate, (control,), token_to_angle(int(token))))
        else:
            events.append(GateEvent(layer, coord, int(token), axis, gate, (control,), None))
    return events


def events_to_gate_sequence(events: Sequence[GateEvent]) -> str:
    parts: list[str] = []
    for e in events:
        coord = ",".join(str(x) for x in e.coord)
        prefix = f"L{e.layer}@({coord})/A{e.axis}:"
        if e.gate in ("rx", "ry", "rz"):
            parts.append(f"{prefix}{e.gate}({e.angle},{e.qubits[0]})")
        elif e.gate == "cx":
            parts.append(f"{prefix}cx({e.qubits[0]},{e.qubits[1]})")
        else:
            parts.append(f"{prefix}{e.gate}({e.qubits[0]})")
    return " ".join(parts)


def build_qiskit_circuits(events: Sequence[GateEvent], qubits: int) -> tuple[Any, Any]:
    """Build quantum and classical-shadow Qiskit circuits. Requires qiskit."""
    from qiskit import QuantumCircuit  # type: ignore

    qc = QuantumCircuit(qubits, name="nd_derived_quantum")
    cc = QuantumCircuit(qubits, name="nd_classical_shadow")
    current_layer = -1
    for e in events:
        if current_layer != -1 and e.layer != current_layer:
            qc.barrier()
        current_layer = e.layer

        if e.gate in ("x", "y", "z", "h", "s", "sdg", "t", "tdg"):
            getattr(qc, e.gate)(e.qubits[0])
            if e.gate == "x":
                cc.x(e.qubits[0])
        elif e.gate == "cx":
            qc.cx(e.qubits[0], e.qubits[1])
            cc.cx(e.qubits[0], e.qubits[1])
        elif e.gate in ("rx", "ry", "rz"):
            getattr(qc, e.gate)(float(e.angle or 0.0), e.qubits[0])
        else:
            raise ValueError(f"unsupported gate: {e.gate}")
    return qc, cc


def save_circuit_png(qc: Any, path: str) -> None:
    from qiskit.visualization import circuit_drawer  # type: ignore

    circuit_drawer(qc, output="mpl", filename=path)


# -----------------------------------------------------------------------------
# Semantic free-category derivation
# -----------------------------------------------------------------------------


def derive_semantic_category(
    indexes: Sequence[int],
    hex_colors: Sequence[str],
    spec: DimensionSpec,
    max_cells: int = 512,
    max_qubits: int = 8,
    reversible_only: bool = False,
) -> dict[str, Any]:
    q = max(1, min(max_qubits, spec.root, len(indexes)))
    window = min(len(indexes), max(1, max_cells))
    obj_set: set[str] = set()
    rgb_sum: dict[str, list[float]] = {}
    rgb_n: dict[str, int] = {}
    gen: dict[tuple[str, str, str], dict[str, Any]] = {}

    def add_obj(sym: str, color: str) -> None:
        obj_set.add(sym)
        r, g, b = hex_to_rgb01(color)
        if sym not in rgb_sum:
            rgb_sum[sym] = [0.0, 0.0, 0.0]
            rgb_n[sym] = 0
        rgb_sum[sym][0] += r
        rgb_sum[sym][1] += g
        rgb_sum[sym][2] += b
        rgb_n[sym] += 1

    def add_edge(src: str, dst: str, kind: str, coord: tuple[int, ...]) -> None:
        key = (src, dst, kind)
        if key not in gen:
            gen[key] = {"src": src, "dst": dst, "kind": kind, "count": 0, "examples": []}
        gen[key]["count"] += 1
        if len(gen[key]["examples"]) < 12:
            gen[key]["examples"].append({"at": list(coord)})

    for flat in range(window):
        coord = index_to_coord(flat, spec.shape)
        token = indexes[flat]
        sym = token_to_semantic_symbol(token, q=q, axis=flat % spec.dimension, reversible_only=reversible_only)
        add_obj(sym, hex_colors[flat])

        for axis, size in enumerate(spec.shape):
            nxt = list(coord)
            if coord[axis] + 1 >= size:
                continue
            nxt[axis] += 1
            nflat = coord_to_index(nxt, spec.shape)
            if nflat >= window:
                continue
            ntoken = indexes[nflat]
            nsym = token_to_semantic_symbol(ntoken, q=q, axis=axis, reversible_only=reversible_only)
            add_obj(nsym, hex_colors[nflat])
            add_edge(sym, nsym, f"A{axis}", coord)

    objects = sorted(obj_set)
    obj_color: dict[str, str] = {}
    for oid in objects:
        n = max(1, rgb_n.get(oid, 1))
        rgb = [rgb_sum[oid][i] / n for i in range(3)]
        obj_color[oid] = "#" + "".join(f"{int(max(0, min(255, round(v * 255)))):02x}" for v in rgb)

    generators = []
    for key, payload in gen.items():
        src, dst, kind = key
        count = int(payload["count"])
        generators.append(
            {
                "src": src,
                "dst": dst,
                "kind": kind,
                "count": count,
                "label": f"{kind}×{count}",
                "examples": payload["examples"],
            }
        )
    generators.sort(key=lambda e: (e["src"], e["dst"], e["kind"]))

    return {
        "kind": "nd_semantic_free_category_presentation",
        "objects": [{"id": oid, "color": obj_color[oid]} for oid in objects],
        "generators": generators,
        "composition": "paths in the free category over the n-dimensional adjacency generator graph",
        "identities": "implicit identity morphism for each object",
        "window": {"cells": window, "q": q, **spec.to_dict()},
        "reversible_only": bool(reversible_only),
    }


# -----------------------------------------------------------------------------
# Rendering helpers
# -----------------------------------------------------------------------------


def combine_pngs_side_by_side(left_path: str, right_path: str, out_path: str) -> None:
    from PIL import Image, ImageChops  # type: ignore

    def trim(im: Any, bg_rgb: tuple[int, int, int] = (255, 255, 255), pad: int = 16) -> Any:
        rgb = im.convert("RGB")
        bg = Image.new("RGB", rgb.size, bg_rgb)
        diff = ImageChops.difference(rgb, bg).convert("L")
        diff = diff.point(lambda p: 255 if p > 10 else 0)
        bbox = diff.getbbox()
        if not bbox:
            return im
        x0, y0, x1, y1 = bbox
        return im.crop((max(0, x0 - pad), max(0, y0 - pad), min(im.width, x1 + pad), min(im.height, y1 + pad)))

    a = trim(Image.open(left_path).convert("RGBA"))
    b = trim(Image.open(right_path).convert("RGBA"))
    target_h = max(a.height, b.height)

    def resize_to_h(im: Any, h: int) -> Any:
        if im.height == h:
            return im
        w = int(round(im.width * (h / float(im.height))))
        return im.resize((w, h), resample=Image.Resampling.LANCZOS)

    a2 = resize_to_h(a, target_h)
    b2 = resize_to_h(b, target_h)
    pad = 24
    canvas = Image.new("RGBA", (a2.width + b2.width + pad * 3, target_h + pad * 2), (255, 255, 255, 255))
    canvas.paste(a2, (pad, pad), a2)
    canvas.paste(b2, (pad * 2 + a2.width, pad), b2)
    canvas.convert("RGB").save(out_path)


def render_projection_png(
    indexes: Sequence[int],
    hex_colors: Sequence[str],
    spec: DimensionSpec,
    out_path: str,
    axis_a: int = 0,
    axis_b: int = 1,
    fixed: Optional[dict[int, int]] = None,
    show_labels: bool = False,
) -> None:
    """Render a 2D slice/projection of an n-dimensional tensor."""
    try:
        import matplotlib.pyplot as plt
        from matplotlib.patches import Rectangle
    except ModuleNotFoundError as e:
        raise ModuleNotFoundError("Missing dependency: matplotlib. Install with: pip install matplotlib") from e

    if spec.dimension < 2:
        raise ValueError("projection rendering requires dimension >= 2")
    if axis_a == axis_b or axis_a < 0 or axis_b < 0 or axis_a >= spec.dimension or axis_b >= spec.dimension:
        raise ValueError("axis_a and axis_b must be distinct valid axes")
    fixed = dict(fixed or {})
    for axis in range(spec.dimension):
        if axis not in (axis_a, axis_b):
            fixed.setdefault(axis, 0)

    w = spec.shape[axis_b]
    h = spec.shape[axis_a]
    fig_w = max(4.0, float(w) * 1.05)
    fig_h = max(4.0, float(h) * 1.05)
    fig, ax = plt.subplots(figsize=(fig_w, fig_h))

    for ra in range(h):
        for cb in range(w):
            coord = [0] * spec.dimension
            for axis, value in fixed.items():
                coord[axis] = value
            coord[axis_a] = ra
            coord[axis_b] = cb
            flat = coord_to_index(coord, spec.shape)
            fc = hex_colors[flat]
            ax.add_patch(Rectangle((cb, -ra - 1), 1, 1, facecolor=fc, edgecolor="black", linewidth=1.0))
            if show_labels:
                txt_color = "white" if hex_luma(fc) < 0.45 else "black"
                ax.text(cb + 0.5, -ra - 0.5, str(indexes[flat]), ha="center", va="center", fontsize=6, color=txt_color)

    ax.set_aspect("equal")
    ax.set_xlim(0, w)
    ax.set_ylim(-h, 0)
    ax.set_title(f"projection axes A{axis_a}/A{axis_b}; fixed={fixed}", fontsize=9)
    ax.axis("off")
    fig.tight_layout()
    fig.savefig(out_path, dpi=220, facecolor="white")
    plt.close(fig)


def render_semantic_category_png(category: dict[str, Any], out_path: str, show_edge_labels: bool = True) -> None:
    try:
        import matplotlib.pyplot as plt
        from matplotlib.patches import Circle, FancyArrowPatch
    except ModuleNotFoundError as e:
        raise ModuleNotFoundError("Missing dependency: matplotlib. Install with: pip install matplotlib") from e

    objects = [o["id"] for o in category.get("objects", [])]
    colors = {o["id"]: o.get("color", "#888888") for o in category.get("objects", [])}
    gens = list(category.get("generators", []))
    n = max(1, len(objects))

    pos: dict[str, tuple[float, float]] = {}
    for i, oid in enumerate(objects):
        ang = (2.0 * math.pi * i) / n
        pos[oid] = (math.cos(ang), math.sin(ang))

    def short_label(oid: str) -> str:
        if len(oid) <= 9:
            return oid
        digest = hashlib.sha256(oid.encode("utf-8")).hexdigest()[:4]
        return oid[:5] + "#" + digest

    idx_map = {oid: i for i, oid in enumerate(objects)}
    fig, ax = plt.subplots(figsize=(8.0, 8.0))

    def curve_for(kind: str, src: str, dst: str) -> float:
        if src == dst:
            return 0.35
        axis_num = int(kind[1:]) if kind.startswith("A") and kind[1:].isdigit() else 0
        base = 0.12 + 0.055 * ((axis_num % 5) - 2)
        base += ((idx_map[src] - idx_map[dst]) % 7 - 3) * 0.01
        return base

    for e in gens:
        src = e.get("src")
        dst = e.get("dst")
        if src not in pos or dst not in pos:
            continue
        x0, y0 = pos[src]
        x1, y1 = pos[dst]
        kind = str(e.get("kind", "A0"))
        rad = curve_for(kind, src, dst)
        arrow = FancyArrowPatch(
            (x0, y0),
            (x1, y1),
            arrowstyle="->",
            mutation_scale=12,
            linewidth=1.1,
            color="black",
            connectionstyle=f"arc3,rad={rad}",
        )
        ax.add_patch(arrow)
        if show_edge_labels and int(e.get("count", 1) or 1) > 1:
            mx, my = (x0 + x1) / 2.0, (y0 + y1) / 2.0
            ax.text(mx, my + rad * 0.35, f"{kind}×{int(e.get('count', 1))}", ha="center", va="center", fontsize=8)

    radius = 0.13
    for oid in objects:
        x, y = pos[oid]
        fc = colors.get(oid, "#888888")
        ax.add_patch(Circle((x, y), radius=radius, facecolor=fc, edgecolor="black", linewidth=1.2))
        txt_color = "white" if hex_luma(fc) < 0.45 else "black"
        ax.text(x, y, short_label(oid), ha="center", va="center", fontsize=8, color=txt_color)

    ax.set_aspect("equal")
    ax.set_xlim(-1.35, 1.35)
    ax.set_ylim(-1.35, 1.35)
    ax.axis("off")
    fig.savefig(out_path, dpi=220, bbox_inches="tight", pad_inches=0.2, facecolor="white")
    plt.close(fig)


# -----------------------------------------------------------------------------
# Commands
# -----------------------------------------------------------------------------


def raw_state_from_args(args: argparse.Namespace) -> str:
    if getattr(args, "state", None):
        return str(args.state)
    if getattr(args, "infile", None):
        return read_text_file(str(args.infile))
    raise ValueError("missing --state or --in")


def accepted_report_from_args(args: argparse.Namespace) -> AcceptanceReport:
    raw = raw_state_from_args(args)
    dec = parse_state_to_decimal_string(raw)
    return verify_acceptance(
        dec,
        dimension=int(args.dimension),
        min_root=int(args.min_root),
        segment_length=int(args.segment_len),
        wrap_adjacency=bool(args.wrap),
    )


def cmd_accept(args: argparse.Namespace) -> int:
    report = accepted_report_from_args(args)
    if args.json:
        print(json.dumps(report.to_dict(), indent=2))
    else:
        print(f"Acceptance check: {report.reason}")
        if report.spec:
            print(f"Tensor: shape={report.spec.shape}, L={report.spec.length}, n={report.spec.dimension}, m={report.spec.root}")
        if args.show_indexes:
            print("Indexes:")
            print(" ".join(str(x) for x in report.indexes))
        if args.show_hex and report.hex_colors:
            print("HEX colours:")
            print(" ".join(report.hex_colors))
    return 0 if report.ok else 1


def cmd_derive(args: argparse.Namespace) -> int:
    report = accepted_report_from_args(args)
    if not report.ok or report.spec is None:
        print(f"Acceptance check: {report.reason}", file=sys.stderr)
        return 1
    ensure_dir(args.outdir)
    events = derive_gate_events(
        report.indexes,
        report.spec,
        max_qubits=args.max_qubits,
        max_layers=args.max_layers,
        reversible_only=args.reversible_only,
    )
    q = max(1, min(args.max_qubits, report.spec.root, len(report.indexes)))

    events_path = os.path.join(args.outdir, "nd_gate_events.json")
    seq_path = os.path.join(args.outdir, "nd_gate_sequence.txt")
    manifest_path = os.path.join(args.outdir, "nd_derive_manifest.json")
    with open(events_path, "w", encoding="utf-8") as f:
        json.dump([e.to_dict() for e in events], f, indent=2)
    with open(seq_path, "w", encoding="utf-8") as f:
        f.write(events_to_gate_sequence(events) + "\n")

    outputs = [events_path, seq_path]
    qiskit_status = "not_requested"
    if not args.no_qiskit:
        try:
            qc, cc = build_qiskit_circuits(events, q)
            qasm_path = os.path.join(args.outdir, "nd_quantum_qiskit.txt")
            cseq_path = os.path.join(args.outdir, "nd_classical_shadow_qiskit.txt")
            with open(qasm_path, "w", encoding="utf-8") as f:
                f.write(str(qc) + "\n")
            with open(cseq_path, "w", encoding="utf-8") as f:
                f.write(str(cc) + "\n")
            outputs.extend([qasm_path, cseq_path])
            qiskit_status = "built"
            if not args.no_png:
                qpng = os.path.join(args.outdir, "nd_quantum.png")
                cpng = os.path.join(args.outdir, "nd_classical_shadow.png")
                save_circuit_png(qc, qpng)
                save_circuit_png(cc, cpng)
                outputs.extend([qpng, cpng])
                try:
                    apng = os.path.join(args.outdir, "nd_circuit_assembly.png")
                    combine_pngs_side_by_side(qpng, cpng, apng)
                    outputs.append(apng)
                except Exception:
                    pass
        except ModuleNotFoundError as e:
            qiskit_status = f"missing optional dependency: {e.name}"
        except Exception as e:
            qiskit_status = f"qiskit/render failed: {e}"

    manifest = {
        "accepted": True,
        "spec": report.spec.to_dict(),
        "max_qubits": args.max_qubits,
        "max_layers": args.max_layers,
        "actual_qubits": q,
        "event_count": len(events),
        "reversible_only": bool(args.reversible_only),
        "qiskit_status": qiskit_status,
        "outputs": outputs,
    }
    with open(manifest_path, "w", encoding="utf-8") as f:
        json.dump(manifest, f, indent=2)
    outputs.append(manifest_path)

    print("Wrote:")
    for p in outputs:
        print(f"  {p}")
    return 0


def cmd_category(args: argparse.Namespace) -> int:
    report = accepted_report_from_args(args)
    if not report.ok or report.spec is None:
        print(f"Acceptance check: {report.reason}", file=sys.stderr)
        return 1
    ensure_dir(args.outdir)
    cat = derive_semantic_category(
        report.indexes,
        report.hex_colors,
        report.spec,
        max_cells=args.max_cells,
        max_qubits=args.max_qubits,
        reversible_only=args.reversible_only,
    )
    cat_path = os.path.join(args.outdir, "nd_category.json")
    with open(cat_path, "w", encoding="utf-8") as f:
        json.dump(cat, f, indent=2)
    outputs = [cat_path]

    if not args.no_png:
        try:
            projection_png = os.path.join(args.outdir, "nd_projection.png")
            render_projection_png(
                report.indexes,
                report.hex_colors,
                report.spec,
                projection_png,
                axis_a=args.axis_a,
                axis_b=args.axis_b,
                fixed=parse_fixed_axes(args.fixed),
                show_labels=args.show_labels,
            )
            outputs.append(projection_png)
        except Exception as e:
            print(f"Projection PNG render failed: {e}", file=sys.stderr)
        try:
            cat_png = os.path.join(args.outdir, "nd_category.png")
            render_semantic_category_png(cat, cat_png, show_edge_labels=not args.hide_edge_labels)
            outputs.append(cat_png)
        except Exception as e:
            print(f"Category PNG render failed: {e}", file=sys.stderr)
        if args.assembly:
            try:
                projection_png = os.path.join(args.outdir, "nd_projection.png")
                cat_png = os.path.join(args.outdir, "nd_category.png")
                assembly_png = os.path.join(args.outdir, "nd_category_assembly.png")
                if os.path.exists(projection_png) and os.path.exists(cat_png):
                    combine_pngs_side_by_side(projection_png, cat_png, assembly_png)
                    outputs.append(assembly_png)
            except Exception:
                pass

    manifest_path = os.path.join(args.outdir, "nd_category_manifest.json")
    with open(manifest_path, "w", encoding="utf-8") as f:
        json.dump({"spec": report.spec.to_dict(), "outputs": outputs, "category_kind": cat["kind"]}, f, indent=2)
    outputs.append(manifest_path)

    print("Wrote:")
    for p in outputs:
        print(f"  {p}")
    return 0


def parse_fixed_axes(text: Optional[str]) -> dict[int, int]:
    if not text:
        return {}
    out: dict[int, int] = {}
    for part in text.split(","):
        part = part.strip()
        if not part:
            continue
        if "=" not in part:
            raise ValueError("fixed axes must use axis=value pairs, e.g. 2=0,3=1")
        k, v = part.split("=", 1)
        out[int(k.strip())] = int(v.strip())
    return out


def cmd_projection(args: argparse.Namespace) -> int:
    report = accepted_report_from_args(args)
    if not report.ok or report.spec is None:
        print(f"Acceptance check: {report.reason}", file=sys.stderr)
        return 1
    ensure_dir(args.outdir)
    out_path = os.path.join(args.outdir, args.output)
    render_projection_png(
        report.indexes,
        report.hex_colors,
        report.spec,
        out_path,
        axis_a=args.axis_a,
        axis_b=args.axis_b,
        fixed=parse_fixed_axes(args.fixed),
        show_labels=args.show_labels,
    )
    print(f"Wrote: {out_path}")
    return 0


def cmd_fabric(args: argparse.Namespace) -> int:
    Lmax = max_primary_length(args.h, args.s, args.p)
    rows = valid_lengths_up_to(Lmax, args.dimension, args.min_root)
    nearest = nearest_valid_length(args.target, args.dimension, args.min_root, Lmax) if args.target else None
    payload = {
        "p": args.p,
        "h": args.h,
        "s": args.s,
        "n": args.dimension,
        "min_root": args.min_root,
        "Lmax": Lmax,
        "valid_lengths": rows,
        "target_nearest": nearest,
    }
    if args.json:
        print(json.dumps(payload, indent=2))
    else:
        print(f"p={args.p} h={args.h} s={args.s} n={args.dimension} min_root={args.min_root}")
        print(f"Lmax={Lmax}; valid L=m^n count={len(rows)}")
        if nearest:
            print(f"nearest(target={args.target})={nearest}")
        print("valid lengths:")
        for r in rows[: args.limit]:
            print(f"  L={r['L']} m={r['m']}")
        if len(rows) > args.limit:
            print(f"  ... {len(rows) - args.limit} more")
    return 0


def cmd_match_text(args: argparse.Namespace) -> int:
    if args.text_file:
        text = read_text_file(args.text_file)
    elif args.text is not None:
        text = args.text
    else:
        text = sys.stdin.read()
    analysis = analyze_text(text)
    matches = rank_text_matches(
        analysis,
        dimension=args.dimension,
        min_root=args.min_root,
        limit=args.limit,
        primary_min=args.primary_min,
        primary_max=args.primary_max,
        h_radius=args.h_radius,
    )
    payload = {"analysis": analysis, "matches": matches}
    if args.json:
        print(json.dumps(payload, indent=2, ensure_ascii=False))
    else:
        print(f"Text length={analysis['char_length']} unique={analysis['unique_count']} lines={analysis['line_count']}")
        print(f"Unique preview: {analysis['preview']}")
        print("Ranked matches:")
        for i, row in enumerate(matches, 1):
            print(
                f"  {i:02d}. score={row['score']} p={row['p']}({row['base_type']}) h={row['h']} "
                f"s={row['s']} n={row['n']} Lmax={row['Lmax']} closest_L={row['closest_L']} "
                f"m={row['m']} gap={row['gap']} exact={row['exact']}"
            )
    return 0


def cmd_menu(_: argparse.Namespace) -> int:
    print("\n9.py — General Dimensional Circuit Composition Utility\n")
    while True:
        try:
            mode = input("1=accept 2=derive 3=category 4=fabric 5=exit: ").strip()
        except (EOFError, KeyboardInterrupt):
            print()
            return 0
        if mode == "5":
            return 0
        if mode not in {"1", "2", "3", "4"}:
            print("Unknown mode.\n")
            continue
        if mode == "4":
            p = int(input("primary alphabet p [7]: ").strip() or "7")
            h = int(input("secondary alphabet h [10]: ").strip() or "10")
            s = int(input("secondary length s [5]: ").strip() or "5")
            n = int(input("dimension n [2]: ").strip() or "2")
            ns = argparse.Namespace(p=p, h=h, s=s, dimension=n, min_root=2, target=0, limit=50, json=False)
            cmd_fabric(ns)
            print()
            continue
        raw = input("Accepted state decimal/0b: ").strip()
        n = int(input("dimension n [2]: ").strip() or "2")
        outdir = input("outdir [out9]: ").strip() or "out9"
        base = argparse.Namespace(
            state=raw,
            infile=None,
            dimension=n,
            min_root=2,
            segment_len=SEGMENT_LEN,
            wrap=False,
            outdir=outdir,
        )
        if mode == "1":
            ns = argparse.Namespace(**base.__dict__, json=False, show_indexes=False, show_hex=False)
            cmd_accept(ns)
        elif mode == "2":
            ns = argparse.Namespace(
                **base.__dict__,
                max_qubits=8,
                max_layers=16,
                reversible_only=False,
                no_qiskit=False,
                no_png=False,
            )
            cmd_derive(ns)
        elif mode == "3":
            ns = argparse.Namespace(
                **base.__dict__,
                max_cells=512,
                max_qubits=8,
                reversible_only=False,
                no_png=False,
                axis_a=0,
                axis_b=1,
                fixed=None,
                show_labels=False,
                hide_edge_labels=False,
                assembly=True,
            )
            cmd_category(ns)
        print()


# -----------------------------------------------------------------------------
# CLI wiring
# -----------------------------------------------------------------------------


def add_state_args(parser: argparse.ArgumentParser) -> None:
    parser.add_argument("--state", type=str, default=None, help="State as decimal or 0b... binary")
    parser.add_argument("--in", dest="infile", type=str, default=None, help="Read state from a text file")
    parser.add_argument("--dimension", "-n", type=int, default=2, help="Tensor dimension n in L=m^n")
    parser.add_argument("--min-root", type=int, default=2, help="Minimum tensor root m")
    parser.add_argument("--segment-len", type=int, default=SEGMENT_LEN, help="Decimal segment length for colour tokens")
    parser.add_argument("--wrap", action="store_true", help="Check wrap-around adjacency along every dimension axis")


def build_parser() -> argparse.ArgumentParser:
    p = argparse.ArgumentParser(
        prog="9.py",
        description="General n-dimensional accepted-state -> circuit/category/projection/fabric utility.",
    )
    sub = p.add_subparsers(dest="cmd", required=False)

    pa = sub.add_parser("accept", help="Run general n-dimensional acceptance check")
    add_state_args(pa)
    pa.add_argument("--json", action="store_true", help="Emit JSON report")
    pa.add_argument("--show-indexes", action="store_true", help="Show decoded colour indexes")
    pa.add_argument("--show-hex", action="store_true", help="Show decoded HEX colours")
    pa.set_defaults(func=cmd_accept)

    pd = sub.add_parser("derive", help="Accept + derive n-dimensional circuit event stream and optional Qiskit outputs")
    add_state_args(pd)
    pd.add_argument("--max-qubits", type=int, default=8)
    pd.add_argument("--max-layers", type=int, default=16)
    pd.add_argument("--reversible-only", action="store_true", help="Force x/cx-only exact classical-shadow mapping")
    pd.add_argument("--outdir", type=str, default="out9")
    pd.add_argument("--no-qiskit", action="store_true", help="Skip Qiskit circuit construction")
    pd.add_argument("--no-png", action="store_true", help="Skip PNG rendering")
    pd.set_defaults(func=cmd_derive)

    pc = sub.add_parser("category", help="Accept + derive n-dimensional semantic free-category presentation")
    add_state_args(pc)
    pc.add_argument("--max-cells", type=int, default=512, help="Maximum tensor cells used for category graph")
    pc.add_argument("--max-qubits", type=int, default=8)
    pc.add_argument("--reversible-only", action="store_true")
    pc.add_argument("--outdir", type=str, default="out9")
    pc.add_argument("--no-png", action="store_true")
    pc.add_argument("--axis-a", type=int, default=0)
    pc.add_argument("--axis-b", type=int, default=1)
    pc.add_argument("--fixed", type=str, default=None, help="Fixed coordinates for other axes, e.g. 2=0,3=1")
    pc.add_argument("--show-labels", action="store_true")
    pc.add_argument("--hide-edge-labels", action="store_true")
    pc.add_argument("--assembly", action="store_true")
    pc.set_defaults(func=cmd_category)

    pp = sub.add_parser("projection", help="Render a 2D projection/slice of an accepted n-dimensional tensor")
    add_state_args(pp)
    pp.add_argument("--outdir", type=str, default="out9")
    pp.add_argument("--output", type=str, default="nd_projection.png")
    pp.add_argument("--axis-a", type=int, default=0)
    pp.add_argument("--axis-b", type=int, default=1)
    pp.add_argument("--fixed", type=str, default=None, help="Fixed coordinates for other axes, e.g. 2=0,3=1")
    pp.add_argument("--show-labels", action="store_true")
    pp.set_defaults(func=cmd_projection)

    pf = sub.add_parser("fabric", help="Compute n-dimensional fabric lengths using p/h/s capacity logic")
    pf.add_argument("--p", type=int, default=7, help="Primary alphabet size")
    pf.add_argument("--h", type=int, default=10, help="Secondary alphabet size")
    pf.add_argument("--s", type=int, default=5, help="Secondary length")
    pf.add_argument("--dimension", "-n", type=int, default=2)
    pf.add_argument("--min-root", type=int, default=2)
    pf.add_argument("--target", type=int, default=0, help="Optional target length for nearest valid search")
    pf.add_argument("--limit", type=int, default=80)
    pf.add_argument("--json", action="store_true")
    pf.set_defaults(func=cmd_fabric)

    pm = sub.add_parser("match-text", help="Analyze pasted/file text and rank dimensional fabric configurations")
    pm.add_argument("--text", type=str, default=None)
    pm.add_argument("--text-file", type=str, default=None)
    pm.add_argument("--dimension", "-n", type=int, default=2)
    pm.add_argument("--min-root", type=int, default=2)
    pm.add_argument("--limit", type=int, default=12)
    pm.add_argument("--primary-min", type=int, default=2)
    pm.add_argument("--primary-max", type=int, default=128)
    pm.add_argument("--h-radius", type=int, default=0)
    pm.add_argument("--json", action="store_true")
    pm.set_defaults(func=cmd_match_text)

    pmenu = sub.add_parser("menu", help="Interactive fallback menu")
    pmenu.set_defaults(func=cmd_menu)

    return p


def main(argv: Optional[Sequence[str]] = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)
    if not getattr(args, "cmd", None):
        return cmd_menu(args)
    if args.cmd in {"accept", "derive", "category", "projection"} and not args.state and not args.infile:
        parser.error(f"{args.cmd} requires --state or --in")
    return int(args.func(args))


if __name__ == "__main__":
    raise SystemExit(main())

```

<a id="file-14"></a>
### [14] `dictionary.md`

- **Bytes:** `69519`
- **Type:** `text`

# Codebase Dictionary

Generated for the uploaded code files on 2026-06-08.

Updated on 2026-06-08 to include detailed documentation for `9.py`, `10.py`, `11.py`, and `12.md`.

This dictionary is a static source-code description. The code was inspected as text; the programs were not executed. The purpose of this file is to name each uploaded code file, identify its internal form, describe its functionality, and state its role within the larger generative/configurational system.

## Source File Inventory

| Uploaded file | Lines | Bytes | SHA-256 prefix |
|---|---:|---:|---|
| `7.php` | 1,711 | 67,375 | `b48a2a5dc051` |
| `0.py` | 619 | 19,171 | `e1407673a805` |
| `6.cpp` | 960 | 31,328 | `5251dc2d6d1f` |
| `5.cpp` | 387 | 11,789 | `091267f86223` |
| `4.cpp` | 459 | 14,184 | `4750deee376f` |
| `8.py` | 5,731 | 287,351 | `fc3b2bf774f6` |
| `3.c` | 1,073 | 25,230 | `c6555efc3721` |
| `2.c` | 89 | 2,128 | `1f0ece842890` |
| `1.c` | 120 | 4,669 | `257c114ae65a` |
| `9.py` | 1,451 | 51,015 | `51119c229482` |
| `10.py` | 291 | 9,259 | `4efc142e3cd5` |
| `11.py` | 371 | 12,424 | `f32f17b33496` |
| `12.md` | 74 | 25,361 | `c68924fe4298` |

## System-Level Reading

The uploaded files form six related layers:

1. **Permutation and record-generation layer** — `1.c`, `2.c`, `3.c`, `4.cpp`, `5.cpp`, and `6.cpp` generate combinations, source records, or configurable fabric outputs.
2. **Boolean-function colour encoding layer** — `0.py` gives a canonical truth-table-to-colour system for Boolean logic gates, circuits, and transition machines.
3. **Interactive command/OS layer** — `7.php` and `8.py` are REPL-oriented systems for n-dimensional fabric analysis and graphical kernel-service control.
4. **General-dimensional circuit/category/rendering layer** — `9.py` extends accepted-state colour-index tensors into deterministic n-dimensional circuit events, semantic free-category presentations, projections, fabric reports, and optional rendered outputs.
5. **Documentation bundle and restoration layer** — `10.py` bundles a project tree into `doc/doc.md`, while `11.py` reverses that bundle back into a recoverable file tree where the original text content is embedded.
6. **Axiomatic conceptual-source layer** — `12.md` supplies a long numeric seed and an axiom set named `The Elements`, giving the philosophical/semantic basis for competence, pain, learning, contradiction, correction, tools, memory, and truth.

The naming pattern shows an evolutionary sequence: a direct C permutation generator, an obfuscated xN-style C variant, a configurable C fabric, safer C++ replacements, symbolic-colour logic encoders, n-dimensional fabric/OS tools, reversible documentation-bundle tooling, and a compact axiom file that supplies conceptual constraints for the wider system.

---

# 1. `7.php`

## Internal Identity

- **Declared/internal name:** `nDCodex.php`
- **Language:** PHP 8-style CLI script with strict typing.
- **Primary form:** Command-line REPL with a minimal browser fallback message.
- **Architectural form:** Object-oriented PHP application composed of mathematical utility classes, text-analysis modules, matching logic, fabric traversal, persistence, and export commands.

## Main Purpose

`7.php` implements an n-dimensional hash-length fabric analysis REPL. Its role is to model relationships between alphabets, hash lengths, dimensional lengths, valid n-dimensional tilings, secondary-system capacity, and pasted text/code analysis.

It is a working command environment rather than a simple library. The user runs it from the terminal, changes configuration keys, analyzes primary/secondary length relations, traverses fabric rows, pastes text for character analysis, ranks matching configurations, applies matches, and saves or exports session state.

## Forms

### Source Form

- Single PHP script.
- Uses `declare(strict_types=1)`.
- Refuses direct browser execution as a functional application and instead displays terminal usage instructions.
- Uses classes rather than loose procedural-only code.
- Maintains internal configuration state in the REPL instance.

### Mathematical Form

The file encodes a fabric model involving:

- primary alphabet size `p`;
- secondary alphabet size `h`;
- secondary length `s`;
- active dimension `n`;
- minimum root constraint `min_root`;
- valid primary lengths `L` satisfying `L = m^n` and `m >= min_root`;
- capacity comparisons such as `h^s > p^(L-1)`;
- nearest valid length search;
- ranked matching of observed text/code against possible alphabet/dimension/length configurations.

### Class Form

The major classes are:

- `BigDec` — decimal-string arithmetic for values that can exceed native integer capacity.
- `MathUtil` — bounded exponentiation, exact/floor nth-root logic, length validity, nearest valid length, centered integer generation, primary/secondary length comparison.
- `Fabric` — fabric-row analysis and traversal over valid or candidate lengths.
- `TextAnalyzer` — text/code character analysis, unique-character counting, preview formatting, and escaping.
- `MatchEngine` — ranks candidate configurations against pasted input and session constraints.
- `NDCodex` — the REPL runtime: command parsing, configuration, help/explain system, analysis commands, save/load/export, and prompt loop.

## Functionalities

### REPL Control

The script offers an interactive command loop with grouped command categories:

- **Core:** `help`, `commands`, `explain`, `show`, `set`, `reset`, `quit`, `exit`.
- **Analysis:** `classify`, `lengths`, `fabric traverse`, `witness`.
- **Matching:** `paste`, `match`, `match apply`, `match refine`.
- **Persistence:** `save`, `load`, `export`.

### Configuration Management

The REPL maintains a mutable configuration with keys such as:

- `primary_alphabet`
- `secondary_alphabet`
- `secondary_length`
- `dimension`
- `min_root`
- matching/traversal/export-related settings

The `set` command validates updates so alphabet sizes and dimensional parameters remain within valid ranges.

### Dimensional Length Analysis

The file determines whether a primary length is valid under an n-dimensional shape constraint. A length is valid when it is an exact nth power and the root is at least the configured minimum root.

### Fabric Traversal

The `fabric traverse` command enumerates fabric rows and renders charts/tables showing candidate lengths, gaps, exactness, capacity state, and related metrics.

### Witness Generation

The `witness` command produces concrete explanatory evidence for a selected length or configuration relationship. Its purpose is to make abstract validity/capacity conditions inspectable.

### Paste and Match System

The paste system accepts multiline text/code blocks terminated by `.end`. The analysis then derives character length, unique-character count, character preview, and likely matching configurations. `match apply` can mutate the current session by applying a ranked candidate. `match refine` narrows candidate generation.

### Persistence and Export

The script can save and load session state and export analysis data. This makes it suitable for repeatable investigations rather than one-off calculations.

## Purpose in the Codebase

`7.php` is the analytical core for n-dimensional Codex/fabric reasoning. It transforms alphabet and length parameters into a configurable, inspectable REPL system. It can be used as a bridge between raw text/code inputs and formal dimensional configuration analysis.

## Inputs

- Terminal commands.
- Numeric configuration values.
- Alphabet-size parameters.
- Multiline pasted text/code.
- Saved state files.

## Outputs

- REPL tables and reports.
- Fabric traversal charts.
- Match rankings.
- Export files.
- Saved session/configuration state.

## Practical Role

Use this file when the goal is to reason about symbolic/code text as a dimensional object: characters become observed structure, alphabets become base systems, and valid lengths become n-dimensional constraints.

---

# 2. `0.py`

## Internal Identity

- **Declared/internal name:** Canonical Truth-Table Colour Encoding, abbreviated **CTCE**.
- **Language:** Python 3.
- **Primary form:** Executable Python module/script.
- **Architectural form:** Functional utilities plus dataclass models for Boolean functions, circuits, and transition machines.

## Main Purpose

`0.py` implements a deterministic mapping from a Boolean function's complete truth-table signature to reproducible colour values. It provides a canonical way to represent primitive gates, arbitrary Boolean functions, composed circuits, and bounded sequential transition systems using truth-table encodings and colours.

The file explicitly distinguishes between:

- the **canonical signature/hash**, which identifies the Boolean behaviour;
- the **RGB/HEX colour**, which is reproducible but not globally unique for all large Boolean systems because 24-bit RGB has finite capacity.

## Forms

### Source Form

- Python script with `#!/usr/bin/env python3`.
- Uses `dataclasses`, `hashlib.sha256`, `itertools.product`, and type hints.
- Contains validation utilities, colour conversion utilities, Boolean function models, circuit models, transition models, and demo/printing helpers.

### Boolean Form

The canonical behaviour is:

```text
Boolean behaviour -> canonical truth-table bits -> integer/hash -> colour
```

The canonical input ordering uses lexicographic binary rows:

```text
n = 2: 00, 01, 10, 11
n = 3: 000, 001, 010, 011, 100, 101, 110, 111
```

### Class Form

The main dataclasses/classes are:

- `BooleanFunction` — stores a function name, input count, and truth-table bits; computes IDs, signatures, hashes, direct colours, hash colours, swatches, and table rows.
- `CircuitNode` — describes an input, gate, or constant node in a combinational circuit.
- `CombinationalCircuit` — evaluates directed circuit nodes and encodes multi-output circuit truth tables.
- `TransitionMachine` — encodes bounded sequential state transitions as canonical transition-table bits and colour encodings.

## Functionalities

### Validation

- Converts values to strict Boolean bits.
- Validates that truth-table strings contain only `0` and `1`.
- Infers input count from truth-table length when the length is a power of two.

### Colour Encoding

- Converts integers to RGB triples.
- Converts RGB triples to HEX strings.
- Hashes arbitrary strings/bytes into reproducible RGB values.
- Maps truth bits directly into scaled RGB for small truth tables.
- Maps truth bits into hash-based RGB for large signatures.
- Produces multi-swatch signatures when a single colour is insufficient.

### Boolean Gate Coverage

The file supports:

- constants;
- unary gates;
- all 16 two-input Boolean functions;
- arbitrary n-input functions;
- named primitive gates such as `AND`, `OR`, `XOR`, `NAND`, `NOR`, `XNOR`, projections, implications, and negations.

### Combinational Circuit Composition

The circuit system composes gate nodes and named dependencies. It includes constructors for:

- half-adder;
- full-adder;
- 2-to-1 multiplexer.

Circuit outputs are encoded by concatenating output bits per canonical input row.

### Sequential Transition Encoding

The transition-machine layer represents bounded state machines by encoding:

```text
current state bits + input bits -> next state bits + output bits
```

It includes a T flip-flop example.

### Demo/CLI Behaviour

The `main()` function prints primitive gate tables and example encodings. This makes the file usable as both a module and a demonstrator.

## Purpose in the Codebase

`0.py` is the symbolic-colour layer. It converts complete Boolean behaviour into reproducible visual identifiers. It can be used to colour-code classical logic gates, composed modules, sequential machines, programmable fabrics, and fixed logic devices.

## Inputs

- Boolean callables.
- Truth-table bit strings.
- Named primitive gates.
- Circuit node graphs.
- Transition functions.

## Outputs

- Canonical signatures.
- SHA-256 hashes.
- Function IDs.
- RGB triples.
- HEX colour values.
- Multi-swatch colour signatures.
- Printed primitive/function descriptions.

## Practical Role

Use this file when the aim is to represent logic behaviour as deterministic colour data. It is suitable for diagrams, gate dictionaries, circuit catalogues, visual programming systems, and combinatorial colour encodings.

---

# 3. `6.cpp`

## Internal Identity

- **Declared/internal name:** `configure_3.cpp`
- **Language:** C++17.
- **Primary form:** Command-line generative fabric tool.
- **Architectural form:** Safe C++ replacement/evolution of the C configurator with object specs, config files, menu input, dry-run planning, bounded execution, and escaped output records.

## Main Purpose

`6.cpp` implements a risk-free C++ generative fabric for `cppdb`. It converts user-specified object definitions into bounded, escaped source records. It supports direct command-line flags, configuration files, sample-config generation, and menu/UI-style prompting.

## Forms

### Source Form

- Single C++17 source file.
- Uses anonymous namespace for internal linkage.
- Uses classes for output sinks and file input.
- Uses explicit structs for options and object specifications.
- Uses exceptions for validation and runtime errors.

### Data Form

The main data structures are:

- `Flow` — enum class with `cartesian`, `literal`, `repeat`, and `reverse` modes.
- `ObjectSpec` — runtime object definition including object name, input name, input value, width, flow name, output name, output target, separator, prefix, and suffix.
- `Options` — execution options such as `execute`, `allow_large`, `all`, `overwrite`, `menu`, `start`, `limit`, config path, and sample-config path.
- `Program` — holds global options plus a direct object spec.

### I/O Form

- `OutputSink` abstract interface.
- `StreamSink` for terminal/stdout output.
- `FileSink` for file output with binary append/write modes.
- `InputFile` wrapper for config-file line reading.
- `Writer` wrapper for generic output writing.

## Functionalities

### Input Modes

The file supports exactly one of the major input modes:

- `--config <path>` — load one or more object specs from a config file.
- `--sample-config <path>` — write a sample config file.
- `--menu` or `--ui` — prompt interactively for a spec.
- Direct object flags — define an object from command-line flags.

### Object Flags

Supported object-level flags include:

- `--object-name` / `-n`
- `--input-name`
- `--input-value` / `-i`
- `--input-width` / `-w`
- `--flow-name`
- `--flow-type` / `-f`
- `--output-name`
- `--output-target` / `-o`
- `--separator`
- `--prefix`
- `--suffix`

### Execution Flags

Supported execution/safety flags include:

- `--start`
- `--limit`
- `--all`
- `--allow-large`
- `--overwrite`
- `--execute`

### Flow Generation

The supported generation flows are:

- `cartesian` — generates fixed-width combinations from the input value.
- `literal` — writes the input value once.
- `repeat` — repeats the input value according to input width.
- `reverse` — writes the reversed input value.

### Escaping and Record Safety

The file escapes fields, supports unescaped text inputs such as `\n`, `\t`, and `\r`, validates text length, bounds cartesian width, caps repeat count, and refuses large output unless explicitly allowed.

### Dry Run

Without `--execute`, the tool prints a plan rather than writing records. This makes the generator inspection-first and fail-closed.

## Purpose in the Codebase

`6.cpp` is the most developed C++ configurator in this group. It generalizes earlier fixed permutation generators into a spec-driven record fabric capable of producing stable database-ready records from named object specifications.

## Inputs

- Command-line flags.
- Config files using key/value object specs and `end` delimiters.
- Menu prompts.
- Sample configuration request.

## Outputs

- Escaped `cppdb` source records.
- Sample config files.
- Dry-run plans.
- File or stdout output.

## Practical Role

Use this file as the current C++ fabric generator when configurable named objects, multiple input modes, and safer bounded output are required.

---

# 4. `5.cpp`

## Internal Identity

- **Declared/internal name:** `configure_2.cpp`
- **Language:** C++17.
- **Primary form:** Command-line numeric permutation/source-record generator.
- **Architectural form:** Risk-free C++ replacement for a numeric generator, using sink abstraction, dry-run planning, safety caps, and escaped `cppdb` records.

## Main Purpose

`5.cpp` generates numeric fixed-width records from the digit alphabet `0123456789`. It writes bounded, escaped `cppdb` source records that can serve as stable generated object handles in a CLI/database system.

## Forms

### Source Form

- Single C++17 source file.
- Anonymous namespace for internal implementation.
- Explicit `Options` struct.
- `OutputSink`, `StreamSink`, `FileSink`, and `Writer` classes.
- Top-level parsing, validation, planning, and output functions.

### Data Form

The main configuration fields are:

- `width` — numeric record width, default 7, maximum 18.
- `start` — first ordinal to emit.
- `limit` — maximum records to emit.
- `all` — emit all records after `start`.
- `allow_large` — allow more than 100,000 records.
- `execute` — actually write the output.
- `overwrite` — permit replacing an existing file.
- `output_path` — default `x33.cppdb.txt`.

## Functionalities

### Numeric Combination Generation

The generator treats each ordinal as a base-10 fixed-width number and emits the corresponding digit string. For width 7, it can generate records like numeric handles from `0000000` through `9999999`, bounded by start/limit controls.

### Source Record Construction

Each generated numeric value is wrapped as a `cppdb` source record. The file creates stable textual records rather than raw lines only.

### Safety Controls

The file includes:

- overflow-aware exponentiation;
- max-width validation;
- default output limit;
- refusal to emit more than 100,000 records unless `--allow-large` is used;
- refusal to overwrite output files unless `--overwrite` is used;
- dry-run plan by default unless `--execute` is supplied.

### CLI Options

Supported options include:

- `--width` / `-w`
- `--start`
- `--limit`
- `--all`
- `--allow-large`
- `--output` / `-o`
- `--overwrite`
- `--execute`
- `--help`

## Purpose in the Codebase

`5.cpp` is the numeric-specialized C++ generator. It is narrower than `6.cpp`, but safer and clearer than the earlier C variants. It is useful where generated object handles must be numeric, fixed-width, bounded, escaped, and database-ready.

## Inputs

- Command-line numeric generation parameters.

## Outputs

- Escaped `cppdb` numeric records to file or stdout.
- Dry-run emission plan.

## Practical Role

Use this file for stable numeric handle generation, especially when the intended alphabet is strictly decimal digits.

---

# 5. `4.cpp`

## Internal Identity

- **Declared/internal name:** `configure_1.cpp`
- **Language:** C++17.
- **Primary form:** Command-line broad-character permutation/source-record generator.
- **Architectural form:** Risk-free C++ replacement for the original broad-character permutation generator.

## Main Purpose

`4.cpp` generates fixed-width combinations over configurable alphabets and emits stable escaped `cppdb` source records. It is broader than `5.cpp` because it supports multiple alphabet choices rather than digits only.

## Forms

### Source Form

- Single C++17 source file.
- Anonymous namespace.
- `Options` struct.
- `OutputSink`, `StreamSink`, `FileSink`, and `Writer` classes.
- Functional stages for parsing, alphabet selection, escaping, hashing, planning, validation, and output.

### Alphabet Form

The generator supports named alphabets:

- `original`
- `safe`
- `digits`
- `lower`

The `original` alphabet reflects the broad-character heritage of the earlier C generator. The safe/digits/lower options allow narrower controlled generation.

### Record Form

Generated values are escaped and emitted as source records. The file also computes record hashes, allowing deterministic identity for generated records.

## Functionalities

### CLI Options

Supported options include:

- `--width` / `-w` — record width, default 4, maximum 16.
- `--alphabet` / `-a` — alphabet name.
- `--start`
- `--limit`
- `--all`
- `--allow-large`
- `--output` / `-o`
- `--overwrite`
- `--execute`
- `--help`

### Generation

The program enumerates ordinal positions over the selected alphabet and maps each ordinal to a fixed-width character record.

### Safety Controls

The file uses:

- overflow-aware power computation;
- width limits;
- output count limits;
- dry-run mode by default;
- explicit `--execute` requirement;
- overwrite protection;
- escaping of embedded whitespace/newlines.

## Purpose in the Codebase

`4.cpp` is the first safer C++ generalization of the original permutation idea. It is the broad-character counterpart of the numeric-only `5.cpp` and a predecessor to the object-spec configurator in `6.cpp`.

## Inputs

- Command-line alphabet and generation parameters.

## Outputs

- Escaped `cppdb` source records.
- Dry-run plan.
- File or stdout output.

## Practical Role

Use this file when the required generated records use a configurable alphabet rather than decimal digits only.

---

# 6. `8.py`

## Internal Identity

- **Declared/internal name:** `Graphical Codex CLI-REPL OS - Tkinter Edition`
- **Probable runtime filename from docstring:** `graphical_codex_cli_repl_os.py`
- **Language:** Python 3.
- **Primary form:** Pure-stdlib Tkinter graphical REPL operating surface.
- **Architectural form:** Kernel-service manager, command gateway, API router, codex graph editor, terminal bridge, Configure4 authoring system, and quadtree desktop surface.

## Main Purpose

`8.py` implements a graphical Codex CLI-REPL OS where services are controlled only through commands submitted to the blue CLI engine. The GUI exists as a visible operating surface, but service activation and execution are routed through the REPL command gate.

The file preserves a JSON/Codex data model while turning the application into a kernel-based, API-routable, command-driven system.

## Forms

### Source Form

- Single large Python script.
- Uses only standard-library modules, including `tkinter`, `json`, `hashlib`, `subprocess`, `shlex`, `pathlib`, and `dataclasses`.
- Defines initial codex nodes, file-tree entries, diagnostics, manifest preview, statuses, service IDs, aliases, validation constants, and UI theme values.

### GUI Form

- Tkinter application.
- Dark theme with a blue CLI engine.
- Header, left service panel, central REPL/command panel, right state/API panels, and optional quadtree desktop window/surface.
- Scrollable UI frame support.

### Kernel-Service Form

The service layer includes:

- `blue.cli.engine`
- `kernel.clock`
- `kernel.memory`
- `kernel.terminal.linux`
- `kernel.terminal.powershell7`
- `api.gateway`
- `codex.manifest`
- `codex.graph`
- `codex.validator`
- `codex.emitter`
- `codex.builder`
- `diagnostics.runtime`
- `version.ledger`
- `configure_4.authoring.fabric`
- `quadtree.desktop`

Services remain dormant unless activated through the blue CLI engine, except the core command gate.

### API Form

The API gateway exposes local route handlers such as:

- `/kernel/status`
- `/kernel/services`
- `/kernel/events`
- `/terminal/linux`
- `/terminal/powershell7`
- `/codex/manifest`
- `/codex/nodes`
- `/codex/files`
- `/codex/diagnostics`
- `/codex/validate`
- `/codex/emit`
- `/codex/build`
- `/version/snapshot`
- `/configure4/status`
- `/configure4/specs`
- `/configure4/validate`
- `/configure4/preview`
- `/configure4/register`
- `/quadtreeDesktop/status`
- `/quadtreeDesktop/modules`
- `/quadtreeDesktop/validate`
- `/quadtreeDesktop/preview`
- `/quadtreeDesktop/apply`
- `/quadtreeDesktop/export`
- `/quadtreeDesktop/import`

### Class Form

The major classes are:

- `Configure4FabricRecord` — typed record for objects in a Configure4 fabric graph.
- `Configure4Spec` — service specification draft with identity, command surface, API surface, execution behaviour, validation rules, persistence behaviour, generated code plan, diagnostics, metadata, and object graph.
- `Configure4Sink` and sink subclasses — memory, console, file-preview, and JSON output sinks.
- `Theme` — static colour palette for the UI.
- `ScrollFrame` — scrollable Tkinter container.
- `GraphicalCodexCliReplOS` — main application, service manager, UI renderer, command dispatcher, API gateway, quadtree desktop, Configure4 runtime, terminal bridge, manifest manager, diagnostics, emitter, builder, and utility layer.

## Functionalities

### Blue CLI Command System

The command system supports:

- service listing/status/activation/deactivation/restart/execution;
- API listing/calling/registration;
- codex node listing, selection, addition, patching, and validation;
- file search;
- manifest show/save/load;
- diagnostics listing and patching;
- emission preview;
- build-run simulation/reporting;
- version snapshots;
- Linux and PowerShell command execution through gated terminal services;
- kernel memory commands: `set`, `get`, and `vars`.

### Configure4 Authoring Fabric

`configure_4.authoring.fabric` supports authoring new runtime service specifications. Its commands include:

- `configure4.help`
- `configure4.status`
- `configure4.mode <manual|semi_auto|auto>`
- `configure4.new {json}`
- `configure4.set <path> <value>`
- `configure4.get [path]`
- `configure4.validate [draft-id|all]`
- `configure4.preview [draft-id]`
- `configure4.register [draft-id]`
- `configure4.export <path>`
- `configure4.import <path>`
- `configure4.sample`
- `configure4.reset [draft-id|all]`
- `configure4.history`
- `configure4.list`

It contains manual, semi-automatic, and automatic modes; validation gates; flow bounds; preview generation; runtime command binding; API route binding; service registration; spec hashing; and history tracking.

### Quadtree Desktop

`quadtree.desktop` maps services, routes, manifest objects, diagnostics, Configure4 drafts, batches, and outputs onto a programmable quadtree desktop. It includes commands for:

- showing/hiding the desktop;
- listing, creating, getting, and patching modules;
- getting/patching layers;
- getting, setting, patching, resetting, subdividing, and binding cells;
- setting selections;
- creating, selecting, patching, validating, previewing, applying, and rolling back batches;
- validating, previewing, applying, exporting, importing, and snapshotting desktop state.

### Terminal Bridge

The system can execute Linux shell commands and PowerShell 7 commands through gated services after activation. It tracks current working directory and terminal timeout.

### Codex Project Model

The file models:

- project manifest;
- security policy;
- schema layer;
- module registry;
- C++ emitter;
- diagnostics runtime;
- file manifests;
- build reports;
- validation results.

## Purpose in the Codebase

`8.py` is the graphical operating surface for the Codex system. It turns codebase/project objects into a service-controlled OS-like interface where every meaningful action flows through a command gate and can also be represented through API-style route calls.

## Inputs

- Blue CLI commands.
- JSON payloads.
- Tkinter UI actions that stage or submit CLI-compatible operations.
- Files for manifest/configure import/export.
- Terminal commands for Linux/PowerShell services.

## Outputs

- GUI-rendered service state.
- REPL console text.
- JSON API responses.
- Manifest files.
- Configure4 exported specs.
- Quadtree manifests/images.
- Build/diagnostic/version reports.

## Practical Role

Use this file as the high-level interactive Codex OS. It is the control plane for service activation, codex object management, API routing, Configure4 service authoring, and quadtree-based graphical state management.

---

# 7. `3.c`

## Internal Identity

- **Declared/internal name:** `configurator_v2.c`
- **Language:** C11.
- **Primary form:** User-specifiable C generative intelligence fabric.
- **Architectural form:** xN-style C system preserving object-coded internal identifiers while allowing runtime names and flow parameters to be user-specified.

## Main Purpose

`3.c` generalizes the earlier C permutation generator into a configurable fabric. It allows users to define runtime object names, input names, input values, widths, flow names, flow types, output names, output targets, separators, prefixes, and suffixes through CLI flags, config files, or a micro UI.

## Forms

### Source Form

- Single C11 source file.
- Uses heavily obfuscated/xN-style internal identifiers such as `x3`, `x15`, `x28`, and `x31`.
- Uses explicit fixed-size buffers and C structs.
- Uses manual string parsing, file output, flow dispatch, and CLI handling.

### Data Form

The internal structure maps to the following conceptual objects:

- output sink structure with context and write callbacks;
- flow enum with four modes;
- object spec with runtime names, input value, width, flow, output target, separator, prefix, and suffix;
- fabric/config container holding multiple object specs.

### Flow Form

Supported flow types are:

- `cartesian` — generate every fixed-width combination from `input_value`.
- `literal` — write `input_value` once.
- `repeat` — write `input_value` `input_width` times.
- `reverse` — write `input_value` reversed once.

### Config Form

The config-file form uses key/value records such as:

```text
object_name=x33
input_name=x1
input_value=0123456789
input_width=7
flow_name=x15
flow_type=cartesian
output_name=x31
output_target=x33.txt
separator=\n
end
```

## Functionalities

### CLI Interface

Supported modes and flags include:

- `--ui`
- `--config <path>`
- `--sample-config <path>`
- direct object flags:
  - `--object-name`
  - `--input-name`
  - `--input-value`
  - `--input-width`
  - `--flow-name`
  - `--flow-type`
  - `--output-name`
  - `--output-target`
  - `--separator`
  - `--prefix`
  - `--suffix`
- aliases: `-n`, `-i`, `-w`, `-f`, `-o`.

### Generation

The file can generate object outputs using the selected flow type and write the resulting payloads to files, stdout, or `-`.

### Config Loading

It parses config files with comments, whitespace trimming, key/value pairs, and `end` delimiters for object specs.

### Sample Config

It can emit a sample config file containing both a cartesian digit object and a reverse text example.

### Micro UI

The `--ui` mode prompts the user for object fields interactively.

## Purpose in the Codebase

`3.c` is the bridge between fixed permutation generation and user-specified fabric generation. It is less type-safe and less readable than the C++ replacements, but it carries the important transition into named runtime objects and multiple flow types.

## Inputs

- CLI flags.
- Config files.
- Micro UI prompt responses.

## Outputs

- Generated text/fabric output.
- Sample config files.
- File/stdout records.

## Practical Role

Use this file as a C-level fabric configurator when the runtime object naming style and xN internal naming style are part of the intended system design.

---

# 8. `2.c`

## Internal Identity

- **Declared/internal name:** Not explicitly declared.
- **Language:** C.
- **Primary form:** Compact/obfuscated numeric permutation generator.
- **Architectural form:** xN-named C implementation of an output-sink permutation generator.

## Main Purpose

`2.c` generates all fixed-length combinations over the digit alphabet `0` through `9` and writes them to `x33.txt`. The hard-coded generation length is 7.

## Forms

### Source Form

- Single C source file.
- Uses fixed-width integer types and Boolean status returns.
- Uses obfuscated identifiers (`x0`, `x1`, `x2`, etc.).
- Uses a sink structure with write callbacks.

### Data Form

- `x0` is the maximum permutation length, set to 10.
- `x1` is the digit alphabet.
- `x2` is the alphabet size.
- `x3` is the output sink structure with a file context, buffer writer, and character writer.
- `x30` in `main()` is the hard-coded generation length, set to 7.

## Functionalities

### Safe Power

The function `x10` computes `base^exp` with overflow checking against `UINT64_MAX`.

### Combination Generation

The function `x15` enumerates every ordinal from `0` to `10^7 - 1`, converts it into a fixed-width base-10 digit string, and writes each record followed by a newline.

### File Output

The functions `x23`, `x25`, `x26`, and `x28` implement file writing and file lifecycle control.

## Purpose in the Codebase

`2.c` is a minimal numeric generator in xN-coded form. It appears to be a compact object-coded variant of the clearer safety-critical generator in `1.c`, specialized to digits and a longer fixed width.

## Inputs

- No runtime user input.
- Hard-coded generation length: 7.
- Hard-coded output file: `x33.txt`.

## Outputs

- `x33.txt`, containing every 7-digit combination from the digit alphabet.

## Practical Role

Use this file as a minimal C generator for numeric object-space enumeration. It is not flexible, but it is structurally simple and deterministic.

---

# 9. `1.c`

## Internal Identity

- **Declared/internal name:** `main.c`
- **Description in header:** Permutation Generator in C, architected to safety-critical standards.
- **Language:** C.
- **Primary form:** Clear, safety-oriented fixed permutation generator.
- **Architectural form:** Direct C implementation with configuration constants, output sink abstraction, safe exponentiation, generator core, file sink, and system assembler `main()`.

## Main Purpose

`1.c` generates all possible character combinations for a fixed length using a predefined character set. It writes the combinations to `system_safe.txt`. It is a complete rewrite of an original concept with clearer structure and safety-critical design principles.

## Forms

### Source Form

- Single C source file.
- Uses descriptive identifiers rather than xN obfuscation.
- Contains a structured header comment.
- Divides code into labelled sections:
  1. configuration and data definitions;
  2. abstract output interface;
  3. core generator logic;
  4. file sink implementation;
  5. system assembler.

### Data Form

- `MAX_PERMUTATION_LENGTH` is 10.
- `ALPHABET` contains lowercase letters, uppercase letters, digits, space, and newline.
- `ALPHABET_SIZE` is computed from the array.
- `OutputSink` abstracts output as `write` and `write_char` callbacks.

## Functionalities

### Safe Exponentiation

`safe_uint64_power` computes the number of combinations while checking for unsigned 64-bit overflow.

### Permutation Generation

`generate_permutations` maps each ordinal into a fixed-width representation over the alphabet and writes the result through the output sink.

### File Output

`file_sink_write`, `file_sink_write_char`, `file_sink_init`, and `file_sink_close` manage output to `system_safe.txt`.

### Main Assembly

`main()` sets `permutation_length` to 4, validates it, opens the file sink, generates permutations, closes the sink, and returns success/failure.

## Purpose in the Codebase

`1.c` is the readable baseline generator. It establishes the core design pattern later reused in other files:

```text
alphabet + fixed length + ordinal enumeration + output sink + safety checks -> generated records
```

## Inputs

- No runtime user input.
- Hard-coded alphabet.
- Hard-coded permutation length: 4.
- Hard-coded output file: `system_safe.txt`.

## Outputs

- `system_safe.txt`, containing all generated fixed-length permutations over the defined alphabet.

## Practical Role

Use this file as the reference implementation for the permutation algorithm and output-sink architecture before moving to the compact xN C version or the configurable C/C++ versions.

---


# 10. `9.py`

## Internal Identity

- **Declared/internal name:** `9.py — General Dimensional Circuit Composition and Rendering Utility`.
- **Language:** Python 3.
- **Primary form:** Executable command-line utility with subcommands and an interactive fallback menu.
- **Architectural form:** General-dimensional accepted-state processor that joins colour-index decoding, n-dimensional tensor validation, deterministic circuit derivation, semantic category construction, projection rendering, and fabric-length matching.

## Main Purpose

`9.py` implements a general-dimensional version of a colour-index/circuit pipeline. Its central transformation is:

```text
accepted state -> colour-index tensor -> deterministic circuit/category/rendering outputs
```

The file treats a decimal or binary state as a sequence of fixed-length decimal colour tokens. It verifies that the token count forms a valid n-dimensional tensor length `L = m^n`, checks token ranges against the 24-bit colour-index domain, rejects direct orthogonal adjacency conflicts when enabled by the acceptance rule, converts accepted tokens to HEX colours, and then derives deterministic circuit, category, projection, and fabric-analysis outputs.

## Forms

### Source Form

- Single Python script.
- Uses `argparse` for CLI command dispatch.
- Uses `dataclasses` for typed data records.
- Uses standard libraries for hashing, JSON, arithmetic, filesystem operations, regular expressions, and typing.
- Treats `qiskit`, `matplotlib`, and `Pillow` as optional dependencies.
- Always emits text/JSON outputs for core derivations; PNG and Qiskit outputs are produced only when the optional dependencies are available.

### Dimensional Fabric Form

The file encodes a fabric model involving:

- token count `L`;
- active dimension `n`;
- root `m` where `L = m^n`;
- minimum-root constraint `m >= min_root`;
- nearest valid length search;
- bounded exponentiation for large comparisons;
- primary/secondary capacity comparison via `h^s` and `p^(L-1)`;
- ranked matching of observed text length and observed alphabet size against possible fabric configurations.

### Colour Form

The state is decoded into fixed-width decimal chunks using `SEGMENT_LEN = 7`. Each token is treated as a colour index in the range:

```text
1 <= index <= 16^6
```

The conversion rule is:

```text
index -> HEX colour = #(index - 1) as six hexadecimal digits
```

This means `1` maps to `#000000` and `16^6` maps to `#ffffff`.

### Tensor Form

A valid accepted state must satisfy the following conditions:

- the state is decimal text or `0b`-prefixed binary text;
- the decoded token count is an exact nth power;
- the root is at least the configured `min_root`;
- every decoded token lies inside the colour-index domain;
- optional wrap or non-wrap orthogonal adjacency checking does not find equal neighbouring tokens.

The tensor is traversed in row-major order. Flat indexes are converted to n-dimensional coordinates, and n-dimensional coordinates are converted back to row-major flat indexes.

### Circuit Form

The circuit derivation maps each accepted token into a deterministic gate event. The default gate alphabet is:

```text
x, y, z, h, s, sdg, t, tdg, rx, ry, rz, cx
```

The derivation rule uses:

- row-major tensor traversal;
- a bounded number of qubits;
- a bounded number of layers;
- coordinate-sensitive source-qubit assignment;
- token- and axis-sensitive controlled-target assignment;
- optional `reversible_only` mode that restricts events to `x` and `cx`.

When Qiskit is available, the event stream can be converted into a quantum circuit and a classical-shadow circuit. The classical shadow only mirrors deterministic classical components such as `x` and `cx`.

### Category Form

The semantic category layer derives a free-category presentation from the n-dimensional tensor. Tokens become semantic symbols, adjacent cells become generators, and the category is represented as:

```text
objects + adjacency-labelled generators + implicit identity morphisms + free path composition
```

The output object colours are averaged from the HEX colours associated with the symbols. The generator labels encode active tensor axes such as `A0`, `A1`, and so on.

### Rendering Form

The script can render:

- 2D projections/slices of accepted n-dimensional tensors;
- semantic category diagrams;
- quantum/classical Qiskit circuit PNGs;
- side-by-side assembly PNGs combining circuit, projection, or category renderings.

Rendering uses `matplotlib` and `Pillow` where available.

### Class Form

The principal dataclasses are:

- `DimensionSpec` — active dimension, root, length, and n-dimensional tensor shape.
- `AcceptanceReport` — acceptance status, reason, dimensional spec, decoded indexes, HEX colours, segment length, and wrap-adjacency setting.
- `GateEvent` — layer, coordinate, source token, active axis, gate name, qubit operands, and optional rotation angle.

## Functionalities

### Acceptance Checking

The `accept` command verifies whether a state is a valid n-dimensional colour-index tensor. It can print a human-readable report, a JSON report, decoded indexes, and decoded HEX colours.

### Circuit Derivation

The `derive` command accepts a valid state and writes:

- `nd_gate_events.json`;
- `nd_gate_sequence.txt`;
- `nd_derive_manifest.json`;
- optional Qiskit circuit text outputs;
- optional PNG circuit renderings;
- optional side-by-side circuit assembly image.

### Semantic Category Derivation

The `category` command accepts a valid state and writes:

- `nd_category.json`;
- optional `nd_projection.png`;
- optional `nd_category.png`;
- optional `nd_category_assembly.png`;
- `nd_category_manifest.json`.

### Projection Rendering

The `projection` command renders a 2D slice of an accepted n-dimensional tensor. It supports choosing two visible axes, fixing the remaining axes, and optionally showing token labels.

### Fabric-Length Analysis

The `fabric` command computes valid dimensional lengths under the primary/secondary capacity model. It reports `Lmax`, valid `L = m^n` lengths, and optionally the nearest valid length to a target.

### Text Matching

The `match-text` command analyzes text from a string, file, or standard input. It computes character length, unique-character count, line count, unique-character preview, and ranked fabric-configuration matches.

### Interactive Menu

The `menu` command provides a fallback terminal interface for acceptance checking, derivation, category generation, and fabric analysis.

## Purpose in the Codebase

`9.py` is the bridge between the colour-index acceptance model, n-dimensional tensor structure, deterministic circuit generation, semantic category construction, and visual rendering. It extends the conceptual line from `0.py` and `7.php`: `0.py` maps Boolean behaviour to colours, `7.php` analyzes dimensional alphabet/hash-length fabrics, and `9.py` turns accepted colour-index states into concrete n-dimensional computational and categorical artifacts.

## Inputs

- Decimal accepted-state strings.
- `0b`-prefixed binary state strings.
- Text files containing accepted-state data.
- Text strings or text files for match analysis.
- Fabric parameters `p`, `h`, `s`, `n`, and `min_root`.
- CLI flags controlling dimensions, adjacency, qubits, layers, projections, fixed axes, labels, and output directories.

## Outputs

- Acceptance reports.
- Decoded colour indexes.
- HEX colour values.
- Gate-event JSON.
- Gate-sequence text.
- Qiskit circuit text where available.
- Circuit, projection, category, and assembly PNGs where available.
- Semantic category JSON.
- Fabric-length reports.
- Text-analysis and ranked-match reports.
- Manifest JSON files listing generated outputs.

## Practical Role

Use this file when the goal is to treat an accepted colour-index state as an n-dimensional computational fabric. It is suitable for generating reproducible circuit streams, categorical graph presentations, visual tensor slices, and dimensional matching reports from symbolic states.

---

# 11. `10.py`

## Internal Identity

- **Declared/internal name:** no explicit module docstring; operationally a repository/documentation bundler.
- **Recommended descriptive name:** `bundle_documentation.py`.
- **Language:** Python 3.
- **Primary form:** Executable command-line script.
- **Architectural form:** Static project-tree scanner and Markdown documentation-bundle generator.

## Main Purpose

`10.py` bundles a project directory into a single Markdown document, normally `doc/doc.md`. It scans a root folder for included text files and PNG images, renders a deterministic filesystem tree, generates a table of contents, and embeds the contents of each supported text file in fenced code blocks. PNG images are not embedded as bytes; instead, their dimensions and relative paths are recorded and Markdown image references are emitted.

The core transformation is:

```text
project directory -> included file list -> tree + table of contents + file sections -> doc/doc.md
```

## Forms

### Source Form

- Single Python script.
- Uses only standard-library modules.
- Uses `pathlib.Path` for filesystem paths.
- Uses `os.walk` for recursive traversal.
- Uses regular expressions to build safe fenced code blocks.
- Uses a small PNG IHDR parser to read dimensions without external image libraries.

### Inclusion Form

The bundler includes text files by extension and special extensionless filenames. Included text extensions are:

```text
.css, .cmd, .js, .php, .hpp, .cpp, .md, .py, .txt, .ps1, .html, .h, .json
```

Special included filenames are:

```text
.env, .gitignore, .htaccess
```

PNG files are also included as image references.

### Ignore Form

The scanner prunes ignored directories so that generated artifacts, vendor folders, dependency folders, editor metadata, and runtime folders are not bundled into the output. Ignored directories include:

```text
.git, .svn, .hg, node_modules, vendor, venv, .venv, __pycache__, dist, build, doc_export, .idea, .vscode, doc, .runtime
```

The `doc` directory is ignored so that the generated documentation bundle does not recursively bundle itself.

### Safety Form

The file uses several safety filters:

- binary detection using NUL-byte sampling;
- a maximum text-file byte limit of 5 MB;
- replacement decoding for text files;
- deterministic escaping of fenced code blocks;
- deterministic tree ordering with directories first and files second.

### Markdown Output Form

The generated Markdown contains:

- document title;
- root path;
- generation timestamp;
- included-file count;
- maximum text-file byte setting;
- ignored-directory list;
- ASCII filesystem tree;
- table of contents;
- one section per included file;
- file byte count;
- file type;
- fenced code content for text files;
- PNG dimensions and image reference for PNG files.

## Functionalities

### Path Inclusion

`iter_included_paths()` walks the project root, prunes ignored directories, and yields files whose names/extensions match the inclusion rules.

### Text Reading

`read_text()` reads text files safely. It skips files that exceed the byte limit or appear to be binary and returns a note explaining the skip condition.

### Language Detection

`detect_code_lang()` maps file extensions to Markdown code-fence language identifiers so code blocks render with suitable syntax highlighting where supported.

### Fence Safety

`fenced_block()` inspects the content for runs of backticks and chooses a fence longer than any existing run, preventing embedded code from prematurely closing the Markdown fence.

### PNG Dimension Reading

`png_dimensions()` reads the PNG signature and IHDR chunk directly to extract image width and height without depending on Pillow.

### Tree Building

`build_tree()` and `render_tree()` construct a deterministic ASCII tree from the included relative paths.

### Documentation Bundle Creation

`create_doc_md()` coordinates scanning, tree rendering, table-of-contents generation, text embedding, PNG referencing, and output writing.

### CLI Behaviour

The CLI accepts:

- `--root` — root folder to scan; defaults to the script directory;
- `--out` — output Markdown path; defaults to `<root>/doc/doc.md`.

On success, it prints the created output path.

## Purpose in the Codebase

`10.py` is the forward documentation-bundle tool. It converts a project’s source tree into a portable Markdown artifact that can be read, archived, indexed, reviewed, or later reconstructed by the companion exporter in `11.py`.

## Inputs

- Root directory path.
- Optional output Markdown path.
- Included project text files.
- Included PNG images.

## Outputs

- A generated Markdown documentation bundle, normally `doc/doc.md`.
- Embedded text-file contents inside fenced code blocks.
- PNG metadata and Markdown image references.
- Filesystem tree and table of contents.

## Practical Role

Use this file when the goal is to preserve or transmit a codebase as one Markdown document. It is suited to review, documentation, model ingestion, archival, and later round-trip export.

---

# 12. `11.py`

## Internal Identity

- **Declared/internal name:** no explicit module docstring; CLI error text identifies the tool as `export-doc-md.py`.
- **Recommended descriptive name:** `export-doc-md.py`.
- **Language:** Python 3.
- **Primary form:** Executable command-line script.
- **Architectural form:** Reverse documentation-bundle exporter that rebuilds a file tree from a `doc.md` bundle created by `10.py`.

## Main Purpose

`11.py` reconstructs a project file tree from a bundled Markdown document. It parses file sections, extracts embedded text from fenced code blocks, writes text files back to their original relative paths, and attempts to recover PNG assets from linked paths or an explicitly provided asset root.

The core transformation is:

```text
doc.md documentation bundle -> parsed file sections -> safe relative paths -> reconstructed file tree
```

## Forms

### Source Form

- Single Python script.
- Uses only standard-library modules.
- Uses compiled regular expressions to identify bundled file sections, file types, notes, and PNG paths.
- Uses dataclasses to represent parsed sections and exported files.
- Uses `pathlib.Path` and `shutil` for filesystem restoration.
- Uses explicit error handling and status reporting.

### Bundle Parsing Form

The parser expects file sections in the form emitted by `10.py`:

```text
<a id="file-N"></a>
### [N] `relative/path`
```

Each section is parsed into:

- numeric index;
- relative path;
- inferred file type;
- raw Markdown section body.

### Text-Recovery Form

For text files, the exporter finds the first fenced code block in the section. Because the bundler chooses a fence longer than any backtick run in the content, the exporter can safely identify the closing fence and remove exactly one sentinel newline before writing the recovered text.

If a section lacks a fenced block, the exporter writes an empty file and optionally records a warning when the original content was skipped.

### PNG-Recovery Form

PNG bytes are not embedded in the Markdown bundle. The exporter therefore attempts to copy PNG files from:

1. the linked path relative to the `doc.md` location;
2. `--asset-root` plus the original relative path, when an asset root is provided.

If neither source exists, the exporter reports a `missing-binary` status.

### Path-Safety Form

The exporter rejects unsafe paths before writing:

- empty paths;
- POSIX absolute paths;
- Windows drive paths;
- UNC-style paths;
- parent-directory segments such as `..`.

This prevents an edited or malicious `doc.md` from writing outside the selected export directory.

### Data Class Form

The principal dataclasses are:

- `ExportedFile` — output record containing file index, relative path, file type, target path, status, and optional warning.
- `ParsedSection` — parsed bundle section containing file index, relative path, file type, and section body.

## Functionalities

### Bundle Section Parsing

`parse_bundle_doc()` splits the Markdown bundle into ordered file sections and identifies each section’s declared file type.

### Text Extraction

`_extract_text_content()` recovers embedded text from fenced code blocks and returns both the recovered content and any warning.

### PNG Copying

`_try_copy_png()` attempts to recover image files from the documentation-relative linked path or from the original project root supplied through `--asset-root`.

### Export Execution

`export_doc_md()` coordinates validation, optional output cleaning, section parsing, text writing, PNG copying, warning collection, and optional strict-mode failure.

### CLI Behaviour

The CLI accepts:

- `--doc` — path to the bundled Markdown file; defaults to `./doc/doc.md` if present, otherwise `./doc.md`;
- `--out` — target export directory; defaults to `./doc_export`;
- `--asset-root` — optional original project root for PNG recovery;
- `--clean` — delete the target export directory before rebuilding it;
- `--strict` — return a non-zero exit code if any file cannot be fully recovered.

The CLI prints a report containing counts for written text files, copied assets, missing assets, skipped unsafe paths, and warnings.

## Purpose in the Codebase

`11.py` is the reverse partner of `10.py`. Together, the two scripts form a Markdown round-trip system:

```text
source tree -> doc/doc.md -> reconstructed source tree
```

Text files can be round-tripped because their bytes are embedded as fenced code content. PNG files require the original linked asset or an asset root because they are referenced rather than embedded.

## Inputs

- Bundled `doc.md` file.
- Optional target output directory.
- Optional asset-root directory for recovering PNG files.
- Optional clean and strict flags.

## Outputs

- Reconstructed text files in the target export directory.
- Copied PNG files when recoverable.
- Warnings for skipped, missing, or unrecoverable content.
- Terminal export report.

## Practical Role

Use this file when the goal is to restore a codebase from a Markdown documentation bundle. It is especially useful as a companion to `10.py`, enabling reversible documentation artifacts for text-heavy projects.

---

# 13. `12.md`

## Internal Identity

- **Declared/internal name:** `The Elements`.
- **Authorship line:** `Dominic Alexander Cooper`.
- **Language/form:** Markdown/plain-text conceptual document.
- **Primary form:** Numeric-seed plus numbered axiom list.
- **Architectural form:** Axiomatic conceptual-source file for a competence, learning, correction, memory, tool, and truth framework.

## Main Purpose

`12.md` defines a compact axiom system named `The Elements`. It begins with a very large decimal integer, then presents axioms numbered `000` through `065`. The document functions as a conceptual source file rather than executable code.

Its core structure is:

```text
long decimal seed + title + author line + axioms 000..065
```

The axioms define a human-centric competence system in which existence, pain, competence, incompetence, learning, contradiction, correction, tools, memory, context, and truth are treated as formalizable system components.

## Forms

### Numeric Seed Form

The first line is a long decimal integer. Within the larger codebase, this can be treated as a seed, identifier, entropy source, system-state marker, or canonical numeric handle. The file itself does not execute this integer; it stores it as textual data.

### Title and Authorship Form

The document title is:

```text
The Elements
```

The named author line is:

```text
Dominic Alexander Cooper
```

### Axiom Form

The axiom list runs from `000` to `065`. Each axiom is represented as:

```text
NNN    statement
```

The list uses zero-padded numeric labels, making the statements directly addressable and suitable for indexing, reference, parsing, validation, or incorporation into a larger symbolic system.

### Foundational Axiom Group

Axioms `000` to `016` establish the base vocabulary:

- existence;
- pain;
- competence;
- incompetence;
- learning;
- contradiction;
- proof by construction;
- proof by contradiction as non-preferred for learning;
- quantifiability of competence and incompetence;
- context-indexed competence;
- innocence;
- competence as functional axial n-dimensional space.

### Derived System Axiom Group

Axioms `017` to `032` restate and derive system-level consequences from the first group. These axioms treat the person as a living information-bearing system, pain as a valid signal, incompetence as a negative state, competence as a positive state, learning as a valid update operation, contradiction as a corruption signal, construction as the preferred proof engine, and correction as non-guilt.

### Mental-Database Axiom Group

Axioms `033` to `063` define the mental human-centric database. The mind ingests, encodes, indexes, queries, validates, rejects, and externalizes information. The environment, tools, notes, diagrams, calendars, checklists, questions, answers, mistakes, correction, and optimization become database components or operations.

### Truth Closure Axiom Group

Axioms `064` and `065` give the closure rules:

- once true, always true;
- what is not true cannot be done.

These provide an absolute truth constraint at the end of the axiom list.

## Functionalities

Although `12.md` is not executable code, it performs several system functions:

- defines stable numbered conceptual records;
- supplies a seed-like numeric identifier;
- gives a competence/incompetence ontology;
- specifies learning as constructive update;
- treats contradiction as a corruption/incompetence signal;
- frames correction as competence restoration rather than guilt;
- models mind, body, environment, and tools as one extended database;
- gives a terminal target of complete competence under innocence and functional dimensional competence-space;
- supplies truth-preservation constraints for downstream systems.

## Purpose in the Codebase

`12.md` is the philosophical and axiomatic substrate for the wider generative/configurational system. Where the C/C++ files generate records, `0.py` maps Boolean behaviour to colours, `7.php` and `9.py` analyze dimensional fabrics, and `8.py` provides a command-gated operating surface, `12.md` supplies the conceptual rules that define what the system is trying to preserve: competence, constructive learning, correction, innocence, and truth.

## Inputs

- No runtime input.
- Static decimal seed.
- Static axiom list.

## Outputs

- A numbered axiom set.
- A conceptual source for parser/indexer/database use.
- A stable reference list for competence, correction, learning, and truth constraints.

## Practical Role

Use this file as the formal conceptual dictionary for systems that need to align with the `The Elements` axiom set. It can be parsed into a database, referenced in documentation, used as validation text, or treated as an axiomatic constraint layer for later Codex, OS, fabric, and learning-system modules.

---

# Cross-File Evolution Map

## Stage 1 — Clear C Baseline

`1.c` defines the readable algorithm: a fixed alphabet, safe power calculation, output-sink abstraction, and fixed-length permutation generation.

## Stage 2 — xN Numeric C Variant

`2.c` compresses the concept into xN-style naming and specializes the alphabet to decimal digits with hard-coded width 7.

## Stage 3 — User-Specified C Fabric

`3.c` adds runtime object names, flow names, input/output names, config files, CLI flags, micro UI, sample config, and multiple flow types.

## Stage 4 — Safer C++ Broad-Character Generator

`4.cpp` recreates the broad-character generator in safer C++ with dry-run planning, overwrite protection, count limits, escaped records, and configurable alphabet choices.

## Stage 5 — Safer C++ Numeric Generator

`5.cpp` specializes the safer C++ approach to decimal numeric records, giving stable digit-only generated handles.

## Stage 6 — Object-Spec C++ Fabric

`6.cpp` generalizes into named object specs, config/menu/direct input modes, multiple output targets, flow dispatch, sample configs, and bounded escaped `cppdb` records.

## Stage 7 — Logic Colour Encoding

`0.py` gives a canonical behavioural representation layer for Boolean gates, circuits, and transition machines by mapping truth-table behaviour to signatures and colours.

## Stage 8 — n-Dimensional Fabric REPL

`7.php` analyzes symbolic/textual structures as alphabet-length-dimensional fabrics and provides matching/persistence/export through a CLI REPL.

## Stage 9 — Graphical Codex REPL OS

`8.py` creates the graphical service-controlled operating surface that can host command, API, Codex, Configure4, terminal, versioning, diagnostic, and quadtree-desktop behaviours.

## Stage 10 — General-Dimensional Circuit/Category Utility

`9.py` turns accepted colour-index states into n-dimensional tensors, deterministic gate events, semantic free-category presentations, projection renders, optional Qiskit circuit artifacts, and dimensional fabric reports.

## Stage 11 — Documentation Bundle Generator

`10.py` converts a project tree into a single Markdown documentation bundle with an included filesystem tree, table of contents, file metadata, embedded text content, and linked PNG image references.

## Stage 12 — Documentation Bundle Exporter

`11.py` reverses the documentation bundle by reconstructing text files from fenced code blocks and recovering PNG assets from linked paths or an original asset root.

## Stage 13 — Axiomatic Source Layer

`12.md` provides `The Elements`: a numeric seed plus axioms `000` through `065`, defining the conceptual basis for competence, learning, correction, innocence, tools, memory, and truth.

---

# Functional Classification Table

| File | Category | Main Functionality | Main Purpose |
|---|---|---|---|
| `1.c` | Baseline C generator | Generate fixed-length permutations from broad alphabet | Establish readable safe permutation architecture |
| `2.c` | xN C generator | Generate 7-digit numeric combinations | Compact deterministic numeric enumeration |
| `3.c` | Configurable C fabric | Generate user-specified cartesian/literal/repeat/reverse outputs | Move from fixed generation to runtime-configurable fabric |
| `4.cpp` | C++ broad generator | Generate escaped records over selected alphabets | Safer C++ replacement for broad-character generation |
| `5.cpp` | C++ numeric generator | Generate escaped numeric `cppdb` records | Stable numeric handle/source-record generation |
| `6.cpp` | C++ object-spec fabric | Generate records from named object specs and config/menu/direct input | Current safer configurable fabric generator |
| `0.py` | Boolean colour encoder | Encode Boolean truth tables/circuits/state machines to colours | Canonical visual representation of logic behaviour |
| `7.php` | nD analysis REPL | Analyze alphabet lengths, dimensions, pasted text, and config matches | Interactive n-dimensional hash-length fabric analysis |
| `8.py` | Graphical REPL OS | Control services, API routes, Codex graph, Configure4, terminals, quadtree desktop | Full graphical command-gated Codex operating surface |
| `9.py` | nD circuit/category utility | Validate accepted states, derive gate events, render projections/categories, analyze fabric lengths | Convert colour-index tensors into computational/categorical artifacts |
| `10.py` | Documentation bundler | Bundle a source tree into `doc/doc.md` with tree, TOC, text contents, and PNG references | Portable Markdown documentation and archival artifact |
| `11.py` | Documentation exporter | Rebuild files from a bundled `doc.md`, including text files and recoverable PNGs | Round-trip restoration of a documentation bundle |
| `12.md` | Axiomatic source document | Store numeric seed and axioms `000` to `065` | Conceptual substrate for competence, learning, correction, and truth constraints |

---

# Dependency and Runtime Notes

## Standard Dependencies

- The C files use standard C headers such as `stdio.h`, `stdint.h`, `stdbool.h`, `limits.h`, `string.h`, `stdlib.h`, and `ctype.h`.
- The C++ files use standard C++ headers such as `iostream`, `fstream`, `sstream`, `string`, `string_view`, `vector`, `map`, `limits`, `stdexcept`, `iomanip`, `cstdio`, and `cstdint`.
- `0.py` uses only standard Python libraries.
- `10.py` and `11.py` use only standard Python libraries.
- `8.py` uses Tkinter and standard Python libraries; Tkinter must be available in the Python installation.
- `7.php` requires PHP CLI for full REPL behaviour.
- `9.py` uses standard Python libraries for core JSON/text outputs and optionally uses `qiskit`, `matplotlib`, and `Pillow` for circuit construction and PNG rendering.
- `12.md` has no runtime dependency.

## Build/Run Summary

```bash
# C baseline
cc -std=c11 -Wall -Wextra -pedantic -O2 1.c -o main

# xN numeric C generator
cc -std=c11 -Wall -Wextra -pedantic -O2 2.c -o generator_xn

# C configurable fabric
cc -std=c11 -Wall -Wextra -pedantic -O2 3.c -o configurator_v2

# C++ generators/fabrics
c++ -std=c++17 -Wall -Wextra -pedantic -O2 4.cpp -o configure_1
c++ -std=c++17 -Wall -Wextra -pedantic -O2 5.cpp -o configure_2
c++ -std=c++17 -Wall -Wextra -pedantic -O2 6.cpp -o configure_3

# Python CTCE
python 0.py

# PHP nDCodex REPL
php 7.php

# Tkinter Graphical Codex CLI-REPL OS
python 8.py

# General n-dimensional circuit/category/projection/fabric utility
python 9.py menu
python 9.py accept --state 0000001000000200000030000004 --dimension 2
python 9.py derive --in state.txt --dimension 3 --outdir out9
python 9.py category --in state.txt --dimension 4 --outdir out9 --assembly
python 9.py projection --in state.txt --dimension 3 --axis-a 0 --axis-b 2
python 9.py fabric --p 7 --h 10 --s 5 --dimension 2
python 9.py match-text --text-file source.txt --dimension 3

# Documentation bundle generator
python 10.py --root . --out doc/doc.md

# Documentation bundle exporter
python 11.py --doc doc/doc.md --out doc_export --asset-root .

# Axiomatic Markdown source
# 12.md is read, parsed, indexed, or referenced as a static document.
```

---

# Design Pattern Summary

The codebase repeatedly applies the same general pattern:

```text
bounded symbolic input space
    -> deterministic enumeration or analysis
    -> safe validation gates
    -> canonical record/signature/output form
    -> file, REPL, colour, API, GUI, render, bundle, export, or axiom representation
```

In the C/C++ layer, the symbolic input space is usually an alphabet and width. In the Python CTCE layer, it is Boolean truth-table behaviour. In the PHP nDCodex layer, it is text/code plus alphabet/dimension/hash-length configuration. In the Tkinter OS layer, it is services, commands, API routes, manifests, quadtree cells, and configurable runtime specs. In `9.py`, it is accepted colour-index state as an n-dimensional tensor that becomes circuits, categories, projections, and fabric reports. In `10.py` and `11.py`, it is a source tree transformed into a reversible Markdown bundle. In `12.md`, it is an axiom-indexed conceptual source that constrains the meaning of competence, correction, learning, and truth.

---

# Recommended Naming Map

The uploaded numeric filenames can be mapped to descriptive project names as follows:

| Uploaded file | Recommended descriptive name |
|---|---|
| `1.c` | `main_permutation_generator.c` |
| `2.c` | `numeric_xn_permutation_generator.c` |
| `3.c` | `configurator_v2.c` |
| `4.cpp` | `configure_1.cpp` |
| `5.cpp` | `configure_2.cpp` |
| `6.cpp` | `configure_3.cpp` |
| `0.py` | `canonical_truth_table_colour_encoding.py` |
| `7.php` | `nDCodex.php` |
| `8.py` | `graphical_codex_cli_repl_os.py` |
| `9.py` | `general_dimensional_circuit_composition_rendering.py` |
| `10.py` | `bundle_documentation.py` |
| `11.py` | `export-doc-md.py` |
| `12.md` | `the_elements.md` |

---

# Maintenance Notes

1. Keep `1.c` as the readable algorithmic baseline.
2. Treat `2.c` and `3.c` as xN-style experimental/system-code forms.
3. Prefer `4.cpp`, `5.cpp`, and `6.cpp` for safer production-style generation because they include dry-run planning, explicit execution, and overwrite protection.
4. Prefer `6.cpp` over `3.c` when object-spec configurability is needed with clearer safety boundaries.
5. Use `0.py` for logic/gate/circuit visual identity systems.
6. Use `7.php` for dimensional alphabet-length/text-analysis REPL work.
7. Use `8.py` as the graphical command-gated operating surface and service/API control plane.
8. Use `9.py` when accepted colour-index states need to become tensor projections, circuit events, semantic categories, Qiskit artifacts, or fabric-length reports.
9. Use `10.py` before external review, archiving, or model ingestion when a project should be bundled into a single Markdown file.
10. Use `11.py` after a documentation bundle needs to be restored into a runnable source tree; provide `--asset-root` when PNG files must be recovered.
11. Use `12.md` as the conceptual axiom source for systems that must remain aligned with `The Elements`.
12. Keep the `10.py` and `11.py` bundle/export formats synchronized. If the section heading format, file metadata fields, or code-fence rules change in `10.py`, update the parser assumptions in `11.py`.
13. Keep optional-render dependencies for `9.py` separate from its core text/JSON behaviour so the file remains usable even without Qiskit, matplotlib, or Pillow.
