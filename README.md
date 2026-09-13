# Spin BuAli — Dataset

Radiology dictations paired with their signed reports, and the ground truth
labels transcribed from those reports.

Data and labels only. No code, no scoring — that lives in
[`Spin_BuAli/benchmark/`](https://github.com/) and reads from here.

## Layout

```
Small_Demo/              a dataset: the audio, and the labels for it
├── DPM89130.MP3         the dictation — Persian, code-switched with English terms
└── labels.csv           one row per case

report_images/           where the labels came from, kept for provenance
└── 89130.jpg            a photo of the signed report on screen
```

Nine cases: `89130 · 89136 · 89151 · 89170 · 89196 · 89197 · 89247 · 89272 · 89273`.

A **dataset folder** holds what a benchmark run consumes: audio in, labels to
score against. Nothing else. Copy or mount that folder and a run works.

`report_images/` is source material, not dataset. The reports were read out of
those photographs once, into `labels.csv`; nothing reads them again. They stay
so a label can be checked against what was actually signed, and they are shared
across datasets rather than duplicated into each.

## labels.csv

Six columns, one row per case:

| Column | |
|---|---|
| `asset_id` | the audio stem — what every score is reported against |
| `audio` | filename, relative to this folder |
| `image` | the photo the label was read from, in `report_images/` — provenance, not a path a run needs |
| `report` | the label: the full report text |
| `modality` | the imaging technique, e.g. `Ultrasound` — see `Spin_BuAli/docs/taxonomy/modalities.csv` for the reference vocabulary |
| `region` | body region(s) examined, `;`-separated when more than one, e.g. `Hepatobiliary;Retroperitoneum;Bladder;Genitourinary (Male)` — see `Spin_BuAli/docs/taxonomy/body_regions.csv` |

`modality`/`region` are known context about the recording -- what a real order
would say -- not something inferred from the audio or report. Fed to the LLM
as a prompt line by `Spin_BuAli/benchmark/context_labels.py`, for both the
`separate` and `multimodal` pipelines. All nine cases here are `Ultrasound`;
`region` differs per case because the exam covers different organs depending
on the patient's sex (`Genitourinary (Male)` vs `Genitourinary (Female)`) and,
for case 89136, whether the pelvic organs were examined at all (that case's
report has no prostate/uterus section, only a note on a VP-shunt-site fluid
collection — tagged `Abdomen` instead).

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

A new folder in the same shape: the audio flat at the top, one `labels.csv`
beside it with those columns. Report photos go into the shared
`report_images/`, not into the dataset folder. `modality`/`region` are
optional -- a `labels.csv` without them still loads fine (see
`Spin_BuAli/benchmark/dataset.py`'s `from_csv`), they just mean no context
line reaches the LLM for that dataset.

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
