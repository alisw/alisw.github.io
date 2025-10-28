# ALICE Software Documentation

Documentation for the ALICE software build infrastructure and development workflows.

## Overview

This repository contains documentation for ALICE collaboration software, including tutorials for Git workflows, build infrastructure details, and operational guides.

## Development

This site is built using [MkDocs](https://www.mkdocs.org/) with the Material theme.

### Requirements

- Python 3.8+
- MkDocs
- MkDocs Material theme

### Installation

Optionally, create a virtual environment:

```bash
python -m venv venv
source venv/bin/activate
```

Install dependencies:

```bash
pip install -e .
```

### Local Development

To serve the documentation locally:

```bash
mkdocs serve
```

The site will be available at `http://127.0.0.1:8000/`

### Building

To build the static site:

```bash
mkdocs build
```
