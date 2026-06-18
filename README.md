# SpinMyGP SDK Documentation Project

This repository contains the documentation for the SpinMyGP SDK, built with [MkDocs](https://www.mkdocs.org/) and the [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme.

## Project Structure

- `docs/`: Source Markdown files for the documentation.
  - `index.md`: The main partner integration guide.
  - `changelog.md`: SDK version history.
- `mkdocs.yml`: Configuration file for the MkDocs site.
- `site/`: The generated static website (after running `mkdocs build`).

## Local Development

To preview the documentation locally:

1.  **Install dependencies:**
    ```bash
    pip install mkdocs mkdocs-material
    ```

2.  **Start the development server:**
    ```bash
    python -m mkdocs serve
    ```
    The site will be available at `http://127.0.0.1:8000/`.

3.  **Build the static site:**
    ```bash
    python -m mkdocs build
    ```
    The output will be in the `site/` directory.
