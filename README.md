# Operations

Documentation and examples for the EOEPCA Operations Building Block.

- [Documentation](https://eoepca.readthedocs.io/projects/operations/)
- [Operating model](docs/operating-model.md)
- [Monitoring stack](docs/monitoring-stack.md)
- [Dashboards and usage](docs/dashboards-and-usage.md)
- [STAC service path](docs/stac-scenario.md)
- [Workspace monitoring](docs/workspace-scenario.md)
- [Deployment sources](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/operations/parts)

## Local checks

To build the documentation and check links within it:

```bash
uv run --no-project --with-requirements docs/requirements.txt mkdocs build --strict
```

To preview it locally:

```bash
uv run --no-project --with-requirements docs/requirements.txt mkdocs serve
```

## License

[Apache 2.0](LICENSE).
