# is456.com

Source for the IS 456 · Agentic AI in Business lab site, built with [Zensical](https://zensical.org/).

| Environment | URL |
|---|---|
| Local dev | http://dev.is456.com:8000 |
| Production (GitHub Pages) | https://www.is456.com |

## Local development

Tooling is pinned in `mise.toml` (Python 3.14, uv). Python dependencies are managed with uv (`pyproject.toml`, `uv.lock`).

```bash
mise install      # install Python + uv
mise run install  # uv sync
mise run dev      # live-reload server on $DEV_ADDR (default localhost:8000)
mise run build    # build static site into site/
```

The dev server binds to `localhost:8000` by default. To reach it from other machines (e.g. via `dev.is456.com`), create a gitignored `mise.local.toml`:

```toml
[env]
DEV_ADDR = "0.0.0.0:8000"
```

`dev.is456.com` needs a DNS record (or hosts-file entry) pointing at the machine running the dev server.

## Deployment

Pushes to `main` build and deploy to GitHub Pages via `.github/workflows/pages.yml`. Pull requests run the build only. `docs/CNAME` sets the custom domain to `www.is456.com`.
