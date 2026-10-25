# Understanding Comfy Texture Pipeline + Animation Pipeline

Study repo exploring two ComfyUI node packages — one focused on high-res texture baking (UltraTex) and one on 3D-aware image-to-3D generation (UniForge3D). The goal is to understand how each pipeline is structured, then adapt them for use with local open-source models.

---

## Source Repositories

| Project | Author | License | Folder |
|---|---|---|---|
| [UltraTex-ComfyUI-nodes](https://github.com/Mstafa-awad/UltraTex-ComfyUI-nodes) | Mstafa-awad | GPL-3.0 | `workflows/ultratex/` |
| [ComfyUI-UniForge3D](https://github.com/Easymode-ai/ComfyUI-UniForge3D) | Easymode-ai | Apache-2.0 | `workflows/uniforge3d/` |

---

## What's in Here

```
workflows/
  ultratex/
    MostAadTech_UltraTex_workflow.json       ← base texture workflow
    MostAadTech_UltraTex_8k.json             ← 8K variant
    MostAadTech_UltraTex_8k_new.json         ← updated 8K variant
  uniforge3d/
    uniforge3d_ultra_high.json               ← high-quality 3D generation workflow

LICENSE-GPL3-UltraTex.txt                    ← GPL-3.0 (UltraTex)
LICENSE-Apache2-UniForge3D.txt               ← Apache-2.0 (UniForge3D)
THIRD_PARTY_NOTICES-UltraTex.md             ← upstream third-party notices
```

---

## Licensing Notes

- **UltraTex** is GPL-3.0. Any derivative work you distribute must also be GPL-3.0 and preserve copyright notices.
- **UniForge3D** is Apache-2.0. You can use and modify it freely; keep the license and mark your changes.
- This repo is **private / personal study** — no distribution, so the copyleft terms don't kick in.

---

## Goals

- [ ] Map out each workflow node by node
- [ ] Identify which models each pipeline depends on
- [ ] Find drop-in replacements using locally available open-source models
- [ ] Mirror the core logic for texture + animation work at BrickHeart Studio
