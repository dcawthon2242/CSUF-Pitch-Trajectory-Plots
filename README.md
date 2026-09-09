# CSUF Pitch Trajectory Plots

Pregame scouting visuals for Cal State Fullerton hitters. For an opposing pitcher, the tool draws the recent trajectories of one pitch type against one batter side and colors each pitch by how good a swing at it would have been. The goal is to give a hitter a picture of what the pitches worth attacking look like out of the hand, and what the ones to leave alone look like, before the ball reaches the point where a decision has to be made.

Built on NCAA TrackMan pitch data and the swing-decision models shared with the
[Hitter Reports](https://github.com/dcawthon2242/CSUFHitterReports).
See also
[Pitcher Reports](https://github.com/dcawthon2242/CSUFPitcherReports) ·
[Catcher Reports](https://github.com/dcawthon2242/CSUFCatcherReports)

<img width="1259" height="838" alt="Trajectory plot example" src="https://github.com/user-attachments/assets/da45b33a-86b8-45b1-9f25-0b84530e5298" />

## Reading the plot

The view is from the batter's box. The batter silhouette is placed on the correct side of the plate for the handedness being shown, the dashed rectangle is the strike zone, and the mound is drawn behind it for depth.

| Element | Meaning |
| --- | --- |
| Colored lines and dots | The 20 most recent pitches of the chosen type to the chosen batter side. Each is drawn from release to the hitter's decision point and colored by swing run value: red is a pitch worth swinging at, blue is a pitch to lay off, gray is neutral. |
| Red line | The average trajectory of the 5 best pitches to swing at (out of the last 50), extended all the way to the plate. |
| Blue line | The average trajectory of the 5 worst pitches to swing at (out of the last 50), extended to the plate. |
| Black line | The average trajectory of all 20 plotted pitches, extended to the plate. |

The decision point is 150 ms before the ball reaches the front of the plate. Trajectories stop there because that is roughly the last moment a hitter can still commit or hold off. The extended average lines show where each group finishes.

## How it works

1. **Select pitches.** Filter the dataset to the pitcher, pitch type, and batter side. Drop pitches whose release point is more than 0.25 ft from the pitcher's mean release for that sample so a stray tagged pitch does not pull the picture around.
2. **Take the most recent 50** by `ZoneTime`, then plot the most recent 20.
3. **Compute trajectories** from the TrackMan 9-parameter fit. Position at time *t* is `p0 + v0 * t + 0.5 * a * t^2` in each axis, using `x0`, `vx0`, `ax0`, `z0`, `vz0`, `az0`. Each pitch is evaluated up to `ZoneTime - 0.15`.
4. **Color by swing run value.** The column `x_swing_run_value` is the modeled change in run expectancy if the hitter swings at that pitch, given the count, from the swing-decision model chain (swing, whiff, contact, batted-ball type). Values are mapped onto a blue-gray-red scale.
5. **Extend the extremes.** Average the trajectory parameters of the 5 highest and 5 lowest run-value pitches in the 50-pitch window and draw those averages to the plate in red and blue. Draw the average of all 20 plotted pitches in black.

## Repository layout

```
PitchPerspectivePlot.R     horizontal_movement_chart(): draws one plot
                           save_pitcher_plots(): exports every pitch type x batter side for a pitcher to PDF
PitchPlotExport.R          Earlier standalone version of the export helper
D1SwingDecisions.R         Trains the swing-decision models that produce the run values (Division I TrackMan, 2025)
```

## Requirements

R packages: `dplyr`, `png`, `shape`. The training script additionally uses `tidyverse`, `lightgbm`, `xgboost`, `caret`, `caTools`, `pROC`, and `rBayesianOptimization`.

Input data: a TrackMan data frame with the following columns.

| Column | Use |
| --- | --- |
| `Pitcher`, `TaggedPitchType`, `BatterSide`, `PitcherThrows` | Filtering and titling |
| `x0`, `z0`, `vx0`, `vz0`, `ax0`, `az0` | Release position, velocity, and acceleration for the trajectory |
| `ZoneTime` | Time from release to the plate, used for recency and for the decision point |
| `x_swing_run_value` | Modeled run value of swinging at the pitch (see above) |

A batter silhouette PNG is also required. The script points at a local path; change `img_path` to your copy.

## Usage

Draw one plot interactively:

```r
source("PitchPerspectivePlot.R")
horizontal_movement_chart("Smith, Dylan", D1TM25, pitch_type = "Sinker", batter_side = "Right")
```

Export every pitch type against both batter sides for a pitcher, one PDF each, into the working directory:

```r
save_pitcher_plots("Smith, Dylan", D1TM25)
# writes Dylan_Smith_Sinker_vs_RightHanded.pdf, Dylan_Smith_Slider_vs_LeftHanded.pdf, ...
```

Pitcher names follow the TrackMan "Last, First" convention. Passing `pitch_type = NULL` plots all of a pitcher's pitch types together.

## Notes

- Twenty pitches is a deliberate cap. More than that and the lines overlap into noise; fewer and one unusual pitch dominates the colors.
- The run values reflect the count each pitch was thrown in. A pitch that would be a fine swing at 2-0 can rate poorly at 0-2, so two nearly identical trajectories can carry different colors.
- The plot is drawn with base R graphics rather than ggplot so the batter image, mound, and trajectories share one coordinate system and export cleanly to PDF.
