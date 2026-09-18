# ulaidotin

Sample documentation site built with [Hugo](https://gohugo.io) and the
[Docsy](https://www.docsy.dev) theme (based on
[google/docsy-example](https://github.com/google/docsy-example)).

Live site: https://docs.ulai.co.in/

## Local development

Requires Hugo **extended** ≥ 0.160.1, Go, and Node.js.

```sh
npm install
npm run serve   # http://localhost:1313/
npm run build   # outputs to ./public
```

## Deployment

Every push to `main` builds the site and deploys it to GitHub Pages via
`.github/workflows/hugo.yaml`.
