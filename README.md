# Food Diary

Food Diary is a Codex skill for turning meal photos, receipts, menus, and notes into a structured food diary.

The bundled skill lives in [`skill/SKILL.md`](skill/SKILL.md) and is named `food-diary`.

## What It Can Do

- Log direct chat images, pasted screenshots, local image paths, and text-only meal notes.
- Classify entries as food, receipt, menu, non-food, text, or unreadable.
- Resolve date, time, and meal type from user text, image metadata, or submission time.
- Estimate portions, look up calories/macros, and write per-entry `.nutrition.json` files.
- Update `index.csv` with calories, protein, carbs, fat, confidence, notes, and record paths.
- Preserve direct user-provided files; only the legacy inbox flow deletes processed inbox images.
- Keep the old desktop `inbox/` flow available when explicitly requested.
- Commit and push the resulting food diary records to the configured health-log repo.

## Reports

Food Diary can generate two Word reports on the Desktop:

- `food-diary-report-YYYY-MM-DD.docx` - diet report from `index.csv`, including daily macro tables, complete-day trend charts, threshold markers, and data-driven diet signals.
- `food-diary-health-report-YYYY-MM-DD.docx` - health report after Apple Health sync, combining Apple Health aggregates with the food diary log.

Apple Health sync imports a local iOS Health export into `health/daily.csv`, `health/workouts.csv`, and `health/ecg/*.csv`, while deliberately excluding GPS route files.

## Data Repo

Food Diary writes source records into a separate private health-log repo, usually configured by `repo_path`:

```text
health-log/
├── index.csv
├── YYYY/MM/DD/<meal_type>/*.nutrition.json
├── YYYY/MM/DD/<meal_type>/*.note.txt
├── health/daily.csv
├── health/workouts.csv
└── inbox/                  # legacy optional input
```

The repo stores source data only. Generated Word reports and charts stay on the Desktop and are not committed.

## Config

The skill resolves the data repo path in this order:

1. A path explicitly supplied by the user.
2. `repo_path` from the Food Diary desktop config.
3. The legacy `kibble/config.toml` path, for existing installs.
4. `/Users/tom/Desktop/health-log`.

The desktop app keeps compatibility with existing `kibble/config.toml` files so old local setups continue to work.

## Optional Desktop App

The `app/` folder contains a small Tauri desktop drop target. It is optional: Food Diary can work directly from images sent to Codex.

If you use the desktop app, it copies dropped images into the configured repo's `inbox/` and pushes them. Later, ask Codex to process the legacy inbox with the `food-diary` skill.

```bash
cd app
pnpm install
pnpm tauri dev
pnpm tauri build
```

## License

MIT
