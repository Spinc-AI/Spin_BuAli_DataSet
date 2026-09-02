# Spin BuAli — Dataset

Paired radiology dictations and their signed reports, and the ground truth
labels derived from them.

```
Small_Demo/
├── DPM89130.MP3        the dictation — Persian, code-switched with English terms
└── 89130.jpg           the signed report — a photo of the typed Word document

labels/
├── report/DPM89130.txt transcribed from the image (done — 9 of 9)
└── transcript/         verbatim Persian of what was said (not yet created)

manifest.json           pairs everything by asset_id
```

Nine cases: `89130 · 89136 · 89151 · 89170 · 89196 · 89197 · 89247 · 89272 · 89273`.

## Read this before scoring anything against these labels

**The audio and the images are not the same text, and not even the same
language.** The dictation is Persian with English radiology terms mixed in. The
signed report is written entirely in English, restructured into house template
order and cleaned up.

So a word error rate between an STT transcript and one of these reports is
meaningless. Measured directly:

| | |
|---|---|
| WER | **0.93** |
| CER | 0.95 |
| chrF | 0.04 |
| medical term F1 | **0.63** |
| number errors | 0 of 3 measurements |

That 0.93 is not a bad model. It is the same study described twice, in two
languages, and any model would score about the same — which makes WER useless
for ranking STT here.

The clinical metrics survive the language change, because
`evaluation/clinical_terms.json` maps Persian and English variants onto shared
concept ids: `stone`, `calculus` and `سنگ` are one concept. Measurements,
negation, laterality and units compare across the two languages as they stand.

## Two kinds of ground truth, and only one exists

| | What it is | Scores | Status |
|---|---|---|---|
| **Report truth** | `labels/report/` — the signed English report | the **whole pipeline**: audio → final report | ✅ 9 of 9 |
| **Transcript truth** | `labels/transcript/` — verbatim Persian of what was said | the **STT stage alone** | ❌ none yet |

The benchmark in `Spin_BuAli/benchmark/` compares STT models against each other.
That needs transcript truth. Report truth cannot substitute for it.

Report truth is not a lesser thing — it is the right reference for the question
*"is the finished report correct?"*, which is what a radiologist actually cares
about, and it is what the controller's pipelines produce. It just answers a
different question from *"did the model hear correctly?"*.

Creating transcript truth means someone listening to each MP3 and typing what
was said, in Persian, including the English terms as spoken. There is no
shortcut from the images.

## How the report labels were made

Transcribed from the photographs by reading them directly, not by OCR — the
images are phone photos of a screen, with moiré banding, glare and perspective
skew that defeat conventional OCR.

**Transcribed verbatim, including the reports' own spelling.** `vessicles`,
`homogenous`, `injunction` and similar appear as written. A reference is what
was signed, not a corrected version of it.

That is a decision worth revisiting: if a model outputs the correct spelling
`vesicles`, it currently scores an error. Normalising the references would fix
that and introduce a different bias — the labels would stop being what the
radiologist actually approved. Left verbatim for now, flagged here.

## Known gaps

**Concept coverage is the limiting factor.** In the measurement above only 10 of
22 reference concepts matched, and 3 of 5 laterality comparisons came out wrong.
Both are vocabulary problems, not metric problems: `clinical_terms.json` is a
41-concept seed whose Persian side is much thinner than its English side.

These nine pairs are the right corpus to mine for that — every case gives a
Persian utterance and its English equivalent for the same finding.

**Nine cases is a demo, not a benchmark.** Enough to validate the plumbing and
find the vocabulary gaps. Not enough for a percentile, a corpus WER anyone
should quote, or a ranking between two close models.

## Using it

The manifest is the format `Spin_BuAli/benchmark/` reads:

```json
{"asset_id": "DPM89130",
 "audio": "Small_Demo/DPM89130.MP3",
 "image": "Small_Demo/89130.jpg",
 "reference": "labels/report/DPM89130.txt",
 "reference_kind": "report"}
```

`reference_kind` says which question the label answers. Keep it accurate as
transcript labels are added — it is the only thing preventing the two from being
scored as though they were interchangeable.
