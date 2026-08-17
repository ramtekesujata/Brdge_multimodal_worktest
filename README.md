# Brdge_multimodal_worktest

# Multimodal RAG — Text + Image Retrieval over a Document Corpus

A multimodal Retrieval-Augmented Generation (RAG) pipeline built to run on Kaggle, combining **sentence-transformers** for text embeddings and **CLIP** for image embeddings, enabling both text-to-text and text-to-image (cross-modal) search over a mixed document corpus (PDF, PPTX, DOCX).

## How it works

1. **Convert to PDF** — `.pptx` and `.docx` files are converted to PDF via LibreOffice so everything downstream is handled uniformly by PyMuPDF.
2. **Extract content** — Text is chunked from each PDF, and each page/slide is also rendered as a full-page image (one render per page, not per embedded object — cheaper than extracting individual shapes and a better retrieval unit since it captures any diagram or chart on that page).
3. **Embed**
   - **Text chunks** → `sentence-transformers` (`all-MiniLM-L6-v2`) — better text-to-text semantic matching than CLIP's text tower, with no 77-token limit.
   - **Page images** → CLIP (`open_clip`, ViT-B-32, LAION-2B weights) — CLIP's shared text/image embedding space also lets a plain text query retrieve a relevant image directly.
4. **Index** — Both embedding sets are stored in separate FAISS indices (`IndexFlatIP`, exact cosine similarity via normalized inner product).
5. **Query** — A query is embedded with both encoders and searched against both indices, returning ranked text passages and matching page images.

## Setup (Kaggle)

1. Attach your document corpus as a Kaggle **Dataset** input (Add Data, right panel).
2. Turn on a **GPU** accelerator (Settings).
3. Run cells top to bottom. Internet access is required (models auto-download from Hugging Face).

## Configuration

Point `DATA_DIR` at your dataset path. Kaggle mounts datasets under `/kaggle/input/<dataset-slug>/...`; the notebook auto-detects the first folder under `/kaggle/input/` if `DATA_DIR` isn't set explicitly — double-check the printed path before continuing.

## Usage

Edit the query string in the search cell and re-run:

```python
text_results, image_results = show_results("how do I use the eisenhower matrix", top_k=5)
```

This returns the top-k matching text chunks and the top-k matching page images, with source file, page number, and similarity score for each.

## Dependencies

- `pymupdf` (PDF parsing/rendering)
- `python-pptx`, `python-docx` (source format handling)
- `pillow`
- `sentence-transformers`
- `open_clip_torch`
- `faiss-cpu`
- LibreOffice (system package, for `.pptx`/`.docx` → PDF conversion)

All Python dependencies are installed in the notebook's first cell.

## Next steps

- **Add a generation step** — feed the top text chunks (and, for a vision-capable model, the top image files directly) as context to an LLM call to answer questions instead of just returning ranked passages.
- **OCR pass** — for graphic-heavy pages, OCR over the page renders would surface literal text-in-image content (axis labels, etc.) into the text index as well.
- **Approximate index** — `IndexFlatIP` is exact and fine at a few thousand vectors; switch to `IndexIVFFlat` or similar for larger corpora.
