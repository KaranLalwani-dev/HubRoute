# Cable Ring Planner (Honeywell PS4, UR Hackathon 2026H2)

Local web app around our solver: pick a ResPlan plan, solve it (minimum number of hubs, then the cheapest cable rings), see the annotated picture,
download the JSON in the slide-8 format (plus SVG and PNG of the picture).

Status: built and tested against **synthetic mock plans only** (the sandbox it was written in has no ResPlan data). First thing to do on your machine: run the smoke test on real plans.

## Layout
```
cells/      the solver, exactly the notebook cells (V1 validator, P1 prepare, S1 ring solver, S11 multi-hub, S18 picture helpers, S19 JSON export, S20 batch runner)
backend/    core.py (loads the cells into one namespace), scene.py (geometry for the page), app.py (FastAPI)
frontend/   index.html, style.css, app.js (plain JavaScript, SVG drawing, no build step)
tests/      test_codec.py (plan file round trip, no data needed), test_smoke.py (needs real plans)
tools/      export_plans.py (ResPlan.pkl to plan files)
```
The algorithm lives only in `cells/`; the notebook and the app run the same files.

## Run
1. Python 3.10+. `python -m venv .venv`, activate it, `pip install -r requirements.txt`.
2. Put `ResPlan.pkl` and `split.json` into `backend/data/` (or set `RESPLAN_PKL=/path/to/ResPlan.pkl`; `split.json` is read from the same folder).
   If the pickle does not load, install the same shapely / networkx versions as in the Kaggle notebook.
3. Put the dataset helper `resplan_utils.py` (the file imported in Kaggle Cell 25) into `backend/`.
4. From the repository root: `python -m uvicorn backend.app:app --port 8000`, then open http://127.0.0.1:8000
   (`?plan=14433` in the address opens a plan directly).
5. Optional check: `python -m pytest -q`.

### Plans in and out
- The page has a plan browser (3 plans per page by default; 3, 6, 12 or 24 per page, page number box, find by id). Each plan shows a small picture and, once solved, its J and hub count.
- **Upload plan list**: one JSON file that holds many plans (a list, or `{"plans": [...]}`). **Upload single plan**: one plan as a JSON file (several files at once also work). You can also drop files onto the picture area.
  Uploaded plans appear under the split "uploaded"; a plan with the same id as a dataset plan replaces it.
- The file layout is the dataset's own plan dict: the same keys (`id`, `area`, `net_area`, `wall_depth`, ..., one geometry per layer) with the geometries as WKT text; see `backend/plan_codec.py`.
  Make such files from the pickle with `tools/export_plans.py`:
  ```
  python tools/export_plans.py --pkl backend/data/ResPlan.pkl --ids 14433 2373 --out samples --verify      (one file per plan)
  python tools/export_plans.py --pkl backend/data/ResPlan.pkl --split val --limit 30 --out samples --bundle   (one file with all of them)
  ```
  `--verify` reads every file back and checks that the geometry is identical. The stored NetworkX graph is not written; the solver builds its own from the layers.
- Without `ResPlan.pkl` the app still starts: use the upload buttons.

Solved plans are cached in `backend/cache/` (shift-click Solve to solve again). Delete that folder after changing anything in `cells/`.

## Where to change what (after the organizers answer)
| Question | Where |
|---|---|
| 1 which rooms are connected | `cells/cellP1_prepare.py` (`prepare`, link sets "helper" and "physical"; the page has a Link rule switch) |
| 2, 8 touching walls, clearance, tolerance | `cells/cellV1_validator.py` (`CFG["tol"]`), `cells/cellP1_prepare.py` (`P_CFG["margin_m"]`), `cells/cellS1_ring_solver.py` (`S_CFG["inset_m"]`; multi-hub: `S11_CFG["inset_m"]` in S11) |
| 3 sliver rooms | `cells/cellP1_prepare.py` (`min_area_m2`, placement tiers) |
| 4, 5 nested rooms, rooms in pieces | `cells/cellP1_prepare.py` (overlap rule, piece handling) |
| 6 hub and device distance | `cells/cellS1_ring_solver.py` (`S_CFG["gap_m"]`, `hub_min_m`) |
| 7 crossings at another ring's node | `cells/cellV1_validator.py` (`v_crossings`), `S_CFG["conservative"]` |
| 9 rooms over 100 m2 | `cells/cellS18_design_picture.py` (`solve_plan`): such plans are not solved yet |
| 10 augmented plans, scale | `cells/cellP1_prepare.py` (`aligned_share`, scale formula in `prepare`) |
| 11 how the hidden plans are run | `cells/cellS20_batch_runner.py` (notebook batch run) or this app |
| 12 JSON details | `cells/cellS19_json_export.py` (`plan_to_json`) |

## API
- `GET /api/status`, `GET /api/plans?split=&q=&limit=&offset=` (one page; adds J and hub count for solved plans), `GET /api/random?split=`
- `POST /api/upload` body: one plan file, a list of them, or `{"plans": [...]}` (dataset layout as JSON)
- `GET /api/plans/{id}` geometry (and the cached solution if there is one)
- `POST /api/plans/{id}/solve` body `{"seconds": 30, "link_set": "physical", "force": false}`
- `GET /api/plans/{id}/result.json` the slide-8 JSON of a solved plan

