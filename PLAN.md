# Project plan: auto-fitting e-NABLE devices from photos or LiDAR scans

## Summary

Upload photos or a LiDAR scan of an arm and get a print-ready kit: a hand scaled to the recipient, a gauntlet fitted to their actual forearm, colors and designs the recipient picks, a printability report, and a build guide for the volunteer. The goal is to cut the hours volunteers spend sizing and hand-fitting each device, so recipients wait less.

## Background

### e-NABLE (Enabling the Future)

- Around 40,000 volunteers in 100+ countries make free 3D-printed hands and arms. Upper limb only, no legs.
- Designs are open source (Phoenix, Kinetic Hand, Paraglider, UnLimbited Arm, and others).
- Current sizing methods, per e-NABLE's own docs:
  - **Blender method**: most reliable, but takes about 21 minutes of tutorials to learn.
  - **Two-measurement spreadsheet**: e-NABLE notes it "may not produce a properly fitting device."
  - **AI HandSizer (photo-based)**: still in testing, Phoenix v3 only, about 1.2 cm average error.
- Gauntlets are often fitted by hand by heating them (heat gun or hot water) and forming them over the limb or a plaster cast. That's slow, skilled work and the step this project replaces.

### Prior art

[ai-parametric-prosthetic-hand-generator](https://github.com/pedrocandeias/ai-parametric-prosthetic-hand-generator) already uses an LLM to suggest parametric hand settings from typed measurements. It does not handle photos or scans and doesn't fit a gauntlet to the limb. That's the gap.

### The gender gap

- A University of Michigan study (*Journal of Clinical and Translational Science*, insurance data on 4,000+ patients) estimated women wait about 33% longer than men for a first prosthesis.
- A VA cohort study found time to prosthetic prescription was significantly shorter for men.
- A 2021 *Journal of Vascular Surgery* paper found women are less likely to be referred to a prosthetist at all.
- **Caveat:** these are lower-limb, clinical figures, not e-NABLE data. Pitch it as evidence of a gender gap in prosthetic care, not as an e-NABLE statistic.

## Scope

- **Upper limb only.** e-NABLE doesn't make lower-limb devices, and a leg socket carries body weight: if it fails, someone falls. That needs a clinician.
- **Start with wrist-powered hands** (the Phoenix/Paraglider family) for people missing part of the hand. Their forearm is intact, so the gauntlet is fitted to a real forearm and teammates are valid test subjects.
- Below-elbow sockets (UnLimbited Arm style) are a stretch goal.

## Input modes

| | Photo mode (primary) | LiDAR mode (high accuracy) |
|---|---|---|
| Input | 3–4 photos of the arm on a printed marker sheet | A scan file (OBJ/STL/PLY) from Polycam, Scaniverse or 3d Scanner App |
| Needs | Any phone and a printer | iPhone/iPad Pro with LiDAR |
| Expected accuracy | About 3–5 mm (to be measured) | About 1–2 mm |
| How the arm's 3D shape is built | From the outline in each photo | Clean up the scan |
| Padding gap inside the gauntlet | 4–5 mm, plus adjustable straps | 2–3 mm |

Photo mode is primary because LiDAR is limited to expensive iPhone Pro models. Many e-NABLE recipients are in low-resource settings with basic Android phones.

### Photo protocol (what the recipient does)

1. Print the marker sheet: one page with four ArUco markers in the corners and a mm ruler. Code finds the markers to get the scale and correct for camera angle. A credit card is a less accurate fallback.
2. **Photo A, affected arm from above:** forearm and hand flat on the sheet, palm down, phone directly overhead.
3. **Photo B, affected arm from the side:** same arm resting on its side, thumb up.
4. **Photo C, sound hand:** flat on the sheet, fingers together. Gives palm width and length for scaling.
5. **Optional:** tape-measure circumferences at the wrist and the widest part of the forearm. This is the single biggest accuracy boost.
6. **Optional, design ideas:** describe the look in words or upload reference images (see [Personalization](#personalization)).

Shoot from about 1 m away with 2× zoom. The arm's edges sit several centimetres above the paper, so up close they look bigger than they are. Correct the rest using the arm thickness from the other view.

## Pipeline

Both input modes produce the same thing: a 3D shape of the arm plus landmarks, measurements and an accuracy estimate. Everything after that is shared.

### Photo path

1. **Photo check (Claude, vision):** are the markers visible, is the whole arm in frame, is it the right view, and which side is it? If something's wrong, return plain-language retake instructions in the recipient's language. Also classify the limb difference.
2. **Straighten (code):** detect the markers (OpenCV) and warp each photo into a flat top-down view in mm. Cut the arm out from the background (rembg or SAM 2; the white sheet helps).
3. **Landmarks (Claude, then code):** Claude marks the wrist crease and where the gauntlet starts and ends on an image with a labelled mm grid drawn over it. Code refines them, for example snapping the wrist to the narrowest point nearby.
4. **Measure (code):** widths every 5 mm along the arm from photo A, thicknesses from photo B, lined up at the wrist.
5. **Build the 3D arm (code):** treat each cross-section as an ellipse, adjust them to the tape measurements if provided, and join them into a smooth, closed mesh.

### LiDAR path

1. **Clean (code):** keep the largest piece of the mesh, rebuild it into a closed surface (Poisson), detect units, decimate, and align the arm's long axis.
2. **Landmarks (Claude, then code):** render 4–6 views with a mm grid. Claude marks the landmarks and code refines them.
3. **Measure (code):** slice cross-sections along the arm for circumferences, widths and lengths.

### Shared downstream

1. **Choose the device (Claude):** pick a design and settings against e-NABLE's rules (which designs need a working wrist, scale limits, age and growth allowance) and explain why.
2. **Design spec (Claude):** turn the recipient's description and reference images into colors for each part and a pattern for the gauntlet (see [Personalization](#personalization)).
3. **Generate parts:**
   - **Hand:** run Paraglider's OpenSCAD files from the command line with the computed scale.
   - **Gauntlet:** compute a distance field around the arm and keep a shell from the padding gap out to the gap plus a 3 mm wall. Cut a C-shaped opening so it slips on, add strap slots, attach the stock hinge pieces at the wrist axis, and add the chosen pattern to the outside.
   - **Fit-test ring:** a 15 mm slice of the gauntlet at its tightest point. It prints in about 15 minutes, so fit can be checked before a 6-hour print.
4. **Printability (code):**
   - The mesh is closed and valid, and walls meet a minimum thickness.
   - Pattern details are big enough to print.
   - Parts fit the printer bed; check overhangs and pick an orientation.
   - Slice with PrusaSlicer for print time and filament use.
   - Export all parts on one build plate as a 3MF, with each part's color.
5. **Visual review (Claude, bounded):** look at renders of the gauntlet over the arm and adjust only a few bounded settings (padding gap, trim lines, opening angle, strap positions). Three passes at most.
6. **Build packet (Claude):** parts list, filament colors, print settings, hardware, assembly steps and a fit checklist, plus a plain-language summary for the recipient in their own language.
7. **Volunteer approval:** nothing gets printed until a volunteer signs off.

## Personalization

Many recipients are kids, and a device a kid helped design is one they'll be excited to wear. Recipients, or a parent, can describe the look they want in words ("purple and teal with stars", "make it look like a dragon") or upload reference images such as a favorite toy, a drawing, or a color swatch.

### How it works

1. **Design spec (Claude, vision):** turn the text and images into a structured spec:
   - a color for each part: palm, fingers, gauntlet, pins and straps
   - a pattern motif for the gauntlet
   - optional text, such as a name

   Map colors to real filament colors, ideally the volunteer's own stock, and suggest substitutes for anything they don't have.
2. **Patterns:** raised (about 0.8 mm) or recessed motifs on the outside of the gauntlet only.
   - Claude picks from a library of tested SVG motifs (stars, hearts, scales, lightning, snowflakes, paw prints) or writes a simple custom SVG, which is rendered and checked before use.
   - The pattern is wrapped around the arm's long axis and added to the gauntlet's outer surface when the shell is built, so it works the same for photo and LiDAR inputs.
3. **Keep-out zones:** no pattern near the strap slots, the hinge, the opening edges, or on the skin side.
4. **Preview and changes:** the 3D viewer shows the colored, patterned device. The recipient can ask for changes in plain words ("bigger stars", "more purple") and approve the final look.
5. **Printing:** the default is one color per part, which works on any single-color printer. If the volunteer has a multi-material printer (Bambu AMS, Prusa MMU), the 3MF can carry a two-color pattern on the gauntlet.

### Guardrails

- **Characters and logos:** turn requests like "Spider-Man" into an inspired-by palette and generic motifs (webs, red and blue) instead of copying trademarked logos.
- **Reference images:** use them only to pull out colors and motifs, and don't store them. Ask families not to include the child's face.
- **Printability:** details at least about 1 mm wide (for a 0.4 mm nozzle), and patterns never make walls thinner than the minimum.

## Where Claude is used

| Step | Claude's job | Why it isn't plain code |
|---|---|---|
| Photo check | Spots bad photos and explains retakes | Varied real-world photos |
| Intake | Reads free-text goals, classifies the limb difference | Unstructured input |
| Landmarks | Finds anatomy on images; code refines the numbers | Recognising anatomy, which plain geometry can't do |
| Device choice | Applies e-NABLE guidelines, flags exceptions | Judgement calls on edge cases |
| Design | Turns "purple dragon with stars" or a reference image into part colors, a pattern and real filament choices | Open-ended creative requests |
| Review | Critiques renders, tweaks settings through tools | Catches things like "the opening is on the wrong side" |
| Build guide | Personalised and translated | e-NABLE works in 100+ countries |

- Model: `claude-opus-5-5`, with vision, structured outputs and strict tool schemas.
- Claude only sees renders, photos and measurements, never the raw scan. That works better and keeps health data local.

## Tech stack

- **Backend:** Python with FastAPI.
- **Images:** OpenCV (markers, straightening) and rembg or SAM 2 (background removal).
- **Meshes:** trimesh, open3d (surface rebuild, distance fields), scikit-image (turning the distance field into a mesh), manifold3d (cutting and joining solids), pyvista (renders).
- **CAD and slicing:** OpenSCAD command line (a recent build with the Manifold backend, which is much faster) and PrusaSlicer command line.
- **Patterns:** SVG motifs turned into images with cairosvg, then wrapped onto the gauntlet's outer surface.
- **Frontend:** React with react-three-fiber. Photo or scan upload, a design box for text and reference images, a 3D viewer showing the colored gauntlet over the arm, a timeline of Claude's reasoning, and downloads.

## Testing without amputees

- The gauntlet goes on the forearm, which is intact for partial-hand recipients, so teammates' forearms are realistic test subjects for the main custom part.
- **Accuracy study:** tape-measure 8–10 people at three points on the forearm, run their photos (and LiDAR scans where possible) through the app, and report the error for each mode in the pitch.
- **Physical proof:** print fit-test rings for 2–3 people and show they fit.
- **Live demo:** a judge puts their arm on the marker sheet, takes two photos, and gets a fitted gauntlet in about a minute.
- Don't use photos of real amputees from the internet; there's no consent. For true residual-limb cases, ask an e-NABLE chapter for consented example data or leave it as future work.

## Women's access angle

- **Works from a phone at home.** No plaster-casting visit and no clinician referral, which is where women already lose ground.
- **Photo mode works on any phone**, not just iPhone Pro models.
- **Fewer volunteer hours per device**, so the queue moves faster.
- **Stretch:** an opt-in chapter dashboard of request-to-delivery time by gender. Nobody measures this gap for upper-limb devices yet.

## Build order (assuming about 24 hours)

| Hours | Work |
|---|---|
| 0–2 | Test the risky parts first: print the marker sheet, take test photos, get Paraglider rendering from the command line, prototype the ellipse arm and gauntlet shell in a notebook |
| 2–8 | Photo pipeline: markers and straightening, background removal, width and thickness measurements, building the 3D arm |
| 4–10 | Gauntlet generator: shell, C-opening, strap slots, hinge pieces, fit-test ring |
| 6–12 | Claude: photo check, landmarks, device choice and settings |
| 8–18 | Frontend |
| 12–16 | Printability checks, slicing, 3MF export |
| 14–18 | Personalization: design spec from text and images, part colors, gauntlet patterns |
| 16–20 | LiDAR input path, review loop, build packet |
| 20–24 | Accuracy study, printed fit rings, backup demo video, pitch |

**Minimum demo:** photos → measurements → fitted gauntlet and scaled hand → printability report → download.

**After that, in order:** personalization (colors first, then patterns and names), LiDAR mode, the review loop, the build packet.

**Stretch:**
- Ventilation hole patterns on the gauntlet.
- Two-color patterns for multi-material printers.
- Claude-written OpenSCAD adapters for specific goals ("hold a paintbrush").
- A below-elbow socket mode.
- The equity dashboard.

## Risks

| Risk | Mitigation |
|---|---|
| Photos make the arm look bigger than it is | Shoot from about 1 m with 2× zoom; correct using the other view's thickness; tape-measure circumferences |
| Scan holes break the distance-field step | Rebuild the scan into a closed surface (Poisson) first |
| Scan scale is wrong | LiDAR is to scale; check it against the marker or a known dimension |
| The custom gauntlet doesn't line up with the stock hinge | Reuse the stock hinge pieces instead of modelling new ones |
| Licensing | Most e-NABLE designs are Creative Commons; check each one and credit the designers |
| Over-trusting automation | Volunteer approval step, bounded settings, fit-test ring before the full print |
| Health data privacy | Process locally, delete after use, send Claude only renders and numbers |
| Requests for trademarked characters | Inspired-by colors and generic motifs, no copied logos |
| Kids' photos in reference images | Use only for colors and motifs, don't store, ask families to leave faces out |
| Patterns too fine to print or weakening the gauntlet | Minimum detail size, keep-out zones, pattern added outside the minimum wall |

## Open questions

- How long is the hackathon, and how many people are on the team?
- Does anyone have an iPhone with LiDAR?
- Do we have access to a 3D printer before judging?
- Can we reach a local e-NABLE chapter for feedback or a quote?

## References

- [Enabling the Future](https://enablingthefuture.org/)
- [e-NABLE device sizing](https://enablingthefuture.org/resources-2/e-nable-device-sizing/)
- [e-NABLE Hub sizing wiki](https://hub.e-nable.org/s/e-nable-devices/wiki/page/view?title=e-NABLE+Device+Sizing)
- [Kinetic Hand](https://hub.e-nable.org/p/devices?p=Kinetic+Hand)
- [UnLimbited Arm v2.1](https://hub.e-nable.org/p/devices?p=Unlimbited+Arm+v2.1)
- [Paraglider (flexible_flyer)](https://github.com/mendenm/flexible_flyer)
- [ai-parametric-prosthetic-hand-generator](https://github.com/pedrocandeias/ai-parametric-prosthetic-hand-generator)
- [Why do women wait longer for prosthetic care? (Living with Amplitude)](https://livingwithamplitude.com/article/women-prosthetic-care-health-disparities/)
- [Differences in prosthetic prescription between men and women veterans](https://pmc.ncbi.nlm.nih.gov/articles/PMC11890125/)
- [Factors that influence time to prosthesis receipt (PM&R)](https://onlinelibrary.wiley.com/doi/full/10.1002/pmrj.12781)
- [Gre-Nable: adapting a socket](https://www.gre-nable.fr/adapter-une-emboiture-pour-une-prothese/)
- [Transactions on Additive Manufacturing Meets Medicine](https://www.journals.infinite-science.de/index.php/ammm/article/download/2722/1496/9945)
- [3D-printing and upper-limb prosthetic sockets: promises and pitfalls](https://doi.org/10.1109/tnsre.2021.3057984)
- [Smartphone-based photogrammetry for residual limb reconstruction](https://doi.org/10.3390/s26041251)
