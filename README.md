# Nepřehlédni to — Hnutí Kruh

Kampaňový web postavený na [Astro](https://astro.build/).

## Požadavky
- Node.js ≥ 18.14 (doporučeno 20+)
- npm (nebo jiný správce balíčků)

## Lokální vývoj
```bash
npm install
npm run dev
```

## Build a náhled buildu
```bash
npm run build
npm run preview
```

## Struktura
- `src/` — stránky, layouty, styly
- `public/` — statická aktiva (obrázky, ikony, favicon)
- `dist/` — produkční build (generuje se)

## Nasazení
Výstup z `dist/` je čisté statické HTML/CSS/JS (lze hostovat na libovolném static hostingu).

## Analytics (Matomo)
Matomo se načte až po souhlasu uživatele (viz consent banner v `src/layouts/Layout.astro`).
## 🧞 Commands

All commands are run from the root of the project, from a terminal:

| Command                   | Action                                           |
| :------------------------ | :----------------------------------------------- |
| `npm install`             | Installs dependencies                            |
| `npm run dev`             | Starts local dev server at `localhost:4321`      |
| `npm run build`           | Build your production site to `./dist/`          |
| `npm run preview`         | Preview your build locally, before deploying     |
| `npm run astro ...`       | Run CLI commands like `astro add`, `astro check` |
| `npm run astro -- --help` | Get help using the Astro CLI                     |

## 👀 Want to learn more?

Feel free to check [our documentation](https://docs.astro.build) or jump into our [Discord server](https://astro.build/chat).
