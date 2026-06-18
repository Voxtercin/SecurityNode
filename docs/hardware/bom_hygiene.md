# BOM Hygiene Checklist — SecurityNode

> **Design-stage cleanup list — not yet fabricated.** The board is drawn in Altium but
> not built. This is a punch-list of data-quality problems in the Altium LiveBOM
> (`assets/SecurityNode.BomDoc`) that should be resolved **before any real fabrication
> or quoting**, plus the open hardware-confirmation items that still need to be checked
> against the Altium source. Each item below is verified against
> `assets/SecurityNode.BomDoc`; the schematic netlist (`assets/SecurityNode.pdf`) is
> authoritative on any disagreement. See `docs/application_node/DESIGN-AUDIT.md` §7 for
> the broader audit context.
>
> Nothing here changes the *design* — these are catalog/metadata defects and unresolved
> per-designator assignments, not topology errors.

---

## 1. Data-quality defects to correct in the BOM

- [ ] **SMF5.0CA TVS mislabeled "TVS 12V".** The user comment / `Comment` field reads
  `TVS 12V`, but the part is a **5 V** bidirectional TVS (the LCSC description itself
  says *"… Bidirectional **5V** SOD-123FL ESD and Surge Protection"*). Correct the
  comment to a 5 V label (e.g. `TVS 5V`) so the value column does not contradict the
  part. *(This is the same part family used at the VBUS input and on the PIR line — see
  the open diode-split item in §2.)*

- [ ] **100 nF capacitor: description says 0201, footprint is 0603.** The `Description`
  reads *"100nF ±20% 6.3V Ceramic Capacitor X5R **0201**"* while `Footprint=0603_cap`.
  The placed footprint (0603) is what the board uses; fix the description so the package
  size matches the footprint (0603), or re-confirm the intended package against the
  Altium source and make both fields agree.

- [ ] **F1 has a stray "YTL " prefix on the part number.** The manufacturer part number
  proper is `1206L075/16V` (Littelfuse Inc.), but the catalog item's design-item ID,
  library reference, and `Comment`/user-comment fields all carry `YTL 1206L075/16V`.
  Strip the stray `YTL ` prefix so the part is unambiguously the Littelfuse `1206L075/16V`
  PTC.

- [ ] **J2 (Mini-USB-B) lists the wrong manufacturer.** Manufacturer is given as
  `EDAC Inc.` while the part number is `KH-MINI-SMT-5P-Cu` — that MPN is a **Shenzhen
  Kinghelm** part (the LCSC datasheet link in the same record points to
  *Kinghelm KH-MINI-SMT-5P-Cu*, C2688795). Correct the manufacturer to **Kinghelm**
  (keep the MPN and supplier part number, which are already correct).

- [ ] **Non-standard / corrupted manufacturer-part-number keys.** Several parts use keys
  that do **not** match the standard `Manufacturer` / `Manufacturer Part Number` columns
  and therefore drop out of a standard BOM export. Normalize these to the standard keys:
  - [ ] `MF=` / `MP=` instead of `Manufacturer` / `Manufacturer Part Number` on:
    **U1 (AP2112K-3.3TRG1)**, **U4 (TPS2553DBVR)**, **U5 (SN74LVC2G66DCTR)**,
    the **ESP32-CAM** module, **D4 (BAT54S)**, and **LS1 (KPEG242-5V)** (the latter
    also has an **empty** `MF=` manufacturer field to fill in).
  - [ ] **U2 (USBLC6-2P6)** uses corrupted **`Manufacturer Pad Number=`** and
    **`Supplier Pad Number=`** keys (should be **"Part Number"**, not "Pad Number");
    these will silently disappear from a standard export. Rename to
    `Manufacturer Part Number` / `Supplier Part Number`.
  - [ ] **F1** uses `Supplier=` / `Supplier Part Number=` (no trailing `1`) where most
    rows use `Supplier 1` / `Supplier Part Number 1`. Make the supplier keys consistent
    so all rows export under the same columns.

- [ ] **YAGEO 100 k resistor links to the wrong datasheet.** The 100 k part
  (`RC0603FR-07100KL`, YAGEO) has a `ComponentLink1URL` that points to an **84.5 Ω**
  part (`…CR0603FA84R5G…`). Repoint the datasheet/product link to the correct
  100 kΩ part so anyone cross-checking the component lands on the right datasheet.

- [ ] **The BOM has no Designator column — it is a value catalog.** Every row is a
  *value/part class* (e.g. "10k 1/10W", "1k 1/10W", "100nF 6.3V"), with **no R#/C#/D#
  designators** mapping a board reference to a specific MPN. Before fabrication or
  quoting, add a **designator → MPN assignment** (a real per-reference BOM), so that
  each placed component on the PCB resolves to exactly one purchasable part. This is a
  prerequisite for the per-designator confirmation items in §2.

---

## 2. Open hardware-confirmation items (to be confirmed against the Altium source)

These are **not** known errors — they are details that the BOM/schematic exports do not
pin down unambiguously, and that must be confirmed against the Altium source (a high-res
schematic render or the project files) before the BOM is finalized.

- [ ] **D1 / D3 / D6 TVS-vs-Schottky designator split — to be confirmed.** The BOM holds
  exactly **2× SMF5.0CA** (TVS 5 V) and **2× PSBD1DF40V1H** (Schottky 40 V / 1 A). Only
  **D2 = Schottky** (door clamp, schematic-confirmed) and **D5 = LED** (XL-2012SURC) are
  **certain**; **D4 = BAT54S** (PIR clamp) is also certain. The remaining split among
  **D1 / D3 / D6** (which is the VBUS input TVS, which is the 5 V-rail Schottky, which is
  the PIR-line TVS) is **inferred from topology only** — the per-designator package is not
  printed in the BOM. Confirm the exact D1/D3/D6 assignment against the Altium source
  before publishing the diode table. *(to be confirmed)*

- [ ] **Per-designator R / C → MPN assignment — to be confirmed.** Because the BOM is a
  value catalog (§1), the one-to-one mapping of each resistor/capacitor reference to its
  exact MPN is inferred from value class plus net context. The load-bearing **values** are
  confirmed (e.g. R3/R4 = 22 R, R8/R20/R21 = 1 k, R9/R12 = 100 R, 10 k pull-ups,
  R11/R18/R19 = 100 k, R13 = 21k5, R15 = 330 R), but the precise split among same-value
  parts (e.g. *which* 100 nF cap is *which* reference) is inference. Confirm each
  designator's MPN against the Altium source, or add a designator column to the BOM so the
  mapping is explicit. *(to be confirmed)*

---

*Source of truth: `assets/SecurityNode.pdf` (schematic netlist, authoritative on
disagreement) and `assets/SecurityNode.BomDoc` (Altium LiveBOM). Cleanup items in §1 are
metadata/data-quality fixes that do not alter the design; items in §2 are unresolved
assignments to confirm before fabrication.*
