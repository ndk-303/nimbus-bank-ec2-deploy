# Face model

On the EC2 host, provide the licensed InsightFace `buffalo_l` weights under `models/buffalo_l/`: `det_10g.onnx` and `w600k_r50.onnx`. Model weights are not included in this repository and are ignored by Git. The face container runs as UID 65532 and must be able to read the two files and directories.
