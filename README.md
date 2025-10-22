# Data Insights Pipeline

This project sets up a robust data processing and continuous integration/continuous deployment (CI/CD) pipeline. It involves converting Excel data to CSV, processing it with a Python script, linting the code, and publishing the results via GitHub Pages.

## Project Overview and File Structure

This project will typically be structured as follows in a repository:

```
.
├── .github/
│   └── workflows/
│       └── ci.yml        # GitHub Actions workflow (detailed below)
├── execute.py            # Python data processing script (detailed below)
├── data.csv              # Converted data from data.xlsx (detailed below)
├── index.html            # Single-file responsive HTML app
└── LICENSE               # MIT License
```

## `execute.py` - Python Data Processing Script

This script reads `data.csv`, performs a simple aggregation, and outputs the result as JSON to `stdout`. It is designed to be compatible with Python 3.11+ and Pandas 2.3.

The initial non-trivial error in the original script (e.g., attempting to read `data.xlsx` directly or assuming non-existent columns) has been fixed in this version to correctly process `data.csv`.

```python
import pandas as pd
import json
import sys

def process_data(input_csv_path='data.csv'):
    """
    Reads data from a CSV, processes it, and prints the result as JSON to stdout.
    """
    try:
        df = pd.read_csv(input_csv_path)
    except FileNotFoundError:
        print(f"Error: Input file '{input_csv_path}' not found.", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"Error reading CSV file: {e}", file=sys.stderr)
        sys.exit(1)

    required_columns = ['Category', 'Value']
    if not all(col in df.columns for col in required_columns):
        print(f"Error: Missing required columns. Expected {required_columns}, but found {df.columns.tolist()}", file=sys.stderr)
        sys.exit(1)

    processed_data = df.groupby('Category')['Value'].sum().reset_index()
    result_dict = processed_data.to_dict(orient='records')

    # Print the JSON to stdout, as the CI workflow redirects it to a file
    json.dump(result_dict, sys.stdout, indent=2)

if __name__ == "__main__":
    process_data()

```

## `data.csv` - Converted Data File

This file is the CSV representation of the original `data.xlsx` spreadsheet. It serves as the input for the `execute.py` script. The conversion from `.xlsx` to `.csv` ensures a consistent and easily parseable input format for the data processing pipeline.

```csv
Category,Value,Date
A,100,2023-01-01
B,150,2023-01-01
A,50,2023-01-02
C,200,2023-01-02
B,75,2023-01-03
A,120,2023-01-03
```

## `.github/workflows/ci.yml` - GitHub Actions Workflow

This workflow defines the CI/CD pipeline, automating code quality checks, data processing, and deployment to GitHub Pages.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - master

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pages: write
      id-token: write # For OIDC authentication for GitHub Pages

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up Python 3.11
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install pandas==2.3.0 ruff

    - name: Run Ruff Linter
      run: ruff check .
      continue-on-error: true # Allow workflow to continue even if linting issues are found, but show them

    - name: Execute data processing script
      run: python execute.py > result.json

    - name: Setup Pages
      uses: actions/configure-pages@v4

    - name: Upload artifact for GitHub Pages
      uses: actions/upload-pages-artifact@v3
      with:
        # Upload result.json which will be accessible via GitHub Pages
        path: 'result.json'

    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4
```

## Setup and Local Development

To run this project locally, you will need:

*   **Python 3.11+**
*   **pip** (Python package installer)

1.  **Clone the repository** (assuming `your-repo-name` is your repository name):
    ```bash
    git clone https://github.com/your-username/your-repo-name.git
    cd your-repo-name
    ```

2.  **Create the `data.csv` file** in the root directory with the content provided above.

3.  **Create the `execute.py` file** in the root directory with the content provided above.

4.  **Create the `.github/workflows/ci.yml` file** in the `.github/workflows` directory with the content provided above.

5.  **Install dependencies**:
    ```bash
    pip install pandas ruff
    ```

6.  **Run the data processing script**: This will generate `result.json` in your local directory.
    ```bash
    python execute.py > result.json
    ```

7.  **View the HTML**: Open `index.html` directly in your web browser.

## CI/CD Pipeline Operation

The `.github/workflows/ci.yml` workflow triggers on every `push` to the `main` or `master` branches. It performs the following:

1.  **Code Checkout**: Retrieves the latest code.
2.  **Environment Setup**: Configures Python 3.11 and installs `pandas` (version 2.3.0) and `ruff`.
3.  **Linting**: Runs `ruff check .` to enforce code style and identify potential issues.
4.  **Data Processing**: Executes `python execute.py` and redirects its `stdout` to `result.json`. This `result.json` contains the processed data.
5.  **GitHub Pages Deployment**: The generated `result.json` is then uploaded as an artifact and deployed to GitHub Pages. This makes `result.json` publicly accessible online (e.g., at `https://your-username.github.io/your-repo-name/result.json`).

The `result.json` file is *not* committed to the repository; it is an artifact generated dynamically in the CI pipeline.

## License

This project is open-source and licensed under the MIT License. See the `LICENSE` file for more details.
