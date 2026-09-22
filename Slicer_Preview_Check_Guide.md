# Checking the Slicer Preview for Disconnected Pieces

An insert can look perfect in the app's 3D view and in the slicer's model view and still print with pieces floating in mid-air. The app checks connectivity on the whole model from above; the slicer prints it one layer at a time. A root tip that is attached at mid-height can be cut loose on the first or last few layers, and nothing will warn you except the layer preview.

This guide walks through the checks that would have caught every disconnection found on this project, in the order worth doing them. It takes about five minutes per insert and saves a multi-hour print.

Instructions use OrcaSlicer 2.4.2. Bambu Studio is nearly identical; PrusaSlicer has the same tools under slightly different names.

---

## 1. Before you slice: rule out a bad export

Two of the "disconnected" reports on this project were not geometry problems at all. Check these first.

**Hard-reload the app before exporting.** Press **Ctrl+Shift+R** on the Insert Designer tab. Browsers can keep serving a cached copy of the app, so a fix can be sitting on disk while you keep exporting the old, broken geometry. This cost a full round of printing once.

![Stale export vs. current app, with the G-code of the stale export](images/slicer_stale_export_gcode.png)

*Left: the arc's top surface in a stale export (red) sits 3.7 mm above the hub it should be inside; the current app (blue) keeps it within the hub. Right: layer 2 of that stale export's G-code — the root tip prints as an isolated loop. Real project data.*

**Re-export any STL with a non-square bore made before HANDOFF §52.** Older exports built the hub ring inside-out whenever Bore edge was not Square or Step-the-bore was on. The slicer then *subtracts* every place an arc overlaps the hub instead of joining them (see §5 below). There is no way to fix an old file in the slicer — export it again.

**Read the app's own readouts before you leave it.**

- **Loose pieces** should read 0.
- **Root in walls** should be at or near 100 % if you use the Wall-depth root anchor.
- Any red note about a **stranded root** means the bore chamfer reaches past the arc roots at the faces. Fix it in the app now; the slicer will only confirm it.

The app's checks are necessary but not sufficient: the loose-piece check looks at the model from above, so it cannot see a piece that is only detached on a few layers near the faces. That is what the slicer checks below are for.

![App readouts — a clean build](images/app_readouts_ok.png)

*A clean build: Loose pieces 0, Root in walls 100 % (Wall-depth anchor), Face land 1.60 mm, and a neutral note.*

![App readouts — stranded-root warning](images/app_readouts_stranded.png)

*The retry4 settings (hub 2.4 mm, Chamfer 1.5 mm, Deep anchor): Root in walls turns red and the note names the root tip radius, the bevel void and the number of bad layers. Click the note to jump to the control it names.*

---

## 2. Set up the preview

1. Slice, then switch to the **Preview** tab.
2. Set the view type to **Line Type** so every extrusion is coloured by feature (outer wall, inner wall, sparse infill, brim, and so on).
3. Switch to the **top view** (numpad 1 or the view-cube) and zoom in on the bore.
4. Use the **vertical layer slider** on the right to move through layers. With the slider selected, the up/down arrow keys step one layer at a time, which is how you should move through the first and last layers.
5. Keep the legend open. Clicking a feature in the legend hides or shows it, which is the key trick in §4.

Do these checks in **Preview**, not the Prepare/model view. The model view draws overlapping shells as if they were joined, and the most convincing "it looks fine" screenshots on this project came from that view.

---

## 3. Check the face layers for orphan beads

This is the check that matters most, and the one that caught the retry4 failure.

When the bore has a chamfer or roundover, the ring gets thinner at each face. If the bevel reaches further out than the tip of each arc's root, those tips have nothing under or around them for a few layers. They print as tiny separate beads — one per arc — sitting inside the bevel with no connection to anything.

![Orphan root tips at a face layer](images/slicer_orphan_root_tips.png)

**Where to look:** step through the **first 10 layers and the last 10 layers** one at a time, zoomed in on the hub. The problem is always at the faces, never in the middle.

![Which layers to check](images/slicer_layers_to_check.png)

**How many bad layers to expect** if the bevel is too big:

```
bad layers per face ≈ (bore radius + bevel size − root tip radius) ÷ layer height
safe when   bevel size ≤ 0.3 × Hub width     (Deep root mode)
```

On retry4 (hub 2.4 mm, bevel 1.5 mm, 0.2 mm layers) that predicted 4 bad layers per face; the slice showed 3. On retry3 (bevel 0.9 mm) it predicted and showed none.

**What you are looking for:** small isolated dots or short loops just inside the hub ring, each lined up with an arc, and not touching the ring. Even one is a failed print at that spot.

![Orphan root tips in the real retry4 G-code](images/slicer_orphan_beads_gcode.png)

*The retry4 slice, same hub sector at four layers. The dots beside each arc on the face layers are the orphan root tips; by layer 7 they are inside the ring again. Parsed from the G-code rather than captured from the Orca window, so the colours follow the legend above, not Orca's.*

---

## 4. Hide sparse infill and check what each root is anchored to

An arc root can be touching the hub and still be anchored to almost nothing, if the part of the hub it sits in is sparse infill.

1. In the legend, **hide Sparse infill.**
2. Go to a mid-height layer and look at the hub band.
3. Every arc root should run into **wall beads** (outer/inner wall colour). If the band shows a wall on each side with a wide empty gap in the middle once infill is hidden, and the roots end in that gap, the roots are anchored in air.

![Hub band walls vs sparse core](images/tpu_hub_band_walls.png)

That is how the 9.6 mm hub band failed: two 0.87 mm walls, 4 %-density infill between them, and the roots buried 7 mm into the infill. A 2.5 mm band at 2 wall loops prints as solid wall beads and has no core to fall into.

**Fix:** narrow the Hub width back toward 2.5 mm, or switch Root anchor to **Wall-depth** and set its Wall loops to match the slicer. Raising infill to 100 % does not help here — see the TPU Printing Guide.

![Narrow vs. widened hub band in real G-code](images/slicer_hub_band_gcode.png)

*Real toolpaths: at 0.9 mm (left) the band is nothing but wall beads; widened (middle, right) it becomes two walls around an infill core (purple). Hide that infill in the Preview legend and whatever is left between the walls is what the roots are anchored to.*

---

## 5. Look for a notch where an arc meets the hub

If each arc visibly **stops short of the hub** — roughly 2 mm of gap — and the hub band has a small **V-shaped bite** taken out of it exactly where the arc should enter, you are looking at an inside-out export.

![V-notch from an inside-out hub ring](images/slicer_vnotch_inside_out.png)

What happened: the hub ring's surface faced the wrong way, so the slicer treated it as negative space and cut away every overlap with it. The app looked fine because it draws both sides of every surface. Raising Shape fill sometimes hid it on other shapes, which made it look like a shape-specific problem when it wasn't.

**Fix:** hard-reload the app and export again. Exports made after HANDOFF §52 do not have this problem. If a fresh export still shows the notch, save the STL and the settings — that would be a new bug.

> 📷 **Still wanted — notch in the Orca preview.** The figure above is rendered from the exported mesh; a real Line Type crop of one arc meeting the hub band (from a pre-§52 export) would be a useful addition.

---

## 6. Quick sweep through the middle

After the targeted checks, drag the layer slider slowly from bottom to top once, watching the hub and the outer rim. You are looking for anything that appears or disappears abruptly: a ring that doesn't close all the way round on the first few layers, a rib that starts a few layers late, or material showing up inside the bore. Geometry poking into the bore hole was a real bug once (fixed in HANDOFF §49); if you see it on a fresh export, report it.

---

## 7. What to do when you find a problem

| What you see | Where | Fix |
|---|---|---|
| Isolated dots inside the hub ring | First/last few layers | Smaller bevel (≤ 0.3 × hub), Square bore, or Wall-depth root anchor |
| Roots ending in empty space with infill hidden | Mid-height, hub band | Narrow the hub to ~2.5 mm, or Wall-depth anchor with matching wall loops |
| Arc stops ~2 mm short, V-notch in hub | Any layer | Hard-reload the app and re-export |
| Hub ring doesn't close on the first layers | First layers | Check Face land in the app; smaller bevel |
| Material inside the bore hole | Any layer | Re-export; if it persists, save the file and report it |
| Everything looks fine in model view but not in Preview | — | Trust Preview |

---

## 8. Sharing screenshots

When you post a preview screenshot to ask for help, crop it before sharing:

- **Crop to the preview pane.** The title bar and project tab show the full file path, which usually includes your Windows user name.
- **Hide the printer and account panel.** Printer names, network printer IPs and cloud account names appear in the top bar and device tab.
- **Keep what's useful:** the layer number from the slider, the Line Type legend, and the file name of the STL (the app's file names encode the shape, size, bore, shore, wall and fill, which is exactly what someone needs to reproduce the issue).

---

## 9. Pre-print checklist

- [ ] App hard-reloaded before export (Ctrl+Shift+R)
- [ ] No non-square-bore STLs from before HANDOFF §52
- [ ] App: Loose pieces 0, no stranded-root warning
- [ ] Preview → Line Type → top view
- [ ] First 10 and last 10 layers stepped one at a time: no orphan beads
- [ ] Sparse infill hidden at mid-height: every root ends in wall beads
- [ ] No notch where arcs meet the hub
- [ ] One slow sweep bottom to top: nothing appears or vanishes abruptly
