# shan-proverbs

**A TikTok-sourced Shan/Tai proverbs corpus with automated NLP processing via Telegram bot, Google Sheets, n8n Docker, and ShanNLP.**

This project automates the entire workflow of collecting Shan/Tai proverbs from TikTok creators through a Telegram bot, storing them in Google Sheets, processing with NLP tools, and pushing cleaned data back to the repository. It enables corpus building, language preservation, and downstream NLP analysis for low-resource Tai languages.

## 🚀 Project Workflow

```
TikTok Creators
     ↓
Telegram Bot (Collection)
     ↓
Google Sheets (Raw Data Storage)
     ↓
n8n Docker Workflow
     ├─→ Download from Google Sheets
     ├─→ Process with ShanNLP Library
     │  ├─ Tokenization (word_tokenize)
     │  ├─ Text normalization
     │  ├─ Digit/date conversion
     │  └─ Unicode standardization
     ├─→ Enrich & Clean Proverbs
     └─→ Git Push to Repository
     ↓
shan-proverbs Repository (Final Corpus)
     ↓
NLP Analysis & Research
```

## 📁 Repository Structure

```
.
├── README.md                      # This file
├── proverbs.json                  # Main proverbs corpus (processed by n8n)
├── shan_proverbs_extra.json       # Extended corpus with metadata
├── docker-compose.yml             # Docker setup for n8n
├── n8n-workflows/                 # n8n workflow JSON files
│   └── telegram-to-repo.json      # Main workflow: Google Sheets → ShanNLP → Git Push
└── data/
    └── exports/
        ├── shan_proverbs.csv
        └── shan_proverbs_clean.json
```

## 🔄 Data Collection Pipeline

### Stage 1: Telegram Bot Collection
Your Telegram bot collects Shan proverbs shared by TikTok creators and stores raw data in Google Sheets with these fields:
- Proverb text (original Shan)
- Creator name/ID
- TikTok link
- Timestamp
- Optional notes

### Stage 2: Google Sheets Storage
Data is stored in a Google Sheet with structured columns for easy import into n8n.

### Data Validation & Cleaning in Google Sheets

Before n8n processes your data, ensure quality with these best practices in your Google Sheet:

#### 1. **Shan Text Validation**
- Use data validation to accept only Shan Unicode characters
  - Go to `Data` → `Data validation`
  - Create custom formula: `=AND(REGEX(A2, "[\u1000-\u109F]+"))` to check for Shan characters
  - Set error alert: "Please enter valid Shan text"
- Check for mixed scripts (avoid English/Burmese in Shan proverb column)

#### 2. **Required Fields Check**
- Mark these columns as mandatory (no empty cells):
  - Proverb text (original Shan)
  - Creator name/TikTok username
  - TikTok link (must start with `https://www.tiktok.com/`)
  - Timestamp
- Use conditional formatting to highlight empty required cells in red

#### 3. **URL Validation**
- Validate TikTok links with data validation formula:
  ```
  =AND(NOT(ISBLANK(C2)), ISNUMBER(SEARCH("tiktok.com", C2)))
  ```
- Ensure URLs are clickable and formatted consistently
- Remove any shortened URLs; use full TikTok video links

#### 4. **Remove Duplicates**
- Regularly run `Data` → `Data cleanup` → `Remove duplicates`
- Check for:
  - Identical proverb text from same creator
  - Duplicate rows with minor spacing differences
  - Same content with different timestamps

#### 5. **Clean Whitespace**
- Use a helper column with formula: `=TRIM(CLEAN(A2))`
- This removes:
  - Leading/trailing spaces
  - Extra spaces between words
  - Non-printing characters
- Copy cleaned values back to original column using paste special (values only)

#### 6. **Consistent Formatting**
- **Timestamps**: Use standard ISO 8601 format `YYYY-MM-DDTHH:MM:SSZ`
  - Google Sheets formula: `=TEXT(NOW(), "YYYY-MM-DDTHH:MM:SSZ")`
- **Creator names**: Use Title Case (e.g., "Creator Name" not "CREATOR NAME")
- **TikTok links**: Remove `?is_copy_url=1&is_from_webapp=v1` query parameters

#### 7. **Add Quality Score Column**
Create a helper column to rate data quality (1-5 scale):
```
=COUNTIF(B2:F2,"<>")/COUNTA(B2:F2)*5
```
Filter to show only scores ≥ 4 before n8n processing

#### 8. **Text Length Checks**
- Proverb should be 10-500 characters (avoid single word entries)
  - Use formula: `=AND(LEN(A2)>=10, LEN(A2)<=500)`
- Creator name: 2-50 characters
- Use conditional formatting to highlight outliers

#### 9. **Manual Review Process**
Before n8n syncs:
1. Sort by `created_at` to identify recent entries
2. Read through for context and accuracy
3. Mark suspicious entries with a "Review" flag column
4. Fix or delete flagged entries before processing
5. Keep a backup sheet of "raw unreviewed data"

#### 10. **Google Sheets Best Practices**
- **Freeze header row**: `View` → `Freeze` → `1 row`
- **Alternate row colors**: Select data → `Format` → `Alternating colors` for readability
- **Data validation dropdown**: For creator names, use a list of approved creators
- **Comments**: Add notes next to questionable entries for context
- **Version history**: Enable to track changes over time

#### 11. **Automated Cleaning with Formulas**

Create a "Cleaned Data" sheet with these formulas:

```excel
# Column A - Cleaned Proverb Text
=IF(AND(LEN(TRIM(A_raw))>10, REGEX(TRIM(A_raw), "[\u1000-\u109F]+")), TRIM(A_raw), "")

# Column B - Creator (Titlecase)
=IF(LEN(B_raw)>0, PROPER(TRIM(B_raw)), "")

# Column C - TikTok URL (clean parameters)
=IF(AND(NOT(ISBLANK(C_raw)), ISNUMBER(SEARCH("tiktok.com", C_raw))), 
   REGEXREPLACE(C_raw, "\\\\?.*", ""), "")

# Column D - Timestamp (ISO format)
=IF(ISDATE(D_raw), TEXT(D_raw, "YYYY-MM-DDTHH:MM:SSZ"), "")
```

Then copy cleaned data back as values before n8n imports.

#### 12. **n8n Pre-flight Checks**

Before importing to n8n, add this validation in a "Status" column:

```excel
=IF(COUNTIF(A2:D2,"<>")<>4, "MISSING_FIELDS",
  IF(LEN(A2)<10, "PROVERB_TOO_SHORT",
  IF(ISERROR(SEARCH("tiktok.com", C2)), "INVALID_URL",
  "READY_TO_PROCESS")))
```

Filter to show only "READY_TO_PROCESS" rows before n8n export.

### Recommended Workflow

1. **Daily**: Review newly added entries in Google Sheets
2. **Weekly**: Run duplicate check and whitespace cleanup
3. **Before n8n sync**: Apply all validation formulas and filter for "READY_TO_PROCESS"
4. **After n8n processing**: Compare results with original to catch processing errors
5. **Monthly**: Audit data quality and update validation rules as needed



### Stage 3: n8n Automated Processing
The n8n Docker container runs a scheduled workflow that:

1. **Authenticates** with Google Sheets API
2. **Downloads** all new proverbs from the spreadsheet
3. **Processes** using ShanNLP library:
   - Text normalization and Unicode standardization
   - Word tokenization using maximal matching or newmm algorithm
   - Digit/date/keyboard conversion utilities
4. **Cleans** and deduplicates data
5. **Enriches** with metadata
6. **Exports** to JSON/CSV formats
7. **Commits & Pushes** to GitHub repository

### Stage 4: Repository Storage
Processed proverbs are stored in this repository for version control and collaboration.

## 📦 Data Format

### JSON Structure
```json
{
  "id": "unique-id-001",
  "proverb_shan": "",
  "definition": "",
  "english_translation": "Meaning in English",
  "tokens": ["", "", ""],
  "source_creator": "TikTok username",
  "source_url": "https://www.tiktok.com/@creator/video/...",
  "collected_at": "2025-01-04T15:30:00Z",
  "processed_at": "2025-01-04T16:45:00Z",
  "tags": ["wisdom", "culture", "tradition"]
}
```

## 🐳 n8n Docker Setup

### Prerequisites
- Docker and Docker Compose
- n8n image
- Google Sheets API credentials
- GitHub personal access token (for git push)
- ShanNLP library installed in n8n environment

### Quick Start

#### 1. Create persistent volumes
```bash
docker volume create n8n_data
```

#### 2. Start n8n with Docker Compose

Create a `docker-compose.yml`:

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
      - WEBHOOK_TUNNEL_URL=http://localhost:5678/
    networks:
      - n8n-network
    restart: unless-stopped

volumes:
  n8n_data:

networks:
  n8n-network:
    driver: bridge
```

Start the service:
```bash
docker-compose up -d
```

#### 3. Access n8n UI
Open `http://localhost:5678` in your browser

#### 4. Configure n8n Credentials

**Google Sheets API**
- In n8n, create a new credential: Google Sheets OAuth2
- Authenticate with your Google account
- Grant permission to access Google Sheets

**GitHub API**
- Create GitHub Personal Access Token with `repo` and `workflow` scopes
- Add token to n8n GitHub credentials

#### 5. Import the Workflow
- Import `n8n-workflows/telegram-to-repo.json`
- Configure node parameters:
  - Google Sheet ID
  - Column mappings
  - Repository URL and branch
  - ShanNLP processing options

#### 6. Schedule the Workflow
- Set trigger to run on schedule (e.g., daily at 2 AM)
- Or trigger manually from n8n UI

## 📊 ShanNLP Integration

The n8n workflow uses the ShanNLP library for text processing:

### Word Tokenization
```python
from shannlp import word_tokenize

text = ""

# Method 1: Maximal Matching (fast)
tokens = word_tokenize(text, engine='mm')
# Output: ['', '', '']

# Method 2: newmm (PyThaiNLP-based)
tokens = word_tokenize(text, engine='newmm')
```

### Utility Functions in n8n
- `digit_to_text()` - Convert digits to Shan words
- `num_to_shanword()` - Convert numbers to Shan text
- `shanword_to_date()` - Parse Shan date formats
- `convert_years()` - Convert between calendar systems (AD, BE, GA, MO)
- `eng_to_shn()` - Keyboard conversion (English to Shan)

## 🛠️ n8n Workflow Nodes

The main workflow includes these node types:

1. **Trigger Node**: Schedule or manual webhook
2. **Google Sheets Node**: Read data from spreadsheet
3. **Function Node**: JavaScript for data transformation
4. **HTTP Request Node**: Call ShanNLP API (if exposed)
5. **Code Node**: Python execution for ShanNLP processing
6. **File Write Node**: Save processed data
7. **Git Node**: Clone, commit, and push to repository
8. **Webhook Node**: Optional notifications

## 📝 Adding ShanNLP to n8n Docker

To use ShanNLP in n8n, you have two options:

### Option 1: Python Function in Code Node (Recommended)
Add ShanNLP to the n8n environment by extending the Docker image:

```dockerfile
FROM docker.n8n.io/n8nio/n8n:latest

USER root
RUN apt-get update && apt-get install -y python3-pip
RUN pip install shannlp
USER node
```

Build and run:
```bash
docker build -t n8n-shannlp .
docker run -it --rm -p 5678:5678 n8n-shannlp
```

### Option 2: Separate Python Service
Run ShanNLP as a microservice in another container and call it via HTTP from n8n.

## 🔐 Environment Variables

Create a `.env` file for sensitive data:

```bash
# Google Sheets
GOOGLE_SHEET_ID=your-sheet-id-here
GOOGLE_API_KEY=your-api-key

# GitHub
GITHUB_TOKEN=your-personal-access-token
GITHUB_REPO_URL=https://github.com/Alexpeain/shan-proverbs.git
GIT_USER_NAME=Alexpeain
GIT_USER_EMAIL=your-email@example.com

# n8n
N8N_EDITOR_BASE_URL=http://localhost:5678
```

## 🔗 Related Projects

- **ShanNLP** - NLP tools for Shan language processing
  - Repository: https://github.com/Alexpeain/ShanNLP
  - Features: Tokenization, digit conversion, date conversion, keyboard conversion
  - Inspired by PyThaiNLP

- **Telegram Bot** - Collects proverbs from TikTok creators
  - Stores data in Google Sheets
  - Validates Shan text input

## 💡 Development & Customization

### Modifying the Workflow
1. Open n8n UI at `http://localhost:5678`
2. Edit the imported workflow
3. Test individual nodes
4. Export updated workflow as JSON
5. Save to `n8n-workflows/` directory

### Adding New Processing Steps
- Use ShanNLP utilities for additional text processing
- Add filtering/validation nodes
- Integrate with other APIs or services
- Export to different formats (Parquet, SQLite, etc.)

### Testing
```python
import json
from shannlp import word_tokenize

# Load processed data
with open('data/exports/shan_proverbs_clean.json') as f:
    proverbs = json.load(f)

# Test tokenization
for proverb in proverbs[:5]:
    tokens = word_tokenize(proverb['proverb_shan'])
    print(f"{proverb['id']}: {tokens}")
```

## 📚 NLP Analysis Examples

### Frequency Analysis
```python
import json
from collections import Counter
from shannlp import word_tokenize

with open('data/exports/shan_proverbs_clean.json') as f:
    proverbs = json.load(f)

# Collect all tokens
all_tokens = []
for proverb in proverbs:
    tokens = word_tokenize(proverb['proverb_shan'])
    all_tokens.extend(tokens)

# Get frequency
freq = Counter(all_tokens)
print("Top 20 most common words:")
for word, count in freq.most_common(20):
    print(f"{word}: {count}")
```

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/improve-workflow`
3. Make changes (to JSON files, workflows, or documentation)
4. Commit: `git commit -m 'feat: improve n8n workflow for ShanNLP integration'`
5. Push: `git push origin feature/improve-workflow`
6. Open a Pull Request

### Areas for Contribution
- Improving tokenization accuracy
- Adding more ShanNLP utilities to the workflow
- Expanding the proverbs corpus
- Creating analysis notebooks
- Improving documentation

## 📄 License

This project is for language preservation and educational purposes.

- **Code**: MIT License
- **Data**: Respect TikTok creator rights and terms of service
- **Processed Data**: Available for research and non-commercial use

## 📚 Resources

- [n8n Documentation](https://docs.n8n.io/)
- [Docker Documentation](https://docs.docker.com/)
- [ShanNLP Repository](https://github.com/Alexpeain/ShanNLP)
- [Google Sheets API](https://developers.google.com/sheets/api)
- [PyThaiNLP](https://github.com/PyThaiNLP/pythainlp) - Parent project inspiration
- [Shan Language](https://en.wikipedia.org/wiki/Shan_language)

## 👤 Author

**Alexpeain** - Self-taught developer focused on NLP for Shan/Tai languages and language preservation.

- GitHub: [@Alexpeain](https://github.com/Alexpeain)
- Projects: Myanmar Book Reviews, ShanNLP, DSA NagBot, Accountability Board
- Location: Shan State, Myanmar

---

**Last updated**: January 2026

---

## 📞 Support

For issues, questions, or suggestions:
- Open an issue on GitHub
- Check existing documentation
- Review n8n workflow logs for debugging
- Consult ShanNLP project for NLP-specific questions
