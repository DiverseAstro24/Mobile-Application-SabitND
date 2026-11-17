# Model Selection Narrative: Multilingual DistilBERT for Filing Approved Documents

## Concept definition
Every document the College President approves has to be filed so it can be found later: by year, by month, and by kind of document. Today that means someone reads each signed document, decides whether it is **procurement**, **financial**, or **general** (and for general ones, whether it came from outside or was issued inside the college), and puts it in the right folder. Finding it again depends on someone remembering when it was approved and who was involved.

This app does the reading and filing. The user scans the signed pages or picks the approved PDF. The system reads the text, suggests the document type, approval date, offices and people involved, and a staff member confirms. The file is then stored under `year / month / type /` and indexed so it can be searched by date, office, name or any word in the document.

The machine learning model has one job: **decide the document type** (procurement, financial, general internal, general external). Everything else is deliberately not machine learning: folder placement follows from the confirmed date and type, search uses a full-text index, and offices are matched against the college's own office list.

## Model choice
`distilbert/distilbert-base-multilingual-cased` from Hugging Face (Apache-2.0), fine-tuned as a four-class text classifier. According to its model card it has 6 layers and 134M parameters, was trained on Wikipedia in 104 languages, and runs about twice as fast as multilingual BERT-base.

Until a fine-tuned model exists, the server uses a keyword baseline so the whole system works from day one. The app shows which one made each suggestion. Plugging the model in is one setting: `MODEL_ID=./models/doc-classifier`.

## Why this model fits better than the alternatives
- **Generative LLMs (prompting).** They can classify, but they need GPUs or a paid API, are slower, and may answer with free text when the app needs exactly one of four labels. More importantly, these are confidential approved documents: a small model running on the college's own server keeps them on campus, while an outside API would not.
- **Larger multilingual encoders (for example XLM-RoBERTa base).** Likely a little more accurate, but roughly twice the parameters and slower, which matters on a modest office server. Document type is usually obvious from the title and opening lines, so the extra capacity buys little.
- **TF-IDF with logistic regression.** This is a serious competitor, not a straw man. Formal documents have very distinctive vocabulary ("purchase request", "disbursement voucher"), and this method needs little data and almost no compute. I would train it first as the benchmark and keep DistilBERT only if it wins on macro-F1. Where I expect DistilBERT to help is mixed English, Filipino and Bisaya text and OCR errors, since subword tokens degrade more gracefully than whole-word features. The multilingual model's language list includes Cebuano, which is relevant locally, but that must be tested on real documents rather than assumed.
- **Keyword rules.** Transparent and instant, but brittle. They stay as the fallback and as a sanity check.

## Dataset and why it is relevant
No public dataset matches this task: the labels are specific to a college office, and the text includes local letterheads, office names and language mix. The right dataset is the office's own approved documents, which is exactly the distribution the model will face once deployed.

- **How it is built.** The app creates the dataset as it is used. The text read from each document is stored with the type a staff member confirmed. `GET /export/training.csv` exports these as `text,label` rows, and `train_classifier.py` fine-tunes from that file. The app also records the model's original suggestion next to the confirmed type, so real-world accuracy can be measured.
- **Size.** Aim for at least 100 documents per class. Around 30 per class is enough only to check that the pipeline works. The `sample_training.csv` included in the project has 16 invented lines that show the format. They are far too few to train anything useful.
- **Evaluation.** Stratified 80/20 split, reporting macro-F1, a per-class report and a confusion matrix. The confusing pair will most likely be general internal versus general external, so that deserves separate attention.
- **Privacy.** Documents stay on the college server. Get the office's approval before collecting, and remove names before sharing any examples outside the office (Data Privacy Act of 2012, RA 10173).

I have not trained on real documents. The accuracy claims above are things to measure, not results.

## Key trade-offs and technical advantages
- **Small and local over largest and best.** Accuracy might be a point or two lower than a bigger model, in exchange for cheap CPU inference and documents that never leave the college.
- **First 512 tokens only.** The model reads the beginning of the document, where the title and subject are. Long attachments such as full contracts are truncated, which is acceptable for deciding type but would not suit tasks that need the whole text.
- **Coarse labels.** Four types are easy to act on but cannot express documents that are both procurement and financial. Suggestions below 60% confidence are marked "please check the type".
- **OCR noise.** Stamps, signatures and skewed photos damage the text. The model tolerates some noise, but a staff confirmation step is required before anything is filed.
- **The model does not verify approval.** It cannot tell a real signature from a forged one. A staff member confirms the document is approved and signed, and the app will not save without that confirmation.
- **A feedback loop.** Each confirmed filing improves the next training run, so the model can improve over time without extra labeling work.

## Conclusion
The best model here is not the biggest one but the one that suits a confidential, multilingual, low-resource office setting: a fine-tuned multilingual DistilBERT, backed by a rule-based fallback and checked by a person before filing. It does one narrow job, and everything around it is plain, auditable software.
