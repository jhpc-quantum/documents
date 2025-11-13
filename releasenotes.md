# Release Notes

## SQC library
### 0.10.0
 - Added
    - Support a new quantum computer, "ibm-kobe-dacc".
 - Fixed
    - All functions support for C++.
### 0.9
First release.

## Qiskit
### 2.0.0
- qiskit-sqc-runtime
    - 1.1
        - Added
            - Add prototype of running quantum circuit on "ibm-kobe-dacc".
        - Changed
            - Change result type of runnning quantum circuit from string to "qiskit" result object.
    - 1.0<br>
        First release.

## TKET
### 2.4.1
- pytket-sqc
    - 1.1
        - Changed
            - Change result type of runnning quantum circuit from string to "pytket" result object.
    - 1.0<br>
        First release.

## CUDA-Q supported JHPC Quantum system.
### 0.11.0
- 2025/11/14
    - Added
        - Add method in "sample_result" class to show OpenQASM string executed on quantum computers or a simulator when backend is "sqcbackend".
    - Fixed
        - Fix function cudaq::sample(...)(C++) and cudaq.sample(...)(Python) so that its return value is returned as "CUDA-Q" result object when backend is "sqcbackend".
- 2025/7/31<br>
    First release.
