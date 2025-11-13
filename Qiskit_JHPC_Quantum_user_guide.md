# 1. はじめに
本書では、量子ソフトウェア開発キットである Qiskit を用いて「富岳」から量子・スパコン連携プラットフォーム（JHPC Quantum）上で量子アプリケーションを実行するための方法について説明します。<br>
この手順書をブラウザで閲覧したい方は以下から参照してください。<br>
[jhpc-quantum・GitHub](https://github.com/jhpc-quantum)<br>
アップデートの情報は[リリースノート](https://github.com/jhpc-quantum/documents/blob/main/releasenotes.md)を参照ください

# 2.　Qiskitとは
Qiskitは量子コンピューター上でプログラムを実行するためのソフトウェアのコレクションです。<br>
その中にある、Qiskit SDKは量子回路や操作を量子コンピュータ上で実行するためのソフトウェア開発キットです。オープンソースソフトウェアでIBM社が開発しています。<br>
Qiskit の詳細については、公式ホームページ (https://www.ibm.com/quantum/qiskit) をご覧ください。<br>

# 3.　利用方法
## 3.1.　環境構築
 
※すでに環境を構築している場合、再度構築する必要はありません。<br>
※以降で説明する各スクリプトは、「富岳」のプリポストサーバと計算ノード(Arm)で共通です。
### 3.1.1.　作業ディレクトリの作成
ユーザーの任意の作業ディレクトリ($WORK)でQiskitディレクトリを作成します。  
```
$ mkdir -p ${WORK}/Qiskit
$ cd ${WORK}/Qiskit
```
### 3.1.2.　Python仮想環境の構築とパッケージのインストール
環境構築スクリプト(venv_setup.sh)を用いて、Python仮想環境の構築とパッケージのインストール行います。   
インストールするパッケージは、qiskitと関連パッケージ、およびJHPC Quantumシステム用のqiskit拡張パッケージ(qiskit-sqc-runtime)です。   
インストール時間は数分です。

共有ディレクトリ（/vol0300/share/ra010014/jhpcq_modules/<***ARCH***>/SDK_latest/<***Qiskit_x.x.x***>）から環境構築スクリプト(venv_setup.sh)と環境変数設定スクリプト(config.sh)をQiskitディレクトリにコピーします。  
※<***ARCH***>には環境に合わせてa64fxまたはx86を指定してください。<br>
※<***Qiskit_x.x.x***>のx.x.xには使用するバージョンを指定してください。
```
$ cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK_latest/<Qiskit_x.x.x>/venv_setup.sh .
$ cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK_latest/<Qiskit_x.x.x>/config.sh .
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
python -m pip install -r ${SHARE_DIR}/requirements.txt

# 5. Install qiskit-sqc-runtime into the virtual environment
pip install ${SHARE_DIR}/qiskit_sqc_runtime-${BACKEND_VERSION}-py3-none-any.whl
deactivate
```
環境変数設定スクリプト(config.sh)  
※富岳での環境構築およびQiskitプログラムの実行の際、Spackを用いて必要なパッケージをロードします。  
下記の環境変数設定スクリプトで使用しているSpackのパッケージのハッシュ値は2025年10月現在のものです。
```
#!/bin/bash

#Framework Name
FRAMEWORK_NAME=Qiskit
#Framework Version
FRAMEWORK_VERSION=2.0.0
#Backend Version
BACKEND_VERSION=1.1
#SQC Library Version
SQC_VERSION=0.10.0
#Architecture
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ]; then
  #Packcage Name：python@3.11.6(yjlixq5)
  SPACK_PKG="/yjlixq5"
  #Target Name
  TARGET_NAME=x86
elif [ "$ARCH" = "aarch64" ]; then
  #Packcage Name：py-numpy@1.25.2(dgmiy5n), python@3.11.6(qbmpmn2)
  SPACK_PKG="/dgmiy5n /qbmpmn2"
  #Target Name
  TARGET_NAME=a64fx
else
  echo "error ${ARCH} is unknown"
  exit 1
fi
#Python Virtual Environment Name
VENV_NAME=venv_${FRAMEWORK_NAME}_${FRAMEWORK_VERSION}_${BACKEND_VERSION}
#Share Directory
SHARE_DIR=/vol0300/share/ra010014/jhpcq_modules/${TARGET_NAME}/SDK/${FRAMEWORK_NAME}_${FRAMEWORK_VERSION}
#SQC Library Directory
SQC_DIR=/vol0300/share/ra010014/jhpcq_modules/${TARGET_NAME}/SDK/SQC_library_${SQC_VERSION}
```
環境構築スクリプトの実行が完了すると、Qiskitディレクトリ配下に以下のような構成でpython仮想環境が構築され、パッケージがインストールされます。  
```
`-- ${WORK}
        |-- Qiskit
        |    |-- <ARCH>
        |    |      `-- ${VENV_NAME} (Python仮想環境)
        |    |-- config.sh
        |    `-- venv_setup.sh
```
## 3.2.　実行
### 3.2.1.　環境設定
以降の手順で使用するスクリプトおよびサンプルプログラムは共有ディレクトリ（/vol0300/share/ra010014/jhpcq_modules/<***ARCH***>/SDK_latest/<***Qiskit_x.x.x***>/sample）に配置されているため必要に応じてコピー・修正を行ってください。    
※<***ARCH***>には環境に合わせてa64fxまたはx86を指定してください。<br>
※<***Qiskit_x.x.x***>のx.x.xには使用するバージョンを指定してください。
<br>
<br>
環境設定スクリプト（backend_setup.sh）  
環境設定スクリプトはqiskitプログラムの実行に必要な各種設定・読み込みを行うスクリプトです。   
※/path/toは各ユーザーの環境に合わせて設定してください。
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

# 4. Add modules paths for SQC library to below environment variables
export LD_LIBRARY_PATH=${SQC_DIR}/lib:${SQC_DIR}/lib64:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=${SQC_DIR}/lib64/pkgconfig
export PATH=${SQC_DIR}/bin:$PATH

# 5. Add sqc library paths to the SQC_LIBRARY_PATH environment variable
export SQC_LIBRARY_PATH=${SQC_DIR}/lib64
```
### 3.2.2.　実行
JHPC Quantumシステム上で量子回路を実行するサンプルプログラムを実行する方法について説明します。<br>
※ SQCBackendの引数は下記の表を参照し、接続先に対応する引数を指定してください。
| 接続先 | SQCBackendの引数 | 
|--------|--------|
| reimei  | qtm-grpc  | 
| reimei-simulator | qtm-sim-grpc  | 
| ibm-kobe-dacc  | ibm-dacc または ibm-kobe-dacc  |

reimei-simulator上で量子回路を実行するサンプルプログラム（sample_reiemi-sim.py）  
```
from qiskit.circuit import QuantumCircuit
from qiskit_sqc_runtime import SQCBackend, SQCSamplerV2

# Construct circuit
qc = QuantumCircuit(2)
qc.h(0)
qc.cx(0, 1)
qc.measure_all()

# Get backend
backend = SQCBackend("qtm-sim-grpc")

# Run quantum circuit
sampler = SQCSamplerV2(backend)
job = sampler.run(qc, shots=10)

# Show result 
result = job.result()
print(result[0].data.meas.get_counts())
print(job.status())
```

ibm-kobe-dacc上で量子回路を実行するサンプルプログラム（sample_ibm.py）  
```
from qiskit.circuit import QuantumCircuit
from qiskit_sqc_runtime import SQCBackend, SQCSamplerV2

# Construct circuit
qc = QuantumCircuit(2)
qc.h(0)
qc.cx(0, 1)
qc.measure_all()

# Get backend
backend = SQCBackend("ibm-dacc")

# Get backend information
transpile_info = backend.transpile_info()

# Convert backend information to Target instance
from qiskit_ibm_runtime.utils.backend_converter import convert_to_target
from qiskit_ibm_runtime.models import BackendProperties, BackendConfiguration
import json
prop_json = json.loads(transpile_info[0])
backend_prop = BackendProperties.from_dict(prop_json)
conf_json = json.loads(transpile_info[1])
backend_conf = BackendConfiguration.from_dict(conf_json)
target = convert_to_target(backend_conf, backend_prop)

# Transpile circuit
from qiskit.transpiler import generate_preset_pass_manager
pm = generate_preset_pass_manager(target=target, optimization_level=1)
isa_circuit = pm.run(qc)

# Run quantum circuit
sampler = SQCSamplerV2(backend)
job = sampler.run(isa_circuit, shots=10)

# Show result 
result = job.result()
print(result) # Return string
print(job.status())
```

<br>
Qiskitのサンプルプログラムを実行するスクリプト（test.sh）を実行します。  

ジョブスクリプト例（プリポストサーバ実行時）  
```
#!/bin/bash
#SBATCH -p mem1            # キューの指定
#SBATCH -n 1               # CPU数の指定
#SBATCH --mem 27G          # メモリ量の指定
#SBATCH --time=00:05:00    # 最大実行時間の指定

./test.sh
```
ジョブスクリプト例（計算ノード(Arm)実行時）  
```
#!/bin/bash
#PJM -L "node=1"
#PJM -L "rscgrp=small"
#PJM -L "elapse=00:05:00"
#PJM -x PJM_LLIO_GFSCACHE=/vol000N:/vol0003:/vol0004
#PJM -g groupname

./test.sh
```
Qiskitのサンプルプログラムを実行するスクリプト（test.sh）<br>
※"/path/to/"には、3.1.2で構築したpython仮想環境へのパスを指定してください。<br>
※ backend_setup.shの引数として、以下のいずれかの接続先量子コンピュータを指定してください。
* reimei
* reimei-simulator
* ibm-kobe-dacc
```
#!/bin/bash

#Testing
#1. Set up Remote Procedure Call (RPC)
source ./backend_setup.sh reimei-simulator

#2. Activate the python virtual environment
source /path/to/${VENV_NAME}/bin/activate

#3. Verification test for qiskit-sqc-runtime installation
python ./sample_reimei-sim.py
```

### 3.2.3.　実行結果  
プリポスト環境と富岳計算ノード（Arm）におけるサンプルプログラムの実行結果です。

reimei-simulator上で量子回路を実行するサンプルプログラム（sample_reimei-sim.py）の実行結果の例
```
{'11': 6, '00': 4}
JobStatus.DONE
```
ibm-kobe-dacc上で量子回路を実行するサンプルプログラム（sample_ibm.py）の実行結果の例
```
{
 "metadata": {
  "execution": {
   "execution_spans": [
    [
     {
      "date": "2025-10-27T07:28:11.887278"
     },
     {
      "date": "2025-10-27T07:28:12.759585"
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
    "meas": {
     "num_bits": 2,
     "samples": [
      "0x0",
      "0x0",
      "0x3",
      "0x0",
      "0x3",
      "0x0",
      "0x3",
      "0x0",
      "0x0",
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
JobStatus.DONE
```

# 4.　既知の問題と対処
## 4.1.　プロセスが終了しない問題
Qiskitを利用してJHPC Quantumの量子コンピュータで実行すると、結果が返却されますがプロセスが終了しない問題が発生します。<br>
<br>
暫定対処法として、以下の方法があります。<br>
- 結果が返却され次第ジョブの制限時間やCtrl+Cなどでプロセスを強制的に終了させる。
- 以下の環境変数の設定のうちどちらか片方を行う。
   
   -
      ```
      $ export OPENBLAS_NUM_THREADS=1
      ```
   -
      ```
      $ export OMP_NUM_THREADS=1
      ```
