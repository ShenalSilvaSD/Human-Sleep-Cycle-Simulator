# Sleep-EDF Expanded data

The simulator uses the **Sleep Cassette** subset of [Sleep-EDF Database Expanded v1.0.0](https://physionet.org/content/sleep-edfx/1.0.0/) by B. Kemp, DOI [10.13026/C2X676](https://doi.org/10.13026/C2X676). It reads expert-scored sleep-stage annotations from EDF hypnograms. The repository does not distribute the raw EDFs, and the notebook does not need the corresponding polysomnography signal files.

Sleep-EDF Expanded is available under the [Open Data Commons Attribution License v1.0](https://physionet.org/content/sleep-edfx/view-license/1.0.0/). The fitted probabilities and results in this project use information from that dataset. The repository’s [MIT licence](../LICENSE) covers the project code.

## Selected recordings and split

The manifest fixes 24 hypnograms across 12 participant prefixes (SC400 through SC411), with two nights each. The recording stems below take the suffix **-Hypnogram.edf**.

~~~
SC4001EC SC4002EC SC4011EH SC4012EC SC4021EH SC4022EJ
SC4031EC SC4032EP SC4041EC SC4042EC SC4051EC SC4052EC
SC4061EC SC4062EC SC4071EC SC4072EH SC4081EC SC4082EP
SC4091EC SC4092EC SC4101EC SC4102EC SC4111EC SC4112EC
~~~

The split uses the sorted participant prefixes and NumPy’s default generator with seed **4042**. It assigns all nights from each participant to the same partition.

| Partition | Participants | Nights | Window epochs | Valid epochs | Valid adjacent transitions |
|---|---:|---:|---:|---:|---:|
| Training | 9 | 18 | 18,083 | 18,053 | 18,006 |
| Held-out | 3 | 6 | 6,917 | 6,912 | 6,901 |

There were 30 movement epochs in the training windows and five in the held-out windows. Neither partition contained unscored or uncovered epochs within the selected windows.

## Preprocessing

1. Read the annotations using MNE. Expand their spans onto a 30-second grid, checking timing and overlaps.
2. Map W to **Wake**, stages 1 and 2 to **Light**, stages 3 and 4 to **Deep**, and R to **REM**.
3. Keep the window from first to last scored sleep with up to 30 minutes of padding on each side, within the recording’s available annotation span.
4. Preserve movement, unscored, and uncovered epochs as invalid positions. Count only consecutive epochs with two valid stages in the same recording. This prevents false transitions across gaps or recording boundaries.

## Reproduce the downloads

Open [the notebook](../notebook/Human_Sleep_Cycle_Simulator.ipynb) and run all cells with internet access. The notebook installs MNE 1.10.2, then fetches the manifest’s files from the versioned [PhysioNet endpoint](https://physionet.org/files/sleep-edfx/1.0.0/sleep-cassette/). If the primary request fails, it tries the [PhysioNet open-data mirror](https://physionet-open.s3.amazonaws.com/sleep-edfx/1.0.0/sleep-cassette/). Files are processed via a temporary directory. The notebook audit records the actual source URL and SHA-256 of each downloaded file alongside the split and epoch counts.
