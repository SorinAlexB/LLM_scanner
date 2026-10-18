# LLM_scanner
This project is meant to scan the most common LLM storage formats in order to find vulnerabilities that might be present.

# Covered formats

# Model file formats: risks and how to check them

A reference for the file formats used to store and ship machine learning models, what can go wrong with each, and how to verify a file before you use it.

Last checked: 2026-10-18. Vulnerability details come from public advisories and vendor write-ups; items marked "not verified" were not confirmed against a primary source.

"Runs code on load" means that simply opening the file can execute code.

## Summary table

| Format | Runs code on load? | Common problems | How to check |
| --- | --- | --- | --- |
| Pickle, joblib, PyTorch `.pt` / `.pth` / `.bin` | Yes | Loading a pickle can execute arbitrary code. "Sleepy Pickle" research showed attackers can patch weights during unpickling. | [Fickling](https://blog.trailofbits.com/2025/09/16/ficklings-new-ai/ml-pickle-file-scanner/) allow-list, [ModelScan](https://github.com/protectai/modelscan), [ModelAudit](https://pypi.org/project/modelaudit/), [Hugging Face pickle guide](https://huggingface.co/docs/hub/security-pickle). Never load the file to check it. |
| NumPy `.npy` / `.npz` | Yes, if loaded with `allow_pickle=True` | Object arrays embed pickle. Scanners have been bypassed with crafted headers. | Load with `allow_pickle=False`; scan with ModelAudit |
| safetensors | No | Stores only weights, with no code, and was audited by Trail of Bits. The file is safe to open, but weights can still be poisoned, so verify provenance. | [safetensors docs](https://huggingface.co/docs/safetensors/en/index), the official parser, hash and signature check |
| GGUF | No | Heap overflows in llama.cpp's GGUF parser (CVE-2024-21825, CVE-2024-23605, CVE-2024-23496) allowed code execution from a crafted file. | Keep llama.cpp and wrappers updated, validate header counts and lengths, parse in a sandbox, [Talos advisory](https://talosintelligence.com/vulnerability_reports/TALOS-2024-1912) |
| GGML (legacy `.bin`) | No | Superseded by GGUF; older parsers are less hardened. | Convert to GGUF or safetensors |
| ONNX | No, but custom operators can load native code | Path traversal through external data references (CVE-2024-27318); CVE-2025-51480 when saving; CVE-2026-34447 (symlink traversal, fixed in onnx 1.21.0). | Upgrade `onnx`; [scanners flag custom operators and external data paths](https://www.promptfoo.dev/docs/model-audit/scanners); keep external data inside the model folder |
| Keras `.h5` / `.hdf5` (legacy) | Yes (Lambda layers) | `safe_mode=True` is not honored when reading `.h5` files (CVE-2025-9905). | Upgrade Keras; avoid `.h5`; scan the config for Lambda and custom objects |
| Keras `.keras` | Yes | Lambda layers in older Keras allow code injection (CVE-2024-3660). Safe-mode bypasses followed (CVE-2025-1550, CVE-2025-8747). CVE-2026-1462 via TFSMLayer was fixed in Keras 3.13.2. | Upgrade Keras, [CERT note](https://kb.cert.org/vuls/id/253266), ModelAudit |
| TensorFlow SavedModel (`.pb`, folder) | Treat as yes | The graph can contain operations with side effects, so it is closer to a program than to data. (From general knowledge; check TensorFlow's security guidance.) | ModelAudit scans `.pb` and SavedModel folders |
| TFLite `.tflite` | No | Flatbuffer parser bugs. No specific CVE verified. | ModelAudit; run only in a sandbox |
| TensorRT `.engine` / `.plan` / `.trt` | Treat as yes | The engine is an opaque, hardware- and version-specific blob and can load plugin code. No specific CVE verified. | Verify hash, signature and build provenance; ModelAudit covers these extensions; deserialize only in a sandbox |
| OpenVINO IR (`.xml` + `.bin`) | No | The XML can hold custom layer definitions or external references. | Scanners flag custom layers; inspect the XML |
| llamafile | Yes, it is an executable | You are running a program. | Treat it as software: sign it and check it like any binary |
| Core ML, ExecuTorch `.pte`, MLX | Core ML and ExecuTorch: not verified. MLX uses safetensors or npz. | Specific issues not verified. | Hash, signature, sandbox |

## Notes

- **Scanners can miss things.** Picklescan had a bypass fixed in 0.0.25 (CVE-2025-71355). Researchers have also published proof-of-concept joblib and `.npy` files that ModelScan and picklescan failed to flag. Prefer formats that cannot hold code, and treat a clean scan as one signal, not proof.
- **ModelAudit** lists pickle, PyTorch, NumPy, joblib, TensorFlow, TFLite, TensorRT, ONNX, H5 and safetensors among the formats it scans without loading them.
- **Check order:** verify hash and signature, then parse, then sandbox, and only then load.
- **Run scanners in a separate, sandboxed process** with no network, so a parser bug cannot compromise the machine that does the checking.

## Sources

- Hugging Face, [Pickle scanning](https://huggingface.co/docs/hub/security-pickle)
- Trail of Bits, [Fickling's new AI/ML pickle file scanner](https://blog.trailofbits.com/2025/09/16/ficklings-new-ai/ml-pickle-file-scanner/)
- CERT/CC, [VU#253266: Keras 2 Lambda layers](https://kb.cert.org/vuls/id/253266)
- JFrog, [Keras safe_mode bypass](https://jfrog.com/blog/keras-safe_mode-bypass-vulnerability/)
- Cisco Talos, [TALOS-2024-1912 (GGUF)](https://talosintelligence.com/vulnerability_reports/TALOS-2024-1912)
- SentinelOne, [CVE-2024-27318 (ONNX)](https://www.sentinelone.com/vulnerability-database/cve-2024-27318/)
- Kodem, [CVE-2026-34447 (ONNX)](https://www.kodemsecurity.com/cve-archive/cve-2026-34447)
- NVD, [CVE-2026-1462 (Keras TFSMLayer)](https://nvd.nist.gov/vuln/detail/CVE-2026-1462)
- Promptfoo, [Model scanners](https://www.promptfoo.dev/docs/model-audit/scanners)
- PyPI, [ModelAudit](https://pypi.org/project/modelaudit/)
- SentinelOne, [CVE-2025-71355 (picklescan)](https://www.sentinelone.com/vulnerability-database/cve-2025-71355/)

