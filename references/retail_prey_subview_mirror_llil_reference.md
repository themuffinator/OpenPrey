# Retail PREY.exe Subview/Mirror LLIL Reference

## Provenance
- Binary analyzed: `C:\Program Files (x86)\R.G. Mechanics\Prey\PREY.exe`
- SHA-256: `0DE3C58A82B6B1EF9C04160DCDF68A763345A0B8C12A88A11BB4B8783C278711`
- Architecture: PE32 / i386
- Primary tooling: `llvm-objdump`, `pefile` relocation/xref scan, manual control-flow reconstruction.

## Method
1. Locate retail subview-related cvar strings in-image (`r_skipSubviews`, `r_subviewOnly`, `r_lockSurfaces`).
2. Resolve static data xrefs via base-relocation entries to identify cvar object pointers.
3. Disassemble code ranges at xref sites and reconstruct LLIL-style behavior.

## Identified Anchors
- `r_skipSubviews` string: `0x7C46AC`, cvar object pointer used at `0x849ABC`.
- `r_subviewOnly` string: `0x7C479C`, cvar object pointer used at `0x849B58`.
- `r_lockSurfaces` string: `0x7C5994`, cvar object pointer used at `0x84A7F0`.

## LLIL-Style Reconstruction

### A) `R_RenderView` core path (function region around `0x4E3480`)
Key disasm (retail):
- `0x4E3509: call 0x4F14E0`  ; generate subviews
- `0x4E3512: mov eax, [0x849B58]`
- `0x4E3517: cmp dword ptr [eax+0x24], 0`
- `0x4E351B: jne 0x4E3559`    ; skip main-view draw when subviewOnly is set

Pseudo-LLIL:
```text
subviews = call R_GenerateSubViews()
if subviews:
    if r_subviewOnly.GetBool():
        return

call DrawMainViewPasses()
```

Observed retail property:
- No evidence of `r_subviewMaxDepth`/`r_subviewMaxCount` style gating in this path.

### B) `R_GenerateSubViews` gate (function region around `0x4F14E0`)
Key disasm (retail):
- `0x4F14E0: mov eax, [0x849ABC]`
- `0x4F14E5: cmp dword ptr [eax+0x24], 0`
- `0x4F14E9: jne 0x4F1632` ; early out

Pseudo-LLIL:
```text
if r_skipSubviews.GetBool():
    return false

for each drawSurf in viewDef->drawSurfs:
    if HasSubview(drawSurf.material):
        if R_GenerateSurfaceSubview(drawSurf):
            any = true
return any
```

### C) `R_GenerateSurfaceSubview` subview-class branch (function region around `0x4F1100`/`0x4F13DD`)
Key disasm (retail):
- `0x4F13DD: mov eax, [ebp+0x8]` ; class selector
- `0x4F13E3: je 0x4F1454`
- `0x4F13E8: je 0x4F1416`
- default -> `0x4F13EC: call 0x4F0B30`
- `0x4F1405..0x4F1414`: xor/toggle mirror flag byte
- `0x4F14A3: call 0x4E3480` ; render subview

Pseudo-LLIL:
```text
switch (subviewClass) {
  case SC_PORTAL:
    parms = BuildPortalSubview(...)
    break
  case SC_PORTAL_SKYBOX:
    parms = BuildSkyboxSubview(...)
    break
  case SC_MIRROR:
  default:
    parms = R_MirrorViewBySurface(drawSurf)
    if (!parms) return false
    parms->isMirror = parms->isMirror XOR tr.viewDef->isMirror
    break
}

parms->superView = tr.viewDef
parms->subviewSurface = drawSurf
R_RenderView(parms)
return true
```

## Retail-vs-Current Implications
- Retail behavior clearly includes `r_skipSubviews` and `r_subviewOnly` in subview/render paths.
- Retail mirror path uses dedicated mirror-view build + mirror-flag xor before rendering subview.
- Retail reference recovered here does **not** show additional subview depth/count cvar gates in the `R_RenderView` path.

## Notes
- This document is LLIL-style reconstruction from retail machine code and xrefs; it is not a direct Binary Ninja IL dump.
