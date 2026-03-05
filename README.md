# ALICE Software Documentation

Documentation for the ALICE software build infrastructure and development workflows.

## Overview

This repository contains documentation for ALICE collaboration software, including tutorials for Git workflows, build infrastructure details, and operational guides.

## Development

Requires Python 3.8+

### Install and run

#### Using `uv` (recommended)

Serve the documentation locally:

```bash
uv run mkdocs serve
```

Build the static site:

```bash
uv run mkdocs build
```

#### Using `pip`

Optionally, create a virtual environment:

```bash
python -m venv venv
source venv/bin/activate
```

Install dependencies:

```bash
pip install -e .
```

```bash
mkdocs serve
```

To build the static site:

```bash
mkdocs build
```
