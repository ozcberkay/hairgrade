# HairGrade AI

Demand validation landing page for HairGrade AI — AI-powered hair loss analysis app.

## Stack

- Astro v5
- Tailwind CSS v4
- Cloudflare Pages deployment
- Tally.so form embed

## Development

```bash
npm install
npm run dev      # localhost:4321
npm run build    # dist/
```

## Deployment

Cloudflare Pages — auto-deploys on push to `master`.

- Build command: `npm run build`
- Output directory: `dist`
- Custom domain: `hairgrade.auraworks.dev`

## Tracking

- Meta Pixel (placeholder in Layout.astro)
- TikTok Pixel (placeholder in Layout.astro)
- UTM parameter parsing → Tally hidden fields
