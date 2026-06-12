# Chapter 2: GUI Beamline Building, Code Generation, and Controls

This chapter is the second 2-hour xrt training session. The first part focuses on building a beamline in xrtQook and generating a runnable script. The second part connects the model to EPICS controls and CSS Phoebus BOB screens, with a final introduction to digital twin mode.

## Goal

Build a beamline from start to end in the GUI, inspect how the beam changes along the way, then generate and run the corresponding Python script.

By the end of this part, you should be able to:

- start xrtQook from the reproducible environment
- navigate the main xrtQook panels and xrtGlow 3D visualizer
- assemble a simple beamline from source to detector
- assign materials to filters, mirrors, and crystals
- add diagnostic plots
- generate, inspect, save, and run a Python script from the GUI model
- expose model parameters as EPICS process variables
- open generated CSS Phoebus BOB screens, or follow the live demo if local Phoebus setup needs more time
- understand how external PV callbacks can update the xrt model in digital twin mode

## Note on Screenshots

This chapter intentionally uses only a few visual guides during the live training. You will build the beamline interactively, inspect the scene yourself, and develop your own sense of what the model should look like. Additional screenshots and reference views will be added after the session for future self-guided readers.

## 1. Welcome Back and Setup

Before starting, review [Chapter 1](chapter1.md): you installed xrt, ran calculation scripts, changed physical parameters, and interpreted plots for sources, filters, mirrors, and crystals.

In this chapter, you move from individual calculations to a complete beamline model. If you already have the xrt source folder from Chapter 1, continue in that folder.

Otherwise, download the current source code archive from the master branch:

```text
https://github.com/kklmn/xrt/archive/refs/heads/master.zip
```

Unpack it into a simple location and open a terminal in the unpacked folder. If you unpack the archive as `xrt-master`, you can either keep that folder name or rename it to `xrt`.

If Git works on your machine, cloning the repository is equivalent:

```bash
git clone https://github.com/kklmn/xrt.git
cd xrt
```

Install the training environment:

```bash
pixi install -e gui
```

The Pixi manifest must install xrt with the `all` optional dependencies, written as `xrt[all]`. This keeps the GUI, xrtGlow, EPICS soft IOC support, `pyepics` callbacks, and BOB-generation support available from the same environment.

Start xrtQook:

```bash
pixi run -e gui xrtQook
```

Checkpoint: xrtQook should open successfully before you start building the beamline.

If you are using a custom environment rather than the Pixi manifest from this repository, install xrt as `xrt[all]`. Installing plain `xrt` may be enough for some calculations, but it can miss packages needed later for xrtQook, xrtGlow, EPICS controls, and Phoebus screen generation.

### Phoebus Setup for Controls

CSS Phoebus will be used later in this chapter to open generated BOB control screens. If Phoebus does not start on your machine during the session, keep following the live demo and finish the local Phoebus setup afterwards.

Download Phoebus release 5.0.5 from:

```text
https://github.com/ControlSystemStudio/phoebus/releases/tag/v5.0.5
```

On Windows, download the Windows archive, unpack it into a simple path such as:

```text
C:\xrt-training\phoebus-5.0.5
```

On macOS, download the macOS archive, unpack it into a simple path such as:

```text
~/xrt-training/phoebus-5.0.5
```

Phoebus also needs a Java runtime.

Download a JDK archive separately and unpack it next to the Phoebus top directory, because this is where the Phoebus startup script looks for the Java binary. A good temporary layout is:

```text
xrt-training/
  phoebus-5.0.5/
  jdk/
```

Use a JDK that matches your operating system and CPU architecture. The exact JDK download link and folder name will be refined after testing the final training machines.

## 2. Quick Tour of xrtQook

xrtQook is the graphical beamline editor for xrt. It is used for:

- creating and editing beamline templates
- inspecting optical elements, materials, beams, and plots
- previewing propagation in the integrated xrtGlow 3D visualizer
- generating Python code from the GUI model

xrtGlow is now included in xrtQook by default and updates live as the beamline changes.

For the overview, load an existing example such as:

```text
examples/withRaycing/_QookBeamlines/BioXAS_Main.xml
```

Use this example to learn the layout of xrtQook. You will build your own beamline afterwards.

### Main Controls

The left toolbar adds new objects to the model, including beamline elements, materials, figure errors, and plots.

The top toolbar contains the main workflow actions:

- create or open a template
- save the current template
- generate Python code
- save generated Python code
- save the generated script and run it
- show OpenCL information

The main window contains several tabs. If `qtpynodeeditor` is available, a `Flow` tab is also shown for the propagation graph.

| Tab | Purpose |
| --- | --- |
| Beamline | Stores beamline elements and connects them through propagation method arguments. |
| Materials | Stores all materials used by the beamline. |
| Figure Error | Stores surface distortion models. |
| Plots | Defines customizable plots for generated scripts and live preview. |
| Job Settings | Controls repeats, processes, and scan generators for generated scripts. |
| Description | Holds presentation notes in reStructuredText format. |
| Code | Appears after pressing the code generation button. |
| Console | Can run generated code for a quick test; a normal IDE is better for sustained work. |

### Live Documentation

The live documentation panel shows docstrings for the currently selected element, material, or plot. Use it whenever you encounter an unfamiliar parameter.

### xrtGlow Panels

xrtGlow controls are grouped into pop-up panels:

| Panel | Useful for |
| --- | --- |
| Navigation / Selection | Show or hide beams, footprints, shapes, and labels. |
| Transformation | Rotate, scale, and translate the scene. Mouse zoom is usually easiest for coordinated scaling. |
| Color | Select the color axis, colormap limits, opacity, and brightness for points and lines. |
| Coordinate grid properties | Adjust grid size, visibility, and beam projections. |
| Scene properties | Set font size and ray-flag visibility, including Good, Out, Over, and Lost rays. Many options are also available from the context menu. |
| Object render properties | Tune visual rendering only, such as thickness, detail level, and source-envelope display. These settings do not change script-based propagation. |
| Scans | Define automated sequences that can be sent to Qook `Job Settings` as generators. |

Try this while exploring the scene: right-click an element in xrtGlow and center the view on it. When mirror orientation becomes confusing, show the local axes for the selected object.

## 3. Build a Beamline Interactively

Before adding elements, sketch a rough beamline layout. The model will follow this optical path:

```text
source -> front-end mask -> filter -> collimating mirror -> direction-restoring mirror -> DCM -> harmonic rejection mirror -> focusing mirror -> sample -> detector
```

Save the template after every major step. GUI experiments are more fun when mistakes are cheap.

Give every element a meaningful name as you create it. Default names are intentionally generic, but names such as `FE_mask`, `diamond_filter`, `collimating_mirror`, `DCM`, `sample`, and `detector` make the layout much easier to navigate later. They also make the generated code, EPICS PVs, and Phoebus screens easier to understand.

### Beamline Elements

Add the following elements one by one. After each addition, inspect the element in xrtGlow and run a quick propagation preview when useful.

| Step | Element | Position | Key settings and checks |
| --- | --- | --- | --- |
| 1 | Wiggler source | `[0, 0, 0]` | Start with the beam source. Confirm that rays are generated before adding optics. |
| 2 | Front-end mask, rectangular aperture | `[0, 10000, 0]` | Use it to define the accepted angular cone. Watch the accepted beam shrink. |
| 3 | Plate placeholder for filter | `[0, 10500, 0]` | Give it finite transverse size, for example `limPhysX = [-10, 10]` and `limPhysY = [-10, 10]` for a 20 x 20 mm plate, or `[-5, 5]` for 10 x 10 mm. Set `pitch` to `90deg` or `np.pi/2`, and set a finite thickness `t` in mm. Material comes later, but the finite thickness is already needed for meaningful propagation through the two plate surfaces. |
| 4 | Toroidal mirror, facing up | `[0, 12000, 0]` | Use `pitch = 0.2 deg`. Set a realistic finite mirror size before using strong curvature. The default physical length is about 2 m, and an aggressive `R` can raise the front face high enough that the beam no longer reaches the surface. Start with infinite curvature radii, then use `p` and `q` to define curvature. |
| 5 | Flat mirror to restore direction | `[0, 14000, auto]` | Use `pitch = -0.2 deg` and keep `center.z = auto`. Set `positionRoll = np.pi`; otherwise the active surface is on the wrong side and the beam may not appear. Use local axes to understand the surface orientation. |
| 6 | Double-crystal monochromator | `[0, 16000, auto]` | Use `bragg`, not `pitch`, to set the DCM angle. Set a finite fixed-exit offset with `fixedOffset = 20` mm; the default zero offset makes the two crystal surfaces overlap visually. Start with empty material, then assign a crystal later. |
| 7 | Flat mirror for harmonic rejection | `[auto, 20000, auto]` | Set both `center.x` and `center.z` to `auto`. There is nothing special to tune here at first; it becomes interesting later when mirror materials and coatings are assigned. |
| 8 | `ParabolicalMirrorParam`, facing up | `[0, 22000, auto]` | Make this the last optical element before the sample. For a parabolic mirror, set one focal distance, normally `q` or `f2`, not both `p` and `q`. It focuses a collimated incoming beam, so tune M1 as a collimating mirror before expecting the spot to land on the sample. Use `isCylindrical` to focus in one dimension. |
| 9 | Sample plate | `[auto, 25000, auto]` | Add a plate as a sample placeholder. Material can be assigned later. |
| 10 | Detector screen | `[auto, 26000, auto]` | Use it as the final beam diagnostic. |

Check these points while building:

- What happens if the mirror pitch sign is wrong?
- How do local axes help explain which way an optic faces?
- Can the DCM face down?
- Can the monochromator be placed for side reflection?
- What should be automatic, and what should be set explicitly?

### Source Acceptance Exercise

The first exercise is to match the wiggler source acceptance to the rectangular aperture opening. The source is a wiggler, not a point source, so its horizontal size is finite. Do not tune only as if all rays start from `[0, 0, 0]`.

Start with a modest front-end opening, for example `10 mm` horizontally and `1 mm` vertically. If you are setting blades directly, this corresponds to about `left = -5`, `right = 5`, `bottom = -0.5`, and `top = 0.5` for a centered aperture.

At first, deliberately keep the source acceptance angles too large. Enable Lost rays in the xrtGlow `Scene properties` ray-flag controls. This shows how much of the full wiggler fan is actually blocked by a realistic front-end aperture.

After adding the source and front-end mask, estimate the angular acceptance from the mask distance and opening:

```text
xPrimeMax ~= horizontal half-opening / source-to-mask distance
zPrimeMax ~= vertical half-opening / source-to-mask distance
```

The source parameters `xPrimeMax` and `zPrimeMax` are in mrad in the Qook tree. With the mask at `y = 10000 mm`, a `1 mm` half-opening corresponds to about `0.1 mrad`. For the example above, the geometric half-angles are about `0.5 mrad` horizontally and `0.05 mrad` vertically before accounting for source size.

Use this estimate only as a starting point. Because the wiggler has finite horizontal extent, the horizontal beam size at the mask is approximately:

```text
source horizontal size + source-to-mask distance * horizontal angle
```

Change `xPrimeMax` and `zPrimeMax`, rerun the preview, and compare the beam footprint with the rectangular aperture. The goal is not to throw away a large amount of flux at the first mask, but to generate roughly the angular range the front end can accept. Once the acceptance is matched, the same ray budget produces more useful rays downstream.

### Toroid Curvature Exercise

Before making the toroidal mirror strongly curved, give it a realistic physical size. The default optical element limits are intentionally broad, about `[-1000, 1000]` mm in the local length direction, so the mirror is effectively 2 m long unless you change its limits.

This matters for a toroid because the meridional surface height changes with distance along the mirror. If `R` is made too small while the mirror is still 2 m long, the front or back edge can be raised so far that the incoming beam no longer reaches the reflecting surface. In xrtGlow this can look like a broken propagation or a bad pitch setting, but it is a geometry problem.

Start with the toroid nearly flat, confirm that the beam hits the surface, then reduce the curvature gradually or switch to `p` and `q` focusing values. If the beam disappears when the curvature becomes aggressive, shorten `limPhysY` to a realistic mirror length or relax `R`. Use local axes, footprints, and ray-flag visibility to check whether rays are reflecting, going outside the optical limits, or missing the surface.

### First Flat Mirror Orientation Exercise

The first flat mirror restores the beam direction after the toroid. Keep its vertical position automatic by setting `center.z = auto`; the actual value will be solved from the incoming beam path.

At first, the beam may not show up on this mirror. Set `positionRoll = np.pi` and run the preview again. This flips the mirror orientation so that the active surface faces the beam.

Use local axes in xrtGlow to make this visible instead of treating `positionRoll` as a magic correction. Rotate the scene around the mirror and compare the incoming beam direction with the side of the optic that is active.

After the beam is visible, open the Inspector and read back the resolved mirror position, especially `center.z`. Then hover the mouse over the mirror in the 3D scene and compare the tooltip position with the Inspector value. This connects the `auto` value in the Qook model with the actual coordinates used in the scene.

### DCM Geometry Exercise

In the DCM, leave `pitch` for whole-monochromator orientation and use `bragg` for the Bragg angle. This keeps the DCM behavior conceptually separate from the mirror pitch exercises.

Set `fixedOffset = 20` mm at first. Then play with this value and watch how the second crystal and outgoing beam move. This is much less confusing than the default `fixedOffset = 0`, where the two crystal surfaces sit on top of each other in the 3D view.

Turn on local axes in xrtGlow to identify the center and orientation of each DCM crystal. This is especially useful because the second crystal has its own local beam, not just a continuation of the first footprint.

Open the DCM in the Inspector and compare `beamLocal1` and `beamLocal2`. These are the local footprints on the first and second crystal. Use their local positions and extents as a guide for realistic crystal sizes:

- `limPhysX` and `limPhysY` for the first crystal
- `limPhysX2` and `limPhysY2` for the second crystal

For this exercise, set the crystal limits explicitly from the Inspector readback. There is a known temporary caveat in the auto-match tool: if the beam footprint is shifted far from the local origin, auto-match can size the optic around the origin rather than around the actual beam position.

After the DCM is roughly aligned, deliberately change the physical sizes of the two crystals independently. Start with the second crystal large enough to catch the full beam, then reduce its size until part of the beam is no longer reflected.

Watch two things happen:

- the beam footprint shifts sideways after the monochromator
- some rays eventually go over the second crystal surface instead of reflecting from it

Use xrtGlow's `Scene properties` panel to toggle ray-flag visibility. Showing and hiding Good, Out, Over, and Lost rays makes it much easier to understand whether the beam is being accepted, clipped, or missing the optic entirely.

Use the same idea in plots with the plot `rayFlag` parameter. For example, compare a normal plot with `rayFlag=(1,)`, an accepted-plus-outside-optical-limits plot with `rayFlag=(1, 2)`, and an over-surface diagnostic plot with `rayFlag=(3,)`.

As a separate orientation exercise, change the DCM `positionRoll`. Set `positionRoll = 90deg` or `np.pi/2` to emulate a side-bouncing monochromator, then use local axes and the 3D beam path to see which side the beam exits on. Then try `positionRoll = 180deg` or `np.pi` to make the DCM bounce the beam downward. In each case, inspect `beamLocal1`, `beamLocal2`, and the outgoing beam before deciding which coordinates can stay automatic.

### Final Focusing Mirror Exercise

The final `ParabolicalMirrorParam` is easier to understand if you first try the wrong thing. Set its `q` to the nominal mirror-to-sample distance and look at the sample screen. The focal spot will not necessarily land on the sample yet.

This is expected if M1 has not actually collimated the beam. A parabolic mirror focuses a parallel incoming beam to its focus; it is not a general point-to-point imaging mirror. If the beam arriving at the final mirror is still divergent or convergent, changing only `q` moves the modeled parabola but does not magically put the waist at the sample.

Go back to the toroidal M1 and tune it as the collimating mirror. Use its `R` and `r` settings, or `p` and `q` Coddington values where appropriate, and watch the beam before the final focusing mirror. Once the incoming beam is close to collimated in the focusing plane, return to the final mirror, set `q` or `f2` for the sample position, and compare the footprint on the sample and detector.

### Add Diagnostic Screens

Add several screens so the beam can be checked at important positions:

| Screen | Position | Purpose |
| --- | --- | --- |
| Before front-end mask | `[0, 9900, auto]` | Compare the original source beam with the clipped beam. |
| Before monochromator | `[0, 15500, auto]` | Check alignment after the upstream mirrors. |
| Before focusing mirror | `[auto, 21000, auto]` | Inspect the beam after harmonic rejection and before the final focus. |
| Detector | `[auto, 26000, auto]` | Final beam profile. |

Try setting one or two screens at 45 degrees. For example, set the screen X axis to:

```text
[0.707, 0.707, 0]
```

Use this step to connect the screen geometry with what appears in the 3D scene.

## 4. Add Materials

Now turn placeholders into physical optics.

Add these materials:

| Material | Use |
| --- | --- |
| `CVDDiamond` | Filter plate. Set `kind` to `plate`; do not change thickness inside the material. The actual filter thickness is the plate element parameter `t`, in mm. |
| Silicon crystal | DCM crystal material. Start with the default `hkl`, then change it later while watching the sample. |
| Bulk silicon | Mirror substrate. |
| Rhodium | Mirror coating. The default `kind = mirror` is fine; you can also set it explicitly. |
| Coated material | Use Rh as `coating` and silicon as `substrate`. The `cThickness` parameter is in angstroms, so use `200` for a 20 nm coating. |

After adding the Rh material, open it in the Inspector and view its reflectivity curve. Set the curve energy range to `Emin (eV) = 5000` and `Emax (eV) = 35000`. Set the grazing incidence angle to match the mirrors: for `pitch = 0.2 deg`, use about `3.5 mrad` in `Grazing angle theta (mrad)`.

After creating the coated material, open a second Inspector window for it. Keep one Inspector on pure Rh and the other on the coated Rh-on-Si material. Use the same curve settings in both windows:

- `Emin (eV) = 5000`
- `Emax (eV) = 35000`
- `Grazing angle theta (mrad) = 3.5`

Compare the two reflectivity curves side by side. This is the first check that the coating choice makes sense over the training energy range. Later, after the material is assigned to an actual mirror, the Inspector can also read the angle from the optical element or its local beam.

For the DCM, first create a silicon crystal with the default `hkl`. Assign this same material to both DCM crystal slots:

- `material` for the first crystal
- `material2` for the second crystal

Then set the DCM `bragg` value from the photon energy in the middle of the source range:

```text
9050 eV
```

Keep looking at the sample screen while you do this. The DCM should now select the center of the source energy range. After the beam is visible, return to the silicon crystal material and change `hkl` to `(3, 1, 1)`. Rerun the preview and watch how the sample signal changes.

With monochromatization enabled, the initial source bandwidth is much wider than the accepted DCM band. If the source still uses `eMin = 9000` and `eMax = 9100`, only a small fraction of the generated rays are near the energy that survives the monochromator. The result is that very few rays may be visible after the DCM, even when the geometry is correct.

Use the sample screen or a sample energy plot to inspect the transmitted bandwidth. Then narrow the source `eMin` and `eMax` around the accepted energy band near `9050 eV`. This is the energy-space version of the earlier source-acceptance exercise: once the DCM energy acceptance is known, spend the ray budget on the part of the source spectrum that can actually reach the sample.

Now return to M1 and deliberately break the collimation. This connects the mirror setup with the energy resolution after the DCM. Start with the toroidal mirror meridional curvature defined by:

```text
R = (12000, 12000)
```

Then change it to:

```text
R = (12000, 1200)
```

Watch the sample screen or a sample energy plot as the beam arriving at the DCM becomes less collimated. The DCM accepts angle and energy together, so a poorly collimated upstream beam can broaden, shift, or weaken the selected band at the sample.

With M1 deliberately broken, ray flags become especially useful. In xrtGlow, toggle Good, Out, Over, and Lost rays in the `Scene properties` panel and compare how the 3D picture changes. Then open the Inspector for the sample screen and look at the beam that actually reaches the sample.

Also open a plot preview from the `Plots` tree for the sample plot. Change the plot `rayFlag` setting and compare a good-rays-only view with diagnostic views that include rays outside optical limits or rays going over a surface. The point is to distinguish a weak monochromatic signal from rays that are present in the model but rejected by geometry.

Also try the sagittal radius `r`. Set it first to a very large number so that the sagittal curvature is almost flat. Then set it to a small value such as:

```text
r = 25
```

Use the local axes, footprints, and the sample screen to separate meridional collimation effects from sagittal focusing or defocusing.

By this point the DCM crystals already have material assigned. Assign the remaining materials to:

- the filter plate
- the mirrors
- the sample plate, if time permits

After each assignment, inspect how the flux changes. Try pure silicon at higher energies and look for where the flux is lost. Compare transmitted flux, reflected flux, and absorbed power.

## 5. Add Plots

Add a few plots before generating the script:

- footprint on the first mirror
- footprint on the final focusing mirror
- beam profile at the sample
- energy spectrum or energy-colored beam profile at the sample
- beam profile at the detector

In the sample plot preview, enable all ray flags with `rayFlag=(0, 1, 2, 3)` and compare that with the 3D scene. You may start seeing stray rays that originated at previous elements rather than at the sample itself. This is a useful design clue: the model may need cleanup slits between the mirrors and another aperture in front of the sample.

Keep the plot set small. A few useful plots are easier to interpret than a long list of diagnostics.

## 6. Generate and Run Code

Save the template, then press the code generation button.

Inspect the generated script and identify the main sections:

- material definitions
- beamline element definitions
- propagation function
- plot definitions
- run settings

Default values are omitted for readability, so the generated script should be shorter than a fully explicit hand-written version.

Save the generated code and run it from the terminal:

```bash
pixi run -e gui python generated_beamline.py
```

Use the actual filename chosen during the session.

Now try changing:

- number of repeats
- number of rays in the source
- number of processes
- plot `saveName` values

Then return to the template, change the same parameters in the tree, regenerate the code, and inspect the difference.

This closes the loop:

```text
GUI model -> generated Python code -> script execution -> plots and diagnostics
```

## 7. EPICS Controls

EPICS is the base control layer at NSLS-II and many other facilities. In this tutorial, EPICS is used to expose selected model parameters as process variables, so the simulated beamline can be controlled like an instrument.

xrt supports two EPICS workflows:

- a standalone EPICS PV tree incorporated into the beamline model and integrated with the xrtGlow control loop
- a monitoring mode where callbacks from real beamline PVs update the beamline model through the API

This chapter starts with the standalone PV-tree workflow.

### Start xrtGlow from the Generated Script

Open the Python script generated in the previous section.

Find the line where the beamline is created:

```python
myBeamline = build_beamline()
```

Immediately after it, add:

```python
myBeamline.glow()
```

Run the script. If xrtGlow opens without errors, pause for a moment and enjoy the picture. Yes: look what we built. Then close it and continue.

Now enable EPICS PV generation by adding an EPICS prefix:

```python
myBeamline.glow(epicsPrefix='VIRT')
```

Use the actual variable name in the generated script. xrtQook names it after the beamline tree root, so it may be `BeamLine`, `beamLine`, or a name you assigned.

Run the script again. xrtGlow prints the generated PV list to the terminal. In the 3D view, double-click an element to see the controls and exposed EPICS PVs for that object.

When `epicsPrefix` is used in dynamic xrtGlow mode, xrt sets the IOC-side EPICS Channel Access environment to localhost before creating the standalone `EpicsDevice`:

```text
EPICS_CA_ADDR_LIST=127.0.0.1
EPICS_CA_AUTO_ADDR_LIST=NO
EPICS_CAS_INTF_ADDR_LIST=127.0.0.1
EPICS_CAS_BEACON_ADDR_LIST=127.0.0.1
EPICS_CAS_AUTO_BEACON_ADDR_LIST=NO
```

### Test Channel Access from Python

Keep the xrtGlow script running. Open a second terminal in the xrt directory and activate the Pixi environment:

```bash
pixi shell -e gui
```

Use the same localhost Channel Access settings in this second terminal before starting Python.

On Linux or macOS:

```bash
export EPICS_CA_ADDR_LIST=127.0.0.1
export EPICS_CA_AUTO_ADDR_LIST=NO
```

On Windows PowerShell:

```powershell
$env:EPICS_CA_ADDR_LIST = "127.0.0.1"
$env:EPICS_CA_AUTO_ADDR_LIST = "NO"
```

Then start a Python console:

```bash
python
```

Import the EPICS client helpers:

```python
from epics import caget, caput
```

Copy one PV name from the xrtGlow PV list and read it:

```python
caget("VIRT:...")
```

Then write to a harmless position PV, for example the sample `center.x` PV:

```python
caput("VIRT:...", 1.0)
```

Watch xrtGlow update. This confirms that Channel Access is working and that PV updates can move the simulated beamline.

## 8. Control the Beamline with CSS Phoebus

Now generate CSS Phoebus BOB screens for the EPICS-enabled beamline.

Follow the instructions in the xrt source tree:

```text
xrt/backends/raycing/epics/README.md
```

Use an absolute path for the generated screens, and run the screen-generation command from the root xrt directory. Use the same prefix that you passed to `glow(epicsPrefix='VIRT')`.

For a saved xrtQook layout:

```bash
pixi run -e gui python -m xrt.backends.raycing.epics.generate_bob --layout path/to/layout.xml --prefix VIRT --output /absolute/path/to/generated-bob
```

For a generated Python script:

```bash
pixi run -e gui python -m xrt.backends.raycing.epics.generate_bob --beamline path/to/generated_beamline.py:build_beamline --prefix VIRT --output /absolute/path/to/generated-bob
```

For example, choose an output directory such as:

```text
C:\xrt-training\generated-bob
```

or:

```text
~/xrt-training/generated-bob
```

### Configure Phoebus for Local EPICS PVs

Create a `settings.ini` file next to the Phoebus startup script.

Add:

```ini
org.phoebus.pv/default=ca
org.phoebus.pv.ca/addr_list=127.0.0.1
org.phoebus.pv.ca/auto_addr_list=false
```

Start Phoebus with these settings. If the startup script does not automatically load the file, pass it explicitly:

```bash
phoebus -settings /absolute/path/to/settings.ini
```

### Open the Generated Screens

In Phoebus, select:

```text
File -> Open
```

Navigate to the generated BOB screen directory.

Open:

- the sample screen, usually under `screens/`
- the detector screen, also under `screens/`
- `propagation/propagation_control.bob`

If auto-update is off, the control loop waits for the `Acquire` signal before retracing. If auto-update is on, retracing happens automatically whenever an exposed element property changes.

Open the screen for the final focusing mirror. Arrange the screens so the sample and detector are on the right, and the propagation controls and mirror controls are on the left. In Phoebus, you can right-click a control-panel tab and split the view vertically or horizontally.

Change `p` or `q` of the final focusing mirror. Watch how the change propagates through:

- the Phoebus controls
- the xrtGlow 3D model
- the sample screen
- the detector screen

At this point, the simulated beamline can be controlled through Phoebus screens.

## 9. Digital Twin Mode

The standalone EPICS workflow creates PVs from the beamline model. Digital twin mode goes in the other direction: existing or simulated device PVs update the beamline model.

This workflow uses PV callback classes from `pyepics`. The callbacks listen to PV updates and apply them to model parameters through the xrt API. For local testing, the device PVs can be served by `pythonSoftIOC`.

Reference example:

```text
https://github.com/yxrmz/xrt_tricks/tree/main/digital_twin_with_callbacks
```

For local testing, create a small IOC that serves two motor readbacks for the last mirror. Save this as `simple_last_mirror_ioc.py`:

```python
from softioc import softioc, builder, asyncio_dispatcher

dispatcher = asyncio_dispatcher.AsyncioDispatcher()

builder.SetDeviceName("DT")

# Motor readbacks in mm. They are writable here only because this is
# a training IOC; on a real beamline these would normally be readbacks.
upstream_z = builder.aOut("FM:UPSTREAM_Z", initial_value=0.0)
downstream_z = builder.aOut("FM:DOWNSTREAM_Z", initial_value=0.0)

builder.LoadDatabase()
softioc.iocInit(dispatcher)

print("Serving DT:FM:UPSTREAM_Z and DT:FM:DOWNSTREAM_Z")
softioc.interactive_ioc(globals())
```

Run it in one terminal:

```bash
pixi run -e gui python simple_last_mirror_ioc.py
```

In the generated beamline script, add the callback imports near the other imports:

```python
import os
os.environ.setdefault("EPICS_CA_ADDR_LIST", "127.0.0.1")
os.environ.setdefault("EPICS_CA_AUTO_ADDR_LIST", "NO")

from math import atan2
from epics import get_pv
```

Then replace the simple `myBeamline = build_beamline()` / `myBeamline.glow()` lines with this hardcoded callback example:

```python
myBeamline = build_beamline()

mirror_name = "focusing_mirror"
mirror_id = myBeamline.oenamesToUUIDs[mirror_name]
mirror = myBeamline.oesDict[mirror_id][0]

up = get_pv("DT:FM:UPSTREAM_Z")
down = get_pv("DT:FM:DOWNSTREAM_Z")
up.wait_for_connection(timeout=5)
down.wait_for_connection(timeout=5)

up0 = float(up.get())
down0 = float(down.get())
pitch0 = float(mirror.pitch)
z0 = float(mirror.center[2])
base = abs(float(mirror.limPhysY[1]) - float(mirror.limPhysY[0]))


def update_last_mirror(pvname=None, value=None, **kwargs):
    if myBeamline.blViewer is None:
        return

    up_delta = float(up.get()) - up0
    down_delta = float(down.get()) - down0

    pitch = pitch0 + atan2(down_delta - up_delta, base)
    center_z = z0 + 0.5 * (up_delta + down_delta)

    myBeamline.blViewer.customGlWidget.update_beamline(
        mirror_id,
        {"pitch": pitch, "center.z": center_z},
        sender="epics")


up.add_callback(update_last_mirror)
down.add_callback(update_last_mirror)

myBeamline.glow()
```

Use the exact name of your last mirror in `mirror_name`. If you followed the suggested naming, this is `focusing_mirror`. If `center.z` is still written as `auto` in the generated script, first read the resolved value from the Inspector and put that number into `z0`.

Move both motors together to change mirror height:

```bash
caput DT:FM:UPSTREAM_Z 1
caput DT:FM:DOWNSTREAM_Z 1
```

Move one motor relative to the other to change pitch:

```bash
caput DT:FM:DOWNSTREAM_Z 2
```

The mirror base length is taken from `limPhysY`:

```text
base = abs(limPhysY[1] - limPhysY[0])
pitch = initial_pitch + atan2(downstream_delta - upstream_delta, base)
center.z = initial_center.z + (upstream_delta + downstream_delta) / 2
```

Check the sign convention against the mirror orientation in xrtGlow. If raising the downstream motor tilts the mirror the wrong way, swap the upstream and downstream PV names or change the sign of the height difference.

As an exercise, extend the same idea to a tripod mirror mount:

- one upstream motor in the middle
- two downstream motors at the back sides of the mirror

Use the three motor positions to calculate pitch and roll, then update the mirror orientation in the xrt model.
