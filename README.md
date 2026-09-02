# Spin BuAli — Dataset

Radiology dictations paired with their signed reports, and the ground truth
labels transcribed from those reports.

Data and labels only. No code, no scoring — that lives in
[`Spin_BuAli/benchmark/`](https://github.com/) and reads from here.

## Layout

**One folder per dataset, self-contained.** `Small_Demo/` is the first:

```
Small_Demo/
├── DPM89130.MP3     the dictation — Persian, code-switched with English terms
├── 89130.jpg        the signed report — a photo of the typed Word document
└── labels.csv       one row per case: the label, and what it pairs with
```

Nine cases: `89130 · 89136 · 89151 · 89170 · 89196 · 89197 · 89247 · 89272 · 89273`.

Nothing about a dataset lives outside its own folder, so a folder can be moved,
copied, zipped or mounted somewhere else and still work.

## labels.csv

Four columns, one row per case:

| Column | |
|---|---|
| `asset_id` | the audio stem — what every score is reported against |
| `audio` | filename, relative to this folder |
| `image` | filename of the report photo the label was read from |
| `report` | the label: the full report text |

`report` holds the report as written, line breaks and all, in a quoted CSV
field. Any spec-compliant reader gives it back unchanged:

```python
import pandas as pd
labels = pd.read_csv("Small_Demo/labels.csv")      # encoding is detected
```

Written **UTF-8 with a BOM**, so Excel opens it correctly — these labels are
meant to be corrected in a spreadsheet. Python's `csv` and pandas both read it
without being told.

## Adding a dataset

A new folder in the same shape: audio and images flat at the top, one
`labels.csv` beside them with those four columns.

Watch the filenames — the audio carries a `DPM` prefix the image does not, so
`89130.jpg` goes with `DPM89130.MP3`. Whatever generates the next `labels.csv`
cannot assume matching stems.

## What these labels can and cannot measure

**The audio and the images are not the same text, and not the same language.**
The dictation is Persian with English radiology terms mixed in. The signed
report is entirely English, restructured into house template order.

So these labels grade **the whole pipeline** — audio in, finished report out.
They cannot grade speech recognition on its own. Measured on case 89130, a
faithful Persian transcript scored against its own English report:

| | |
|---|---|
| WER | **0.93** |
| CER | 0.95 |
| chrF | 0.04 |
| medical term F1 | 0.63 |
| number errors | 0 of 3 measurements |

That 0.93 is not a bad model. It is one study described twice in two languages,
and every model scores about the same — so word error rate cannot rank speech
recognition here.

The clinical metrics survive the language change, because
`evaluation/clinical_terms.json` maps Persian and English variants onto shared
concept ids: `stone`, `calculus` and `سنگ` are one concept.

Grading speech recognition on its own would need verbatim Persian transcripts of
what was said. Those cannot be recovered from the images, and none exist — which
is why `labels.csv` has a `report` column and nothing else. If transcript labels
are ever made, they belong in a second column, not mixed into this one.

## Don't rank on word error rate either

Even comparing English report to English report, WER is close to useless on this
data, because the reports are templated. Two lines appear verbatim in all nine.
Scored against report 89130:

| | WER | laterality errors | term F1 |
|---|---|---|---|
| Correct report, reworded | 0.404 | 0 | 1.00 |
| Correct report, **left/right flipped** | 0.413 | **3** | 1.00 |
| A **different patient's** report | 0.476 | 2 | 0.79 |

Across all nine reports, WER between two unrelated patients runs 0.390–1.03,
median 0.615 — overlapping the 0.404 a *correct* report scores. And flipping
left for right, the error that sends a surgeon to the wrong kidney, moves WER by
0.009.

Rank on the clinical metrics: negation, laterality, number, unit, critical
omissions, term F1. They separated all three cases cleanly. Keep WER as a rough
"how much rewriting is left" signal only.

## How the report labels were made

Transcribed from the photographs by reading them directly, not by OCR — they are
phone photos of a screen, with moiré banding, glare and perspective skew.

**Verbatim, including the reports' own spelling.** `vessicles`, `homogenous`,
`injunction` and similar appear as written, because a reference is what was
signed rather than a corrected version of it.

Worth revisiting: a model that spells `vesicles` correctly currently scores an
error. Normalising the labels would fix that and introduce the opposite bias —
they would stop being what the radiologist approved. Left verbatim, flagged here.

## Known gaps

**Vocabulary is the limiting factor.** In the measurement above only 10 of 22
reference concepts matched, and 3 of 5 laterality comparisons came out wrong.
Both are vocabulary problems rather than metric problems:
`clinical_terms.json` is a 41-concept seed whose Persian side is much thinner
than its English side.

These pairs are good material for fixing that — each case gives the Persian
spoken form and the English written form of the same finding.

**Nine cases is a demo, not a benchmark.** Enough to prove the machinery and
find the vocabulary holes. Not enough for a percentile, a corpus WER worth
quoting, or a ranking between two close models.

## Privacy

The reports carry no patient names. The **audio has not been checked** — a
radiologist may say a name aloud. Treat the MP3s as containing patient
information and keep this repository private.
