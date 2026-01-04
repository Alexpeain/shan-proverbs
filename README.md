# shan-proverbs

**A TikTok-sourced Shan/Tai proverbs corpus with NLP processing via local n8n Docker workflow.**

This project automates the collection, normalization, and storage of Shan/Tai proverbs shared by TikTok creators, enabling downstream NLP analysis, language preservation, and corpus building for low-resource Tai languages.

## 🎯 Project Overview

- **Source**: TikTok videos and captions containing Shan/Tai proverbs
- **Collection**: Manual curation + automated integration via n8n
- **Processing**: Text normalization, Unicode handling, and optional LLM-based annotation
- **Storage**: JSON/CSV formats for easy integration with NLP pipelines
- **Orchestration**: n8n running in Docker for reproducible, local-first workflows

## 📁 Repository Structure

```
.
├── README.md                      # This file
├── proverbs.json                  # Main proverbs corpus (Shan text + definitions)
├── shan_proverbs_extra.json       # Extended corpus with additional metadata
├── docker-compose.yml             # Docker setup for n8n (optional)
├── n8n-workflows/                 # n8n workflow JSON exports
│   └── shan-proverbs-processor.json
└── data/                          # Local storage volume mount
    └── exports/
        ├── shan_proverbs.csv
        └── shan_proverbs_clean.json
```

## 📦 Data Format

### JSON Structure

Each proverb entry follows this structure:

```json
{
  "id": "unique-id-001",
  "proverb_shan": "ပိုင်ဆိုင်သည့် စကား",
  "proverb_thai": "สุภาษิต ในภาษาไทย",
  "english_translation": "Meaning in English",
  "explanation": "Context and cultural significance",
  "source_url": "https://www.tiktok.com/@username/video/...",
  "source_creator": "TikTok creator username",
  "created_at": "2025-01-04T15:30:00Z",
  "tags": ["wisdom", "culture", "tradition"]
}
```

### CSV Format

For analysis and integration with data tools:

| id | proverb_shan | english_translation | source_url | created_at |
|----|--------------|---------------------|------------|------------|
| unique-id-001 | ပိုင်ဆိုင်သည့် စကား | Meaning in English | https://... | 2025-01-04 |

## 🐳 Local Setup with Docker & n8n

### Prerequisites

- Docker and Docker Compose installed
- n8n image (auto-pulled on first run)
- Optional: Python 3.8+ for additional NLP analysis

### Quick Start

1. **Create n8n persistent volume:**

```bash
docker volume create n8n_data
```

2. **Start n8n container:**

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -v $(pwd)/data:/home/node/data \
  docker.n8n.io/n8nio/n8n
```

3. **Access n8n UI:**

Open `http://localhost:5678` in your browser

4. **Import workflow:**

- Use the web UI to import `n8n-workflows/shan-proverbs-processor.json`
- Or copy the JSON directly into n8n's workflow editor

5. **Configure data paths:**

- Update File Read node to point to your input data location
- Set File Write node output to `/home/node/data/exports/`

6. **Run the workflow:**

- Click "Execute Workflow" to process your proverbs
- Check the `data/exports/` folder for cleaned output

### Using docker-compose.yml

Alternatively, create a `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: shan-proverbs-n8n
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
      - ./data:/home/node/data
    environment:
      - N8N_EDITOR_BASE_URL=http://localhost:5678
    networks:
      - n8n-network

volumes:
  n8n_data:

networks:
  n8n-network:
    driver: bridge
```

Then run:

```bash
docker-compose up
```

## 🔄 Workflow Overview

The n8n workflow processes proverbs through these stages:

1. **Input** → Read from JSON/CSV file or HTTP API
2. **Normalization** → Unicode normalization, deduplication, trimming
3. **Cleaning** → Remove non-Shan characters, standardize text
4. **Enrichment** (optional) → Add translations, explanations via LLM
5. **Output** → Write to CSV/JSON files in `/home/node/data/exports/`

## 📊 NLP Integration

Once exported, use the corpus with popular NLP libraries:

### Python Example

```python
import json
import pandas as pd
from pathlib import Path

# Load proverbs
with open('data/exports/shan_proverbs_clean.json') as f:
    proverbs = json.load(f)

# Convert to DataFrame
df = pd.DataFrame(proverbs)

# Basic statistics
print(f"Total proverbs: {len(df)}")
print(f"Unique creators: {df['source_creator'].nunique()}")

# Example: Tokenization for Shan text
from pythainlp.tokenize import word_tokenize

for idx, row in df.iterrows():
    tokens = word_tokenize(row['proverb_shan'], engine='newmm')
    print(f"{row['id']}: {tokens}")
```

## 🛠️ Development

### Adding New Proverbs

1. Edit `proverbs.json` or `shan_proverbs_extra.json` directly
2. Follow the JSON structure above
3. Commit and push to trigger n8n workflow (if scheduled)

### Extending the Workflow

- Add sentiment analysis nodes in n8n
- Integrate database output (PostgreSQL, SQLite)
- Add scheduled scraping from TikTok (with API credentials)
- Export to different formats (Parquet, Excel)

## 📝 License

This project is for language preservation and educational purposes. Please respect intellectual property rights when collecting TikTok content.

- **Code**: MIT License
- **Data**: See individual video creators' terms and conditions

## 🤝 Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/add-proverbs`
3. Commit changes: `git commit -m 'Add new Shan proverbs'`
4. Push to branch: `git push origin feature/add-proverbs`
5. Open a Pull Request

## 📚 Resources

- [n8n Documentation](https://docs.n8n.io/)
- [Docker Documentation](https://docs.docker.com/)
- [PyThaiNLP](https://github.com/PyThaiNLP/pythainlp) - Tai language tokenization
- [Shan Language Resources](https://en.wikipedia.org/wiki/Shan_language)

## 👤 Author

**Alexpeain** - Self-taught developer focused on NLP for Shan/Tai languages and language preservation.

---

**Last updated**: January 2025
