# Pipeline Breakdown

Two workflows, two very different jobs. UltraTex takes a photo + a 3D mesh and wraps a texture around
it. UniForge3D takes a photo (or a few) and builds the 3D mesh from scratch, then optionally textures it.
They're designed to chain together: UniForge3D makes the geometry, UltraTex paints it.

---

## 1. UltraTex — `MostAadTech_UltraTex_workflow.json`

**What it does:** Takes a reference photo + an existing GLB mesh, generates multi-view texture images
using Flux-based diffusion, then bakes them onto the mesh and exports a textured GLB.

**Version tag in file:** UltraTex Modular FAST v0.2.3

### Models required

| Role | File | Type |
|---|---|---|
| Diffusion backbone | `flux-2-klein-base-9b-Q4_0.gguf` | UNet (GGUF quantized) |
| Text encoder | `Qwen3-8B-Q6_K.gguf` | CLIP/text encoder (GGUF, Flux2 type) |
| VAE | `flux2-vae.safetensors` | Variational autoencoder |
| Texture LoRA — albedo | `ultratex_flux2.safetensors` | LoRA |
| Texture LoRA — metallic/roughness | `ultratex_flux2_mr.safetensors` | LoRA |
| Decoder | `ultratex_flux2_decoder.pt` | Custom UltraTex decoder |
| Background remover | `birefnet.safetensors` | BiRefNet segmentation |

### Node-by-node walkthrough

```
INPUTS
  [Node 53] UTXLoadMeshV2          ← Load a .glb from disk (example: deer.glb)
  [Node 3]  LoadImage              ← Load the reference photo (example: piebald deer)

STAGE 1 — PREPARE
  [Node 30] LoadBackgroundRemovalModel  ← Load BiRefNet
  [Node 31] RemoveBackground            ← Cut the subject out of the reference photo → MASK
  [Node 44] UTXUnwrapMeshUVV25          ← UV-unwrap the mesh (4096 atlas, comfy_gpu mode, 16px padding)
  [Node 45] UTXPrepareTextureV25        ← Align mesh + reference + mask into a shared view coordinate
                                           system. Outputs: prepared_source, normal_views, aligned_ref,
                                           aligned_mask. This is the "source of truth" for all generates.

STAGE 2A — GENERATE ALBEDO VIEWS
  [Node 7]  CLIPLoaderGGUF         ← Load Qwen3-8B text encoder
  [Node 6]  VAELoader              ← Load flux2-vae
  [Node 5]  UnetLoaderGGUF         ← Load flux-2-klein-base-9b (diffusion model)
  [Node 8]  UTXTextEncode          ← Encode positive prompt (empty by default — texture-guided)
  [Node 47] UTXTextEncode          ← Encode negative/MR prompt (also empty by default)
  [Node 9]  UTXGenerateTextureViewsV23  ← Run albedo generation
                                           · LoRA: ultratex_flux2.safetensors (strength 1)
                                           · Decoder: ultratex_flux2_decoder.pt
                                           · Task: albedo
                                           · Resolution: 2048 per view, 12 views
                                           · Steps: 4, guidance: 4, denoise: 1
                                           · sparse_retention: 0.2 (keeps 20% of sparse views)
                                           · extra_angle_mode: guided
                                           · tiled_decode: true (saves VRAM)
                                   → Outputs: UTX_VIEW_COLLECTION + contact_sheet preview + SETUP

STAGE 2B — GENERATE METALLIC/ROUGHNESS VIEWS (runs in parallel with 2A)
  [Node 20] UTXGenerateTextureViewsV23  ← Same setup, but:
                                           · LoRA: ultratex_flux2_mr.safetensors
                                           · Task: mr (metallic/roughness)
                                   → Outputs: UTX_VIEW_COLLECTION + MR_SETUP

STAGE 3 — BAKE
  [Node 49] UTXFastSurfaceTextureBaker  ← Projects all views onto the UV atlas
                                           · Takes albedo_setup + albedo_views + mr_setup + mr_views
                                           · Texture size: 4096
                                           · Completion: surface_fill (fills gaps)
                                           · Upscale: fast_detail
                                   → Outputs: MESH, base_color image, roughness, metallic

OUTPUT
  [Node 13] SaveGLB               ← Save the textured mesh as a .glb
  [Node 51] PreviewImage          ← Preview the baked base_color texture
  [Node 10] PreviewImage          ← Preview albedo contact sheet
  [Node 21] PreviewImage          ← Preview MR contact sheet
  [Node 11] UTXSaveViews          ← Save raw view images to disk (prefix: "ultratex_views")
```

### Key design choices to understand

- **No text prompt needed.** The texture is entirely guided by the reference photo. The text encoder is
  loaded but both prompts are blank — UltraTex uses the photo as the condition.
- **Two separate generate passes.** Albedo (color) and MR (metallic/roughness) are separate LoRAs on
  the same Flux2 backbone, run as separate nodes. They share the same prepared_source.
- **4 steps is intentional.** This is the "FAST" variant. The note in the file says no GPU benchmark
  was run — it's trading quality for speed.
- **The decoder (`ultratex_flux2_decoder.pt`) is proprietary** to this project. It's a custom head that
  converts the diffusion latents into texture-appropriate outputs. Not swappable with a standard VAE.
- **tiled_decode: true** — decodes the 2048 views in tiles to reduce VRAM usage.

---

## 2. UniForge3D — `uniforge3d_ultra_high.json`

**What it does:** Takes 1–3 reference photos and reconstructs a full 3D mesh from them using a
diffusion-based 3D generation model (the xibushu/UniForge3D model, which is a TRELLIS-style sparse
voxel approach). Then optionally applies UV texturing. Output is a .glb.

### Models required

| Role | File/ID | Type |
|---|---|---|
| Main 3D model | `xibushu/UniForge3D` (HuggingFace) | Multi-stage diffusion + VAE |
| Upscaler (pre-process) | `4xNomos8kDAT.pth` | Real-ESRGAN style upscaler |

### Node-by-node walkthrough

The workflow has named groups which make the stages readable:

```
INPUTS (up to 3 images, run through upscale + resize preprocessing)
  [Node 157] LoadImage            ← Primary reference image (example: bag)
  [Node 263] LoadImage            ← Optional 2nd view (disabled in this workflow)
  [Node 270] LoadImage            ← Optional 3rd view (disabled)

  [Node 153] UpscaleModelLoader   ← Load 4xNomos8kDAT upscaler
  [Node 158] ImageUpscaleWithModel ← 4x upscale primary image
  [Node 165] ImageResizeKJv2      ← Resize to 1024×1080, pad black, keep proportion
  [Node 160] BatchImagesNode      ← Combine up to 3 images into a batch (slots 1+2 optional)

GROUP: "VGGT Image Encoding"
  [Node 90]  UniForge3DModelLoader     ← Load the xibushu/UniForge3D model (GPU 0, seed 1)
  [Node 89]  UniForge3DPreprocessImages ← Normalize/tokenize image batch for the model
  [Node 97]  UniForge3DEncodeDino512   ← Extract DINO visual features at 512 resolution
  [Node 98]  UniForge3DEncodeDino1024  ← Extract DINO visual features at 1024 resolution
  → DINO features are the "visual prompt" — they replace text conditioning entirely.

GROUP: "Uniforge" — Sparse structure generation
  [Node 129] UniForge3DSparseStructure ← First diffusion pass: generate a 32³ and 64³
                                          sparse voxel structure
                                          · 40 steps, guidance 7.5, sigma_min 0.00001
                                          · Uses DINO_FEATURES_512 + image batch
  [Node 217] UniForge3DSparsePostProcess ← Clean up sparse_structure_32 (fill holes,
                                            keep only shell)
  [Node 215] UniForge3DSparsePostProcess ← Clean up sparse_structure_64

  [Node 257] UniForge3DShape512Mesh    ← Second diffusion pass: decode sparse structure
                                          into a 512-resolution mesh
                                          · 30 steps, guidance 7.5
                                          · Uses cleaned sparse coords + DINO_FEATURES_512
                                          → mesh512 + raw coords

GROUP: "Clean up 512 Representation with FaithC"
  [Node 230] MeshReconstructWithQuad   ← Remesh to quads (1024 resolution, remove floaters)
  [Node 223] Trellis2MeshToTrimesh     ← Convert to trimesh format
  [Node 224] CuMeshSimplify            ← Decimate to 200k faces
  [Node 225] Trellis2TrimeshToMesh     ← Convert back to MESH
  [Node 220] RotateMesh                ← Rotate -90° on X axis (Y-up to Z-up correction)
  [Node 260] SaveGLB                   ← Save the clean 512 mesh as intermediate .glb

  [Node 195] UniForge3DCoordinateUpsample ← Upsample the voxel coords (64³ resolution)
  [Node 216] UniForge3DSparsePostProcess  ← Clean the upsampled coords (disabled in workflow)

GROUP: "Uniforge" continued — 1024 shape refinement
  [Node 267] UniForge3DShape1024       ← Third diffusion pass: refine to 1024 resolution
                                          · 25 steps, guidance 7.5
                                          · Uses DINO_FEATURES_1024 + upsampled coords
                                          → shape1024_vae_latent + view_assignment

  [Node 183] UniForge3DVaeDecoder      ← Decode latent into a 2048-resolution mesh

GROUP: "White mesh"
  [Node 245] MeshReconstructWithQuad   ← Remesh at 2048 resolution (2M face budget)
  [Node 117] WTiVoNativeMeshToMesh     ← Watertight voxel repair (2048³, 12M proxy points)
  [Node 136] Trellis2MeshToTrimesh     ← Convert to trimesh
  [Node 137] CuMeshExportMesh          ← Export untextured mesh as .glb

GROUP: "Texture" (disabled in this workflow — nodes have mode: 4)
  [Node 251] Uniforge3DGenerateTexture ← Texture the mesh using DINO features + view assignment
  [Node 250] CuMeshExportMesh          ← Export textured .glb
```

### Key design choices to understand

- **Three diffusion passes, each at increasing resolution.** Sparse structure (64³) → Shape512 → Shape1024.
  This is a coarse-to-fine strategy. Each pass conditions on the previous one plus the DINO image features.
- **DINO, not CLIP.** There's no text prompt anywhere. The model conditions on visual features extracted
  by a DINO vision transformer (two resolutions: 512 and 1024). This is how it understands "what the
  object looks like" from the photo.
- **FaithC / WTiVo are mesh repair tools** also by Mstafa-awad. They make the mesh watertight and
  clean up floaters/holes before export. These are the same author as UltraTex — the two repos are
  designed to work together.
- **The texture step is disabled.** `mode: 4` in ComfyUI means "bypassed/muted." The texturing nodes
  (`Uniforge3DGenerateTexture`) are there but turned off — the intended workflow is to take the
  untextured mesh and run it through UltraTex separately.
- **Multi-view support.** The `BatchImagesNode` accepts up to 3 images. With multiple views from
  different angles, the reconstruction is significantly better. Most of the disabled input nodes are
  the 2nd and 3rd view loaders.

---

## How they chain together

```
Your photo(s)
     │
     ▼
[UniForge3D]
  Image → 4x upscale → DINO encode
       → SparseStructure (64³)
       → Shape512 mesh + remesh/repair
       → Shape1024 latent → VAE decode
       → 2048 mesh → watertight repair
       → untextured .glb
     │
     ▼
[UltraTex]
  untextured .glb + original reference photo
       → BiRefNet mask
       → UV unwrap
       → Flux2 albedo generate (12 views @ 2048)
       → Flux2 MR generate (12 views @ 2048)
       → Bake to 4096 atlas
       → textured .glb ✓
```

---

## What models you need to run locally

| Model | Where to get it | Notes |
|---|---|---|
| `flux-2-klein-base-9b-Q4_0.gguf` | HuggingFace | Quantized Flux2 UNet. ~5GB. |
| `Qwen3-8B-Q6_K.gguf` | HuggingFace | Qwen3 text encoder for Flux2. ~6GB. |
| `flux2-vae.safetensors` | HuggingFace | Standard Flux2 VAE. |
| `ultratex_flux2.safetensors` | UltraTex repo / HuggingFace | Proprietary LoRA. |
| `ultratex_flux2_mr.safetensors` | UltraTex repo / HuggingFace | Proprietary LoRA. |
| `ultratex_flux2_decoder.pt` | UltraTex repo | Proprietary. Required. |
| `birefnet.safetensors` | HuggingFace (ZhengPeng7/BiRefNet) | Background removal. |
| `xibushu/UniForge3D` (HF model) | HuggingFace | Multi-stage 3D diffusion. Large. |
| `4xNomos8kDAT.pth` | OpenModelDB | 4x upscaler. |

### Potential local open-source swaps (to research)

| Component | What it does | Possible alternatives |
|---|---|---|
| Flux2 backbone | Diffusion backbone for texture gen | Flux.1-dev (fp8/GGUF), any Flux2 compatible |
| BiRefNet | Background removal | RMBG-2.0, IS-Net |
| 4xNomos8kDAT | Upscaler | 4xRealWebPhoto, 4xUltraSharp, RealESRGAN |
| UniForge3D itself | 3D generation | TRELLIS (open-source upstream it builds on), TripoSG, InstantMesh |
| UltraTex LoRA/decoder | Texture baking | This is the hard one — no direct drop-in. Could explore ProjectTexture, TEXTure, or training your own. |
