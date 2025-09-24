# 1. はじめに
本書では、量子ソフトウェア開発キットである TKET を用いて「富岳」から量子・スパコン連携プラットフォーム（JHPC Quantum）上で量子アプリケーションを実行するための方法について説明します。  
この手順書をブラウザで閲覧したい方は以下から参照してください。<br>
[jhpc-quantum・GitHub](https://github.com/jhpc-quantum)
# 2.　TKETとは
TKET はゲート型量子コンピュータ向けのプログラムを作成・実行するためのソフトウェア開発キットです。  
拡張モジュールも豊富で、現在提供されている様々な量子コンピュータやシミュレータに対して活用できるほか、その他の量子ソフトウェアライブラリとの互換性も有します。  
TKET の詳細については、公式ホームページ (https://quantinuum.co.jp/business/tket) をご覧ください。

# 3.　利用方法
## 3.1.　環境構築
 
※すでに環境を構築している場合、再度構築する必要はありません。<br>
※以降で説明する各スクリプトは、「富岳」のプリポストサーバと計算ノード(Arm)で共通です。
### 3.1.1.　作業ディレクトリの作成
ユーザーの任意の作業ディレクトリ($WORK)でTKETディレクトリを作成します。  
```
$ mkdir -p ${WORK}/TKET
$ cd ${WORK}/TKET
```
### 3.1.2.　Python仮想環境の構築とパッケージのインストール
環境構築スクリプト(venv_setup.sh)を用いて、Python仮想環境の構築とパッケージのインストール行います。   
インストールするパッケージは、TKET(pytket)と関連パッケージ、およびJHPC Quantumシステム用のpytket拡張パッケージ(pytket-sqc)です。   
インストール時間は数分です。

共有ディレクトリ（/vol0300/share/ra010014/jhpcq_modules/<***ARCH***>/SDK/<***TKET_x.x.x***>）から環境構築スクリプト(venv_setup.sh)と環境変数設定スクリプト(config.sh)をTKETディレクトリにコピーします。  
※<***ARCH***>には環境に合わせてa64fxまたはx86を指定してください。  
※<***TKET_x.x.x***>のx.x.xには使用するバージョンを指定してください。
```
$ cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK/<TKET_x.x.x>/venv_setup.sh .
$ cp -p /vol0300/share/ra010014/jhpcq_modules/<ARCH>/SDK/<TKET_x.x.x>/config.sh .
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
source /vol0004/apps/oss/spack/share/spack/setup-env.sh
spack load ${SPACK_PKG}

# 3. Set up a Python virtual environment in each user's working directory
mkdir -p ${TARGET_NAME}
cd ${TARGET_NAME}
python -m venv ${VENV_NAME}

# 4. Install packages such as pytket into the virtual environment
source ./${VENV_NAME}/bin/activate
python -m pip install -r ${SHARE_DIR}/requirements.txt

# 5. Install pytket-sqc into the virtual environment
pip install ${SHARE_DIR}/pytket_sqc-1.0-py3-none-any.whl
deactivate
```
環境変数設定スクリプト(config.sh)  
※富岳での環境構築およびTKETプログラムの実行の際、Spackを用いて必要なパッケージをロードします。  
下記の環境変数設定スクリプトで使用しているSpackのパッケージのハッシュ値は2025年7月現在のものです。
```
#!/bin/bash

#Framework Name
FRAMEWORK_NAME=TKET
#Framework Version
FRAMEWORK_VERSION=2.4.1
#SQC Library Version
SQC_VERSION=0.9
#Architecture
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ]; then
  #Packcage Name：python@3.11.6(yjlixq5), gcc@13.2.0(77gzpid)
  SPACK_PKG="/yjlixq5 /77gzpid"
  #Target Name
  TARGET_NAME=x86
elif [ "$ARCH" = "aarch64" ]; then
  #Packcage Name：py-numpy@1.25.2(dgmiy5n), python@3.11.6(qbmpmn2), gcc@13.2.0 (abihbe7)
  SPACK_PKG="/dgmiy5n /qbmpmn2 /abihbe7"
  #Target Name
  TARGET_NAME=a64fx
else
  echo "error ${ARCH} is unknown"
  exit 1
fi
#Python Virtual Environment Name
VENV_NAME=venv_python
#Share Directory
SHARE_DIR=/vol0300/share/ra010014/jhpcq_modules/${TARGET_NAME}/SDK/${FRAMEWORK_NAME}_${FRAMEWORK_VERSION}
#SQC Library Directory
SQC_DIR=/vol0300/share/ra010014/jhpcq_modules/${TARGET_NAME}/SDK/SQC_library_${SQC_VERSION}

```
環境構築スクリプトの実行が完了すると、TKETディレクトリ配下に以下のような構成でpython仮想環境が構築され、パッケージがインストールされます。  
```
`-- ${WORK}
        |-- TKET
        |    |-- <ARCH>
        |    |      `-- ${VENV_NAME} (Python仮想環境)
        |    |-- config.sh
        |    `-- venv_setup.sh
```
## 3.2.　実行
### 3.2.1.　環境設定
以降の手順で使用するスクリプトおよびサンプルプログラムは共有ディレクトリ（/vol0300/share/ra010014/jhpcq_modules/<***ARCH***>/SDK/sample/TKET）に配置されているため必要に応じてコピー・修正を行ってください。    
※<***ARCH***>には環境に合わせてa64fxまたはx86を指定してください。
<br>
<br>
環境設定スクリプト（backend_setup.sh）  
環境設定スクリプトはTKET(pytket)プログラムの実行に必要な各種設定・読み込みを行うスクリプトです。   
※"/path/to"は各ユーザーの環境に合わせて設定してください。
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
source /vol0004/apps/oss/spack/share/spack/setup-env.sh
spack load ${SPACK_PKG}

# 4. Add modules paths for SQC library to below environment variables
export LD_LIBRARY_PATH=${SQC_DIR}/lib:${SQC_DIR}/lib64:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=${SQC_DIR}/lib64/pkgconfig
export PATH=${SQC_DIR}/bin:$PATH

# 5. Add sqc library paths to the SQC_LIBRARY_PATH environment variable
export SQC_LIBRARY_PATH=${SQC_DIR}/lib64
```
### 3.2.2.　実行
JHPC Quantumシステム上で量子回路を実行するサンプルプログラム（sample.py）を実行する方法について説明します。  

JHPC Quantumシステム上で量子回路を実行するサンプルプログラム（sample.py）  

```
from pytket import Circuit
from pytket_sqc.sqcbackend import SqcBackend
# 回路の作成
circ = Circuit(2)
circ.H(0)
circ.CX(0,1)
circ.measure_all()
# バックエンドの作成
b = SqcBackend('qtm-sim-grpc')
result = b.process_circuit(circ, 10)
print(result)
```
※ ***#バックエンド作成*** のSqcBackendの引数は下記の表を参照し、接続先に対応する引数を指定してください。
| 接続先 | SqcBackendの引数 | 
|--------|--------|
| reimei  | qtm-grpc  | 
| reimei-simulator | qtm-sim-grpc  | 
| ibm-kobe-dacc  | 未定  |   

<br>
TKETのサンプルプログラムを実行するスクリプト（test.sh）を実行します。  

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
TKETのサンプルプログラムを実行するスクリプト（test.sh）  
※"/path/to/"には、3.1.2で構築したpython仮想環境へのパスを指定してください。  
※ backend_setup.shの引数として、以下のいずれかの接続先量子コンピュータを指定してください。
* reimei
* reimei-simulator
* ibm-kobe-dacc(現在利用不可)
```
#!/bin/bash

#Testing
#1. Set up Remote Procedure Call (RPC)
source ./backend_setup.sh reimei-simulator

#2. Activate the python virtual environment
source /path/to/${VENV_NAME}/bin/activate

#3. Verification test for pytket-sqc installation
echo "Result sample.py"
python ./sample.py
```

### 3.2.3.　実行結果  
プリポスト環境と富岳計算ノード（Arm）における”sample.py”の実行結果です。

JHPC Quantumシステム上で量子回路を実行するサンプルプログラム（sample.py）の実行結果の例
```
Result sample.py
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
   c    : "11"
}
{
   c    : "00"
}
```