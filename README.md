# Code Review: CAFA-evaluator

**BIOL 7180 — Code Review Assignment**

**Paper:** Piovesan D, Zago D, Joshi P, et al. (2024). CAFA-evaluator: a Python tool for benchmarking ontological classification methods. *Bioinformatics Advances*, 4(1), vbae043. DOI: [10.1093/bioadv/vbae043](https://doi.org/10.1093/bioadv/vbae043)

**Code repository:** [https://github.com/BioComputingUP/CAFA-evaluator](https://github.com/BioComputingUP/CAFA-evaluator)

**PyPI package:** [https://pypi.org/project/cafaeval/](https://pypi.org/project/cafaeval/)

---

## Summary of the Paper

Piovesan et al. present CAFA-evaluator, an open-source Python tool for benchmarking prediction methods whose targets are organized in hierarchical ontologies, such as the Gene Ontology (GO). The tool was developed to address a long-standing gap in the protein function prediction community: the lack of a well-documented, easy-to-use, and language-accessible evaluation framework. Prior solutions, including the original MATLAB code from the CAFA2 challenge, were difficult to install, poorly documented, and hard-coded for specific ontologies.

CAFA-evaluator generalizes the evaluation of multi-label classifiers to directed acyclic graphs (DAGs), using matrix computation and topological sorting for efficiency. It computes standard metrics including precision, recall, F-measure (both macro- and micro-averaged), weighted F-measure, semantic distance (S-score), and generates precision–recall curves. The software accepts an ontology file in OBO format, a ground truth annotation file, and a folder of prediction files. An optional information accretion file enables weighted evaluation. The tool was validated by replicating published results from CAFA2 and CAFA3 and was adopted as the official evaluation software for the CAFA5 challenge on Kaggle.

The software is distributed as a pip-installable package (`cafaeval`) and also provides a command-line interface and a Jupyter Notebook for plotting results. Its only dependencies are NumPy, Pandas, and Matplotlib.

As requested by my professor, my review below focuses on the code associated with this manuscript.

---

## High-Level Feedback

1. **Installation and basic usage are straightforward.** The package installed cleanly via `pip install cafaeval` and also from a local clone of the repository using `pip install .` with no errors. Running the command-line tool on the bundled example data produced output within seconds, and the output matched the format described in the README. This is a major strength—many bioinformatics tools fail at this first hurdle.

2. **The codebase is impressively compact and focused.** The entire Python source consists of only four files totaling approximately 663 lines of code (`__main__.py`, `evaluation.py`, `parser.py`, `graph.py`). This small footprint makes the code relatively easy to read and audit, and it aligns with the paper's claim that the tool is easy to maintain. The separation of concerns across the four modules (CLI, evaluation logic, file parsing, graph data structures) is logical and appropriate.

3. **Documentation quality is mixed.** The GitHub README is detailed and provides clear instructions for both command-line and library usage, including example input/output formats and a comprehensive parameter table. The GitHub Wiki is excellent—it provides algorithmic explanations with illustrative figures showing the effects of different propagation strategies, normalization modes, and threshold settings. However, the in-code documentation (docstrings and inline comments) is sparse relative to the complexity of the evaluation logic. Several core functions lack docstrings entirely or have incomplete ones, which would make it harder for a new contributor or advanced user to understand the implementation details without consulting the Wiki separately.

4. **There is no automated test suite.** The repository contains no unit tests, integration tests, or any testing framework (e.g., pytest). For software that serves as the official evaluation tool for an international benchmarking competition, this is a significant concern. While the authors state that the tool has been validated against CAFA2 and CAFA3 published results, there is no way for a user or contributor to verify that changes to the code do not break existing functionality. Adding even a small set of tests using the bundled example data would greatly improve confidence in the software's correctness and make it safer to accept community contributions.

5. **There is no continuous integration (CI) or automated quality assurance.** The repository has no `.github/workflows` directory or equivalent CI configuration. Setting up a basic GitHub Actions workflow to run linting (e.g., `flake8` or `ruff`) and tests on each push or pull request would be a low-effort, high-value addition, especially since the project does accept external pull requests (as evidenced by merged PRs from community contributors).

6. **No formal environment specification is provided.** While dependencies are listed in `pyproject.toml` (NumPy, Pandas, Matplotlib), no version pins or constraints are specified. The `requires-python` field is set to `>=3.6`, but Python 3.6 reached end-of-life in December 2021. Given that the code uses features like `with (open(...) as f):` (parenthesized context managers, a Python 3.10+ feature in `parser.py` line 142), the minimum version should be updated to at least `>=3.10`. The absence of version constraints on dependencies (e.g., NumPy 2.x introduced breaking changes) could cause installation failures or subtle bugs in the future. A `requirements.txt` or pinned dependency ranges would help with reproducibility.

7. **Reproducibility of the paper's results is partially supported.** The Wiki provides links to CAFA2 and CAFA3 ground truth and ontology files, which is commendable. However, the paper does not include a specific script or command that a reader could run end-to-end to reproduce the exact figures or tables in the manuscript. The Jupyter Notebook (`plot.ipynb`) requires manual modification of hardcoded paths and parameters in its first cell, which introduces a barrier to reproducibility. A wrapper script or a Makefile that automates the full pipeline from input data to final figures would significantly strengthen reproducibility.

---

## Specific, Lower-Level Feedback

- **`__init__.py` is empty.** The package's `__init__.py` file contains no imports or version information. Adding a `__version__` variable (e.g., `__version__ = "1.3.0"`) and importing the main `cafa_eval` function would make the package more Pythonic and allow users to check the version programmatically with `cafaeval.__version__`.

- **Incomplete docstrings on core functions.** The `compute_confusion_matrix` function in `evaluation.py` (line 28) has a docstring that begins with "Perform the evaluation at the matrix level for all tau thresholds" but then trails off with "The calculation is" and never finishes the sentence. Several other important functions, such as `normalize` (line 109), `evaluate_prediction` (line 144), and `cafa_eval` (line 166), have no docstrings at all. Since these are the functions that advanced users and developers interact with, documenting their parameters, return types, and behavior would improve usability considerably.

- **No type hints anywhere in the codebase.** The code does not use Python type annotations on any function signatures. Adding type hints (e.g., `def cafa_eval(obo_file: str, pred_dir: str, gt_file: str, ...) -> tuple[pd.DataFrame, dict]:`) would make the API clearer, enable better IDE support, and allow static analysis tools like `mypy` to catch potential bugs.

- **The `Graph.__init__` method ends with `return`.** In `graph.py` line 74, the `__init__` method ends with an explicit `return` statement. While this is not an error, it is unnecessary and non-idiomatic in Python. Removing it would clean up the code slightly.

- **Magic numbers in the code.** In `evaluation.py`, the metrics array is initialized with `np.zeros((len(tau_arr), 6), dtype='float')` (line 34), and the six columns are later referenced by positional index (`metrics[i, 0]`, `metrics[i, 1]`, etc.) without named constants. While there is a corresponding `columns` list in `compute_metrics`, making the mapping between column names and indices explicit via named constants or an enum would improve readability and reduce the risk of indexing errors.

- **Potential issue with the DAG stored as a dense boolean matrix.** In `graph.py` line 41, the adjacency matrix for the ontology is stored as a dense NumPy boolean array: `self.dag = np.zeros((self.idxs, self.idxs), dtype='bool')`. For large ontologies like the full Gene Ontology (which has over 40,000 terms), this creates a matrix of approximately 40,000 × 40,000 = 1.6 billion elements. Even as booleans (1 byte each), this would consume about 1.6 GB of memory just for the DAG. The paper mentions "sparse matrices," but the actual implementation uses dense NumPy arrays. Using `scipy.sparse` matrices would significantly reduce memory usage, especially since the DAG is inherently sparse (each term has only a few parents).

- **Topological sort uses `list.pop(0)`, which is O(n).** In `graph.py` line 98, the topological sort implementation uses `queue.pop(0)` to dequeue elements. For a Python list, `pop(0)` is O(n) because it requires shifting all remaining elements. Using `collections.deque` and its `popleft()` method would make this O(1) and is the standard approach for BFS-style algorithms in Python. For small ontologies this is negligible, but for large ontologies with tens of thousands of terms it could matter.

- **Typo in the README output description.** The README states "The same files are gerated by both the command line and the `write_results` function" — "gerated" should be "generated."

- **The `plot.ipynb` notebook is disproportionately large.** The notebook file is approximately 995 KB, which is very large relative to the rest of the codebase. This is likely because it contains embedded output cells (plots, dataframes). Clearing output cells before committing would reduce the repository size and avoid cluttering diffs in version control.

- **The CHANGELOG references a "current" version with no date.** The top entry in `CHANGELOG.md` reads `## [current] -` with a note about bootstrap confidence intervals being in progress. While this communicates the development roadmap, it would be cleaner to use a standard convention like `## [Unreleased]` (as recommended by [keepachangelog.com](https://keepachangelog.com/)).

- **Error handling in parsers could be more informative.** The file parsers in `parser.py` open files with bare `open()` calls and do not catch or re-raise exceptions with context. If a user provides a malformed OBO file or a ground truth file with unexpected formatting, the resulting error message (e.g., a generic `IndexError` or `ValueError`) would not clearly indicate what went wrong or which line of the input file caused the problem. Adding try-except blocks with informative error messages around the parsing logic would improve the user experience, especially for users who are new to the CAFA input formats.

- **The `Prediction.__str__` method may not be useful.** In `graph.py` line 137, the `Prediction` class defines a `__str__` method that prints each row's index, the full matrix row, and namespace. For large matrices, this would produce an enormous and unreadable string. A more useful implementation might show a summary (e.g., number of targets, number of non-zero predictions, namespace).

- **No license header in source files.** While the repository includes a `LICENCE.md` file (GPL-3.0), none of the individual Python source files contain a license header or copyright notice. Adding a brief header to each file is a common best practice for open-source projects and helps clarify the licensing terms when files are shared or reused individually.

- **The `setup.py` file is mostly redundant.** The repository contains both a `pyproject.toml` and a `setup.py`. The `setup.py` file contains only `from setuptools import setup; setup()`, which is a legacy shim. Since the project already has a properly configured `pyproject.toml` with all metadata, the `setup.py` can be removed to simplify the build configuration, unless backward compatibility with very old pip versions is a concern.

- **Variable naming could be more descriptive in places.** Several variables use short, non-descriptive names that make the code harder to follow for someone unfamiliar with the CAFA evaluation framework. For example, in `evaluation.py`: `g` for ground truth matrix, `p` for thresholded predictions, `ne` for the number of elements, `w` for weights, and `toi` for "terms of interest." Expanding these to more descriptive names (e.g., `gt_matrix`, `pred_binary`, `n_gt_targets`, `ic_weights`, `terms_of_interest`) would improve readability without affecting performance.

---

## Summary

CAFA-evaluator is a well-conceived, focused tool that fills an important gap in the protein function prediction community. Its small codebase, minimal dependencies, clean pip-installable packaging, and extensive Wiki documentation are notable strengths. The tool installed and ran successfully on the first attempt with the bundled example data, which is a testament to the authors' attention to usability.

The most impactful improvements the authors could make are: (1) adding an automated test suite, even a minimal one based on the example data, to guard against regressions; (2) expanding in-code documentation with complete docstrings and type hints; (3) pinning dependency versions and correcting the minimum Python version; and (4) providing a reproducibility script that generates the paper's results end-to-end. These changes would further solidify CAFA-evaluator's role as a reliable, community-standard benchmarking tool.
