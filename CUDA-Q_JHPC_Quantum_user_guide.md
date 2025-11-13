# 1. はじめに

本書では、量子ソフトウェア開発キットである CUDA-Q を用いて「富岳」から量子・スパコン連携プラットフォーム（JHPC Quantum）上で量子アプリケーションを実行するための方法について説明します。<br>
この手順書をブラウザで閲覧したい方は以下から参照してください。<br>
[jhpc-quantum・GitHub](https://github.com/jhpc-quantum)<br>
アップデートの情報は[リリースノート](https://github.com/jhpc-quantum/documents/blob/main/releasenotes.md)を参照ください

# 2. CUDA-Qとは

CUDA-QはNVIDIAが開発したオープンソースの量子-古典ハイブリッド計算プラットフォームです。
PythonとC++でのプログラミングをサポートしています。  
CUDA-Q の詳細については、公式ホームページ (<https://developer.nvidia.com/cuda-q>) をご覧ください。

# 3. 利用方法

## 3.1. 環境構築

※すでに環境を構築している場合、再度構築する必要はありません。<br>
※以降で説明する各スクリプトは、「富岳」の計算ノード(Arm)とプリポストサーバで共通です。

### 3.1.1. 作業ディレクトリの作成

ユーザーの任意の作業ディレクトリ($WORK)でCUDA-Qディレクトリを作成します。  

```
mkdir -p ${WORK}/CUDA-Q
cd ${WORK}/CUDA-Q
```

### 3.1.2.  構築

#### 3.1.2.1  C++版

C++版のCUDA-Qは共有ディレクトリ（/vol0300/share/ra010014/jhpcq_modules/<***ARCH***>/SDK_latest/<***CUDA-Q_x.x.x***>）にインストールされています。
CUDA-Qを使用するための環境変数設定スクリプト(config.sh)を共有ディレクトリからCUDA-Qディレクトリにコピーしてください。  
※<***ARCH***>には環境に合わせてa64fxまたはx86を指定してください。  
※<***CUDA-Q_x.x.x***>のx.x.xには使用するバージョンを指定してください。  

```
cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK_latest/<CUDA-Q_x.x.x>/config.sh .
```

環境変数設定スクリプト(config.sh)  
※Spackを用いて必要なパッケージをロードします。  
下記の環境変数設定スクリプトで使用しているSpackのパッケージのハッシュ値は2025年10月現在のものです。

```
#!/bin/bash

#Framework Name
FRAMEWORK_NAME=CUDA-Q
#Framework Version
FRAMEWORK_VERSION=0.11.0
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
  #Packcage Name：gcc@11.4.0(tdt2ro4), py-numpy@1.25.2(dgmiy5n), python@3.11.6(qbmpmn2)
  SPACK_PKG="/tdt2ro4 /dgmiy5n /qbmpmn2"
  #Target Name
  TARGET_NAME=a64fx
else
  echo "error ${ARCH} is unknown"
  exit 1
fi
#Python Virtual Environment Name
VENV_NAME=venv_${FRAMEWORK_NAME}_${FRAMEWORK_VERSION}
#Share Directory
SHARE_DIR=/vol0300/share/ra010014/jhpcq_modules/${TARGET_NAME}/SDK/${FRAMEWORK_NAME}_${FRAMEWORK_VERSION}
#SQC Library Directory
SQC_DIR=/vol0300/share/ra010014/jhpcq_modules/${TARGET_NAME}/SDK/SQC_library_${SQC_VERSION}
```

#### 3.1.2.2  Python版

環境構築スクリプト(venv_setup.sh)を用いて、Python仮想環境の構築とパッケージのインストール行います。
インストールするパッケージは、CUDA-Qの関連パッケージです。  
インストール時間は数分です。

共有ディレクトリから環境構築スクリプト(venv_setup.sh)と環境変数設定スクリプト(config.sh)をCUDA-Qディレクトリにコピーします。  
※<***ARCH***>には環境に合わせてa64fxまたはx86を指定してください。  
※<***CUDA-Q_x.x.x***>のx.x.xには使用するバージョンを指定してください。  

```
cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK_latest/<CUDA-Q_x.x.x>/venv_setup.sh .
cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK_latest/<CUDA-Q_x.x.x>/config.sh .
```

環境構築スクリプト（venv_setup.sh）を実行します。

ジョブスクリプト例（プリポストサーバ実行時）

```
#!/bin/bash
#SBATCH -p mem1            # キューの指定
#SBATCH -n 1               # CPU数の指定
#SBATCH --mem 27G          # メモリ量の指定
#SBATCH --time=00:05:00    # 最大実行時間の指定

./venv_setup.sh
```

ジョブスクリプト例（計算ノード(Arm)実行時）

```
#!/bin/bash
#PJM -L "node=1"
#PJM -L "rscgrp=small"
#PJM -L "elapse=00:05:00"
#PJM -x PJM_LLIO_GFSCACHE=/vol000N:/vol0003:/vol0004
#PJM -g groupname

./venv_setup.sh
```

環境構築スクリプト（venv_setup.sh）

```
#!/bin/bash

# Setting up a virtual environment and installing packages
# 1. Set up environment variables
source ./config.sh

# 2. Load related packages with Spack
source /vol0004/apps/oss/spack-v0.21/share/spack/setup-env.sh
spack load ${SPACK_PKG}

# 3. Set up a Python virtual environment in each user's working directory
mkdir -p ${TARGET_NAME}
cd ${TARGET_NAME}
python -m venv ${VENV_NAME}

# 4. Install packages such as cuda-q into the virtual environment
source ./${VENV_NAME}/bin/activate
python -m pip install -r ${SHARE_DIR}/requirements.txt

deactivate
```

環境構築スクリプトの実行が完了すると、CUDA-Qディレクトリ配下に以下のような構成でpython仮想環境が構築され、パッケージがインストールされます。  

```
`-- ${WORK}
        |-- CUDA-Q
        |    |-- <ARCH>
        |    |      `-- ${VENV_NAME} (Python仮想環境)
        |    |-- config.sh
        |    `-- venv_setup.sh
```

## 3.2.  実行

### 3.2.1.  環境設定

以降の手順で使用するスクリプトおよびサンプルプログラムは共有ディレクトリ（/vol0300/share/ra010014/jhpcq_modules/<***ARCH***>/SDK_latest/<***CUDA-Q_x.x.x***/sample）に配置されているため必要に応じてコピー・修正を行ってください。<br>
※<***ARCH***>には環境に合わせてa64fxまたはx86を指定してください。<br>
※<***CUDA-Q_x.x.x***>のx.x.xには使用するバージョンを指定してください。  
<br>
<br>
環境設定スクリプト（backend_setup.sh）  
環境設定スクリプトはCUDA-Qプログラムの実行に必要な各種設定・読み込みを行うスクリプトです。<br>
※/path/toは各ユーザーの環境に合わせて設定してください。<br>
※ backend_setup.shの引数として、以下のいずれかの接続先量子コンピュータを指定してください。
* reimei
* reimei-simulator
* ibm-kobe-dacc(現在利用不可)
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
     echo "not available"
     exit 1
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

# 4. Add grpc and grpc-c-wrapper paths to the LD_LIBRARY_PATH environment variable
export LD_LIBRARY_PATH=${SQC_DIR}/lib:${SQC_DIR}/lib64:$LD_LIBRARY_PATH

# 5. Add sqc library paths to the SQC_LIBRARY_PATH environment variable
export SQC_LIBRARY_PATH=${SQC_DIR}/lib64

# 6. Set up CUDA-Q enviroment
source ${SHARE_DIR}/cudaq/set_env.sh

# 7. Set up PYTHONPATH
export PYTHONPATH=${SHARE_DIR}/cudaq:${PYTHONPATH}
```

### 3.2.2.  実行

JHPC Quantumシステム上で量子回路を実行するサンプルプログラムを実行する方法について説明します。  

#### 3.2.2.1. C++版

JHPC Quantumシステム上で量子回路を実行するサンプルプログラム（sample.cpp)

```
#include <cudaq.h>
#include <iostream>

__qpu__ void kernel(int qubit_count) {
  cudaq::qvector qubits(qubit_count);
  h(qubits[0]);
  for (auto i = 1; i < qubit_count; ++i) {
    cx(qubits[0], qubits[i]);
  }
  mz(qubits);
}

int main(int argc, char *argv[]) {
  auto qubit_count = 1 < argc ? atoi(argv[1]) : 2;
  int shots_count = 5;
  auto result = cudaq::sample(shots_count, kernel, qubit_count);
  std::cout << result.get_execution_qasm() << "\n" << std::endl;
  result.dump();
}
```

サンプルプログラムを翻訳するスクリプト（test_comp_cpp.sh）を実行します。

ジョブスクリプト例（プリポストサーバ実行時）

```
#!/bin/bash
#SBATCH -p mem1            # キューの指定
#SBATCH -n 1               # CPU数の指定
#SBATCH --mem 27G          # メモリ量の指定
#SBATCH --time=00:10:00    # 最大実行時間の指定

./test_comp_cpp.sh
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
#PJM -N "test_comp_cpp"

./test_comp_cpp.sh
```

サンプルプログラムを翻訳するスクリプト（test_comp_cpp.sh）

```
#!/bin/bash

# Testing
# 1. Set up Remote Procedure Call (RPC)
source ./backend_setup.sh reimei-simulator

# 2. Compile program
nvq++ --target sqcbackend --sqcbackend-machine qtm-sim-grpc sample.cpp -o sample.out
```

※ nvq++コマンドのオプション--sqcbackend-machineには、接続先に対応する値を指定してください。

| 接続先 | --sqcbackend-machineの値 |
|--------|--------|
| reimei  | qtm-grpc  |
| reimei-simulator | qtm-sim-grpc  |
| ibm-kobe-dacc  | 未定  |

<br>
サンプルプログラムを実行するスクリプト（test_exec_cpp.sh）を実行します。
<br>

ジョブスクリプト例（プリポストサーバ実行時）  
```
#!/bin/bash
#SBATCH -p mem1            # キューの指定
#SBATCH -n 1               # CPU数の指定
#SBATCH --mem 27G          # メモリ量の指定
#SBATCH --time=00:10:00    # 最大実行時間の指定

./test_exec_cpp.sh
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

./test_exec_cpp.sh
```
サンプルプログラムを実行するスクリプト（test_exec_cpp.sh）
```
#!/bin/bash

# Testing
# 1. Set up Remote Procedure Call (RPC)
source ./backend_setup.sh reimei-simulator

# 2. Execute program
./sample.out
```

#### 3.2.2.2. Python版
JHPC Quantumシステム上で量子回路を実行するサンプルプログラム（sample.py)
```
import cudaq

cudaq.set_target('sqcbackend', machine='qtm-sim-grpc')
kernel = cudaq.make_kernel()
qubits = kernel.qalloc(2)

kernel.h(qubits[0])
kernel.cx(qubits[0], qubits[1])

kernel.mz(qubits)

result = cudaq.sample(kernel, shots_count=5)
print(f"{result.get_execution_qasm()}\n")
print(result)
```
※ cudaq.set_target()の引数machineには、接続先に対応する値を指定してください。
| 接続先 | machineの値 | 
|--------|--------|
| reimei  | qtm-grpc  | 
| reimei-simulator | qtm-sim-grpc  | 
| ibm-kobe-dacc  | 未定  |

<br>
サンプルプログラムを実行するスクリプト（test_python.sh）を実行します。  

ジョブスクリプト例（プリポストサーバ実行時）  
```
#!/bin/bash
#SBATCH -p mem1            # キューの指定
#SBATCH -n 1               # CPU数の指定
#SBATCH --mem 27G          # メモリ量の指定
#SBATCH --time=00:05:00    # 最大実行時間の指定

./test_python.sh
```
ジョブスクリプト例（計算ノード(Arm)実行時）   
```
#!/bin/bash
#PJM -L "node=1"
#PJM -L "rscgrp=small"
#PJM -L "elapse=00:05:00"
#PJM -x PJM_LLIO_GFSCACHE=/vol000N:/vol0003:/vol0004
#PJM -g groupname

./test_python.sh
```

サンプルプログラムを実行するスクリプト（test_python.sh）  
※"/path/to/"には、3.1.2.2で構築したpython仮想環境へのパスを指定してください。
```
#!/bin/bash

# Testing
# 1. Set up Remote Procedure Call (RPC)
source ./backend_setup.sh reimei-simulator

# 2. Activate the python virtual environment
source /path/to/${VENV_NAME}/bin/activate

# 3. Execute program
python ./sample.py
```

### 3.2.3.  実行結果  
プリポスト環境と富岳計算ノード（Arm）におけるサンプルプログラムの実行結果です。

JHPC Quantumシステム上で量子回路を実行するサンプルプログラムの実行結果の例
```
// Code generated by NVIDIA's nvq++ compiler
OPENQASM 2.0;

include "qelib1.inc";

qreg var0[2];
h var0[0];
cx var0[0], var0[1];
creg var3[2];
measure var0 -> var3;

{ 
   var3 : { 00:2 11:3 }
}
```
