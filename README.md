# Zapper-One-Wicked-Cricket-Prototype

Zapper_Proto_All_Unlocked.p2s is a savestate to load into PCSX2 with everything unlocked.

zappersave.lee is the save data of the Steam version of the game with everything unlocked.

This is how to trigger the end of a level if glitched or if eggs are missing :

<img width="1137" height="828" alt="DEBUG" src="https://github.com/user-attachments/assets/5ee9bb59-428f-4e92-b8b4-692f38264f11" />


<img width="1119" height="834" alt="Main_Title" src="https://github.com/user-attachments/assets/78314245-a3d5-4c56-a973-61436dd1baf3" />


# Camera Manipulation — Technical Findings

------------------------------------------------------------------------

## Stage 1 --- Camera Euler Angles ⚠️ PARTIALLY LOCATED

A set of 3 floats (yaw, pitch, roll) passed directly to
`bdSetViewOrientation`:

-   Passed via float registers **f12, f13, f14** at call time
-   Computed from upstream angle logic involving `sinf` calls and
    arithmetic on s0-relative data
-   `bdSetViewOrientation` converts these angles into a rotation matrix

**Ideal manipulation target:** modifying these register values or their
source memory produces a correctly rotated camera with no side effects.

Located by tracing backwards from the `jal bdSetViewOrientation` call at
`001DB600`.

------------------------------------------------------------------------

## Stage 2 --- VU0 Matrix Construction ✅ FOUND

Code block: `00203648–002036AC`

  -----------------------------------------------------------------------
  Address                 Instruction                     Role
  ----------------------- ------------------------------- ---------------
  00203648                lqc2 vf14, 0x0(a2)              Load source
                                                          matrix row 0
                                                          from a2

  0020364C                lqc2 vf15, 0x10(a2)             Load source
                                                          matrix row 1

  00203650                lqc2 vf16, 0x20(a2)             Load source
                                                          matrix row 2

  00203654                lqc2 vf17, 0x30(a2)             Load source
                                                          matrix row 3

  00203658--00203694      vmulax/vmaday/vmaddaz/vmaddw    Full VU0 matrix
                                                          multiply (4x3
                                                          rows x 3
                                                          columns)

  00203698                sqc2 vf14, 0x0(a0)              Store row 0 -\>
                                                          002CC000

  0020369C                sqc2 vf15, 0x10(a0)             Store row 1 -\>
                                                          002CC010

  002036A0                sqc2 vf16, 0x20(a0)             Store row 2 -\>
                                                          002CC020

  002036A8                sqc2 vf17, 0x30(a0)             Store row 3 -\>
                                                          002CC030

  002036A4                jr ra                           Return
  -----------------------------------------------------------------------

-   Input matrix loaded from `a2` (source buffer)
-   Output written to `a0 = 002CC000`
-   Called from `001DB6DC` inside UpdateScene / SetupView chain

------------------------------------------------------------------------

## Stage 3 --- Intermediate Matrix Buffer ✅ FOUND (Not Final)

  Address    Content                                   Status
  ---------- ----------------------------------------- --------
  002CC000   Row 0 - X axis vector                     Found
  002CC010   Row 1 - Y axis vector (nearly constant)   Found
  002CC020   Row 2 - Z axis vector                     Found
  002CC080   Alternate buffer slot (double buffer B)   Found

### Observations

-   Matrix repeats every `0x80` bytes -\> confirmed double/circular
    buffer (2 alternating frames)
-   `s0 = 002CC080`, `s1 = 002CC140`, `s2 = 002CC300`
-   Writing rotated values here caused black screen at tested angles:
    10, 20, 45, 90 degrees
-   Values at `002CC080` decode as near-identity matrix row (0.9999,
    0.00185, 0.00147)

**Conclusion:** This is an intermediate buffer, not the final rendering
matrix.

------------------------------------------------------------------------

## Stage 4 --- View x Projection Pipeline ✅ PARTIALLY FOUND

Call chain inside `001DB630–001DB750`:

  -----------------------------------------------------------------------
  Address                 Instruction                     Role
  ----------------------- ------------------------------- ---------------
  001DB644                jal bmMatIdentity               Initialize
                                                          matrix

  001DB66C                jal bmMatMultiply               First multiply

  001DB6EC                jal bmMatCopy                   Copy matrix

  001DB700                jal bmMatMultiply               Second multiply

  001DB710                jal bmMatMultiply               Third multiply

  001DB720                jal bmMatMultiply               Fourth multiply

  001DB730                jal bmMatMultiply               Fifth multiply

  001DB600                jal bdSetViewOrientation        Sets camera
                                                          Euler angles

  001DB680                jal bdCalcPerspProjectionMatrix Builds
                                                          projection
                                                          matrix

  001DB68C                jal bdSetProjectionMode         Sets projection
                                                          mode
  -----------------------------------------------------------------------

-   `s0 = 002CC000` used as working matrix base
-   `s2 = 002CC300`, `s3 = 002CBFC0`, `s4 = 002CC0C0` used as additional
    matrix slots

------------------------------------------------------------------------

## Stage 5 --- GIF / VIF DMA -\> Graphics Synthesizer ❌ NOT FOUND

-   Final matrix likely uploaded via VIF1 DMA -\> VU1 or directly via
    GIF PATH3
-   Hardware register range: `0x10000000–0x1000FFFF`
-   Identifying this confirms the final matrix before rasterization

------------------------------------------------------------------------
