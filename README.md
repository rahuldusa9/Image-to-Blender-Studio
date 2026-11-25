# Image to Blender Studio

Image to Blender Studio is a pair of single-cell Kaggle notebooks that turn one reference image into a 3D asset. Both open a Gradio studio with an image uploader, a four-view gallery, a mesh preview, and a downloadable ZIP. You can import the resulting GLB into Blender or open it in a glTF 2.0 viewer.

## Choose a notebook

| Notebook | What it produces | Approximate model downloads |
| --- | --- | ---: |
| [Mesh-only notebook](image-to-blender-studio-dual-gpu.ipynb) | An untextured GLB for editing geometry, blocking out a scene, or assigning your own materials | 22.2 GB |
| [Paint notebook](image-to-blender-studio-paint.ipynb) | A GLB with an embedded color texture when Paint completes, or vertex colors when the fallback is used | 40.5 GB |

These are weight download sizes in decimal GB. Allow extra disk space for temporary files, repositories, extensions, caches, and exports. Disk download size and GPU memory usage measure different things.

## Run on Kaggle

1. Import the notebook you want to use into Kaggle.
2. In Notebook Settings, enable **Internet** and select a **GPU** accelerator. Choose **GPU T4 ×2** if it is available and you want parallel generation of some Qwen views.
3. Run the notebook’s single code cell. Open the **Image to Blender Studio** link printed beneath it.
4. Sign in with username **studio** and the generated password shown in the cell output. Keep the link and password private.
5. Upload one image. If several objects are visible, enter a short description in **Object to reconstruct**, such as “the green armchair.”
6. Start with **Draft** and select **Generate 3D asset**. The first run installs dependencies and downloads model weights. Repeated runs can reuse cached weights while the Kaggle session stays alive.
7. Download the ZIP when the studio reports **Ready**. Extract it and open **asset.glb**.

The upload limit is 12 MB. A missing studio link usually calls for checking the notebook cell output for the Gradio launch or share tunnel error. Keep the Kaggle session active while using the studio.

## Prepare a reference image

Use one clearly visible subject with space around its outline and a simple background. For a character, use a full-body reference that shows the head, hands, clothing, and shoes. The studio can start from any angle, but a clear front or three-quarter view gives the model more visible evidence about the subject.

Qwen generates a canonical front view first, then uses it as an extra reference for the left, back, and right views. The generated views can invent details hidden by the original crop, such as shoes, the back of a shirt, or the far side of a pair of glasses. Inspect the four views for matching proportions, clothing, colors, and orientation. Disagreement between views can carry into the 3D mesh. If you see extra objects or conflicting details, try a cleaner crop, a more precise subject description, or another seed.

## What runs under the hood

1. **Qwen-Image-2.1 INT8**, hosted in ComfyUI, generates the four object views. It uses a Qwen3-VL 8B INT8 text encoder and a Qwen-Image-2.1 VAE.
2. **Hunyuan3D-2mv FP16** combines the four labeled views into one mesh. The notebook downloads its weights in the background while Qwen runs.
3. In the **Paint notebook**, mesh cleanup checks for tall, thin panels detached from the central geometry. Hunyuan3D-Paint Turbo and Hunyuan3D-Delight then use the generated front image to create and bake a texture. The Paint and Delight weights download separately from the mesh weights, so the mesh can start before the Paint download finishes.
4. The studio packages the GLB and the four generated views into a ZIP.

Hunyuan’s mesh worker uses an isolated Transformers 4.57.6 installation for compatibility with the DINOv2 checkpoint. The private dependency directory is added only to Hunyuan worker processes.

## Controls and quality presets

Both notebooks expose the same reconstruction presets. **Draft** is a good way to find out whether the source image and generated views are suitable. Higher settings increase computation and can preserve more geometry when the input views agree.

| Preset | Qwen view size | Qwen steps per view | Hunyuan mesh steps | Mesh extraction resolution |
| --- | ---: | ---: | ---: | ---: |
| Draft | 512 px | 12 | 20 | 196 |
| Balanced | 640 px | 18 | 30 | 256 |
| Detail | 832 px | 25 | 40 | 384 |

**Seed** accepts an integer from 0 to 4294967295. Leave it blank to choose a random seed. Save the seed from a successful run if you want to repeat the same generation settings. Software changes and differences between GPU environments can still affect the result.

## How the GPUs are used

With two visible GPUs and the default **PARALLEL_QWEN = True**, GPU 0 makes the front view first. Once the front is ready, GPU 0 makes the left and right views while GPU 1 can generate the back view. The Hunyuan mesh worker then uses GPU 1. The Paint worker also uses GPU 1 after the mesh completes. A single-GPU runtime runs these stages on its available GPU.

Each parallel Qwen worker loads its own in-memory model instance. If running both workers puts too much pressure on memory, set **PARALLEL_QWEN = False** in the cell and restart the studio. This uses one Qwen worker and still allows Hunyuan to use GPU 1 on a two-GPU runtime.

A GPU can show 0% during model downloads, extension compilation, CPU-based UV processing, or a gap between GPU operations. Check the current stage alongside GPU utilization. The Paint notebook includes a **Live GPU status** panel. In Kaggle you can also run **!nvidia-smi** in a separate cell when the kernel is available.

## Mesh-only workflow

Choose the mesh-only notebook if you want to work on shape first. The ZIP contains the untextured **asset.glb** plus the four generated PNGs. The PNGs are reference images. The GLB contains the geometry, ready for material work or retopology in Blender. This workflow skips Paint, UV unwrapping, and the Paint and Delight downloads.

Use **Retry mesh from a previous run** with the run’s 12-character ID to reuse all four saved views in the same Kaggle session. An incomplete set of views cannot be used for this retry.

## Paint workflow

The Paint notebook first builds the same type of mesh, then tries to create a texture map. It validates that the Paint result contains an embedded base-color texture before offering the textured GLB. You can upload **asset.glb** directly to an online viewer that supports glTF 2.0 materials.

Paint starts in GPU-first mode by default. If CUDA reports an out-of-memory error, the notebook retries with CPU offload. GPU-assisted UV unwrapping is also enabled by default. **CuMesh** runs mesh clustering on CUDA, and **xatlas** calculates and packs the UV charts on the CPU. The notebook falls back to CPU xatlas if CuMesh fails to install or run. The first CuMesh build needs a CUDA compiler and may take time. Diagnostics reports the UV path used.

If Paint fails or emits no new inference output for five minutes, the notebook can bake **vertex colors** from the four generated views onto the completed mesh. The studio labels the resulting color method. For the fallback GLB, use an online viewer that supports glTF vertex colors. Fine details may look softer because vertex-color detail depends on mesh density.

**Retry mesh from a previous run** can reuse both the four views and a completed mesh from the same session. If no saved mesh exists, it rebuilds the mesh using the four saved views.

## Open the result

The exported ZIP has this layout in both workflows.

```text
blender_asset.zip
├── asset.glb
└── views/
    ├── front.png
    ├── left.png
    ├── back.png
    └── right.png
```

In Blender, choose **File → Import → glTF 2.0**, then select **asset.glb**. Switch to **Material Preview** to inspect color in the Paint version. For an online viewer, unzip the download and select the GLB file itself. If only the studio preview is blank, try the downloaded file in Blender or another viewer before assuming the export failed.

Generated meshes are a starting point for editing. Inspect the face, glasses, thin parts, and silhouette from several angles. Check the mesh topology and scale before committing to a shot or animation. A character may need retopology, an armature, or texture cleanup for close-ups and posing.

## Diagnose a failed or slow run

Open **Diagnostics** in the studio and download **Full run log** when available. Save the 12-character run ID shown on failure. Run folders live under **/kaggle/working/blender_assets/**, while model caches and some setup logs live under **/kaggle/temp/image-to-blender/**.

| What you see | What to check |
| --- | --- |
| No GPU detected | Confirm a GPU accelerator is enabled. Use **!nvidia-smi -L** to see which devices Kaggle exposes. |
| First run stays in setup | Check the Kaggle cell output and the ComfyUI setup log. Initial installs and model downloads can be sizable. |
| Images finish and the GPU goes idle | Look for a Hunyuan weight download or dependency installation. These run before GPU mesh inference begins. |
| Mesh generation fails | Check **hunyuan-setup.log** and **hunyuan-inference.log**. The first records dependency setup, and the second records model loading and mesh execution. |
| Paint is quiet | Check the live phase and elapsed time. UV unwrap, normal and position rendering, denoising, texture projection, and gap filling have separate timing messages. |
| CuMesh build fails | Check **cumesh-build.log**. The notebook can continue with CPU xatlas. |
| GPU-first Paint runs out of VRAM | Check **hunyuan-paint-gpu-oom.log** and the following offload attempt. |
| A colored GLB opens gray | Check whether the studio reported a Paint texture or vertex colors, then confirm the viewer supports that color format. |
| Unwanted slabs or distorted details remain | Check the four source views and inspect the GLB from the side. The Paint notebook’s cleanup covers certain detached panels. Connected artifacts and warped anatomy may require a different reference or manual editing. |

The notebooks have been checked for valid notebook structure and tested locally with simulated workers for several generation, retry, and fallback paths. Actual performance, CuMesh build compatibility, and Paint reliability depend on the Kaggle runtime and should be verified with a live run.

## Upstream projects

- [Kaggle Notebooks and accelerators](https://www.kaggle.com/docs/notebooks)
- [ComfyUI](https://github.com/Comfy-Org/ComfyUI)
- [Qwen-Image-2.1 weights for ComfyUI](https://huggingface.co/Comfy-Org/Qwen-Image-2.1)
- [Hunyuan3D-2mv weights and model terms](https://huggingface.co/tencent/Hunyuan3D-2mv)
- [Hunyuan3D-2 Paint and mesh pipeline](https://github.com/Tencent-Hunyuan/Hunyuan3D-2)
- [CuMesh](https://github.com/JeffreyXiang/CuMesh)

