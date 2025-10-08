# CSIRT-MU Writeups

This page contains CTF writeups and blog posts created by the CSIRT-MU Threat Management team. All content is provided "as is" for educational purposes only.

Feel free to contribute by submitting pull requests with new writeups or improvements to existing ones.

## Mirror

The branch `publish` is automatically mirrored to [https://github.com/CSIRT-MU/writeups](https://github.com/CSIRT-MU/writeups).

Merge from master to publish to trigger the mirroring.

## Deploy

```bash
poetry install
poetry run mkdocs build -d public
python -m http.server --directory public 8000
```

## Content

Markdown documents should have the following metadata at the top:
```
---
authors:
    - <author name>
date: <DD-MM-YYYY>
---
```