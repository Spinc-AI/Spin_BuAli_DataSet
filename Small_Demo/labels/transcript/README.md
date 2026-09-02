# Transcript labels — not yet created

This folder is for **verbatim Persian transcripts**: what was actually said in
each dictation, word for word, with the English radiology terms written as they
were spoken.

One file per recording, named to match it:

```
labels/transcript/DPM89130.txt   ←   DPM89130.MP3
```

It is empty, and this file exists only so the folder survives a clone.

## Why these cannot be generated from the images

The images in this dataset are the **signed English report** — a different
language from the dictation, restructured into template order. There is no way
to recover from them what the radiologist actually said.

Creating these means listening to each MP3 and typing it out.

## Why they matter

Two different questions need two different labels:

| Question | Label | Have it? |
|---|---|---|
| Is the finished report correct? | `../report/` | yes |
| Did the model hear the words correctly? | here | no |

Until this folder is filled, speech recognition can only be judged through its
effect on the final report — never on its own.

## The cheaper way to fill it

Do not type from scratch. Run an STT model over the audio first, then correct
its draft with the signed report open alongside: the report tells you what the
findings should be, so mishearings stand out immediately.

The benchmark already writes those drafts to `transcripts.json` for recordings
that have no label yet.
