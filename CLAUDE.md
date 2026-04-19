# mx-brief

**Public static output repo.** Content here is not authored directly — it is generated and pushed by the sibling private repo `mx-brief-source` (see `../mx-brief-source/`).

## Orchestration

- **No dev server, no build, no deploy pipeline in this repo.** Just static HTML (`index.html`, `posts/`, `archive/`).
- **Ship**: handled upstream — `pnpm publish` in `mx-brief-source` writes files here and pushes.
- **Preview locally**: open `index.html` in a browser, or serve with any static file server.

### Repo pattern

- This repo: **public** — `joshtune/mx-brief` (GitHub Pages)
- Source: **private** — `joshtune/mx-brief-source`
- **Do not hand-edit posts here.** Changes will be overwritten on the next publish. Edit generators / templates in `mx-brief-source` instead.
