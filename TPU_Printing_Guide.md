# Printing Inserts in TPU — Walls, Shells and Low-Density Infill

This guide covers how to slice and print an insert exported from the Insert Designer so that it comes off the bed as soft as the app designed it and in one piece. Everything here was learned the hard way on this project: five sliced retries, a sheared HalfArc, a 215 g "solid" plate that should have weighed a fraction of that, and a hub ring that turned out to be mostly air.

The baseline is the app's default target: **95A TPU, 0.4 mm nozzle, direct-drive extruder, OrcaSlicer (tested on 2.4.2), 0.2 mm layers.** Setting names below are Orca's; Bambu Studio and PrusaSlicer use similar names.

---

## 1. The one idea to keep in mind

**The app decides how soft the insert is. The slicer's job is to print that geometry faithfully and to fill any sealed band as lightly as possible.**

Almost every lattice shape (Grid, Honeycomb, HalfArc, S-bend, Chevron, and so on) is built from ribbons that are already the width of two extrusion lines. The slicer cannot add infill inside a 0.8 mm ribbon, so the infill slider does nothing to those parts. The only places where slicer infill matters are the regions the app deliberately leaves hollow for the slicer to fill: the **Solid** shape, rows set to **Solid** under Vary-shape-by-layer, and the hub band itself. Those are the regions this guide is mostly about.

---

## 2. Recommended settings at a glance

| Setting (Orca name) | Value | Why |
|---|---|---|
| Wall generator | Arachne | Handles the variable-width ribbons cleanly; a narrow hub band fills with beads instead of gaps |
| Wall loops (`wall_loops`) | **2** | Matches the app's 0.8 mm ribbons (two 0.4 mm lines); see §3 |
| Top shell layers | **0** | Leaves the lattice open at the top face; see §4 |
| Bottom shell layers | **0** | Same at the bottom face |
| Sparse infill density | **3–10 %** (3.8 % proven) | Only affects sealed bands; see §5 |
| Sparse infill pattern | Gyroid (proven) or TPMS-D | Isotropic, no crossing lines, compresses evenly |
| Layer height | 0.2 mm | All layer-count numbers in these guides assume this |
| Line widths | ~0.42 outer / 0.45 inner | Orca defaults for a 0.4 nozzle; fine as-is |
| Brim | On, **inner and outer** | Nothing else holds the part down with 0 bottom shells; see §7 |
| Supports | Off | Inserts are self-supporting when laid flat |

Print temperature, fan and speed are material-dependent. Start from the filament maker's numbers and keep speeds modest (roughly 20–35 mm/s for walls on a direct drive). Dry the filament before a long print; wet TPU strings badly and prints weak layer bonds, which is exactly where inserts fail.

---

## 3. Why only two wall loops

**The ribbons are already two beads wide.** The default wall is 0.8 mm, which Arachne prints as exactly two 0.4 mm lines. Raising wall loops to 4 or 6 does *not* make those ribbons thicker — wall count is a ceiling, not a quota. What it *does* change is everything wider than the ribbons: sealed bands grow an extra bead on each side, and the hub band gets more perimeter, which quietly stiffens the part.

**Two loops also sets how wide a hub band the slicer can fill solidly.** Two loops at ~0.43 mm reach about 0.87 mm in from each edge, so roughly 1.7 mm of any band is wall. On a real print, a 2.5 mm hub band came out 99.6 % wall beads (Arachne widens the lines to close the gap). A 9.6 mm hub band came out as a thin skin on each side around a core of 3.8 % infill — and the arc roots buried 7 mm deep in that band were anchored to nearly nothing. That is what sheared the HalfArc at the bore.

![Hub band width vs wall loops](images/tpu_hub_band_walls.png)

![Narrow vs. widened hub band in real G-code](images/slicer_hub_band_gcode.png)

*The same thing in real toolpaths from this project: a narrow band (left) is all wall beads; widened (middle, right) it turns into two walls around an infill core.*

**Rule of thumb:** keep the Hub width near the 2.5 mm default at 2 wall loops. If you need a wider hub, switch the app's **Root anchor** to **Wall-depth** and set its **Wall loops** control to the same number you use in the slicer, so the app keeps each root inside the printed walls instead of pushing it into the sparse core.

---

## 4. Why remove the top and bottom layers

With the default top/bottom shells on, the slicer caps every sealed band with several solid layers. That turns an open-cell lattice into a closed drum: the skins take load in tension across the face, the trapped air resists compression, and the insert ends up noticeably stiffer than the app's geometry alone suggests.

Setting both to **0** lets the infill run straight out to both faces. Only the walls stay continuous from bottom to top, which is what you want: the walls carry the shape, the infill supports it lightly, and the whole thing collapses and recovers like foam.

![Top/bottom shells on vs off](images/tpu_shells_on_off.png)

Two side effects to plan for:

- **The first layer is mostly infill and wall lines, not a solid sheet**, so it has much less bed contact. Use a brim (see §7).
- **The faces are open.** That is the point, but it means a face-down print shows the infill pattern, and any debris inside the tyre can work into the cells. Neither has caused a problem in use.

---

## 5. Low infill with gyroid or TPMS

This section applies to the **Solid** shape, Solid rows, and the hub band. For the lattice shapes, leave infill at whatever you like; it has no effect.

**Start at 3.8 %.** It is the value used on every successful print in this project. Workable range is roughly 3–10 %: lower is softer and faster, higher is firmer. Adjust softness in the app first (wall, cells, Shape fill, build preset) and use infill density only to fine-tune a Solid build.

**Do not use 100 % infill to "strengthen the hub".** It doesn't reach the hub. On one real plate, all 3,172 m of solid infill landed in the outer sealed band, none in the hub ring (which was already solid wall beads). The plate weighed 215 g and took 18 hours. Dropping back to 3.8 % saved roughly 24 g and 4 hours *per insert* and brought the compliance back. If the hub needs to be stronger, the fix is in the app (narrower band, Wall-depth anchor), not the infill slider.

**Pattern choice:**

- **Gyroid** is the proven pattern here. It has no crossing lines within a layer, so there are no stiff nodes, and it is close to equally compliant in every direction.
- **TPMS-D** (and TPMS-FK on newer Orca builds) is the same family of smooth, triply-periodic surfaces and should behave similarly. It has not been printed on this project yet, so treat it as an experiment and compare against a gyroid print.
- Avoid grid, cubic, triangles and lines for inserts. Crossing lines make stiff nodes, and straight lines give a strongly directional feel.

At very low density the infill lines span long distances between walls. That is fine in 95A TPU at the speeds above; if you see sagging or stringy infill, slow the sparse-infill speed before raising density.

---

## 6. Hub band and bore chamfer

The bore edge interacts with the walls and the arc roots, and it has caused more failed prints than anything else in this project. The short version:

- **Square bore edge** is the safest choice and needs nothing special.
- **If you need a lead-in chamfer** to seat the insert over the beadlock ring, keep it small: **Profile size ≤ 0.3 × Hub width** in the default (Deep) root mode — 0.75 mm on a 2.5 mm hub. Past that, the chamfer cuts under each arc's root tip for a few layers at each face and those tips print as loose beads in mid-air.
- **Need a bigger chamfer?** Use **Wall-depth** root anchoring, which moves the roots outboard of the chamfer while keeping them inside the printed walls. The app's **Root in walls** readout and its stranded-root warning tell you whether you are safe.

![Root anchor set to Wall depth in the app](images/app_root_anchor_walldepth.png)

*Sidewall band with a 6 mm hub, Chamfer 1.5 mm and Root anchor = Wall depth; Slicer wall loops matches the slicer's 2.*
- **Re-export any older STL with a non-square bore.** Exports made before the inside-out hub-ring fix (HANDOFF §52) cause the slicer to cut a notch where every root should join the hub.

The companion guide, *Checking the Slicer Preview for Disconnected Pieces*, shows exactly how to spot each of these before you print.

---

## 7. Getting it to stay on the bed

With zero bottom shells, the part's grip on the bed comes from wall lines and sparse infill lines only.

- **Brim on both the inner and outer surfaces.** The "outer only" brim setting puts nothing inside the bore, so the hub ring — the smallest footprint on the plate — has nothing holding it.
- **Slow the first layer** (around 15–20 mm/s) and keep the first-layer line width at or slightly above normal.
- **Mind the bed surface.** TPU can bond very strongly to smooth PEI; a light glue-stick layer works as a release agent. Textured PEI usually releases cleanly once cool.

---

## 8. Quick-start recipe

1. In the app: keep Hub width at 2.5 mm, Bore edge Square (or Chamfer ≤ 0.75 mm), and check that **Loose pieces** reads 0 and no red warnings are showing.
2. Hard-reload the app page (Ctrl+Shift+R) before exporting, so you are not exporting from a cached old version.
3. In Orca: Arachne, 2 wall loops, top and bottom shells 0, gyroid at 3.8 %, 0.2 mm layers, brim inner + outer, supports off.
4. Slice, then run through the preview checks in the companion guide.
5. Print, and note the file name — the app encodes shape, size, bore, shore, wall and fill in it, which makes comparing prints much easier.

---

## 9. Troubleshooting

| Symptom | Most likely cause | Fix |
|---|---|---|
| Insert much stiffer than expected | Top/bottom shells left on, or high infill | Shells 0, infill 3–10 % |
| Very heavy, very long print time | 100 % infill in a sealed band | Drop to 3.8 %; the hub is already solid walls |
| Ribs shear off at the bore | Wide hub band with sparse core; roots anchored in infill | Hub back to ~2.5 mm, or Wall-depth anchor with matching wall loops |
| Arcs look detached from the hub in the slicer | Old inside-out export, or chamfer too large | Re-export; keep Profile ≤ 0.3 × hub or use Wall-depth |
| Part lifts at the bore | Brim set to outer only, or no brim | Brim inner + outer |
| Stringing, rough walls, weak layer bonds | Wet filament, or too fast | Dry the filament; slow down |
| Raising wall loops didn't firm up the lattice | Ribbons are already two beads | Change wall/cells/fill in the app instead |
