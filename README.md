# EOEPCA Operations Building Block

This repository contains the documentation source for the EOEPCA Operations Building Block. Its job is to publish the Read the Docs pages and provide a small set of examples that explain why the building block exists for EO platform operators.

The deployment sources live in the [`EOEPCA/eoepca-plus`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/operations) repository, especially under `argocd/operations`.

## Update the docs

Edit the content in [`docs/`](./docs) and verify the site locally:

```bash
uv run --with-requirements docs/requirements.txt mkdocs serve
```

## Read the Docs

The published documentation is available at:

<https://eoepca.readthedocs.io/projects/operations/>
