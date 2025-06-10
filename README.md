# Solana Kit Docs

A comprehensive documentation project for Solana development tools and utilities built with MkDocs.

## Installation

### Prerequisites

- Python 3.8 or higher
- pip package manager

### Setup Virtual Environment

```bash
# Create virtual environment
python -m venv venv

# Activate virtual environment
# On Windows:
venv\Scripts\activate
# On macOS/Linux:
source venv/bin/activate
```

### Install Dependencies

```bash
pip install -r requirements.txt
```

## Development

### Start Development Server

```bash
mkdocs serve
```

### Build Documentation

```bash
mkdocs build
```

### Deploy to GitHub Pages

```bash
mkdocs gh-deploy
```

## Project Structure

```
solana-kit-docs/
├── docs/           # Documentation files
├── mkdocs.yml      # MkDocs configuration
├── requirements.txt # Python dependencies
├── venv/           # Virtual environment
└── README.md       # This file
```

## License

MIT License
