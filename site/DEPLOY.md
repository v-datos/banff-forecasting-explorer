# Publishing the Interactive Banff Explorer as a Public Site

[`index.html`](index.html) is a **100% standalone, zero-dependency static HTML/CSS/JS application** (`~101 KB`), deterministically built from pipeline CSV exports via:

```bash
banff-forecast --rebuild-site
```

It requires no Node build step in production, no backend API server, and no external CDN scripts.

---

## Live Deployment (Two-Repo Public Pages Architecture)

- **Public Live URL:** <https://v-datos.github.io/banff-forecasting-explorer/>
- **Public Explorer Repo:** <https://github.com/v-datos/banff-forecasting-explorer> (`site/`, `outputs/`, `docs/`)
- **Private Pipeline Repo:** <https://github.com/v-datos/banff-forecasting-pipeline> (`src/`, `data/`, `tests/`, `notebooks/`)
- **Automated Workflows:**
  - Public `deploy-pages.yml` (`v-datos/banff-forecasting-explorer`) automatically publishes `site/index.html` to GitHub Pages whenever `site/**` updates on `main`.
  - Private `ci.yml` (`v-datos/banff-forecasting-pipeline`) runs `banff-forecast --rebuild-site`, `pytest -v` (including Python-vs-JS scenario parity across all 6 presets), and Node JS syntax validation on every push/PR.
  - Private `refresh-forecasts.yml` (`5th of each month @ 08:15 UTC`) refreshes Town of Banff Tableau Public (`BanffTrafficData-GS.twb`), StatCan, Bank of Canada, Alberta Economic Dashboard, and ECCC data, commits updated outputs to the private repo, and (when `EXPLORER_PUSH_TOKEN` is set) syncs `site/`, `outputs/`, and `docs/` to `v-datos/banff-forecasting-explorer` (with `sync-explorer.yml` also handling manual/main pushes).

---

## Alternative Static Hosts (Zero Git Required)

You can also host the single [`site/index.html`](index.html) file directly on any HTTPS static provider:

- **Netlify Drop:** Visit <https://app.netlify.com/drop> and drag the `site/` folder onto the page for an instant `https://<site-name>.netlify.app` URL.
- **Cloudflare Pages:** Create a **Direct Upload** project in the Cloudflare Dashboard and upload `site/`.
- **Vercel:** Run `npx vercel site --prod` inside the project root.

> [!NOTE]
> Google Drive links preview or download `.html` files as raw documents rather than executing client-side JavaScript as a live website; use GitHub Pages (<https://v-datos.github.io/banff-forecasting-explorer/>) or another HTTPS static host for interactive public sharing.
