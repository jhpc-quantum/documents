# 1. はじめに
本書では、SQCを用いて「富岳」から量子・スパコン連携プラットフォーム（JHPC Quantum）上で量子アプリケーションを実行するための方法について説明します。<br>
この手順書をブラウザで閲覧したい方は以下から参照してください。<br>
[jhpc-quantum・GitHub](https://github.com/jhpc-quantum)<br>
アップデートの情報は[リリースノート](https://github.com/jhpc-quantum/documents/blob/main/releasenotes.md)を参照ください
# 2.　SQCとは
SQCは、以下のためのC/C++インターフェース(C-API)を提供するライブラリです。
* 量子回路の作成・読み込み
* 富岳に直接接続された量子コンピュータに対応したトランスパイル(IBM量子コンピュータのみ)
* 富岳に直接接続された量子コンピュータ上での量子回路の実行

利用可能なC-API一覧については、公式ドキュメント ([SQC.md](https://github.com/jhpc-quantum/SQC/blob/main/docs/SQC.md)) をご覧ください。

# 3.　利用方法
## 3.1.　環境構築
 
※すでに環境を構築している場合、再度構築する必要はありません。<br>
※以降で説明する各スクリプトは、「富岳」のプリポストサーバと計算ノード(Arm)で共通です。
### 3.1.1.　作業ディレクトリの作成
ユーザーの任意の作業ディレクトリ($WORK)でSQCディレクトリを作成します。  
```
$ mkdir -p ${WORK}/SQC
$ cd ${WORK}/SQC
```
### 3.1.2.　Python仮想環境の構築とパッケージのインストール
こちらは接続先がIBM量子コンピュータの時のみ必要な手順です。<br>
環境構築スクリプト(venv_setup.sh)を用いて、Python仮想環境の構築とパッケージのインストールを行います。<br>  
インストールするパッケージは、qiskit,qiskit-ibm-runtime,qiskit-qasm3-importと関連パッケージです。<br>
インストール時間は数分です。

共有ディレクトリ（/vol0300/share/ra010014/jhpcq_modules/<***ARCH***>/SDK_latest/<***SQC_library_x.x.x***>）から環境構築スクリプト(venv_setup.sh)と環境変数設定スクリプト(config.sh)をSQCディレクトリにコピーします。  
※<***ARCH***>には環境に合わせてa64fxまたはx86を指定してください。<br>
※<***SQC_library_x.x.x***>のx.xには使用するバージョンを指定してください。
```
$ cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK_latest/<SQC_library_x.x.x>/venv_setup.sh .
$ cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK_latest/<SQC_library_x.x.x>/config.sh .
```
環境構築スクリプト（venv_setup.sh）を実行します。  
ジョブスクリプト例（プリポストサーバ実行時）  
```
#!/bin/bash
#SBATCH -p mem1            # キューの指定
#SBATCH -n 1               # CPU数の指定
#SBATCH --mem 27G          # メモリ量の指定
#SBATCH --time=00:10:00    # 最大実行時間の指定

./venv_setup.sh
```
ジョブスクリプト例（計算ノード(Arm)実行時）  
```
#!/bin/bash
#PJM -L "node=1"
#PJM -L "rscgrp=small"
#PJM -L "elapse=00:10:00"
#PJM -x PJM_LLIO_GFSCACHE=/vol000N:/vol0003:/vol0004
#PJM -g groupname

./venv_setup.sh
```

環境構築スクリプト（venv_setup.sh）
```
#!/bin/bash

#Setting up a virtual environment and installing packages
# 1. Set up environment variables
source ./config.sh

# 2. Load related packages with Spack
source /vol0004/apps/oss/spack-v0.21/share/spack/setup-env.sh
spack load ${SPACK_PKG}

# 3. Set up a Python virtual environment in each user's working directory
mkdir -p ${TARGET_NAME}
cd ${TARGET_NAME}
python -m venv ${VENV_NAME}

# 4. Install packages such as qiskit into the virtual environment
source ./${VENV_NAME}/bin/activate
python -m pip install -r ${SQC_DIR}/requirements.txt
deactivate
```

環境変数設定スクリプト(config.sh)  
※富岳での環境構築およびSQCプログラムの実行の際、Spackを用いて必要なパッケージをロードします。  
下記の環境変数設定スクリプトで使用しているSpackのパッケージのハッシュ値は2025年10月現在のものです。
```
#!/bin/bash

#SQC Library Version
SQC_VERSION=0.10.0
#Architecture
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ]; then
  #Packcage Name：python@3.11.6(yjlixq5), gcc@13.2.0(77gzpid)
  SPACK_PKG="/yjlixq5 /77gzpid"
  #Target Name
  TARGET_NAME=x86
elif [ "$ARCH" = "aarch64" ]; then
  echo "This architecture is not supported."
  exit 1
  #Packcage Name：gcc@14.1.0(6mygk5q), py-numpy@1.25.2(dgmiy5n), python@3.11.6(qbmpmn2)
  SPACK_PKG="/6mygk5q /dgmiy5n /qbmpmn2"
  #Target Name
  TARGET_NAME=a64fx
else
  echo "error ${ARCH} is unknown"
  exit 1
fi
#Python Virtual Environment Name
VENV_NAME=venv_SQC_${SQC_VERSION}
#SQC Library Directory
SQC_DIR=/vol0300/share/ra010014/jhpcq_modules/${TARGET_NAME}/SDK/SQC_library_${SQC_VERSION}
```
環境構築スクリプトの実行が完了すると、SQCディレクトリ配下に以下のような構成でpython仮想環境が構築され、パッケージがインストールされます。  
```
`-- ${WORK}
        |-- SQC
        |    |-- <ARCH>
        |    |      `-- ${VENV_NAME} (Python仮想環境)
        |    |-- config.sh
        |    `-- venv_setup.sh
```
## 3.2.　実行
### 3.2.1.　環境設定
以降の手順で使用するスクリプトおよびサンプルプログラムは共有ディレクトリ（/vol0300/share/ra010014/jhpcq_modules/<***ARCH***>/SDK_latest/<***SQC_library_x.x.x***>/sample）に配置されているため必要に応じてコピー・修正を行ってください。    
※<***ARCH***>には環境に合わせてa64fxまたはx86を指定してください。<br>
※<***SQC_library_x.x.x***>のx.xには使用するバージョンを指定してください。
<br>
<br>
環境設定スクリプト（backend_setup.sh）  
環境設定スクリプトはSQCプログラムの実行に必要な各種設定・読み込みを行うスクリプトです。   
※/path/toは各ユーザーの環境に合わせて設定してください。<br>
※ backend_setup.shの引数として、以下のいずれかの接続先量子コンピュータを指定してください。
* reimei
* reimei-simulator
* ibm-kobe-dacc
```
#!/bin/bash

#Setting up Remote Procedure Call (RPC)
# 1. Set up IP address of quantum computer to SQC_RPC_SERVER according to a argument
if [ "$#" -eq 1 ]; then
  QPU="$1"
  if [ $QPU = "reimei" ]; then
     # The server port number will be changed for different target
     export SQC_RPC_SERVER=ip:port
  elif [ $QPU = "reimei-simulator" ]; then
     export SQC_RPC_SERVER=ip:port
  elif [ $QPU = "ibm-kobe-dacc" ]; then
     export SQC_RPC_SERVER=ip:port
  else
     echo "invalid qpu $QPU"
  fi
else
 echo "usage: /path/to/backend_setup.sh <target-qpu-name>"
fi

# 2. Set up environment variables
source /path/to/config.sh

# 3. Load related packages with Spack
source /vol0004/apps/oss/spack-v0.21/share/spack/setup-env.sh
spack load ${SPACK_PKG}

# 4. Add SQC library paths to various environment variables
export LD_LIBRARY_PATH=${SQC_DIR}/lib:${SQC_DIR}/lib64:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=${SQC_DIR}/lib64/pkgconfig
export PYTHONPATH=${SQC_DIR}/python:${PYTHONPATH}

# 5. Set SQC_COMPILE_OPTIONS variable to compile program using SQC C-API.
SQC_LIBS="-lsqc_api  -lsqc_rpc -lsqc_reqsched -lsqc_reqinvoker -lqtmd_sim_invoker -lsqc_dbmgr\
 -lsqc_util -lsqc_rpccommon -luuid -lsqlite3 -lprotobuf -labsl_leak_check -labsl_die_if_null\
 -labsl_log_initialize -lutf8_validity -lutf8_range -lgrpc++ -lgrpc -laddress_sorting -lupb_textformat_lib\
 -lupb_json_lib -lupb_wire_lib -lupb_message_lib -lutf8_range_lib -lupb_mini_descriptor_lib -lupb_mem_lib\
 -lupb_base_lib -labsl_statusor -lgpr -labsl_log_internal_check_op -labsl_flags_internal\
 -labsl_flags_reflection -labsl_flags_private_handle_accessor -labsl_flags_commandlineflag\
 -labsl_flags_commandlineflag_internal -labsl_flags_config -labsl_flags_program_name -labsl_raw_hash_set\
 -labsl_hashtablez_sampler -labsl_flags_marshalling -labsl_log_internal_conditions\
 -labsl_log_internal_message -labsl_examine_stack -labsl_log_internal_format -labsl_log_internal_proto\
 -labsl_log_internal_nullguard -labsl_log_internal_log_sink_set -labsl_log_internal_globals -labsl_log_sink\
 -labsl_log_entry -labsl_log_globals -labsl_hash -labsl_city -labsl_low_level_hash\
 -labsl_vlog_config_internal -labsl_log_internal_fnmatch -labsl_random_distributions\
 -labsl_random_seed_sequences -labsl_random_internal_pool_urbg -labsl_random_internal_randen\
 -labsl_random_internal_randen_hwaes -labsl_random_internal_randen_hwaes_impl\
 -labsl_random_internal_randen_slow -labsl_random_internal_platform -labsl_random_internal_seed_material\
 -labsl_random_seed_gen_exception -labsl_status -labsl_cord -labsl_cordz_info -labsl_cord_internal\
 -labsl_cordz_functions -labsl_exponential_biased -labsl_cordz_handle -labsl_crc_cord_state -labsl_crc32c\
 -labsl_crc_internal -labsl_crc_cpu_detect -labsl_bad_optional_access -labsl_strerror\
 -labsl_str_format_internal -labsl_synchronization -labsl_graphcycles_internal\
 -labsl_kernel_timeout_internal -labsl_stacktrace -labsl_symbolize -labsl_debugging_internal\
 -labsl_demangle_internal -labsl_malloc_internal -labsl_time -labsl_civil_time -labsl_strings\
 -labsl_strings_internal -labsl_string_view -labsl_base -labsl_spinlock_wait -labsl_int128\
 -labsl_throw_delegate -labsl_time_zone -labsl_bad_variant_access -labsl_raw_logging_internal\
 -labsl_log_severity -lcares -lssl -lre2 -lz -lmunge -lcrypto -lsqc_rpccommon -lprotobuf-c -lrt\
 -lpthread -ldl -lnuma -pthread"
SQC_INCS="-I${SQC_DIR}/include/ -I${SQC_DIR}/include"
PY_PATH=$(readlink -f $(which python3.11) | sed 's@/bin/python3.11@@g')
PYLIB="-L${PY_PATH}/lib -lpython3.11"

SQC_COMPILE_OPTIONS="${SQC_INCS} -L${SQC_DIR}/lib -L${SQC_DIR}/lib64 ${SQC_LIBS} ${PYLIB}"
```

### 3.2.2.  実行

JHPC Quantumシステム上で量子回路を実行するサンプルプログラムを実行する方法について説明します。

サンプルプログラムを翻訳するスクリプト（test_comp.sh）を実行します。

ジョブスクリプト例（プリポストサーバ実行時）

```
#!/bin/bash
#SBATCH -p mem1            # キューの指定
#SBATCH -n 1               # CPU数の指定
#SBATCH --mem 27G          # メモリ量の指定
#SBATCH --time=00:10:00    # 最大実行時間の指定

./test_comp.sh
```

ジョブスクリプト例（計算ノード(Arm)実行時）

```
#!/bin/bash
#PJM -L "node=1"
#PJM -L "rscgrp=small"
#PJM -L "elapse=00:10:00"
#PJM -g groupname
#PJM -x PJM_LLIO_GFSCACHE=/vol000N:/vol0003:/vol0004
#PJM -S

./test_comp.sh
```

サンプルプログラムを翻訳するスクリプト（test_comp.sh）

```
#!/bin/bash

# 1. Set up Remote Procedure Call (RPC)
source ./backend_setup.sh reimei-simulator

# 2. Compile program
gcc sample_api.c -o sample.out ${SQC_COMPILE_OPTIONS} 
```

<br>
サンプルプログラムを実行するスクリプト（test_exec.sh）を実行します。
<br>

ジョブスクリプト例（プリポストサーバ実行時）  
```
#!/bin/bash
#SBATCH -p mem1            # キューの指定
#SBATCH -n 1               # CPU数の指定
#SBATCH --mem 27G          # メモリ量の指定
#SBATCH --time=00:10:00    # 最大実行時間の指定

./test_exec.sh
```
ジョブスクリプト例（計算ノード(Arm)実行時）   
```
#!/bin/bash
#PJM -L "node=1"
#PJM -L "rscgrp=small"
#PJM -L "elapse=00:10:00"
#PJM -g groupname
#PJM -x PJM_LLIO_GFSCACHE=/vol000N:/vol0003:/vol0004
#PJM -S

./test_exec.sh
```
サンプルプログラムを実行するスクリプト（test_exec.sh）<br>
※"/path/to/"には、3.1.2で構築したpython仮想環境へのパスを指定してください。
```
#!/bin/bash

# 1. Set up Remote Procedure Call (RPC)
source ./backend_setup.sh reimei-simulator

# 2. Activate the python virtual environment when the connection destination is IBM quantum computer
source /path/to/${VENV_NAME}/bin/activate

# 3. Execute program
./sample.out
```

### 3.2.3. サンプルプログラムとその実行結果
#### 3.2.3.1. C-APIで作成した回路の実行
C-APIで作成した回路を実行するサンプルプログラム（sample_api.c）
```
#include "sqc_api.h"
#include "sqc_ecode.h"
#include <stdlib.h>
#include <string.h>

int main(int argc, char *argv[])
{
  // Specify backend
  sqcBackend backend = SQC_RPC_SCHED_QC_TYPE_QTM_SIM_GRPC;

  // Initialize C-API
  sqcInitOptions* init_options = sqcMallocInitOptions();
  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC){
    init_options->use_qiskit = 1;
  } else {
    init_options->use_qiskit = 0;
  }
  sqcInitialize(init_options);

  // Construct circuit
  const int qubits = 2;
  sqcQC* qcir = sqcQuantumCircuit(qubits);
  sqcHGate(qcir, 0);
  sqcCXGate(qcir, 0, 1);
  sqcMeasure(qcir, 0, 0, NULL);
  sqcMeasure(qcir, 1, 1, NULL);

  // Set run option
  sqcRunOptions* run_options = (sqcRunOptions*)malloc(sizeof(sqcRunOptions));
  sqcInitializeRunOpt(run_options);
  run_options->nshots = 10;
  run_options->qubits = qubits;
  run_options->outFormat = SQC_OUT_RAW;

  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC) {
    // Convert quantum circuit to OpenQASM string
    qcir->qasm = (char*)malloc(500);
    sqcConvQASMtoMemory(qcir, backend, qcir->qasm, 500);
    
    // Transpile quantum circuit
    sqcTranspile(qcir, backend, NULL);
    printf("QASM after transpile: %s\n", qcir->qasm);
  }

  // Run quantum circuit
  sqcOut* result_out = (sqcOut *)malloc(sizeof(sqcOut));
  int error_code = sqcQCRun(qcir, backend, *run_options, result_out);

  // Show error_code
  printf("error_code:%d\n", error_code);

  // Write result to file
  if ( error_code == SQC_RESULT_OK ) {
    FILE *file;
    file = fopen("result.txt", "w");
    if (file == NULL) {
      printf("Error opening file.\n");
    } else {
      sqcPrintQCResult(file, result_out, run_options->outFormat);
      fclose(file);
    }
  }

  // End processing of C-API
  sqcFreeOut(result_out, run_options->outFormat);
  free(result_out);
  free(run_options);
  sqcDestroyQuantumCircuit(qcir);
  sqcFinalize(init_options);
  sqcFreeInitOptions(init_options);

  return 0;
}
```
上記サンプルプログラムの実行結果ファイル（result.txt）の例<br>
reimeiまたはreimei-simulatorの場合
```
{
   c    : "00"
}
{
   c    : "00"
}
{
   c    : "11"
}
{
   c    : "11"
}
{
   c    : "11"
}
{
   c    : "11"
}
{
   c    : "00"
}
{
   c    : "00"
}
{
   c    : "00"
}
{
   c    : "11"
}
```
ibm-kobe-daccの場合
```
{
 "metadata": {
  "execution": {
   "execution_spans": [
    [
     {
      "date": "2025-10-27T06:15:03.863615"
     },
     {
      "date": "2025-10-27T06:15:04.781847"
     },
     {
      "0": [
       [
        10
       ],
       [
        0,
        1
       ],
       [
        0,
        10
       ]
      ]
     }
    ]
   ]
  },
  "version": 2
 },
 "results": [
  {
   "data": {
    "c": {
     "num_bits": 2,
     "samples": [
      "0x3",
      "0x3",
      "0x0",
      "0x3",
      "0x3",
      "0x0",
      "0x3",
      "0x3",
      "0x3",
      "0x3"
     ]
    }
   },
   "metadata": {
    "circuit_metadata": {}
   }
  }
 ]
}
```
#### 3.2.3.2. OpenQASMファイルの実行
OpenQASMファイルを実行するサンプルプログラム（sample_qasm.c）
```
#include "sqc_api.h"
#include "sqc_ecode.h"
#include <stdlib.h>
#include <string.h>

#define MAX_QASM_LEN (1024*1024)

int main(int argc, char *argv[])
{
  // Specify backend and 
  sqcBackend backend = SQC_RPC_SCHED_QC_TYPE_QTM_SIM_GRPC;

  // Initialize C-API
  sqcInitOptions* init_options = sqcMallocInitOptions();
  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC){
    init_options->use_qiskit = 1;
  } else {
    init_options->use_qiskit = 0;
  }
  sqcInitialize(init_options); 

  // Read OpenQASM file
  const int qubits = 2;
  sqcQC* qcir = sqcQuantumCircuit(qubits);
  qcir->qasm = (char *)calloc(sizeof(char), MAX_QASM_LEN);
  
  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC){
    sqcReadQasmFile(&(qcir->qasm), "./sample.qasm3", MAX_QASM_LEN);
  } else {
    sqcReadQasmFile(&(qcir->qasm), "./sample.qasm2", MAX_QASM_LEN);
  }

  // Set run option
  sqcRunOptions* run_options = (sqcRunOptions*)malloc(sizeof(sqcRunOptions));
  sqcInitializeRunOpt(run_options);
  run_options->nshots = 10;
  run_options->qubits = qubits;
  run_options->outFormat = SQC_OUT_RAW;

  // Transpile quantum circuit
  if (backend == SQC_RPC_SCHED_QC_TYPE_IBM_DACC) {
    sqcTranspile(qcir, backend, NULL);
    printf("QASM after transpile: %s\n", qcir->qasm);
  }

  // Run quantum circuit
  sqcOut* result_out = (sqcOut *)malloc(sizeof(sqcOut));
  int error_code = sqcQCRun(qcir, backend, *run_options, result_out);
  
  // Show error_code
  printf("error_code:%d\n", error_code);

  // Write result to file
  if ( error_code == SQC_RESULT_OK ) {
    FILE *file;
    file = fopen("result_qasm.txt", "w");
    if (file == NULL) {
      printf("Error opening file.\n");
    } else {
      sqcPrintQCResult(file, result_out, run_options->outFormat);
      fclose(file);
    }
  }

  // End processing of C-API
  sqcFreeOut(result_out, run_options->outFormat);
  free(result_out);
  free(run_options);
  sqcDestroyQuantumCircuit(qcir);
  sqcFinalize(init_options);
  sqcFreeInitOptions(init_options);
  
  return 0;
}
```
上記で利用するOpenQASMファイルの例<br>
reimeiまたはreimei-simulatorの場合（sample.qasm2）
```
OPENQASM 2.0;
include "qelib1.inc";
qreg q[2];
creg c[2];
h q[0];
cx q[0], q[1];
measure q[0] -> c[0];
measure q[1] -> c[1];
```
ibm-kobe-daccの場合（sample.qasm3）
```
OPENQASM 3.0;
include "stdgates.inc";
bit[2] c;
qubit[2] q;
h q[0];
cx q[0], q[1];
c[0] = measure q[0];
c[1] = measure q[1];
```
上記サンプルプログラムの実行結果ファイル（result_qasm.txt）の例<br>
reimeiまたはreimei-simulatorの場合
```
{
   c    : "00"
}
{
   c    : "00"
}
{
   c    : "11"
}
{
   c    : "11"
}
{
   c    : "11"
}
{
   c    : "00"
}
{
   c    : "11"
}
{
   c    : "00"
}
{
   c    : "00"
}
{
   c    : "11"
}
```
ibm-kobe-daccの場合
```
{
 "metadata": {
  "execution": {
   "execution_spans": [
    [
     {
      "date": "2025-10-27T06:24:56.046955"
     },
     {
      "date": "2025-10-27T06:24:56.941759"
     },
     {
      "0": [
       [
        10
       ],
       [
        0,
        1
       ],
       [
        0,
        10
       ]
      ]
     }
    ]
   ]
  },
  "version": 2
 },
 "results": [
  {
   "data": {
    "c": {
     "num_bits": 2,
     "samples": [
      "0x0",
      "0x3",
      "0x0",
      "0x3",
      "0x0",
      "0x3",
      "0x0",
      "0x2",
      "0x0",
      "0x0"
     ]
    }
   },
   "metadata": {
    "circuit_metadata": {}
   }
  }
 ]
}
```
