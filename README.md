# Zapper-One-Wicked-Cricket-Prototype

Zapper_Proto_All_Unlocked.p2s is a savestate to load into PCSX2 with everything unlocked.

zappersave.lee is the save data of the Steam version of the game with everything unlocked.

This is how to trigger the end of a level if glitched or if eggs are missing :

<img width="1137" height="828" alt="DEBUG" src="https://github.com/user-attachments/assets/5ee9bb59-428f-4e92-b8b4-692f38264f11" />


<img width="1119" height="834" alt="Main_Title" src="https://github.com/user-attachments/assets/78314245-a3d5-4c56-a973-61436dd1baf3" />


# PS2 Camera Manipulation — Technical Findings

---

## Stage 1 — Camera Angle Source ❌ NOT FOUND

A single float (or yaw + pitch pair) somewhere in RAM:

- Updated every frame by game logic (controller input or scripted path)
- Ideal manipulation target  
- Modifying this value allows the game’s internal math to generate a perfectly rotated camera
- No side effects when altered correctly
- Will be located by tracing backwards from Stage 2 code

---

## Stage 2 — VU0 Matrix Construction ✅ PARTIALLY FOUND

**Code block:** `00203678–002036AC`

| Address   | Instruction                           | Role |
|------------|----------------------------------------|------|
| 00203678   | `vmulax.xyzw ACC, vf10, vf16x`        | Multiply X column |
| 0020367C   | `vmaday.xyzw ACC, vf11, vf16y`        | Accumulate Y |
| 00203680   | `vmaddaz.xyzw ACC, vf12, vf16z`       | Accumulate Z |
| 00203688   | `vmulax.xyzw ACC, vf10, vf17x`        | Next row X |
| 002036A4   | `jr ra`                               | End of function (return) |
| 002036A8   | `sqc2 vf17, ...`                      | Store row to `002CC000` |
| 002036AC   | `sqc2 ...`                            | Store row to `002CC010/20` |

**Function behavior:**

- Reads Stage 1 angle
- Computes `sin` / `cos`
- Assembles a **3×3 rotation matrix**

`jr ra` at `002036A4` confirms this is the tail of a dedicated camera matrix function.

---

## Stage 3 — Intermediate Matrix Buffer ✅ FOUND (but not final)

| Address   | Content | Status |
|------------|----------|--------|
| 002CC000   | Row 0 — X axis vector | ✅ Found |
| 002CC010   | Row 1 — Y axis vector (nearly constant) | ✅ Found |
| 002CC020   | Row 2 — Z axis vector | ✅ Found |

### Observations

- Matrix repeats every `0x80` bytes  
- Indicates **double / circular buffering** (2 alternating frames)
- Writing rotated values here caused a **black screen** at all tested angles:
  - 10°
  - 20°
  - 45°
  - 90°

### Conclusion

This is **not the final output matrix**.

It is an intermediate buffer that is further transformed before rendering.

---

## Stage 4 — View × Projection Combine ❌ NOT FOUND

Expected behavior:

- Code reads `002CC000`
- Multiplies it with a projection matrix
- Produces a final combined matrix elsewhere in RAM

### Debug Strategy

- Set a memory write breakpoint on `002CC000`
- Capture the responsible PC
- Trace:
  - **Forward** → final matrix address
  - **Backward** → Stage 1 angle source

---

## Stage 5 — GIF / VIF DMA → Graphics Synthesizer ❌ NOT FOUND

Final matrix likely uploaded via:

- **VIF1 DMA → VU1**
- Or directly via **GIF PATH3**

Hardware register range:

```
0x10000000–0x1000FFFF
```

Identifying this address would reveal the **true last matrix** before pixels are drawn.

---

# Next Step

1. Set a memory write breakpoint on:
   ```
   002CC000 (size: 16 bytes)
   ```
2. Run execution
3. Catch the Stage 2 → 3 `sqc2` write
4. Trace the source VU register backwards
5. Locate the Stage 1 raw angle float
6. Modify that single value for clean, stable camera rotation
