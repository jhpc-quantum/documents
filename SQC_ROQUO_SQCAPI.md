# Writing programs with the SQC C API

Use the SQC C API to construct quantum circuits in your own C program, convert
circuits to OpenQASM, and submit them to the scheduler. The public entry point
is `sqc_api.h`; link with `libsqc_api.so`. This installation provides the API
headers in `include/` and runtime libraries in `lib/`.

## Set up your shell

Use Bash on the ARM64 system where this installation is available:

```bash
source /shared/hps1/jhpcq/sqc/latest/env.sh reimei-simulator YOUR_GROUP
```

`PREFIX` is `/shared/hps1/jhpcq/sqc/latest`. Replace `YOUR_GROUP` with your assigned
group. The script adds the installed commands and libraries to the shell paths
and configures Python, `SQC_RPC_SERVER`, and `SQC_RPC_GROUP_ID`.

Choose a matching environment and backend constant in your program:

| Environment argument | C API backend constant |
| --- | --- |
| `reimei-simulator` | `SQC_RPC_SCHED_QC_TYPE_QTM_SIM_GRPC` |
| `reimei` | `SQC_RPC_SCHED_QC_TYPE_QTM_GRPC` |
| `ibm-kobe-dacc` | `SQC_RPC_SCHED_QC_TYPE_IBM_DACC` |

Selecting an environment does not change a backend constant compiled into your
program. Scheduler submission requires your configured credentials and client
configuration in `~/.sqc-scheduler`; sourcing `env.sh` does not obtain credentials.
The example uses JWT authentication. Offline circuit generation needs no
scheduler connection.

## Complete example: construct a Bell circuit

Save the following as `bell.c` in your own working directory. Running it without
arguments prints OpenQASM. Running it with `--run` also submits 100 shots to the
Reimei simulator, waits for completion, and prints the raw result.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "sqc_api.h"

int main(int argc, char **argv)
{
    int status = EXIT_FAILURE;
    int initialized = 0;
    sqcInitOptions *init = NULL;
    sqcQC *qc = NULL;
    sqcOut result = {0};
    sqc_handle_t handle = {0};
    const sqcBackend backend = SQC_RPC_SCHED_QC_TYPE_QTM_SIM_GRPC;
    const int submit = argc == 2 && strcmp(argv[1], "--run") == 0;

    if (argc > 2 || (argc == 2 && !submit)) {
        fprintf(stderr, "Usage: %s [--run]\n", argv[0]);
        return EXIT_FAILURE;
    }

    init = sqcMallocInitOptions();
    if (init == NULL) goto cleanup;
    init->use_qiskit = 0; /* Reimei circuit generation needs no Qiskit. */
    if (sqcInitialize(init) != E_SUCCESS) goto cleanup;
    initialized = 1;

    qc = sqcQuantumCircuit(2);
    if (qc == NULL) goto cleanup;
    sqcHGate(qc, 0);
    sqcCXGate(qc, 0, 1); /* control qubit 0, target qubit 1 */
    sqcMeasureAll(qc);

    /* Allocate a generous conversion buffer explicitly. The circuit owns it
       and sqcDestroyQuantumCircuit() will free it. */
    const size_t capacity = 1024 * 1024;
    qc->qasm = calloc(capacity, 1);
    if (qc->qasm == NULL) goto cleanup;
    int rc = sqcConvQASMtoMemory(qc, backend, qc->qasm, capacity);
    if (rc <= 0) {
        fprintf(stderr, "QASM conversion failed: %d\n", rc);
        goto cleanup;
    }
    puts(qc->qasm);

    if (!submit) {
        status = EXIT_SUCCESS;
        goto cleanup;
    }

    /* Zero-initialize first: the initializer does not set every field. */
    sqcRunOptions run = {0};
    sqcInitializeRunOpt(&run);
    run.qubits = 2;
    run.nshots = 100;
    run.priority = 0;
    run.inFormat = SQC_IN_QASM;
    run.outFormat = SQC_OUT_RAW;
    run.auth_method = RPC_AUTH_METHOD_JWT;

    rc = sqcQCRunAsync(qc, backend, run, &handle);
    if (rc != 0) {
        fprintf(stderr, "Submission failed: %d\n", rc);
        goto cleanup;
    }
    fprintf(stderr, "Scheduler job ID: %s\n", handle.job_id);
    /* Independent classical work can be performed here. */
    rc = sqcQCWait(&result, handle);
    /* For a valid submitted handle, Wait releases it on success or failure.
       Do not call sqcDestroyHandle() again. */
    if (rc != 0) {
        fprintf(stderr, "Waiting for result failed: %d\n", rc);
        goto cleanup;
    }
    if (result.result == NULL) {
        fprintf(stderr, "No raw result returned\n");
        goto cleanup;
    }
    sqcPrintQCResult(stdout, &result, SQC_OUT_RAW);
    status = EXIT_SUCCESS;

cleanup:
    sqcFreeOut(&result, SQC_OUT_RAW);
    if (qc != NULL) sqcDestroyQuantumCircuit(qc);
    if (initialized) sqcFinalize(init);
    sqcFreeInitOptions(init);
    return status;
}
```

Compile C with GCC and link with G++ because the API implementation uses C++:

```bash
gcc -std=gnu99 -D_GNU_SOURCE -Wall -Wextra \
  -I"$PREFIX/include" -c bell.c -o bell.o
g++ bell.o -L"$PREFIX/lib" -Wl,-rpath,"$PREFIX/lib" \
  -lsqc_api -lsqc_rpccommon -o bell

# Offline: generate and inspect the circuit.
./bell

# Submit to the scheduler using your configured credentials.
./bell --run
```

Source `env.sh` in every new shell before running your program. It supplies
search paths for transitive library dependencies in the dependency directories.
The source and object files above belong in your own directory; you do not need
the SQC implementation source tree to compile this client.

## API workflow and ownership

1. Allocate `sqcInitOptions`, select `use_qiskit`, and call `sqcInitialize()` once
   per process. Keep the options until finalization.
2. Create a circuit with `sqcQuantumCircuit(nqubits)`. Qubit and classical-bit
   indices start at zero. Classical-bit count is assumed to match qubit count.
3. Append gates and measurements. For example, `sqcHGate(qc, 0)`,
   `sqcRXGate(qc, angle_in_radians, 0)`, `sqcCXGate(qc, control, target)`, and
   `sqcMeasure(qc, qubit, classical_bit, NULL)`.
4. Convert with `sqcConvQASMtoMemory()`. A positive return is the number of bytes
   including the terminating NUL. `E_SHORTAGE_SIZE` means increase the buffer;
   `E_NULL_POINTER` means the output pointer was NULL. Pass a nonzero capacity.
   Reimei backends generate QASM 2; other backends use the QASM 3 path.
5. Zero-initialize `sqcRunOptions`, call `sqcInitializeRunOpt()`, and set shots,
   qubits, priority, input/output formats, and authentication. Use positive shot
   counts and priorities from 0 to 255. Use `SQC_OUT_RAW` for these clients.
6. Call `sqcQCRunAsync()` and check for zero success before using its handle.
   `sqcQCWait()` polls until terminal status and consumes a valid handle.
   A successful wait can also correspond to cancellation; inspect the returned
   backend result instead of assuming measurements were produced.
7. Release results with `sqcFreeOut()`, destroy the circuit, then call
   `sqcFinalize()` and `sqcFreeInitOptions()`. Destroy Python-backed circuits
   before finalizing Python. Do not repeatedly initialize/finalize in one process.

For synchronous submission, `sqcQCRun(qc, backend, run, &result)` submits and waits
in one call; no asynchronous handle is returned.

For asynchronous jobs, `sqcQCCancel(run, handle)` requests cancellation and
releases the handle after an attempted RPC, including on RPC failure. Validate
that the handle and `SQC_RPC_SERVER` are set first. Do not reuse a consumed
handle. `sqcDestroyHandle(handle)` releases local resources without requesting
remote cancellation; use it only for a live handle you will no longer wait on.

The gate append functions do not validate qubit indices or the circuit capacity.
Keep indices within the allocated circuit and stay below `MAX_N_GATES` (40000)
operations. Measurement, reset, delay, and barrier operations also occupy entries.
If you append more gates after generating `qc->qasm`, regenerate the QASM before
submission: an existing `qc->qasm` is used as the submitted program.

## Submit existing QASM

Instead of appending gates, allocate `qc->qasm` with `malloc`/`calloc` and read a
NUL-terminated QASM string into it. Match `run.qubits` to the circuit and set
`run.inFormat = SQC_IN_QASM`. The circuit destructor owns and frees `qc->qasm`,
so do not assign a string literal, stack buffer, or separately freed pointer.
The supplied `sqcReadQasmFile()` helper is also available; size and validate your
input buffer before using it.

## IBM Kobe DACC and transpilation

Use `source /shared/hps1/jhpcq/sqc/latest/env.sh ibm-kobe-dacc YOUR_GROUP` and select
`SQC_RPC_SCHED_QC_TYPE_IBM_DACC` in your code. Set `init->use_qiskit = 1` before
`sqcInitialize()` to enable the installed Python/Qiskit integration.

After constructing a circuit or loading QASM, obtain backend information and
transpile before submission:

```c
int rc = sqcIbmdTranspileInfo(qc, SQC_RPC_SCHED_QC_TYPE_IBM_DACC);
if (rc != 0) {
    /* Report the error and clean up; do not continue to transpilation. */
} else {
    sqcTranspile(qc, SQC_RPC_SCHED_QC_TYPE_IBM_DACC, NULL);
    /* Convert the transpiled circuit using sqcConvQASMtoMemory(), then submit. */
}
```

When converting a loaded circuit after transpilation, use a fresh, sufficiently
large buffer and replace/free the old `qc->qasm` after successful conversion.
`sqcTranspile()` returns `void`; inspect Python errors and generated QASM.
Backend information retrieval requires the configured service and credentials.
The Bell example above is specifically for the Reimei simulator; switching IBM
requires this additional initialization/transpilation flow.

## Troubleshooting

- Compiler cannot find `sqc_api.h`: source `env.sh` and use `-I"$PREFIX/include"`.
- Linker cannot find `libsqc_api`: use `-L"$PREFIX/lib"` and link with G++.
- Loader reports a missing library: source the installed `env.sh` in that shell.
- Submission/authentication fails: check the selected backend, endpoint, group,
  and your `~/.sqc-scheduler` credentials. API return codes are not measurement
  results; check every submission/wait return value.
- See `README.md` for the installed `sqcsub`, `sqclist`, `sqcstat`, `sqccancel`,
  and `sqctranspile` commands.
