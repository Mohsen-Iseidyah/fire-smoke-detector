# Results

Pending hardware deployment.

Once running on Raspberry Pi 4 + Coral Edge TPU, record here:

- inference latency per frame (`edgetpu` vs `cpu`)
- sustained camera count before the shared queue saturates
- false-positive rate before and after training on the merged dataset with
  negative samples
- detection range: smallest flame reliably detected, with and without `tiles: 2`
- thermal behaviour under `libedgetpu1-std` vs `libedgetpu1-max`

Method: state the camera resolution, `inference_interval_seconds`, and the
number of enabled cameras alongside every figure.
