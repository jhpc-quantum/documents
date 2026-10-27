# SQC command-line tools

## Install Cert and JWT files

```bash
 . /shared/hps1/jhpcq/sqc/latest/scripts/install-cert-files.sh BACKEND
 python3 /shared/hps1/jhpcq/sqc/latest/scripts/fetch_qtm_jwt.py
 mv ~/.qtm.jwt ~/.sqc-scheduler/jwt.token
```

## Set environment variables first

Source the installed environment script in Bash,
selecting the backend and your scheduler group:

```bash
source /shared/hps1/jhpcq/sqc/latest/env.sh reimei-simulator YOUR_GROUP
```

For IBM, use:

```bash
source /shared/hps1/jhpcq/sqc/latest/env.sh ibm-kobe-dacc YOUR_GROUP
```

The general syntax is:

```bash
source /shared/hps1/jhpcq/sqc/latest/env.sh BACKEND GROUP_ID
# Equivalent Bash syntax:
. /shared/hps1/jhpcq/sqc/latest/env.sh BACKEND GROUP_ID
```

Replace `YOUR_GROUP` or `GROUP_ID` with your assigned scheduler group, for
example `group1`. For Reimei hardware, use `reimei` as the backend. Source the
script in the same shell where you will run the commands so its
exported variables apply to them. Repeat this setup in each new shell, and
source it again with the appropriate backend when switching systems. Executing
it as `bash /shared/hps1/jhpcq/sqc/latest/env.sh ...` does not set variables in your current shell.

The script sets the installed libraries, Python runtime, scheduler endpoints,
and logging configuration. The installation prefix is `/shared/hps1/jhpcq/sqc/latest`.
Sourcing without arguments preserves configured endpoints and group; when unset,
the endpoints default to IBM Kobe DACC. Omitting the group selects `default` and
prints a warning. The script also accepts `reimei`; its IBM aliases `ibm-kobe`
and `ibm_kobe` select the environment only. Use `ibm-kobe-dacc` with `--qpu`.

| Variable | Purpose |
| --- | --- |
| `SQC_RPC_SERVER` | Scheduler RPC endpoint in `host:port` form. Required explicitly by `sqccancel`, `sqclist`, and `sqcstat`; also used by the SQC API. |
| `SQC_GRPC_SERVER` | gRPC endpoint configured by the environment script. |
| `SQC_RPC_GROUP_ID` | Scheduler group configured for the client. |
| `LD_LIBRARY_PATH` | Runtime library search paths. |
| `PYTHONHOME`, `PYTHONPATH` | Installed Python runtime and modules used for IBM transpilation. |
| `SQC_LOG_FILE`, `SQC_LOG_LEVEL`, `SQC_LOG_DEBUGLEVEL`, `SQC_LOG_ROTATESIZE` | Scheduler logging configuration. |

JWT authentication is the default. Configure the credentials and client
configuration required by your installation before contacting the scheduler;
sourcing the script does not itself obtain credentials.

You can check the selected endpoints and group after sourcing:

```bash
printf 'RPC server: %s\ngRPC server: %s\nGroup: %s\n' \
  "$SQC_RPC_SERVER" "$SQC_GRPC_SERVER" "$SQC_RPC_GROUP_ID"
```

The script adds the installation’s `bin` directory to `PATH`, so installed
commands can be run as `sqcsub`, `sqccancel`, `sqclist`, `sqcstat`, and `sqctranspile` without
`./`. The examples below use those installed commands.

## Commands

| Command | Function |
| --- | --- |
| `sqcsub` | Submit a circuit, wait for completion, and save its result. |
| `sqccancel` | Request cancellation of a scheduler job. |
| `sqclist` | List jobs returned by the scheduler. |
| `sqcstat` | Query the current status of one scheduler job. |
| `sqctranspile` | Transpile a circuit for IBM Kobe DACC and save the QASM. |

The commands and their runtime dependencies are already installed. Provide your
own input QASM files; source code, build tools, and sample circuits are not
included in this runtime installation.

## sqcsub

Read a QASM circuit, submit it to the selected backend, wait for its result,
print the result, and write it to a file. For IBM Kobe DACC, the command fetches
backend information and transpiles the circuit before submission.

```text
sqcsub --nqubits N --nshots N --ifile INPUT --iformat qasm \
  --ofile OUTPUT --qpu BACKEND [--oformat raw]
```

| Argument | Required | Meaning / default |
| --- | --- | --- |
| `--nqubits N` | Yes | Number of circuit qubits; must be greater than zero. Match the input circuit. |
| `--nshots N` | Yes | Number of measurement shots; must be greater than zero. |
| `--ifile PATH` | Yes | Input QASM file. Path must be shorter than 256 bytes. |
| `--iformat qasm` | Yes | Input format. QIR is currently disabled. |
| `--ofile PATH` | Yes | Result file, opened for writing and overwritten if it exists. Path must be shorter than 256 bytes. |
| `--qpu BACKEND` | Yes | Backend name from the table below. |
| `--oformat raw` | No | Backend raw result format; default: `raw`. Other output formats are currently unavailable. |

| Backend | Meaning |
| --- | --- |
| `reimei` | Reimei hardware through the Quantinuum gRPC backend. |
| `reimei-simulator` | Reimei simulator through the Quantinuum gRPC backend. |
| `qtm-grpc` | Alias for `reimei`. |
| `qtm-sim-grpc` | Alias for `reimei-simulator`. |
| `ibm-kobe-dacc` | IBM Kobe DACC backend. |

Example using your own three-qubit circuit (`circuit.qasm`):

```bash
source /shared/hps1/jhpcq/sqc/latest/env.sh reimei-simulator YOUR_GROUP
sqcsub --nqubits 3 --nshots 100 \
  --ifile ./circuit.qasm --iformat qasm \
  --ofile result.txt --oformat raw --qpu reimei-simulator
```

Although the implementation uses the asynchronous API internally, the command
calls `sqcQCWait` and remains running until the result is obtained or an error
occurs. It is not a submit-and-exit command. The result format is backend
dependent; `raw` does not promise a common histogram or bitstring layout.

The parser recognizes `bitstrings_txt` and `histogram`, but rejects them before
submission. `--auth_method` is not implemented for this command; authentication
is fixed to JWT. There is no implemented `--help` option. Always supply all
required arguments and a value after each option: this parser does not reliably
validate missing values, missing file paths, or unknown options.

## sqccancel

Request cancellation of a job identified by its **scheduler job ID**.

```text
sqccancel --jobid JOBID [--auth_method METHOD]
sqccancel --help
```

| Argument | Required | Meaning / default |
| --- | --- | --- |
| `--jobid JOBID` | Yes | Nonempty scheduler job ID; must not start with `-`. |
| `--auth_method METHOD` | No | `none`, `mutual-tls`, `munge`, or `jwt`; default: `jwt`. The server must support the selected method. |
| `--help` | No | Print usage and exit. |

```bash
sqccancel --jobid JOB_ID_FROM_SQCLIST
```

The command prints `Cancelling job: ...` and reports an error if the cancellation
call fails. Cancellation behavior depends on the job state and backend. Use
`sqcstat` afterward to check the resulting state.

## sqclist

Query the scheduler for its job list using JWT authentication.

```bash
sqclist
```

This command accepts **no arguments**, including no `--help` or `--auth_method`.
It uses `SQC_RPC_SERVER` and the scheduler client's default configuration.
Which jobs are returned depends on the server and your credentials.

The output includes RPC result information followed by entries of this form:

```text
jobs:
  id=..., status=... (...), qc_job_id=...
```

Use the `id` value with `sqcstat --jobid` or `sqccancel --jobid`.
`qc_job_id` is the backend's job ID, which may differ from the scheduler ID.

## sqcstat

Query one scheduler job and print its current status. When the server provides
them, the output also includes the backend job ID and result.

```text
sqcstat --jobid JOBID [--auth_method METHOD]
sqcstat --help
```

| Argument | Required | Meaning / default |
| --- | --- | --- |
| `--jobid JOBID` | Yes | Nonempty scheduler job ID; must not start with `-`. |
| `--auth_method METHOD` | No | `none`, `mutual-tls`, `munge`, or `jwt`; default: `jwt`. |
| `--help` | No | Print usage and exit. |

```bash
sqclist
sqcstat --jobid JOB_ID_FROM_SQCLIST
```

The output includes a numeric status and its textual name. The client handles
created, queued, running, done, and cancelled states. This wrapper performs a
single query; it has no wait or polling option. A successful query of a queued
or running job does not mean that execution has finished.

## sqctranspile

Read QASM, retrieve IBM backend information, transpile the circuit using the
SQC/Qiskit integration, and write the resulting QASM. This command does not
submit the circuit for execution.

```text
sqctranspile --nqubits N --ifile INPUT --iformat qasm \
  --ofile OUTPUT --qpu ibm-kobe-dacc [--oformat raw]
```

| Argument | Required | Meaning / default |
| --- | --- | --- |
| `--nqubits N` | Yes | Number of circuit qubits; must be greater than zero. |
| `--ifile PATH` | Yes | Input QASM file. Path must be shorter than 256 bytes. |
| `--iformat qasm` | Yes | Input format; QIR is disabled. |
| `--ofile PATH` | Yes | Output QASM file, overwritten if it exists. Path must be shorter than 256 bytes. |
| `--qpu ibm-kobe-dacc` | Yes | The only backend currently supported for this command. |
| `--oformat raw` | No | Default: `raw`. Required internally even though the output is transpiled QASM; other formats are rejected. |
| `--nshots N` | No | Parsed and printed; default: `1`. No shots are executed. |

```bash
source /shared/hps1/jhpcq/sqc/latest/env.sh ibm-kobe-dacc YOUR_GROUP
sqctranspile --nqubits 3 \
  --ifile ./circuit.qasm --iformat qasm \
  --ofile transpiled.qasm --qpu ibm-kobe-dacc
```

Backend information retrieval requires the configured service and credentials;
this is not a fully offline conversion. Authentication is fixed to JWT;
`--auth_method` and `--help` are not implemented. Supply each option and its
value as separate arguments.

## Common usage notes

- Use `--option value`, rather than `--option=value`, with these wrappers.
- Both QASM-reading tools allocate an initial 1 MiB input buffer. Check the
  underlying reader's limits before using larger circuits.
- A missing shared library usually means the installation environment has not
  been sourced or its paths need adjustment.
- For connection or authentication errors, check the selected backend,
  `SQC_RPC_SERVER`, group, credentials, and scheduler logs.
- `sqccancel`, `sqclist`, and `sqcstat` return `0` for a successful operation and
  a nonzero status for reported failures. `sqcsub` and `sqctranspile` also report
  detected failures with nonzero exits, but some underlying API calls are not
  checked by these wrappers; inspect their output and generated files as well.
