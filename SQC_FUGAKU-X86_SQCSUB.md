# SQC x86 installation

Installation prefix: `/vol0003/share/ra010014/jhpcq/x86_3`

## Set up a Bash session
```bash
. /vol0004/apps/oss/spack/share/spack/setup-env.sh

spack load gcc@16.2.0/rdlj3rj
spack load boost@1.88.0/xoe4a3r
```

```bash
source /vol0003/share/ra010014/jhpcq/x86_3/scripts/env.sh reimei your-group-id
```

This sources `sqc-scheduler-env.sh` and `setenv_sqc.sh`, loads the configured GCC and Boost Spack packages,
and configures PATH, library paths, headers, Python 3.13, and the installed
`gen_sampler` module using the installed C API files.

The optional backend argument selects the RPC and gRPC scheduler endpoints:
`reimei` and `reimei-simulator` use `10.132.0.10`, while `ibm-kobe-dacc`
uses `10.132.0.11` (ports 30011 and 30012 respectively). With a backend,
the second argument sets the group ID; omitting it selects `default` and prints
a warning. Without arguments, existing endpoint settings are preserved, with
defaults on `10.132.0.11`, and the group ID is left unchanged.

Supply authentication settings for your account. To use a custom scheduler,
source `env.sh` without arguments and set the connection and group settings:

```bash
export SQC_RPC_SERVER='scheduler-host:30011'
export SQC_GRPC_SERVER='scheduler-host:30012'
export SQC_RPC_USER_TOKEN='your-token'
export SQC_RPC_GROUP_ID='your-group-id'
```

## Install certificates and a JWT token

Use the same backend selected when sourcing `env.sh`. Replace `BACKEND` below
with `reimei`, `reimei-simulator`, or `ibm-kobe-dacc`.

The certificate installer is included in this installation. The JWT helper is
available in the shared `jhpcq/bin` directory.

```bash
. /vol0003/share/ra010014/jhpcq/x86_3/scripts/install-cert-files.sh BACKEND
python3 /vol0003/share/ra010014/jhpcq/bin/fetch_qtm_jwt.py
mv ~/.qtm.jwt ~/.sqc-scheduler/jwt.token
```

Run these commands after setting up the Bash session above so that `python3`
uses the installed Python 3.13. The first command installs certificates under
`~/.sqc-scheduler` using the backend configuration under
`/vol0300/share/ra010014/jhpcq/x86/etc/`. The final command installs the fetched
JWT as `~/.sqc-scheduler/jwt.token`, replacing any existing token.

## Installed programs

| Program | Purpose |
| --- | --- |
| `sqcsub` | Submit a quantum circuit |
| `sqctranspile` | Transpile a circuit |
| `sqccancel` | Cancel a job |
| `sqclist` | List jobs |
| `sqcstat` | Query job status |

The scheduler programs `sqc_rpc_client`, `sqc_rpc_perf_client`, and
`sqc_rpc_sched` are also present in `bin/`.

Examples below contact the configured scheduler. Replace filenames and job IDs
with your own values.

```bash
sqcsub --nqubits 10 --nshots 30 --ifile simple.qasm --iformat qasm \
  --ofile result.txt --oformat histogram --qpu reimei
sqctranspile --nqubits 10 --ifile simple.qasm --iformat qasm \
  --ofile transpiled.qasm --qpu ibm-kobe-dacc
sqclist
sqcstat --jobid JOBID
sqccancel --jobid JOBID
```

`sqcstat` and `sqccancel` accept `--auth_method` with `none`, `mutual-tls`,
`munge`, or `jwt` (default). QIR input is currently disabled in these circuit
clients. Backend availability depends on scheduler configuration.

## Libraries and support files

- `lib/libsqc_api.so` and `lib/libsqc_api_tket.so`: C API shared libraries.
- `include/sqc_api.h`, `include/sqc_sqc.h`, `include/sqc_out.h`: API headers.
- `share/sqc/capi/gen_sampler.py`: Python circuit support module.
- `lib/`: scheduler and quantum system client libraries.
- `rex-apis/lib/`, `cpprestsdk/lib64/`, `grpc-shared/lib/`, and
  `grpc-shared/lib64/`: related runtime dependencies.
- `python/`: installed Python 3.13 and Python packages.

