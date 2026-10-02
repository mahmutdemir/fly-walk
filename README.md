![](images/Orientation_method_1_croppng.gif)

# flyWalk

flyWalk tracks many fruit flies walking freely in an arena while an odour plume moves over them.
For every fly in every frame, it records the fly's position, which way its head points, and the
odour concentration at its two antennae. That is hard for three reasons. From above, a fly is a
near-symmetric dark ellipse, so head and tail look alike. Flies touch, collide and walk across
each other (including flies on opposite surfaces of the arena), so individuals merge into one blob
and must be split and re-identified afterwards. And the odour a fly senses is not the odour at
its centre: it has to be read at each antenna, which depends on getting orientation right first.

## Authorship

I (Mahmut Demir) wrote this code as the analysis toolkit for my work in the
[Emonet lab](https://github.com/emonetlab) at Yale. That covers roughly 33k of the repository's
47k MATLAB lines: multi-fly tracking through collisions, orientation recovery, antenna-level
odour-signal extraction, the manual-annotation GUI, and the acquisition and analysis code.
Everything not listed below is mine:

- It is built on [`movieAnalyser`](https://github.com/sg-s/movie-analyser), a base class by
  my labmate Srinivas Gorur-Shandilya. A modified copy is included as `src/movieAnalyser.m`.
  The core class file `src/@flyWalk/flyWalk.m` is jointly copyrighted to him and me.
- Some acquisition and annotation files are his or adapted from his: `annotateVideo`,
  `maskVideo`, `oval`, `dep/DeviceControl/Kontroller_Walk.m` (from
  [kontroller](https://github.com/sg-s/kontroller)) and `HandleFlyWalkVideo.m`.
- Third-party MATLAB utilities are bundled with their original headers and licenses:
  `export_fig`, `geom2d`, `mask2poly`, `fitellipse`, `inpaint_nans`, `smooth2a`,
  `subtightplot`, `plotboxpos` and others in `dep/`.

The canonical copy is [emonetlab/fly-walk](https://github.com/emonetlab/fly-walk). This
repository is a fork of it at the same commit.

## Publications

This software produced the tracking and odour-signal data in:

- Demir M, Kadakia N, Anderson HD, Clark DA, Emonet T (2020). Walking *Drosophila* navigate
  complex plumes using stochastic decisions biased by the timing of odor encounters.
  *eLife* 9:e57524. https://doi.org/10.7554/eLife.57524
- Kadakia N, Demir M, Michaelis BT, et al. (2022). Odour motion sensing enhances navigation of
  complex plumes. *Nature* 611, 754–761. https://doi.org/10.1038/s41586-022-05423-4

## Status

Research code. It was written to produce the results above and was last changed in 2020. There
is no automated test suite. Instead, the tracker was validated against frame-by-frame human
annotation (see [Validation](#validation)). The repository history is a single import of the
finished code, not a record of its development. Expect hard-coded paths in the acquisition
scripts (`dep/DeviceControl/`) and MATLAB-era conventions throughout.

## How it works

`flyWalk` is a MATLAB handle class (`src/@flyWalk/`) that subclasses `movieAnalyser`, which
supplies movie loading, frame iteration and a basic player GUI. `flyWalk` overrides
`operateOnFrame` so that each frame is processed as it is visited:

1. **Detect.** `findAllObjectsInFrame` thresholds the background-subtracted frame and measures
   candidate blobs with `regionprops`.
2. **Assign identities.** `mapObjectsOntoFlies` links blobs to existing tracks
   (`assignObjectsToTheseFlies`, gated by a maximum plausible walking speed) and starts new
   tracks with `assignObjectsToNewFlies`.
3. **Resolve collisions.** `findSuspectedMergedObjects` flags blobs that are too large given the
   previous frame. `resolveCollidingObjects` / `splitObject` split them (k-means on pixel
   positions). `resolveInteractingObjects` and `resolveOverpassingObjects` handle flies that touch,
   or pass over each other on opposite surfaces (told apart using their reflections).
4. **Orientation.** `getFlyOrientations` and the orientation post-processing functions decide
   which end is the head and correct head–tail flips over time.
5. **Odour signal.** Virtual antennae are placed from the fitted body ellipse and orientation,
   and the plume intensity is read at each one (`extractSignal`, `reOptAntenna`).
6. **Events and export.** `sortFlyWalk` and `sortEvents` classify odour encounters and
   interactions, and `GetExpMatrixFlyWalk` collects many tracked videos into one experiment
   matrix for analysis.

Main entry points on the class: `track`, `trackNgetOrientations`, `trackNRunSignalModules`,
`extractSignal`, `reProcessSignal`, `createGUI`, `playVideo`, `makeVideo`, `save`.

### Manual annotation and validation

`annotateFlyWalk(f)` opens a GUI for frame-by-frame human annotation of a tracked video. For each
fly and frame, the annotator records:

- movement state (stop, walk, back up, jump)
- grooming (wing, hind leg, foreleg)
- whether the antenna signal is clean
- orientation, and whether the tracker's orientation had to be flipped
- collisions, flies passing over each other, antenna overlap, and reflections

The tracker's output pre-fills each frame (`guessAnnotation`), so the annotator corrects
labels rather than entering every one. Annotation resumes where it was left, and is saved with
the `flyWalk` object in `f.annotation_info`.

### Validation

For every fly in every frame, the tracker decides whether the antenna signal can be trusted. It
rejects a frame when any of these hold:

- the fly is within about 8 mm of the arena border, where its apparent size changes and
  collisions become hard to resolve;
- the fly is colliding with, or passing over, another fly;
- the fly is jumping, so its shape, orientation and antenna position are unreliable;
- a virtual antenna overlaps any fly, including itself, or any reflection predicted from the
  arena geometry.

To measure how well this works, signal reliability was annotated by hand, frame by frame, in the
annotation GUI and compared with the tracker's decision.

| Video | Fly-frames | Agreement | Accepted frames that were bad | Usable frames rejected |
|---|---|---|---|---|
| 17 flies, 5 border flies excluded | 47,335 | 92.5% | 22 of 39,797 (0.06%) | 3,527 of 43,302 (8%) |
| Same video, all 17 flies | 70,153 | 66% | 23 of 41,275 (0.06%) | 23,752 of 65,004 (37%) |
| A second video | 31,122 | 30% | 1 of 445 (0.2%) | 21,787 of 22,231 (98%) |

The rule is designed to keep bad signal out of the analysis, at the expense of losing good
signal. It almost never accepts a bad frame: across all annotated frames, 24 of the 41,720 it
accepted were bad. The price is the usable frames it rejects, and how many depends strongly on
the video. The signal from rejected frames is discarded.

Supporting code in `dep/` covers acquisition (`DeviceControl/`: camera capture, odour-delivery
control, conversion from `.avi` to `.mat`), PIV flow-field analysis of the plume (wrapping PIVlab),
and the trajectory statistics used in the papers.

## Usage

```matlab
% Create a flyWalk object, open the movie and display the GUI
fileName = 'my_movie.mat';
gui_on = 1;
f = openFileFlyWalk(fileName, gui_on);
```

You can now watch the tracking in real time using the `Play` button.

### Video format

flyWalk reads videos stored as `.mat` files. To convert `.avi` files to `.mat`, use `Vid2Mat`.
flyWalk also needs per-video metadata: the source and camera location, the pixel-to-mm
conversion, exclusion borders and so on. Set these with `HandleFlyWalkVideo`.

### Useful properties

* `show_live` — show tracking live as it happens
* `fly_body_threshold` — the image is thresholded at this value (8-bit)
* `min_fly_area` — objects smaller than this (mm²) are ignored
* `maximum_distance_to_link_trajectories` — maximum plausible speed (mm/s) for linking a track
* `start_frame` — frame to start tracking from
* `ft_debug` — print debug messages
* `current_raw_frame`, `current_objects`, `current_object_status` — state of the frame being processed
* `tracking_info` — all tracking output (see below)

### Output

The goal is total data capture: for every fly, in every frame, all relevant tracking information.
It is stored in the structure `tracking_info`. Each field is a matrix with one row per fly and one
column per frame.

## License

GPL-3.0. See [LICENSE](LICENSE). Bundled third-party files keep their own copyright notices and
licenses (for example `dep/altmany-export_fig/LICENSE` and `dep/fitellips_license.txt`).
