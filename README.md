See config.yaml and fill data CSVs. Run the notebook and use the corrected validator to compare production-only metrics.

## Demo mode
- Set `demo_mode: true` in `config.yaml` to auto-fill blanks **from `data/demo_defaults.yaml`**.
- You can edit `demo_defaults.yaml` to change demo values.
- With `demo_mode:false`, **all blanks are errors** (no fabrication).

