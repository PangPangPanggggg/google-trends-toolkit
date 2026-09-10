# Data engineering guide

**Maintainer:** Nitisart Srijunpho

**CV reference:** Experience → Bank of Thailand, Northeastern Region Office

This guide documents the implemented system. Commands below are run from the repository root.

## Layer boundaries and lineage

1. **Definitions:** `keywords.csv` identifies active keywords. `reference/keywords_tried.csv` records the wider screening history.
2. **Acquisition:** the Chrome extension exports full-window Google Trends CSVs. Queue state, authenticated browser state and `incoming/` are local staging assets.
3. **Validation and ingestion:** `collector/ingest.py` validates exports before replacing a canonical series. Invalid exports go to `incoming/review/`.
4. **Canonical archive:** `data/series/` stores one CSV per keyword × geography; `data/catalog.json` records collection metadata and availability.
5. **Presentation output:** `collector/build_site_data.py` derives `data.js`, including data-health metadata.
6. **Analytical output:** `analysis/` reads the archive and creates `derived/sa_pipeline_v3/`. Its manifest records source digest, output hashes, method and environment versions.

The presentation and analytical branches are separate consumers of the archive. An analytical rebuild does not replace raw series.

## Data contracts

| Dataset | Grain / key | Contract |
|---|---|---|
| Canonical CSV | Keyword ID × geography × month | Header `Month,Value`; unique, continuous monthly observations; finite values from 0 to 100 |
| Collection catalog | Keyword ID × geography | Collection period and status; confirmed no-data has evidence and is distinct from missing files |
| Raw dashboard | Selected keywords and geographies | The computed `ISAN` series records `support_n`, `support_total` and `support_geos`; its support may vary by keyword |
| Analytical series | Case × scope × month × stage | Long-format output follows [the analytical contract](../analysis/README.md); explicit T1/T2, SA, rebase and centred-MA3 stages |
| Analytical manifest and sidecars | Release / case × scope | Hashes and source digest, method choices, fallback reasons, diagnostics, coverage and provisional endpoint flags |

National raw history starts in January 2004; provincial raw history starts in January 2014. Ingestion requires continuity through the required completed month and removes an identified partial current month.

A refresh replaces the entire canonical series from a full-window export. Google Trends rescales each request; appending differently scaled windows would break comparability.

## Quality gates

- **Structure:** `python -X utf8 collector/audit.py --strict`
- **Generated presentation consistency:** `python -X utf8 collector/build_site_data.py --check`
- **Analytical lineage and schema:** `python -X utf8 -m analysis.build --audit`
- **Reproducibility:** the Windows X-13 job rebuilds in staging and byte-compares analytical output with the committed release.
- **Regression tests:** `python -X utf8 -m unittest discover -s tests -v`
- **Data-release freshness:** the CI freshness job runs when canonical/derived data or keyword definitions change. Documentation-only updates do not assert that the data has been refreshed.

The production update command runs the relevant checks together; see [operations](OPERATIONS.md). Individual read-only checks are useful for review but do not replace the full data-release process.

## Analytical semantics

The current analytical snapshot has 31 cases (23 individual keywords and eight families), or 62 case-scope series across `TH` and `REG_ISAN5`. Its provincial support requirements, rebasing, X-13/STL fallback rules and centred MA3 are defined in [analysis/README.md](../analysis/README.md).

The public keyword dashboard applies its own raw-composite logic, client-side rebasing and trailing MA3. Feeding the analytical output into that path unchanged would apply transformations twice.

## Failure, recovery and ownership

- Review rejected imports at their source before retrying; retain the last accepted archive until a replacement passes validation.
- CAPTCHA/login steps remain human checkpoints in the collection process.
- `WEAK`, all-zero, confirmed no-data and missing are different states and remain visible in quality reporting.
- A failed structural, analytical or required freshness gate blocks the associated release.
- Keep browser sessions, temporary exports, credentials and generated queues out of Git; keep code, definitions, accepted snapshots and analytical manifests versioned.
- Changes to methods must identify the affected outputs and pass their consistency checks.

Nitisart Srijunpho maintains the research tooling with AI coding assistance. Google Trends supplies the observations; interpretation and software maintenance are separate responsibilities.
