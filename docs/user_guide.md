# SimpleTrack ML user guide

Back to the [README](../README.md).

## Workflow in detail

1. For a generic TIFF without channel metadata, enter its channel count before opening. OME-TIFF and ImageJ metadata are read automatically.
2. Open the TIFF or DV movie and confirm the displayed T, C, Z, Y, and X dimensions.
3. Assign a display color to each channel. Colors affect only the preview, never the numeric measurements.
4. Select the detection channel and Z plane, adjust detection/tracking settings, and click **Analyze**.
   - To analyze only part of the field, choose **Rectangle** or **Freehand** under **Analysis region (ROI)**, click **Draw ROI**, then drag on the image. For Freehand, hold the button and trace the outline; releasing closes it. Press Esc to cancel drawing. The ROI is outlined in dashed cyan. Click **Whole field** to remove it. After changing the ROI, click **Analyze** again.
   - With an ROI, puncta are detected only inside it, and the detection threshold's noise estimate uses only ROI pixels. Tracks end where a punctum leaves the ROI. Intensities and background annuli are still measured from the full image, so a punctum near the ROI edge is measured correctly.
5. Select a track from the results list. Its full XY trajectory is drawn in yellow, and its channel intensity traces appear below the movie. The XY tab shows an equal-scale coordinate plot with start, end, and current-frame markers.
   - Use the mouse wheel over the movie to zoom toward the cursor, drag to pan, and double-click to reset the view.
   - The **Channel histogram** tab shows one histogram per channel. Drag the blue black-level marker or the red white-level marker to change that channel's displayed contrast. Double-click a histogram to reset it to 1.0-99.8%. Display contrast never alters the source image or the results.
6. Select one or more tracks and label them **Good**, **Possible**, **Bad**, or clear the label. **Accept ML prediction** labels the selected tracks with their prediction. **Next unlabelled** jumps to the next track to review. Click the column headings to sort by track ID, label, ML prediction, fold change, or number of frames.
7. To make a figure of one track, select it and click **Snapshot...**.
   - The montage has one row per channel plus a Merge row, and one column per chosen frame, cropped around the particle.
   - Enter frames as 1-based numbers, ranges, and steps, for example `10, 20-60:10, 75`.
   - By default the crops follow the particle. Gaps are interpolated and shown with a gray circle.
   - By default each channel uses **one display range for all frames**, taken from the histogram percentiles pooled over all crops. This keeps tile brightness comparable between frames. **Per frame (auto)** contrast is available but hides intensity changes.
   - Optionally add the intensity trace with the chosen frames marked, a scale bar (enter the pixel size), and time stamps.
   - **Save montage (PNG)** writes the figure. **Save raw crops (TIFF)** writes the unscaled crop pixels as an ImageJ T,C,Y,X hyperstack (NaN outside the image) for further adjustment in Fiji.
8. Choose the export scope (**All**, **Selected**, **Good**, **Possible**, or **Good + Possible** tracks), then export to Excel or CSV.

**Save Project** stores everything needed to resume without rerunning the analysis in a `.simpletrack` file:

- the source-image reference, settings, ROI, and analysis results;
- labels, sorting, the selected track, and the current frame;
- channel colors, zoom/pan, and histogram display limits.

**Open Project** restores it. The original TIFF or DV file is referenced, not copied. If it has moved, SimpleTrack asks you to locate it.

## Assembly ML assistant

- It uses the **detection (tracking) channel** only. An assembly event must rise to about **3-fold** from the start of the track to its peak. Change **Required fold change (peak/start)** to use a different fold, then press Enter or click **Update predictions**.
- Fold change = peak of the 3-frame median-smoothed background-corrected mean ÷ median of the first few points (15% of the track, 2-5 points). The start level is floored at the trace noise, so a near-zero start cannot give an absurd fold.
- The intensity plot shows the start level (dotted), the peak (dashed), the fold, and the prediction.
- Until there are at least 2 Good and 2 Bad labels, predictions come from the fold rule alone. After that, a logistic-regression model learns from 10 trace features. The learned model's weight grows as labels accumulate (equal to the rule at 10 labels).
- A track below the required fold is never predicted Good.
- **Possible** tracks are stored in the model file but not used as training examples.
- Labelled examples accumulate across movies in `%USERPROFILE%\.simpletrack\assembly_model.json`. **Reset model...** deletes them all.

## Image dimensions

- Data are normalized internally to `T,C,Z,Y,X`.
- For generic unlabelled sequential pages, SimpleTrack assumes channel-fast order: `T0-C1, T0-C2, ..., T1-C1, ...`.
- If the movie contains Z planes, the selected plane is used for 2D tracking.
- X increases rightward and Y increases downward, in pixel coordinates.
- Large uncompressed TIFFs and standard DeltaVision DV movies are memory-mapped when possible to limit RAM use.

## Output columns

Excel export contains:

- `measurements`: one row per tracked time point, with time, XY, and all channel intensities.
- `track_summary`: track duration, start/end XY, intensity ranges, and net intensity changes. These are given for both mean intensity (`chN_start`, `chN_max`, ...) and sum intensity (`chN_sum_start`, `chN_sum_max`, ...).
- `settings`: analysis parameters (including the ROI vertices, or `whole field`), channel colors, input path, source axes, tracking channel, fold threshold, and model file.

For every channel, measurements include:

- the raw disk mean and raw disk sum (`chN_raw_sum`);
- the local annulus background;
- the background-corrected mean and background-corrected sum (`chN_corrected_integrated`);
- the change from the preceding detection and from the track's first detection, for both the mean and the sum (`chN_sum_delta_from_previous`, `chN_sum_change_from_first`).

CSV export contains the measurement table only.

`frame` is the 1-based frame number; `frame_index` is the 0-based array index. `user_label` is `Good`, `Possible`, `Bad`, or `Unlabelled`. The ML columns are `ml_prediction`, `ml_probability`, `fold_change`, and `meets_fold_threshold`.

The **Intensity traces** tab has a **Plot** selector to switch between mean and sum intensity. In the Snapshot window, tick **Trace shows sum intensity** to plot the background-corrected sum instead.

## Parameter guide

- **Detection channel**: the channel with the clearest puncta. All channels are still measured.
- **Threshold sigma**: increase to remove false spots; decrease to recover dim spots.
- **Spot radius**: approximate punctum radius in pixels.
- **Max link distance**: the largest expected movement between adjacent frames.
- **Allowed gap**: the number of missed frames a track may bridge.
- **Minimum track length**: discard tracks with fewer detections than this.
- **Intensity disk radius** (default 2): pixels whose centres lie within this radius of the punctum centre are averaged as its signal. For a diffraction-limited spot about 3 px wide at half maximum, a radius of 2 captures roughly 70% of the signal and tolerates about 0.5 px of centroid error or channel misregistration.
- **Background radii** (defaults 4 and 7): the median of pixels with inner < distance ≤ outer is the local background. Keep the inner radius at least 1-2 px beyond the disk, so the spot's tail is not counted as background. All three values are radii, not diameters.
