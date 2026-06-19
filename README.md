# ResuMatch 

**MIT License**

ResuMatch computes semantic and structural similarity between resumes and job descriptions using NLP and ML. It implements a multi-component scoring pipeline — sentence embeddings, TF-IDF vectorization, regex-based skill extraction, and NER — to produce a weighted compatibility score with actionable recommendations.

## Architecture

Upload → TextExtractor → TextPreprocessor → SimilarityEngine → JSON Response

| Module             | File                   | Responsibility                                     |  
|--------            |------                  |----------------                                    |
| API Server         | `main.py`              | Routes, file validation, CORS, async processing    |  
| Text Extractor     | `text_extractor.py`    | Format detection, parser dispatch, OCR fallback    |
| Text Preprocessor  | `text_preprocessor.py` | Tokenization, lemmatization, NER, skill extraction |
| Similarity Engine  | `similarity_engine.py` | Embeddings, TF-IDF, scoring aggregation            |
| Configuration      | `config.py`            | Environment variables, model paths, weights        |


## Pipeline

### 1. Text Extraction

| Format        | Primary Parser | Fallback |
|--------       |--------------- |----------|
| PDF (text)    | pdfplumber     | PyPDF2   |
| PDF (scanned) | pytesseract    | —        |
| DOCX          | python-docx    | —        |
| TXT           | Built-in I/O   | —        |
| PNG/JPG/JPEG  | pytesseract    | —        |

Max file size: 50 MB (configurable).

### 2. Preprocessing

1. Sentence segmentation ([Spacy](https://spacy.io))
2. Tokenization (spaCy + NLTK fallback)
3. Stopword removal (custom list preserving domain terms)
4. Lemmatization (spaCy)
5. NER — extracts PERSON, ORG, GPE, DATE, EDUCATION
6. Skill extraction — regex matching against 200+ skills across 8 categories (languages, frameworks, databases, cloud, tools, soft skills, certifications, domains)

### 3. Similarity Scoring

Five components, weighted sum:

| Component           | Weight | Method                                             |
|---------------------|--------|----------------------------------------------------|
| Semantic Similarity | 0.35   | SentenceTransformer embeddings → cosine similarity |
| Skill Match         | 0.25   | Extracted skill sets → Jaccard similarity          |
| Experience Match    | 0.15   | Year ranges + role title overlap                   |
| Education Match     | 0.10   | Degree level mapping + field relevance             |
| Keyword Match       | 0.15   | TF-IDF vectorization → cosine similarity           |


final_score = Σ(weight_i × score_i) × 100

### Score Interpretation

| Range  | Classification |
|------- |--------------- |
| 75–100 | Strong match   |
| 60–74  | Moderate match |
| 45–59  | Weak match     |
| 0–44   | Poor match     |

## API Reference

### `POST /analyze`

**Request:** `multipart/form-data` with `resume` (file) and `job_description` (string).

**Response (200):**
```json
{
 "analysis_id": "uuid",
 "overall_score": 78.3,
 "component_scores": {
 "semantic_similarity": 0.82,
 "skill_match": 0.71,
 "experience_match": 0.76,
 "education_match": 0.88,
 "keyword_match": 0.74
 },
 "matched_skills": "python", "fastapi", "postgresql",
 "missing_skills": "kubernetes", "aws",
 "recommendations": "Add cloud deployment experience",
 "extracted_entities": {
 "companies": "Tech Corp",
 "education": "B.Tech Computer Science"
 }
}

**Errors:** `400` (invalid format), `413` (file too large), `422` (corrupted file), `500` (server error).

### `POST /batch-analyze`

Accepts array of resume files + single job description. Max batch size: 10 (configurable).

### `GET /analysis/{analysis_id}`

Retrieve stored analysis results.

### `GET /health`

Returns server status, model load state, uptime.

## Configuration

| Variable         | Default           | Description         |
|----------        |---------          |-------------        |
| HOST             | `0.0.0.0`         | Bind address        |
| PORT             | `8000`            | Server port         |
| DEBUG            | `False`           | Debug mode          |
| SENTENCE_MODEL   | `all-MiniLM-L6-v2`| Embedding model     |
| USE_GPU          | `False`           | CUDA inference      |
| MAX_FILE_SIZE    | `52428800`        | Max upload (50 MB)  |
| MAX_BATCH_SIZE   | `10`              | Max files per batch |
| WEIGHT_SEMANTIC  | `0.35`            | Semantic weight     |
| WEIGHT_SKILL     | `0.25`            | Skill weight        |
| WEIGHT_EXPERIENCE| `0.15`            | Experience weight   |
| WEIGHT_EDUCATION | `0.10`            | Education weight    |
| WEIGHT_KEYWORD   | `0.15`            | Keyword weight      |


## Installation

### Prerequisites

- Python 3.8–3.12
- Node.js 16+ (frontend)
- Tesseract OCR 5.x (for image processing)

### Backend

```bash
cd backend
python -m venv.venv
source.venv/bin/activate
pip install -r requirements.txt
python -m spacy download en_core_web_sm
```

### Frontend

```bash
cd frontend
npm install
```

### Run

```bash
# Terminal 1
cd backend && python main.py

# Terminal 2
cd frontend && npm run dev
```

Backend: `[http://localhost:8000\`](http://localhost:8000`) | Frontend: `[http://localhost:3000\`](http://localhost:3000`)

---

## Project Structure

```
ResuMatch/
├── backend/
│ ├── main.py # FastAPI entry point
│ ├── config.py # Configuration
│ ├── similarity_engine.py # Scoring logic
│ ├── text_extractor.py # File parsing + OCR
│ ├── text_preprocessor.py # NLP pipeline
│ ├── models.py # SQLAlchemy models
│ ├── schemas.py # Pydantic schemas
│ ├── database.py # DB connection
│ ├── requirements.txt
│ ├── uploads/
│ ├── results/
│ └── tests/
├── frontend/
│ ├── pages/
│ ├── components/
│ ├── services/
│ ├── hooks/
│ └── styles/
├── dev.sh
└── README.md
```

---

## Performance

| Operation | Avg Time (CPU) |
|-----------|---------------|
| Text extraction (PDF) | 200–500 ms |
| Preprocessing | 100–300 ms |
| Embedding computation | 500–800 ms |
| Full pipeline | 1.5–3.0 s |

Memory: ~150 MB idle, ~400–600 MB during analysis.

---

## Testing

```bash
cd backend
pip install pytest pytest-asyncio httpx
pytest tests/ -v
pytest --cov=. tests/ --cov-report=term-missing
```

---

## Roadmap

- PostgreSQL integration
- JWT authentication
- Analytics dashboard
- Multi-language support
- Fine-tuned embedding model
- Parallel batch processing
- Docker containerization
- CI/CD pipeline

---

## License

MIT. See `LICENSE`.

---

## References

- Reimers & Gurevych (2019). *Sentence-BERT*. EMNLP.
- spaCy: [Spacy](https://spacy.io)
- FastAPI: [Tiangolo](https://fastapi.tiangolo.com)
- scikit-learn: Pedregosa et al. (2011). *JMLR*.
