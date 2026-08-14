# Antenna design and parameter calculations

## Metadata

- **Sources**: design conversation (August 2026); NXP NT3H2111/NT3H2211 datasheet; NXP AN11578; inductance model from Mohan et al. (1999)
- **Note date**: 2026-08-14
- **Purpose**: Record the antenna and energy-harvesting calculation trail for this credit-card NFC board so later revisions (and other 13.56 MHz coils) can reuse it
- **Status**: First lot fabricated at JLCPCB (2-layer, 1 oz, 0.8 mm, black mask, lead-free HASL). Chip-only assembly reads at about 1 cm. Factory NDEF format issue found and fixed.

## Executive summary

The board is an ISO/IEC 7810 ID-1 card (85.6 mm x 54 mm, R3.18 corners) with an NXP NT3H2111W0FT1 (NTAG I2C plus, SO8) and a 6-turn planar coil. The chip fixes the antenna capacitance, so the coil is sized for a free resonance slightly above 13.56 MHz.

A from-scratch Mohan/Wheeler estimate for a 4-turn full-card loop gave about 2.4 uH. The layout instead reused a proven 6-turn 24 mm x 41.5 mm coil. The same formula gives about 2.19 uH, or about 14.8 MHz with the chip's 50 pF plus a 3 pF parasitic allowance. A 6.8 pF C0G trim capacitor (C1) across LA/LB would pull that to about 14.0 MHz.

Unloaded Q is far too high for the 848 kHz load-modulation sidebands, but the chip's internal load drops Q to about 10, so no damping resistor is required. The LED hangs from harvested `VOUT` into the open-drain field-detect pin (`FD`) and is sized from AN11578's load table.

Bench results matched the model: with C1 empty the card reads at about 1 cm. Write failures were not a coupling problem; NTAG I2C parts ship without a factory NDEF capability container.

## 1. Architecture and part selection

**Chip.** NT3H2111W0FT1 (NTAG I2C plus 1k, SO8, 1.27 mm pitch). Chosen over NTAG21x inlays because it is a solderable IC, and over the XQFN8 option because SO8 is hand-solderable. Suffix: NT3H2111 (die), W0 (wafer rev), FT1 (SO8). Orderable reel code is typically `NT3H2111W0FT1X`.

The datasheet lists on-chip antenna capacitance $C_\mathrm{IC}$ between LA and LB as 44 pF min, **50 pF typ**, 56 pF max, at 13.56 MHz and $V_\mathrm{LA-LB} = 2.4\,\mathrm{V_{RMS}}$.

**Final circuit** (simplified from a two-LED reference):

| Ref | Role |
| --- | --- |
| C1 | 6.8 pF C0G across LA/LB (resonance trim; unpopulated on board 1) |
| C2 | 150 nF X7R from VOUT to GND (datasheet: 150 nF typical, 220 nF absolute max, close to pins 7 and 2) |
| R1 | 0 Ω VOUT-to-VCC jumper (self-supply from harvested energy; leave off to disable) |
| R2 | 100 kΩ pull-up from VOUT to FD |
| LED1 + R3 | Red 0603 LED and 2.7 kΩ in series from VOUT into FD |

`FD` is open-drain and, with the default `FD_ON = 00b`, asserts low when an RF field is present. The LED therefore lights while a phone is coupled, with no transistor. A reference NPN stage inverted that logic and was removed.

## 2. Antenna design

### 2.1 Capacitance budget

The chip capacitance is fixed, so the coil is designed to it:

$$
C_\mathrm{total} = C_\mathrm{IC} + C_\mathrm{para} + C_\mathrm{tune} = 50 + 3 + 0 = 53~\mathrm{pF}
$$

The 3 pF term is an allowance for trace and coil parasitics. $C_\mathrm{tune}$ starts at 0 and is added after measurement.

### 2.2 Target frequency

A phone reader coil loads the tag and pulls resonance down, so the free tag is tuned high. Target: $f_0 = 14.0~\mathrm{MHz}$.

### 2.3 Required inductance

$$
f_0 = \frac{1}{2\pi\sqrt{LC}} \quad\Rightarrow\quad L = \frac{1}{(2\pi f_0)^2\, C_\mathrm{total}}
$$

With $f_0 = 14.0~\mathrm{MHz}$ and $C = 53~\mathrm{pF}$:

1. $2\pi f_0 = 8.7965\times 10^{7}~\mathrm{rad/s}$
2. Squared: $7.7378\times 10^{15}$
3. Times $C$: $7.7378\times 10^{15} \times 53\times 10^{-12} = 4.1010\times 10^{5}$
4. Reciprocal: **$L = 2.44~\mu\mathrm{H}$**

### 2.4 From-scratch geometry (Mohan / modified Wheeler)

Square planar spiral (Mohan et al., 1999):

$$
L = \frac{K_1 \mu_0 n^2 d_\mathrm{avg}}{1 + K_2 \rho},
\qquad
d_\mathrm{avg} = \frac{d_\mathrm{out}+d_\mathrm{in}}{2},
\qquad
\rho = \frac{d_\mathrm{out}-d_\mathrm{in}}{d_\mathrm{out}+d_\mathrm{in}}
$$

For a square, $K_1 = 2.34$ and $K_2 = 2.75$. A rectangular coil is mapped to the equal-perimeter square. For a full-card loop 76 mm × 44 mm: perimeter 240 mm, so $d_\mathrm{out} = 60~\mathrm{mm}$. With $w = s = 0.4~\mathrm{mm}$ (pitch 0.8 mm):

| Turns $n$ | Band $b$ | $d_\mathrm{in}$ | $d_\mathrm{avg}$ | $\rho$ | $L$ | Verdict |
| --- | --- | --- | --- | --- | --- | --- |
| 4 | 2.8 mm | 54.4 mm | 57.2 mm | 0.0490 | 2.37 μH | 3% under 2.44 μH; accept and trim |
| 5 | 3.6 mm | 52.8 mm | 56.4 mm | 0.0638 | 3.53 μH | 45% over; reject |

Turn count is a coarse, highly sensitive knob. A trim capacitor only absorbs a few percent.

### 2.5 Adopted geometry

The layout reused a proven open-source coil (extracted from a KiCad reference as DXF) instead of the full-card 4-turn loop:

- 6 turns
- Outer size 24.0 mm × 41.5 mm
- $w = 0.30~\mathrm{mm}$, $s = 0.45~\mathrm{mm}$ (pitch 0.75 mm)
- Total copper length 651.5 mm
- Inner terminal brought out by one 0.3 mm bottom-layer jumper that crosses the turns at a right angle through two 0.4 / 1.2 mm vias

Same formula: perimeter 131 mm so $d_\mathrm{out} = 32.75~\mathrm{mm}$; $b = 6(0.30)+5(0.45) = 4.05~\mathrm{mm}$; $d_\mathrm{in} = 24.65~\mathrm{mm}$; $d_\mathrm{avg} = 28.70~\mathrm{mm}$; $\rho = 0.1411$. Numerator $K_1\mu_0 n^2 d_\mathrm{avg} = 3.038\times 10^{-6}$; denominator $1.388$; **$L = 2.19~\mu\mathrm{H}$**.

Predicted free resonance with 53 pF:

$$
f_0 = \frac{1}{2\pi\sqrt{2.19~\mu\mathrm{H}\cdot 53~\mathrm{pF}}} \approx 14.8~\mathrm{MHz}
$$

High side, as that reference intended (it shipped with no trim cap).

### 2.6 Trim capacitor

To move a measured resonance $f_\mathrm{meas}$ to $f_\mathrm{target}$ at fixed $L$:

$$
\Delta C = \frac{1}{(2\pi)^2 L}\left(\frac{1}{f_\mathrm{target}^2} - \frac{1}{f_\mathrm{meas}^2}\right)
$$

For 14.8 MHz → 14.0 MHz with $L = 2.19~\mu\mathrm{H}$: $(2\pi)^2 L = 8.6457\times 10^{-5}$; $1/f_t^2 - 1/f_m^2 = 5.366\times 10^{-16}$; **$\Delta C = 6.2~\mathrm{pF}$**. Populate **C1 = 6.8 pF C0G**, and keep 3.3 / 4.7 / 10 pF on the bench for trimming.

Board 1 left C1 empty. That is consistent with the 14.8 MHz prediction and the measured ~1 cm read range.

## 3. Losses, Q, and bandwidth

### 3.1 DC resistance and skin depth

Copper length $\ell \approx 0.652~\mathrm{m}$, 1 oz ($t = 35~\mu\mathrm{m}$). Using the design-stage $w = 0.4~\mathrm{mm}$ numbers: $A = 1.4\times 10^{-8}~\mathrm{m}^2$, $R_\mathrm{DC} = \rho_\mathrm{Cu}\,\ell / A \approx 1.2$ to $1.5~\Omega$. Skin depth at 13.56 MHz ($\rho_\mathrm{Cu} \approx 1.72\times 10^{-8}~\Omega\cdot\mathrm{m}$):

$$
\delta = \sqrt{\frac{\rho_\mathrm{Cu}}{\pi f \mu_0}} \approx 17.9~\mu\mathrm{m}
$$

Traces are wide and thin, so effective area is about $2\delta(w+t)$. Going from 1 oz to 2 oz only grows $(w+t)$ from 435 μm to 470 μm: about **8% less resistance, not worth 2 oz**. Widening the trace is the real lever ($w$ 0.4 → 0.6 mm cuts $R$ by about 31%), at the cost of re-running Section 2.4.

### 3.2 Q and the 848 kHz sidebands

Unloaded, using the 4-turn estimate $L = 2.37~\mu\mathrm{H}$ and $R = 1.5~\Omega$ (the adopted 2.19 μH coil is in the same range):

$$
Q = \frac{\omega L}{R} = \frac{8.52\times 10^{7}\cdot 2.37\times 10^{-6}}{1.5} \approx 135
\quad\Rightarrow\quad
\mathrm{BW} = \frac{f_0}{Q} \approx 100~\mathrm{kHz}
$$

Load-modulation sidebands sit at $\pm 848~\mathrm{kHz}$, well outside that bandwidth. The chip's internal equivalent load (order $R_p \sim 2~\mathrm{k}\Omega$) dominates:

$$
Q_\mathrm{loaded} = \frac{R_p}{\omega L} \approx \frac{2000}{202} \approx 10
\quad\Rightarrow\quad
\mathrm{BW} \approx 1.4~\mathrm{MHz}
$$

No damping resistor is required. A 0 Ω placeholder in the loop is cheap insurance if a later spin needs it.

## 4. Energy harvesting and the LED

NXP AN11578 (Rev. 1.0, 2016) Table 1, measured on a Class 5 antenna in an ISO/IEC 10373-6 field, gives the minimum field and the minimum `VOUT` during modulation for a constant current load:

| Load (mA) | $H_\mathrm{min}$ (A/m) | $V_\mathrm{OUT,min}$ (V) |
| --- | --- | --- |
| 1 | 1.2 | 2.7 |
| 2 | 1.9 | 2.5 |
| 3 | 2.7 | 2.4 |
| 4 | 3.5 | 2.2 |
| 5 | 4.3 | 2.0 |

ISO/IEC 14443-2 specifies a PCD field of 1.5 to 7.5 A/m, so the 1 mA row is inside the guaranteed window. AN11578 also states that current drawn from `VOUT` is taken from the reader and shortens range.

LED branch (`VOUT` → LED → R3 → `FD`, $V_\mathrm{OL} \approx 0.2~\mathrm{V}$, red $V_f \approx 1.9~\mathrm{V}$ at sub-mA):

$$
I = \frac{V_\mathrm{OUT} - V_f - V_\mathrm{OL}}{R_3} = \frac{2.7 - 1.9 - 0.2}{2700} \approx 220~\mu\mathrm{A}
$$

That sits well below the 1 mA table row. White, blue, and green LEDs are excluded: their $V_f$ (about 2.7 to 3.2 V) meets or exceeds the harvested rail. Every milliamp from `VOUT` is subtracted from load modulation, so brightness trades directly against range. 2.7 kΩ is the conservative first-spin value; 470 Ω to 1 kΩ parts were ordered as brighter spares.

C2 rule from the datasheet: 150 nF typical, 220 nF absolute max, tight to pins 7 and 2. It holds `VOUT` through load-modulation dips. Too large and the rail cannot charge during field ramp-up.

## 5. Layout rules

- **Keep-out.** No copper of any kind, either layer, inside the coil window or under the turns. The single perpendicular bottom jumper is the only exception. A stray pour here was the predicted number-one failure mode; the released Gerbers were audited and the window is clean.
- **Feed length.** LA/LB traces stay short and paired. Frequency shift from extra inductance is small:

$$
\frac{\Delta f}{f_0} \approx -\frac{1}{2}\frac{\Delta L}{L}
$$

20 mm of feed (about 20 nH) moves $f_0$ by only about 65 kHz. The real reasons to keep the feed tight are the uncontrolled pickup loop and coupling into the high-Q node, not detuning.
- C1 straddles LA/LB at the chip. C2 is within about 5 mm of pins 7 and 2. Vias are tented. The coil is under soldermask (cosmetic; an ENIG-exposed coil is equally valid but forces that finish).
- Board: 0.8 mm (wallet feel), ID-1 outline, R3.18 corners, copper-to-edge at least 0.5 mm.

## 6. QR code on silkscreen

A 22-byte URL fits QR Version 2 at ECC M (26-byte capacity). Modules per side $n = 4V + 17 = 25$; a 4-module quiet zone on each side gives $N = 33$. At 12 mm printed width the module is $12/33 \approx 0.36~\mathrm{mm}$. Silkscreen resolution is about 0.15 mm, so 0.4 to 0.5 mm modules are the comfortable floor.

White silk on black mask is an inverted code. Modern phone decoders accept that; it was checked against the rendered Gerber.

## 7. Bench results versus the model (2026-08-14)

- Chip-only assembly (no C1, no passives): stable reads at **about 1 cm**. Consistent with the predicted 14.8 MHz detune: marginal but usable. C1 was left unpopulated on board 1.
- Reads always worked; writes always failed on iOS with "tag not supported". That was **not** coupling. The NTAG I2C family ships with a blank capability container (no factory NDEF format), and iOS cannot format it through an NDEF session. One raw-session "Format as NDEF" (Smart NFC app, Advanced) per board fixes it. After that, tag status is Read/Write and NDEF writes succeed. This is a one-time production step for every remaining unit.

## Open items

- [ ] Populate C1 = 6.8 pF on one board and A/B the read range against the bare-chip 1 cm baseline
- [ ] Populate R1 / R2 / R3 / C2 / LED and confirm FD-driven lighting (flicker during polling, solid during a held read)
- [ ] Boards 2 to 10: solder, NDEF format, write URL, verify tap-to-open, optional password protect
- [ ] If range matters later: NanoVNA plus a coupling loop to measure true $f_0$, then trim with Section 2.6

## References

Verified before inclusion:

1. NXP Semiconductors. *NT3H2111/NT3H2211 NTAG I2C plus: NFC Forum T2T with I2C interface, password protection and energy harvesting* (datasheet NT3H2111_2211). $C_i$ (LA/LB) = 44 / 50 / 56 pF; VOUT hold-up capacitor 150 nF typical, 220 nF max; FD is open-drain. <https://www.nxp.com/docs/en/data-sheet/NT3H2111_2211.pdf>
2. NXP Semiconductors. *AN11578: Energy harvesting with the NTAG I2C and NTAG I2C plus*, Rev. 1.0, 1 February 2016. Table 1: 1 mA load needs $H_\mathrm{min} = 1.2~\mathrm{A/m}$ at $V_\mathrm{OUT,min} = 2.7~\mathrm{V}$. <https://cache.nxp.com/docs/en/application-note/AN11578.pdf>
3. S. S. Mohan, M. Hershenson, S. P. Boyd, and T. Lee. Simple accurate expressions for planar spiral inductances. *IEEE Journal of Solid-State Circuits*, 1999. DOI: [10.1109/4.792620](https://doi.org/10.1109/4.792620). Semantic Scholar paperId `c237b4c0cb773ab2141037c0cc8dcd8dda155225`. Square-coil coefficients used here: $K_1 = 2.34$, $K_2 = 2.75$.
