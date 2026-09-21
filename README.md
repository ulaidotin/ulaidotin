# ulaidotin

Documentation site for the [Ulai SIP Gateway](https://github.com/ulaidotin/ulai-sip-module),
built with [Hugo](https://gohugo.io) and the [Docsy](https://www.docsy.dev) theme.

Live site: https://docs.ulai.co.in/

## Local development

Requires Hugo **extended** ≥ 0.160.1, Go, and Node.js.

```sh
npm install
npm run serve   # http://localhost:1313/
npm run build   # outputs to ./public
```

## Content layout

All documentation lives under `content/docs/`:

| Path | Covers |
| --- | --- |
| `overview.md` | What the gateway is and where it sits |
| `getting-started/` | Prerequisites, build, first inbound and outbound call |
| `concepts/` | Call flows, routing plane, media pipeline |
| `configuration.md` | Every environment variable and built-in timeout |
| `http-api.md` | `/health`, `/calls`, `/sip/originate` |
| `operations/` | Deployment, networking, observability, troubleshooting |

## Deployment

Every push to `main` builds the site and deploys it to GitHub Pages via
`.github/workflows/hugo.yaml`.
