# AFpeer-to-peerEngagement
This repository contains data handling and analyses files for the paper 'Message Signals and Peer Response in an Atrial Fibrillation Online Health Community: Observational Study of Reddit Posts Using Large Language Model Annotation'

# Message signals and peer response in an atrial fibrillation online health community

Analysis code for the manuscript *Message Signals and Peer Response in an Atrial Fibrillation Online Health Community: Observational Study of Reddit Posts Using Large Language Model Annotation* (Kuipers, Konus, Brundel, Birbil).

The study analyzes 5028 posts and 71,592 comments from the r/AFIB subreddit (February 2023 to November 2024). Each post was annotated with a large language model for its intentions, topics, and affective tone. Two message signals, medical terminology density and emotional intensity, were then related to the number, length, and lexical richness of the comments the post received.

The Reddit data and the human coders' workbooks are not included (see [Data](#data)). The code is shared so that every step from the raw export to the reported estimates can be inspected, and so that steps 7 to 10 can be rerun by anyone who obtains the data.

## Repository layout

```
.
├── README.md                this file
├── requirements.txt         Python packages and the versions used
├── .env.example             template for the Groq API key (steps 2 and 6 only)
├── data/                    input and intermediate files (not shared, see data/README.md)
├── glossary/                AFib glossary used for medical terminology density
│   ├── glossary_original.txt
│   ├── glossary_v2.txt         revised glossary used in the analysis
│   └── glossary_v2_changes.csv every change from the original, with the reason
├── src/                     the scripts, numbered in the order they are run
└── results/                 created by steps 8 to 10
```

## Pipeline

Run the scripts from the repository root, in order, for example `python3 src/09_main_models.py`. Every script starts with a short description of what it does, what it reads, and what it writes.

| Step | Script | What it does | Manuscript |
|---|---|---|---|
| 1 | `01_clean_raw_data.py` | Cleans the raw export: combines title and body, removes deleted content, strips URLs and Markdown, removes duplicate rows | Methods, Data Collection |
| 2 | `02_annotate_messages.py` | Annotates each post with LLaMA 3-70B-8192 (intentions, topics, disease outlook, coping confidence, emotional intensity) and computes message richness and the comment measures | Methods, Annotation Procedure and Measures; Appendix 1, Section A1 |
| 3 | `03_fill_comment_dates.py` | Restores missing comment dates | |
| 4 | `04_aggregate_by_message.py` | Aggregates to one row per post and derives season, time of day, and week part | Methods, Control Variables |
| 5 | `05_remove_unlabeled_and_merge.py` | Removes posts without any label and finalizes the label set | Methods, Data Collection |
| 6 | `06_relabel_after_expert_review.py` | Re-annotates seven labels with the definitions revised after expert review | Methods, Annotation Procedure; Appendix 1, Section A2 |
| 7 | `07_build_glossary_v2.py` | Builds the revised glossary from the original list | Methods, Measures; Appendix 1, Section A3 |
| 8 | `08_validation.py` | Agreement among the three human coders, and between the coders and the model | Methods, Validation of Annotations; Appendix 1, Section A4 |
| 9 | `09_main_models.py` | Descriptive statistics and the four nested models for each outcome, simple slopes, and diagnostics | Results, Tables 1 and 4 to 6 |
| 10 | `10_ztnb_robustness.py` | Zero-truncated negative binomial robustness check for comment count | Results, Robustness Check; Appendix 1, Section A5 |

`src/analysis_common.py` is imported by steps 9 and 10. It holds the definitions shared by both: the medical terminology density measure, the construction of the analytic sample, and the model specification.

### Steps 1 to 6: data preparation and annotation

These are the scripts as they were run, exported from the original Jupyter notebooks. Only the file paths, the handling of the API key, and the descriptive headers were changed. They are included for transparency and cannot be rerun exactly: they need the raw export, steps 2 and 6 call the Groq API, and LLaMA 3-70B-8192 has since been retired by Groq. Step 6 also produces the analytic file, `analytic_file.csv`, that steps 8 to 10 start from.

### Steps 7 to 10: final analysis

These scripts produce the numbers reported in the manuscript. Running them on the analytic file reproduces the published estimates exactly. The analysis decisions they implement are:

1. **Analytic sample.** Of the 5103 posts in the analytic file, 9 with a comment counter of zero are excluded, and 62 whose body was deleted or removed after posting are excluded because only their title could be annotated. Posts written as a title only are kept. Finally, the 4 posts from January 2023 are excluded, because the annotation run did not cover that month, so the analyzed period is February 2023 to November 2024. This leaves 5028 posts.
2. **Medical terminology density** is the share of a post's word tokens covered by terms from the revised glossary. Multiword terms are matched as token sequences, longest match first, without overlaps, and a final *s* is accepted as a plural.
3. **Centering.** Density and emotional intensity are mean-centered before the interaction is formed, so the main effects in Model 4 are effects at the mean of the other signal.
4. **Comment count** is modeled with a negative binomial regression (NB2) in which the dispersion parameter is estimated. **Mean comment length** and **comment richness** are modeled with OLS. Mean comment length is not transformed.

## How to run

Python 3.10 was used. From the repository root:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Place the data files in `data/` as described in `data/README.md`. Then:

```bash
python3 src/07_build_glossary_v2.py
python3 src/08_validation.py
python3 src/09_main_models.py
python3 src/10_ztnb_robustness.py
```

Steps 9 and 10 take a few minutes, most of it for the zero-truncated models. Results are written to `results/`.

Steps 2 and 6 need a Groq API key. Copy `.env.example` to `.env`, add your own key, and load it into the environment (for example `export $(cat .env)`) before running them. The key is read from the environment and is never stored in the code. `.env` is excluded from version control.

## Notes for readers of the code

- **Annotation settings.** All model calls use temperature 0.1. Step 2 sends the full post and allows 1000 output tokens for the labels and 500 for the affective scores. Step 6 sends the first 2000 characters of each post.
- **Labels in the step 2 prompt.** The prompt also asked for four motivation labels. They were not used in the analysis. In step 5, the kin-care motivation was merged into the intention *about family*, and the other three were dropped.
- **Failed API calls** return zeros in steps 2 and 6. Step 5 removes the 200 posts that received no label at all.
- **Medical terminology density in steps 2 to 6.** The column `health_competence_term_ratio` in the intermediate files was computed with the original glossary and a simpler matching rule. It is not used in the final analysis, which recomputes density in `analysis_common.py`.
- **Labels not modeled.** *Educational material* and *trigger or cause* are reported in the descriptive statistics but left out of the regression models, as explained in the manuscript.
- **Timestamps.** Time of day is derived from the post timestamps as stored in the export (UTC), without conversion to local time.
- **Validation sample.** Two of the 50 sampled posts are not in the analytic file. Model-to-coder agreement is therefore computed on 48 posts, and agreement among the coders on all 50.
- **Not included.** The data collection script (Reddit API, PRAW) is not part of this repository.

## Data

The data are not shared. They consist of public posts and comments from r/AFIB, and the coding workbooks of the three human coders. Redistributing Reddit content is restricted by the Reddit Data API Terms, and the study committed to not publishing content that could be traced to individual accounts (Ethics Committee of the Faculty of Economics and Business, University of Amsterdam, reference EB-18313). See `data/README.md` for the expected files and their columns.

## Citation

If you use this code, please cite the article:

> 

## License

Released under the MIT License, see `LICENSE`.
