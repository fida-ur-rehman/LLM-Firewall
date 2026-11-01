# LLM Firewall: Detecting Prompt-Injection Attacks of Unseen Types

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/fida-ur-rehman/LLM-Firewall/blob/main/LLM_Firewall.ipynb)

A filter that scores every prompt before it reaches a large language model and returns **ALLOW**, **REVIEW** or **BLOCK**. The project compares three NLP classifiers, each with and without a decoding step, on prompt-injection attack types that never appear in the training data.

## Research question

Which NLP approach best detects prompt-injection attacks of a type it never saw during training, and can a decoding pre-processing step close the gap?

## How it works

```
prompt -> decoder -> classifier -> risk score -> ALLOW (< 0.35) / REVIEW (0.35 to 0.5) / BLOCK (>= 0.5)
```

1. **Decoder.** Undoes base64, hex, rot13, reversed text and leetspeak. It keeps a decoding only if the result looks more like English than the original, so it does not depend on the prompt naming the encoding.
2. **Classifier.** Gives a risk score between 0 and 1.
3. **Policy.** Maps the score to a decision and reports the tricks found and the decoded text the model saw.

## Models compared

| Model | Text representation |
|---|---|
| TF-IDF + Logistic Regression | Word unigrams and bigrams |
| Sentence embeddings + Logistic Regression | `paraphrase-multilingual-MiniLM-L12-v2` (384-d) |
| Fine-tuned DistilBERT | `distilbert-base-uncased`, 2 epochs |
| ProtectAI DeBERTa-v3 (reference only) | `protectai/deberta-v3-base-prompt-injection-v2` |

## Data

- **Main:** [Prompt Injection Detection Dataset](https://github.com/mirzaakhi/prompt-injection-detection-dataset) (Mirza Akhi, CC BY 4.0). 5,441 synthetic prompts. Obfuscation, payload splitting and multilingual attacks (488 test prompts) appear only in the test split.
- **External test:** [deepset/prompt-injections](https://huggingface.co/datasets/deepset/prompt-injections), 116 prompts, partly German. Never used for training.

Both are downloaded by the notebook. No data is stored in this repository.

## Results

Main test split (1,229 prompts), threshold 0.5.

| Model | Input | Recall | F1 | False-positive rate | Unseen-type recall | External F1 |
|---|---|---|---|---|---|---|
| TF-IDF + LR | raw | 0.687 | 0.798 | 0.067 | 0.553 | 0.456 |
| TF-IDF + LR | decoded | 0.930 | 0.947 | 0.067 | 0.957 | 0.456 |
| Embeddings + LR | raw | 0.776 | 0.844 | 0.122 | 0.668 | 0.351 |
| Embeddings + LR | decoded | 0.954 | 0.946 | 0.122 | 0.965 | 0.351 |
| DistilBERT | raw | 0.806 | 0.852 | 0.170 | 0.678 | 0.356 |
| DistilBERT | decoded | 0.990 | 0.953 | 0.170 | 0.984 | 0.356 |
| ProtectAI (reference) | decoded | 0.916 | 0.950 | 0.026 | 0.996 | 0.537 |

Main findings:

- Decoding raised recall on unseen attack types from 55% to 96% for TF-IDF and from 68% to 98% for DistilBERT, with no new false alarms.
- DistilBERT with decoding had the best F1 but flagged 17% of benign prompts (41% of benign "summarise this document" requests) and missed all 8 multilingual attacks.
- TF-IDF with decoding came within 0.006 F1 of DistilBERT, with 6.7% false alarms and a 0.5 second training time.
- Every model dropped to 0.35 to 0.54 F1 on the external dataset, so results on the synthetic data do not transfer well.

The charts are in [`results/`](results/) and the full write-up is in [`report/`](report/).

## Run it

1. Click **Open in Colab** above.
2. Choose *Runtime > Change runtime type > T4 GPU*.
3. Choose *Runtime > Run all*. It takes about 10 to 15 minutes.
4. The last cells start a Gradio demo and download the charts and results table as a zip file.

To run locally instead:

```bash
pip install -r requirements.txt
jupyter notebook LLM_Firewall.ipynb
```

A CUDA GPU is recommended for DistilBERT fine-tuning.

## Live demo

The notebook launches a Gradio interface with `share=True`. That link only works while the Colab session is running and expires after at most one week, so it is not linked here. Run the notebook to get a fresh link.

## Repository structure

```
LLM-Firewall/
├── LLM_Firewall.ipynb      # full pipeline: data, decoder, models, evaluation, demo
├── requirements.txt
├── results/
│   ├── results.csv
│   ├── data_overview.png
│   ├── chart_raw_vs_decoded.png
│   ├── chart_recall_by_category.png
│   ├── chart_generalisation_gap.png
│   └── confusion_best.png
├── report/
│   ├── LLM_Firewall_Report.pdf
└── README.md
```

## Limitations

- The main dataset is synthetic and mostly English. The multilingual (8 attacks) and external (116 prompts) tests are small.
- The decoder was designed after inspecting missed test attacks, so its 300/300 decoding rate is optimistic.
- Validation uses only seen attack types and scored about 1.00 for every model, so the final model was chosen on the test set.
- One fixed threshold (0.5) and one random seed were used.
- A detector is one layer of defence. Deployments still need least-privilege tool access, output checks and human review.

## References

- Akhi, M. (2025). *Prompt injection detection dataset* [Data set]. GitHub.
- deepset. (2023). *prompt-injections* [Data set]. Hugging Face.
- ProtectAI. (2024). *deberta-v3-base-prompt-injection-v2* [Model]. Hugging Face.
- Reimers, N., & Gurevych, I. (2020). Making monolingual sentence embeddings multilingual using knowledge distillation. *Proceedings of EMNLP 2020*, 4512-4525.
- Sanh, V., Debut, L., Chaumond, J., & Wolf, T. (2019). DistilBERT, a distilled version of BERT: Smaller, faster, cheaper and lighter. *arXiv:1910.01108*.
