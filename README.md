# Gelaratios

Minimal gelato ratio calculator (Svelte + Vite).

Pick style (latte/uovo) and sweetener (dextrose/glucose syrup). Set batch size, edit any ingredient, and the rest adjust proportionally. Ratios come from 3000 g reference datasets.

## Run (dev)
```bash
npm install
npm run dev
```

## Build
```bash
npm run build
npm run preview
```

## Docker
```bash
docker build -t gelaratios:latest .
docker run --rm -p 8080:80 gelaratios:latest
```

## GHCR image
```bash
docker pull ghcr.io/hamletdufromage/gelaratios:latest
docker run --rm -p 8080:80 ghcr.io/hamletdufromage/gelaratios:latest
```

## Config
- Batch increment: `INCREMENT` in `src/App.svelte`
