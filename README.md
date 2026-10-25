# EOEPCA Operations Building Block

This repository contains the documentation source for the EOEPCA Operations
Building Block. The pages describe the monitoring tools and include examples
for EO platform operators. They are published on Read the Docs.

The deployment sources live in the
[`EOEPCA/eoepca-plus`](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations)
repository under `argocd/eoepca/operations`. The deployable parts are in the
`parts/` directory.

## Update the docs

Edit the content in [`docs/`](./docs) and verify the site locally:

```bash
uv run --with-requirements docs/requirements.txt mkdocs serve
```

## Read the Docs

The published documentation is available at:

<https://eoepca.readthedocs.io/projects/operations/>
