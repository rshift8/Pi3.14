# π³·¹⁴: Training-Free Acceleration for Visual Geometry Reconstruction

Richard Shi · University of Toronto

This repository explores training-free token merging for visual geometry reconstruction, as described in the [project report](report/report.pdf). Built on [π³ / Pi3X](https://github.com/yyfz/Pi3), it investigates whether reducing the sequence length of global attention can accelerate reconstruction in a permutation-equivariant architecture.

The project implements a FastVGGT-inspired baseline and **π³·¹⁴**, a frame clustering approach using K-Medoids. Neither requires retraining. Experiments show faster inference for larger scenes, with substantial reconstruction quality degradation for the proposed method. Smaller scenes can become slower because merging adds overhead.

## Approach

Pi3X encodes images with a DINOv2 backbone and alternates between attention within individual frames and global attention across frames. This fork applies merging only to the global decoder layers:

1. **Partition:** The baseline (`FastVGGTMerging`) uses region-based sampling and similarity matching. The proposed method (`KMedoidsMerging`) averages token features into frame descriptors and clusters similar frames with K-Medoids.
2. **Merge:** Average corresponding tokens within each cluster to reduce the sequence processed by attention.
3. **Unmerge:** Replicate the merged outputs back to their original token positions so the decoder can produce dense predictions at the original resolution.

The merge ratio `r` controls the requested fraction of tokens removed. Frame clustering rounds this to a whole number of frames and retains at least one cluster. Merging is disabled by default in `Pi3X`; pass both a positive `merge_ratio` and a reducer class to enable it.

## Reported results

The [report](report/report.pdf) evaluates camera pose estimation on Sintel and point map estimation on Neural-NRGBD using an RTX 3060 Laptop GPU with 6 GB VRAM. Both tables are reproduced below in full. The merge ratio `r` defines the requested fraction of tokens removed; ↓ indicates lower is better and ↑ indicates higher is better.

### Table 1: Camera pose estimation on Sintel

ATE is Absolute Trajectory Error; RPE-t and RPE-r are Relative Pose Error for translation and rotation, respectively.

| Method | ATE ↓ | RPE-t ↓ | RPE-r ↓ | Time (s) ↓ |
| --- | ---: | ---: | ---: | ---: |
| π³ | 0.125 | 0.041 | 0.294 | 384.871 |
| Baseline (r = 0.3) | 0.587 | 0.353 | 4.3142 | 289.368 |
| Baseline (r = 0.6) | 0.256 | 0.174 | 1.159 | 231.316 |
| Baseline (r = 0.9) | 0.121 | 0.058 | 0.443 | 227.268 |
| π³·¹⁴ (r = 0.3) | 0.660 | 0.244 | 4.631 | 252.742 |
| π³·¹⁴ (r = 0.6) | 0.649 | 0.212 | 3.789 | 203.870 |
| π³·¹⁴ (r = 0.9) | 0.595 | 0.279 | 3.010 | 196.516 |

### Table 2: Point map estimation on Neural-NRGBD

Acc. is Accuracy, Comp. is Completion, and N.C. is Normal Consistency. Mean and median values are reported for both sparse and dense sampling.

| View | Method | Acc. mean ↓ | Acc. median ↓ | Comp. mean ↓ | Comp. median ↓ | N.C. mean ↑ | N.C. median ↑ | Time (s) ↓ |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Sparse | π³ | 0.030 | 0.018 | 0.032 | 0.016 | 0.908 | 0.994 | 20.603 |
| Sparse | Baseline (r = 0.3) | 0.224 | 0.181 | 0.549 | 0.440 | 0.604 | 0.649 | 21.396 |
| Sparse | Baseline (r = 0.6) | 0.240 | 0.187 | 0.394 | 0.321 | 0.656 | 0.742 | 21.758 |
| Sparse | Baseline (r = 0.9) | 0.062 | 0.039 | 0.084 | 0.050 | 0.829 | 0.963 | 21.872 |
| Sparse | π³·¹⁴ (r = 0.3) | 0.250 | 0.227 | 0.678 | 0.631 | 0.625 | 0.674 | 21.852 |
| Sparse | π³·¹⁴ (r = 0.6) | 0.240 | 0.198 | 0.774 | 0.667 | 0.624 | 0.665 | 22.345 |
| Sparse | π³·¹⁴ (r = 0.9) | 0.246 | 0.206 | 0.726 | 0.627 | 0.632 | 0.686 | 22.474 |
| Dense | π³ | 0.012 | 0.007 | 0.014 | 0.006 | 0.883 | 0.986 | 140.167 |
| Dense | Baseline (r = 0.3) | 0.233 | 0.209 | 0.730 | 0.575 | 0.580 | 0.628 | 124.048 |
| Dense | Baseline (r = 0.6) | 0.166 | 0.116 | 0.440 | 0.368 | 0.635 | 0.723 | 108.145 |
| Dense | Baseline (r = 0.9) | 0.037 | 0.025 | 0.030 | 0.013 | 0.815 | 0.957 | 102.612 |
| Dense | π³·¹⁴ (r = 0.3) | 0.268 | 0.241 | 0.785 | 0.651 | 0.581 | 0.622 | 112.408 |
| Dense | π³·¹⁴ (r = 0.6) | 0.230 | 0.208 | 0.755 | 0.613 | 0.580 | 0.614 | 100.113 |
| Dense | π³·¹⁴ (r = 0.9) | 0.248 | 0.225 | 0.758 | 0.596 | 0.577 | 0.616 | 90.964 |

These are historical measurements from the report, not benchmarks run by the setup below. The report names the base model π³; the merging implementation in this checkout is in `Pi3X`. This checkout does not include the Sintel/NRGBD evaluation datasets or a complete benchmark evaluation pipeline.

### Results summary

At `r = 0.9`, π³·¹⁴ reduced inference time from 384.871 s to 196.516 s on Sintel (1.96× speedup) and from 140.167 s to 90.964 s on dense NRGBD (1.54× speedup). These gains came with substantial accuracy losses: Sintel ATE increased from 0.125 to 0.595, while dense NRGBD mean accuracy error increased from 0.012 to 0.248.

The FastVGGT-inspired baseline offered a better balance at `r = 0.9`: on Sintel, ATE was 0.121 versus 0.125 for the original model, although both relative pose errors increased; on dense NRGBD, reconstruction errors increased but remained substantially below those of π³·¹⁴. Its inference times were 227.268 s and 102.612 s, respectively.

Merging did not help sparse NRGBD scenes: every tested merging configuration was slower than the original model's 20.603 s. Overall, the results support token merging as a way to accelerate larger scenes, but frame clustering needs further work to preserve reconstruction quality, and the merge ratio must be selected with the task and input size in mind.

## Setup

Use Python 3.10 or 3.11 and an NVIDIA GPU with a CUDA-enabled PyTorch installation for the supplied inference and demo workflows. Although some scripts accept `--device cpu`, several execution paths explicitly use CUDA, so that flag does not provide a complete CPU workflow. Memory requirements depend on the number of frames and image resolution; the report's 6 GB GPU is not a guarantee that every example fits.

```bash
git clone https://github.com/rshift8/Pi3.14.git
cd Pi3.14
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt

# Verify that this environment can use the GPU.
python -c "import torch; print('PyTorch:', torch.__version__); print('CUDA available:', torch.cuda.is_available())"
```

The requirements pin PyTorch 2.5.1, torchvision 0.20.1, and NumPy 1.26.4. They also install `scikit-learn-extra`, which supplies K-Medoids and is imported by the reducer module even when merging is disabled. Run the commands from the repository root so the local `pi3` package and bundled example paths resolve. If the CUDA check prints `False`, resolve the GPU driver/PyTorch installation before running the CUDA examples.

The first model load downloads pretrained weights from Hugging Face and requires internet access. The Pi3X scripts use `yyfz233/Pi3X`; the original Pi3 script uses `yyfz233/Pi3`. For scripts with `--ckpt`, you can instead supply the matching local `model.safetensors` checkpoint.

## Execution

### Interactive reconstruction with π³·¹⁴

```bash
python -m pip install -r requirements_demo.txt
python demo_gradio_mm.py
```

Open the URL printed by Gradio, upload images or a video, and run reconstruction. This demo currently loads **Pi3X with `KMedoidsMerging` and `merge_ratio=0.9`**, disables multimodal conditioning, and updates the patch dimensions for the input resolution. Change the model initialization near the end of `demo_gradio_mm.py` to select `FastVGGTMerging`, another ratio, or an unmerged model. Both demo scripts launch with `share=True`, which also requests a public Gradio sharing link.

For the original Pi3 demo:

```bash
python demo_gradio.py
```

### Run token merging from Python

The command-line scripts do not expose a merge-ratio flag. The following complete example loads the bundled house images, enables the proposed reducer, and saves a filtered colored point cloud to `output/house_kmedoids.ply`:

```bash
python - <<'PYCODE'
from pathlib import Path
import torch
from pi3.models.pi3x import Pi3X
from pi3.models.layers.reducer import KMedoidsMerging
from pi3.utils.basic import load_multimodal_data, write_ply
from pi3.utils.geometry import depth_edge

if not torch.cuda.is_available():
    raise RuntimeError("This example requires a CUDA-enabled GPU.")

device = torch.device("cuda")
model = Pi3X.from_pretrained(
    "yyfz233/Pi3X",
    merge_ratio=0.9,
    token_reducer_class=KMedoidsMerging,
)
model.disable_multimodal()
model = model.to(device).eval()

# The loader returns (N, 3, H, W); the model expects (B, N, 3, H, W).
imgs, _ = load_multimodal_data("examples/house", interval=1, device=device)
h, w = imgs.shape[-2:]
model.update_patch_dimensions(patch_width=w // 14, patch_height=h // 14)
dtype = torch.bfloat16 if torch.cuda.get_device_capability()[0] >= 8 else torch.float16
with torch.inference_mode(), torch.amp.autocast("cuda", dtype=dtype):
    result = model(imgs[None])

mask = (torch.sigmoid(result["conf"][..., 0]) > 0.1)
mask &= ~depth_edge(result["local_points"][..., 2], rtol=0.03)
mask = mask[0]
output = Path("output/house_kmedoids.ply")
output.parent.mkdir(parents=True, exist_ok=True)
write_ply(
    result["points"][0][mask].cpu(),
    imgs.permute(0, 2, 3, 1)[mask].cpu(),
    str(output),
)
print(f"Saved {output}")
PYCODE
```

Replace `KMedoidsMerging` with `FastVGGTMerging` (including the import) to try the baseline. For an unmerged comparison, omit both merging arguments when loading Pi3X. Keep the input data, sampling interval, and checkpoint fixed across comparisons. Use batch size 1 for the frame clustering implementation. Always update the patch dimensions after loading images; the reducers rely on the actual patch grid and five register tokens per frame.

### Original Pi3 command-line inference

```bash
# Bundled video; writes examples/result.ply.
python example.py

# Image directory; writes a colored PLY point cloud.
mkdir -p output
python example.py --data_path examples/house --save_path output/house_pi3.ply --device cuda

# Use a local checkpoint matching the original Pi3 model.
python example.py --data_path examples/house --ckpt /path/to/Pi3/model.safetensors --save_path output/house_pi3.ply
```

| Argument | Meaning | Default |
| --- | --- | --- |
| `--data_path` | Image directory or `.mp4` video | `examples/skating.mp4` |
| `--save_path` | Output point cloud path | `examples/result.ply` |
| `--interval` | Sample every nth image/frame | 1 for image directories, 10 for `.mp4` |
| `--ckpt` | Local model checkpoint | Download pretrained weights |
| `--device` | Inference device | `cuda` |

Increase `--interval` to reduce the number of frames if GPU memory is insufficient.

### Runtime characterization

```bash
python runtime_characterization.py --data_path examples/house --device cuda --save_dir output/profile
```

This profiles one Pi3X forward pass and writes `output/profile/summary.txt` (operator timing and memory statistics) and `output/profile/trace.json` (a Chrome-compatible trace). It also accepts `--interval`, `--ckpt`, and `--conditions_path`. It does not save a point cloud or enable token merging by default. To profile a reducer, update its model initialization using the merging arguments and patch-dimension update shown above. Profiling includes instrumentation overhead and does not reproduce the report's evaluation metrics or benchmark timing procedure.

### Multimodal inputs and current script limitations

Pi3X supports optional camera poses, intrinsics, and depth. The scripts that accept `--conditions_path` expect an `.npz` containing all three keys:

| Key | Shape before loading | Convention |
| --- | --- | --- |
| `poses` | `(N, 4, 4)` | OpenCV camera-to-world matrices |
| `intrinsics` | `(N, 3, 3)` | Camera intrinsics in pixel coordinates |
| `depths` | `(N, H, W)` | Depth maps; invalid values set to 0 |

The loader samples the conditions with the images, resizes depth maps, and rescales intrinsics. An example is included at `examples/room/condition.npz` with RGB images in `examples/room/rgb`.

- `example_mm.py` currently passes an unbatched image tensor to Pi3X and later indexes images as though they were batched. Before using that script, add the batch dimension to the loaded images (e.g. `imgs = imgs[None]` immediately after loading). The Python example above uses the correct model input shape.
- `residual_characterization.py` runs inference and then reads `residual_stats.pkl` from the working directory. The current model does not write that file, and the script's `--save_path` is unused, so it is not a complete residual-statistics collection workflow.
- `example_vo.py` is an experimental visual-odometry entry point and is separate from the report's evaluation workflow.

## Model outputs

Pi3X accepts RGB tensors of shape `(B, N, 3, H, W)` with values in `[0, 1]` and returns:

| Key | Shape | Meaning |
| --- | --- | --- |
| `points` | `(B, N, H, W, 3)` | Global 3D point maps |
| `local_points` | `(B, N, H, W, 3)` | Per-view local 3D point maps |
| `conf` | `(B, N, H, W, 1)` | Confidence logits; apply `torch.sigmoid` |
| `camera_poses` | `(B, N, 4, 4)` | OpenCV camera-to-world transformations |

## Repository guide

| Path | Purpose |
| --- | --- |
| [report/report.pdf](report/report.pdf) | Project method, experiments, and discussion |
| [pi3/models/layers/reducer.py](pi3/models/layers/reducer.py) | Reducer interface, FastVGGT-inspired baseline, and K-Medoids merging |
| [pi3/models/layers/attention.py](pi3/models/layers/attention.py) | Merge → attention → unmerge integration |
| [pi3/models/pi3x.py](pi3/models/pi3x.py) | Pi3X model and global-layer merging configuration |
| [demo_gradio_mm.py](demo_gradio_mm.py) | Interactive Pi3X reconstruction with π³·¹⁴ enabled |
| [runtime_characterization.py](runtime_characterization.py) | PyTorch runtime and memory profiling |
| [examples](examples) | Bundled images, videos, and conditioning example |

## Upstream attribution

This project extends the [Pi3 repository](https://github.com/yyfz/Pi3) by Yifan Wang and collaborators. The pretrained models and original reconstruction architecture are their work. The report and token merging experiments in this fork are by Richard Shi. The baseline adapts ideas from [FastVGGT](https://arxiv.org/abs/2509.02560).

## 🙏 Acknowledgements

Our work builds upon several fantastic open-source projects. We'd like to express our gratitude to the authors of:

  * [DUSt3R](https://github.com/naver/dust3r)
  * [CUT3R](https://github.com/CUT3R/CUT3R)
  * [VGGT](https://github.com/facebookresearch/vggt)


## 📜 Citation

If you find our work useful, please consider citing:

```bibtex
@article{wang2025pi,
  title={$$\backslash$pi\^{} 3$: Permutation-Equivariant Visual Geometry Learning},
  author={Wang, Yifan and Zhou, Jianjun and Zhu, Haoyi and Chang, Wenzheng and Zhou, Yang and Li, Zizun and Chen, Junyi and Pang, Jiangmiao and Shen, Chunhua and He, Tong},
  journal={arXiv preprint arXiv:2507.13347},
  year={2025}
}
```




## 📄 License
This project adopts a dual-licensing strategy:

| Component | License | Commercial Use |
| :--- | :--- | :--- |
| **Code** (Scripts, Tools, Logic) | [BSD 3-Clause](LICENSE) | **Permitted** |
| **Model Weights** (Pi3 and Pi3X Weights) | [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) | **Strictly Non-Commercial** |

**Note on Model Weights:** Due to the nature of the training datasets, the model weights are restricted to non-commercial research and educational purposes only. Redistribution of the weights must maintain this restriction.
