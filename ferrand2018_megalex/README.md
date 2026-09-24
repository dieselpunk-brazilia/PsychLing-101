# ferrand2018_megalex — MEGALEX, French visual lexical decision

This folder contains the standardized preprocessing scripts and LLM prompts for the **visual lexical decision** part of MEGALEX, contributed to the PsychLing-101 database. The auditory part of the megastudy is not included.

## Citation
Ferrand, L., Méot, A., Spinelli, E., New, B., Pallier, C., Bonin, P., Dufau, S., Mathôt, S., & Grainger, J. (2018). MEGALEX: A megastudy of visual and auditory word recognition. *Behavior Research Methods*, 50(3), 1285–1307. https://doi.org/10.3758/s13428-017-0943-1

## Data Source
OSF project of the authors: [https://osf.io/7z6th/](https://osf.io/7z6th/), folder *MEGALEX Word data trial-by-trial Visual + Auditory*, archive `By trials Auditory & Visual-MEGALEX.zip`.

## Task
Visual lexical decision in French. Participants decided, as quickly and as accurately as possible, whether a letter string was a French word or not, using a gamepad: "yes" (word) with the right index finger and "no" (pseudoword) with the left index finger.

Each trial began with a fixation point (250 ms), followed by the letter string, which stayed on screen until the response or for a maximum of 2 s. At the end of each block participants saw their mean reaction time and error rate for words.

## Design
96 participants are in the data file. The item set is 28,466 French words and the same number of pseudowords; each participant saw half of it, about 27,000 trials in 109 blocks, over several weeks. Each word was seen by about 46 participants.

| | trials | unique strings | accuracy | median RT (ms) |
|---|---|---|---|---|
| word | 1,311,832 | 28,466 | 0.889 | 567 |
| pseudoword | 1,284,263 | 27,843 | 0.927 | 620 |

## Folder contents

```
ferrand2018_megalex/
├── original_data/        # Author files, unmodified
├── preprocess_data.py    # Reads original_data/, writes processed_data/exp1.csv
├── processed_data/       # exp1.csv, codebook-canonical column names
├── CODEBOOK.csv          # Descriptions of the columns used here
├── generate_prompts.py   # Reads processed_data/, writes prompts.jsonl.zip
├── prompts.jsonl.zip     # Natural-language prompts, several per participant
└── README.md             # This file
```

### `original_data/`
Extracted without modification from `By trials Auditory & Visual-MEGALEX.zip`:

- `Visual.WpuispW.essais_ElTec.txt` — trial-level visual lexical decision data, words and pseudowords, 2,596,095 rows. Semicolon-separated; the header names 9 columns but every row has 10 fields, the first one being an unnamed row index written by R.
- `Infos sur libellés variables.txt` — the authors' short description of the columns (in French).

### `processed_data/exp1.csv`
One row per trial, 2,596,095 rows, sorted by participant and presentation order. Produced by `preprocess_data.py` (about 10 s).

| column | original column | notes |
|---|---|---|
| `participant_id` | `subject` | |
| `trial_id` | `numinit` | unique across the file, increases with presentation order within a participant, not contiguous |
| `trial_order` | — | 0-indexed presentation order, derived by sorting each participant's trials on `numinit` |
| `phase_id` | `block` | 1–109 |
| `trial_in_block` | `trial_num_block` | position within the block, 1-indexed |
| `stimulus_type` | `category` | `w` → `word`, `p` → `pseudoword` |
| `stimulus` | `target` | |
| `response` | — | derived, see below |
| `accuracy` | `correct` | 1 = correct |
| `rt` | `rt` | ms, unchanged |
| `repetition` | `repetition` | unchanged, see below |

## Author-side preprocessing vs. this repository
The author file already has 3,939 technical failures removed, which leaves 2,596,095 trials. The article removes more trials before analysis:

| step | trials removed | remaining |
|---|---|---|
| trials in the author file | | 2,596,095 |
| RT < 300 ms or > 2,000 ms | 8,658 | 2,587,437 |
| 3 participants (1004, 1005, 1006) | 73,272 | 2,514,165 |
| 8 blocks | 1,981 | **2,512,184** (as reported in the article) |

**None of these exclusions is applied here.** `exp1.csv` and the prompts contain all 2,596,095 trials, so users can apply the authors' criteria or their own. The RT and participant steps can be reproduced from `exp1.csv` (8,655 trials under 300 ms and 3 over 2,000 ms; participants 1004–1006 have 80,423 trials, of which 7,151 fall under the RT criterion). The 8 excluded blocks are not identified in the article, so that step cannot be reproduced.

## Notes
- **Instruction is reconstructed.** The original instruction text is not part of the OSF deposit. The instruction in the prompts is reconstructed from the Method section of the article, with the participant's randomised button labels in place of the gamepad buttons.
- **`response` is derived.** The data file records accuracy but not which button was pressed. In a two-choice task the response follows from the stimulus type and accuracy: word & correct → `word`, word & incorrect → `nonword`, pseudoword & correct → `nonword`, pseudoword & incorrect → `word`. The 3 trials with RT > 2,000 ms may be timeouts rather than responses; the data file does not distinguish them, and they are kept and coded like the others.
- **`repetition` is not documented by the authors.** Their column description leaves it blank. It is 1 on 1,029 trials (937 pseudoword, 92 word trials, 45 distinct strings) and 0 otherwise. In this file every second presentation of the same string to the same participant (990 trials) has `repetition = 1`; the remaining 39 flagged trials are the only presentation of their string to that participant left in the file, so their first presentation was probably removed with the technical failures. The column is kept unchanged.
- **Positions 1–10 of each block are missing.** In almost every block the recorded `trial_in_block` values start at 11 and run to 260 (to 100 in block 109). The first ten trials of each block are not in the author file; the article and the column description do not say why. `trial_in_block` is kept as recorded.
- **Not included:** the OSF project also contains the auditory lexical decision data (trial-level and item-level), separate pseudoword trial files for both modalities, item-level tables for visual and auditory words (`vw.ParItems.DVandIV.txt`, `aw.ParItems.DVandIV.txt`), and the audio files of the auditory pseudowords. None of them is part of this contribution.

## Prompts
`generate_prompts.py` writes `prompts.jsonl.zip` (about 30 s). A session of ~27,000 trials is far above the 32K-token limit, so each participant's session is split into chunks of 4 consecutive blocks (blocks 1–4, 5–8, …, 109), 2,686 prompts in total. Every chunk opens with the instruction and states which blocks it covers; trials are numbered from 1 within each block.

Fields: `text`, `experiment` (`ferrand2018_megalex/exp1`), `participant_id`, `blocks` (the block numbers in the chunk) and `rt` (one value per trial, in prompt order).

The two button labels are two distinct capital letters drawn per participant, from a seed derived from the participant identifier, so they are the same in all chunks of one participant and the archive is reproduced byte for byte on re-running. Only the participant's button press is wrapped in `<< >>`.

Each trial ends with `Correct.` or `Incorrect.`, following example A of the repository README. The article describes feedback only at the end of each block (mean RT and error rate for words); the per-trial word is part of the prompt format and does not reproduce the display. The block-end feedback screens are not in the data and are not reconstructed.

```
In this experiment you will see letter strings presented one at a time in the centre of the screen. [...] Press F for a real word and E for a pseudoword. [...]

The trials below are from blocks 1-4 of the experiment.

Block 1:
Trial 1: The letter string is 'haboua'. You press <<E>>. Correct.
Trial 2: The letter string is 'toquée'. You press <<E>>. Incorrect.
Trial 3: The letter string is 'empêche'. You press <<F>>. Correct.
```

Largest prompt: 21,043 tokens (tiktoken `cl100k_base`) and 70,197 characters. The script checks every prompt against the limit, using tiktoken when it is installed and a conservative estimate otherwise.

## License
TODO: no license on OSF; requested from the authors.
