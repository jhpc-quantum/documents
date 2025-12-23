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
- qiskit-ibm-runtime-jhpcq
    - 0.41.0
        - Added
            - QiskitRuntimeService, SamplerV2 support JHPC Quantum. 

- qiskit-sqc-runtime
    - 1.2
        - Added
            - Add SQCRuntimeService class.
        - Changed
            - Change connection destination strings that can be specified for SQCBackend constructor.
            - Change so that the usage is the same as IBM Cloud.
                - Change return value of SQCBackend.target from None to qiskit.transpiler.Target when backend is "ibm_sqc".
                - Change "circuit: QuantumCircuit" argument of SQCSamplerV2.run to "pubs: Iterable[SamplerPubLike]". However, multiple circuits and qiskit.circuit.Parameter is not supported.
                - Change result type of runnning quantum circuit from string to qiskit.primitives.containers.PrimitiveResult when backend is "ibm_sqc".
        - Removed
            - Remove SQCBackend.transpile_info method. Use SQCBackend.target instead of this method.
    - 1.1
        - Added
            - Add prototype of running quantum circuit on "ibm-kobe-dacc".
        - Changed
            - Change result type of runnning quantum circuit from string to qiskit.primitives.containers.PrimitiveResult when backend is "qtm-sim-grpc" and "qtm-grpc".
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
