# BMA SBA Benchmark Model - Technology Stack

## Document Control

| Field | Value |
|---|---|
| **Project Name** | BMA SBA Benchmark Model |
| **Document Type** | Technology Stack & Justification |
| **Version** | 1.0 (Draft for Discussion) |
| **Date** | March 2026 |

---

## 1. Technology Decisions Summary

| Layer | Technology | Version | License |
|---|---|---|---|
| **Language** | Python | 3.11+ | PSF (open source) |
| **Yield Curves & Bond Pricing** | QuantLib-Python | Pinned (e.g., 1.33) | BSD (open source) |
| **Data Manipulation** | pandas | 2.x | BSD |
| **Numerical Computing** | NumPy | 1.26+ | BSD |
| **Root Finding** | SciPy | 1.12+ | BSD |
| **Data Validation** | Pydantic | 2.x | MIT |
| **Excel Output** | openpyxl | 3.1+ | MIT |
| **Configuration** | PyYAML / ruamel.yaml | Latest | MIT |
| **Audit Storage** | SQLite (built-in) + PyArrow (Parquet) | Built-in / Latest | Public domain / Apache 2.0 |
| **Testing** | pytest + pytest-cov | Latest | MIT |
| **Property Testing** | hypothesis | Latest | MPL 2.0 |
| **CLI Interface** | click or typer | Latest | BSD / MIT |
| **Notebooks** | Jupyter / JupyterLab | Latest | BSD |
| **Package Management** | pip + pyproject.toml | Built-in | - |
| **Environment Management** | conda (recommended) or venv | Latest | BSD |
| **Version Control** | Git + GitHub | Latest | - |
| **IDE** | VS Code + Python Extension | Latest | MIT |
| **AI Assistant** | GitHub Copilot | Latest | Commercial |

---

## 2. Technology Justifications

### 2.1 Python 3.11+

**Why Python?**
- The most widely used language for data science, actuarial modeling, and quantitative finance
- Rich ecosystem of financial and scientific libraries
- Readable syntax - actuaries and auditors can understand the code
- GitHub Copilot has the most training data for Python, maximizing AI assistance
- Growing adoption in the actuarial profession (lifelib, cashflower, heavymodel)
- The Project Lead's path from Prophet to Python is well-trodden in the industry

**Why 3.11+?**
- Significant performance improvements over 3.9/3.10 (10-60% faster)
- Better error messages (helps a Python newcomer)
- `tomllib` built-in (for pyproject.toml parsing)
- All required libraries support 3.11+

**What about Julia?** The previous Hybrid_Implementation docs proposed Julia for computational speed. For this project, Julia is unnecessary: the computational workload (9 scenarios x 50 periods x ~1000 assets) runs in seconds in Python. Adding Julia would double the skill requirements without measurable benefit.

### 2.2 QuantLib-Python

**Why QuantLib?**
- Industry-standard library for bond pricing and yield curve construction
- Used by banks, hedge funds, and financial software companies worldwide
- Open source with active maintenance (>20 years of development)
- Provides correct handling of day counts, compounding, calendar conventions - details that are easy to get wrong in a custom implementation
- Using a recognized library adds credibility to the benchmark model

**Why NOT more QuantLib?**
- QuantLib's internals are C++ - opaque to auditors
- Poor Python documentation (most docs are for C++)
- Error messages are cryptic C++ exceptions
- For a benchmark model, the projection logic MUST be transparent
- **Decision: QuantLib for curves and pricing ONLY. Projection logic in pure Python.**

**QuantLib-Python Installation:**
- **Recommended:** `conda install -c conda-forge quantlib-python` (pre-compiled, most reliable)
- **Alternative:** `pip install QuantLib-Python` (may require C++ build tools on Windows)
- **Pin the version** in pyproject.toml to ensure reproducibility
- **Test on BMA workstations in Week 1** - installation issues should be caught before development begins

### 2.3 pandas 2.x

**Why pandas?**
- The standard for tabular data in Python
- Natural fit for actuaries coming from Prophet (model points ARE tables)
- Excellent I/O: read_csv, read_excel, to_excel, to_parquet
- Powerful operations: groupby, merge, pivot (essential for cash flow aggregation)
- DataFrames are inspectable, printable, and debuggable

**Why 2.x?**
- Copy-on-write (reduces memory bugs)
- Arrow-backed string type (faster string operations)
- Better nullable integer support

### 2.4 Pydantic v2

**Why Pydantic?**
- Type-safe data validation at the boundary (model point loading)
- Clear, human-readable error messages when data is invalid
- Auto-generates JSON schemas for documentation
- Enforces the "Fail Loudly" principle - invalid data stops the run immediately
- Popular in the Python ecosystem; well-supported by Copilot

**Example:**
```python
# If a model point has par_value = -100, Pydantic catches it immediately:
# ValidationError: par_value - Input should be greater than 0
```

### 2.5 openpyxl

**Why openpyxl?**
- Full control over Excel formatting (fonts, colors, borders, conditional formatting, charts)
- BMA analysts live in Excel - the output must look professional and regulatory-quality
- Can create multi-tab workbooks with precise layout control
- Pure Python (no Excel installation required on the machine)

**Why NOT xlsxwriter?** openpyxl can both read AND write Excel files (needed for importing company submissions). xlsxwriter is write-only.

### 2.6 SQLite + Parquet + JSON Lines (Audit Trail)

**Why this combination instead of a database?**

| Requirement | Solution | Why Not PostgreSQL/SQL Server? |
|---|---|---|
| Run metadata (who, when, what) | SQLite | SQLite is a file, not a server. Zero infrastructure. |
| Period-by-period cash flows | Parquet | Columnar format. Efficient for large DataFrames. One line in pandas to read/write. |
| Rule invocation traces | JSON Lines | One JSON object per line. Human-readable. grep-able. |

**PostgreSQL/SQL Server would require:**
- Database server installation and maintenance
- DBA support
- Network configuration
- Authentication setup
- Backup procedures

**SQLite + Parquet + JSON Lines require:**
- Nothing. They're files. Copy them. Email them. Open them in any tool.

For a desktop calculation engine used by a small team, files are simpler, more portable, and more than sufficient.

### 2.7 pytest

**Why pytest?**
- The standard Python testing framework
- Excellent VS Code integration (test explorer, inline results)
- Fixtures for shared setup (reusable test portfolios, curves)
- Parametrize for running the same test with different scenarios
- pytest-cov for coverage reporting

**Testing Philosophy:**
- Every BMA rule function has a unit test
- The BMA illustrative calculation is the golden integration test
- Golden file regression tests prevent unintended changes

### 2.8 Git + GitHub

**Why GitHub (not GitLab/Bitbucket/Azure DevOps)?**
- Already in use (the repo exists on GitHub)
- GitHub Actions for CI/CD (run tests on every push)
- GitHub Copilot integration is tightest with GitHub
- Pull request workflow for code review
- Issue tracking for bugs and feature requests

---

## 3. What is NOT in the Stack (and Why)

| Technology | Why NOT |
|---|---|
| **FastAPI / Flask / Django** | This is a calculation engine, not a web application. No API needed. |
| **React / TypeScript / Vue** | No web frontend. Output is Excel. BMA analysts don't need a dashboard. |
| **PostgreSQL / SQL Server** | Overkill for a desktop tool. SQLite + files are sufficient and simpler. |
| **Docker / Kubernetes** | Runs on a workstation. Containerization adds complexity with no benefit. Can be added later if deployment to a server is needed. |
| **Julia** | Computational workload doesn't justify a second language. Python is fast enough. |
| **Apache Spark / Dask** | Data volumes are small (thousands of records, not millions). pandas is sufficient. |
| **MongoDB / Redis** | No need for document storage or caching. |
| **Terraform / Ansible** | No cloud infrastructure to manage. |
| **Celery / RabbitMQ** | No async job processing needed. Batch runs are sequential or use Python multiprocessing. |
| **Prophet (FIS)** | Commercial license, closed source, not suitable for a transparent benchmark model. |
| **AXIS / MoSes** | Same - commercial, closed source. |

### The Principle: Minimum Viable Technology Stack

Every technology in the stack must earn its place by solving a problem that cannot be solved more simply. A benchmark model should be simple enough that a new developer can understand the full stack in a day.

```
Full Stack for This Project:
  Python + QuantLib + pandas + Pydantic + openpyxl + pytest + Git

That's it. Everything else is optional complexity.
```

---

## 4. Development Environment Setup

### 4.1 Required Installation (Every Developer)

```bash
# 1. Install Miniconda (recommended over pip for QuantLib)
# Download from: https://docs.conda.io/en/latest/miniconda.html

# 2. Create project environment
conda create -n bma_sba python=3.11
conda activate bma_sba

# 3. Install QuantLib (conda-forge has pre-compiled binaries)
conda install -c conda-forge quantlib-python

# 4. Install remaining dependencies
pip install pandas pydantic openpyxl pyyaml pyarrow scipy
pip install pytest pytest-cov hypothesis
pip install click  # or typer
pip install jupyterlab

# 5. Install the project in development mode
pip install -e .
```

### 4.2 VS Code Extensions

| Extension | Purpose |
|---|---|
| Python (ms-python) | Core Python support |
| Pylance | Type checking, IntelliSense |
| GitHub Copilot | AI code completion |
| Jupyter | Notebook support |
| GitLens | Git history and blame |
| Python Test Explorer | Run/debug tests from VS Code |

### 4.3 pyproject.toml Structure

```toml
[project]
name = "bma-sba-benchmark"
version = "0.1.0"
description = "BMA SBA Regulatory Benchmark Model"
requires-python = ">=3.11"
dependencies = [
    "pandas>=2.0",
    "numpy>=1.26",
    "scipy>=1.12",
    "pydantic>=2.0",
    "openpyxl>=3.1",
    "pyyaml>=6.0",
    "pyarrow>=14.0",
    "click>=8.0",
    "QuantLib-Python>=1.33",  # Pin specific version
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
    "hypothesis>=6.0",
    "jupyterlab>=4.0",
]

[project.scripts]
bma-sba = "bma_sba_benchmark.main:cli"

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=bma_sba_benchmark --cov-report=html"
```

---

## 5. CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: windows-latest  # Match BMA workstations
    steps:
      - uses: actions/checkout@v4
      - uses: conda-incubator/setup-miniconda@v3
        with:
          python-version: "3.11"
      - name: Install QuantLib
        run: conda install -c conda-forge quantlib-python
      - name: Install dependencies
        run: pip install -e ".[dev]"
      - name: Run tests
        run: pytest --cov=bma_sba_benchmark
      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: htmlcov/
```

---

## 6. Dependency Management

### 6.1 Version Pinning Strategy

| Category | Strategy | Rationale |
|---|---|---|
| **QuantLib-Python** | Pin exact version (e.g., ==1.33) | Minor version changes can alter numerical results |
| **pandas, numpy** | Pin major.minor (e.g., >=2.0,<3.0) | API stability within major version |
| **Pydantic** | Pin major (>=2.0,<3.0) | v1 to v2 was a breaking change |
| **pytest, openpyxl** | Pin major (>=7.0, >=3.1) | Low risk of breaking changes |
| **Everything else** | Latest compatible | Minimal risk |

### 6.2 Lock File

Use `pip freeze > requirements-lock.txt` after every dependency change. This ensures any environment can be reproduced exactly.

### 6.3 Dependency Update Process

- Update dependencies quarterly (not with every release)
- Run full test suite after any dependency update
- If QuantLib version changes, re-validate all bond pricing against independent sources

---

## 7. Alternatives Considered

### 7.1 Open-Source Actuarial Frameworks

| Framework | Considered For | Decision |
|---|---|---|
| **lifelib** | Complete actuarial modeling | Rejected: too liability-focused; not designed for asset CF projection |
| **cashflower** | Cash flow projection | Rejected: simpler than what we need; would need heavy customization |
| **heavymodel** | Cash flow modeling | Rejected: early stage, small community |

**Verdict:** These frameworks are designed for liability projection. Our model projects asset cash flows. The overlap is insufficient to justify adopting them as a foundation.

### 7.2 Pure Python (No QuantLib)

| Pro | Con |
|---|---|
| Maximum transparency | Must implement yield curve bootstrapping from scratch |
| No installation issues | Must implement all day count conventions |
| No C++ dependency | Must implement bond pricing (error-prone for complex instruments) |
| | Less credibility than using an industry-standard library |

**Verdict:** The risk of implementing bond pricing incorrectly outweighs the transparency benefit. QuantLib is quarantined in `curves/` with an audit wrapper - acceptable trade-off.

### 7.3 Excel/VBA

| Pro | Con |
|---|---|
| BMA analysts already know Excel | Not scalable beyond simple portfolios |
| No installation needed | No version control (Git) |
| | No automated testing |
| | Difficult to audit complex logic in VBA |
| | No rule traceability framework possible |

**Verdict:** Excel is the OUTPUT format, not the computation platform. The model produces Excel reports, but the calculations happen in Python where they can be tested, versioned, and audited.

---

## 8. Security Considerations

| Concern | Mitigation |
|---|---|
| **Source code security** | Private GitHub repository; access limited to team members |
| **Company data confidentiality** | No company data stored in the repo; data files in .gitignore; audit trail files secured per BMA data classification |
| **Dependency supply chain** | Use conda-forge (curated, signed packages); pin versions; periodic security audit with `pip-audit` |
| **No internet at runtime** | Model runs entirely offline; all dependencies installed during setup |
| **Audit trail integrity** | SQLite databases are write-once-per-run; Parquet and JSON files are immutable after creation |

---

## 9. IT Requirements for BMA

### Pre-Project IT Checklist

- [ ] Python 3.11+ approved for installation on BMA workstations
- [ ] Conda (Miniconda) approved for installation
- [ ] QuantLib-Python package approved (conda-forge channel)
- [ ] VS Code approved for installation
- [ ] GitHub Copilot license approved (requires internet for AI features)
- [ ] GitHub access (for version control and CI/CD)
- [ ] Sufficient disk space (~2 GB for environment + packages)
- [ ] Admin/elevated privileges for initial setup (or IT installs for developer)

### Ongoing IT Needs

- None - once installed, the model runs without internet or admin privileges
- Quarterly dependency updates may require temporary internet access or IT assistance
