# Industrial-CO2-emissions---Nordic-Baltic-countries-Power-BI-
A Power BI dashboard answering a simple question: which facilities and industrial sectors emit the most CO₂ in the Nordic and Baltic countries, and how is this changing? It includes a focus on mineral products (cement, lime, ceramics).


## Repository content

```
nordic-industrial-emissions-powerbi/
├── nordic-industrial-emissions.pbix   # Power BI report
├── screenshots/
│   ├── overview.png
│   ├── mineral-products.png
│   └── data-quality.png
└── README.md
```

## Data source

European Environment Agency (2026), *Industrial Reporting under the Industrial Emissions Directive 2010/75/EU and E-PRTR Regulation (EC) No 166/2006*, version 16.0 (February 2026).
DOI: [10.2909/657ac3cb-affa-4295-a4a9-27b4f539adab](https://doi.org/10.2909/657ac3cb-affa-4295-a4a9-27b4f539adab) · Licence: CC-BY 4.0 · [European Industrial Emissions Portal](https://industry.eea.europa.eu/)

Files used: `F1_4_Air_Releases_Facilities` (facility-level data) and `F1_1_Air_Releases_National` (validation).

## Scope

| Dimension | Selection |
|---|---|
| Countries | Denmark, Sweden, Norway, Finland, Estonia, Latvia, Lithuania |
| Pollutant | Total CO₂ (the "CO₂ excluding biomass" variant was dropped to avoid double counting) |
| Period | 2015–2024 |
| Unit | Tonnes (source data in kg, divided by 1,000) |

## Data preparation (Power Query)

1. **Base query**: import, column types, filters (countries, pollutant, years), conversion kg → t, cleaning of facility names (embedded line breaks removed with *Clean* and *Trim*).
2. **Outlier exclusion**: a named step (`Excluded outliers`) removes two implausible values (see Data quality).
3. **Model tables** built as *references* to the Base query, so every correction is made in one place:
   - `Emissions` (fact table): facility ID, year, emissions (t). Grain: one facility × one year.
   - `Sites` (dimension): one row per facility (duplicates removed on the ID, keeping the most recent record), with name, city, country, sector, Annex I activity, and validated coordinates (`Lat_map`, `Lon_map`).
   - `Years` (dimension): 2015–2024.
4. **Activity labels**: Annex I codes for the mineral industry (3(a)–3(g)) mapped to readable names (cement & lime, ceramics, mineral fibres…).

## Data model

Star schema, single-direction, many-to-one relationships:

```
   Sites (1) ──────┐
                   ▼ (*)
                Emissions
                   ▲ (*)
   Years (1) ──────┘
```

All measures are stored in a dedicated `_Measures` table.

## DAX measures

```dax
Emissions (t) = SUM ( Emissions[Releases_t] )

Emissions PY =
VAR y = SELECTEDVALUE ( Years[Year] )
RETURN CALCULATE ( [Emissions (t)], Years[Year] = y - 1 )

YoY % =
IF (
    ISBLANK ( [Emissions (t)] ) || ISBLANK ( [Emissions PY] ),
    BLANK (),
    DIVIDE ( [Emissions (t)] - [Emissions PY], [Emissions PY] )
)

Sector share =
DIVIDE ( [Emissions (t)], CALCULATE ( [Emissions (t)], ALL ( Sites[EPRTR_SectorName] ) ) )

Site count = DISTINCTCOUNT ( Emissions[FacilityInspireId] )

-- Latest year in the selection, identical for every row (ignores facility/country filters)
Ref year = CALCULATE ( MAX ( Emissions[reportingYear] ), REMOVEFILTERS ( Sites ) )

Emissions ref year =
VAR a = [Ref year]
RETURN CALCULATE ( [Emissions (t)], Years[Year] = a )

YoY ref year =
VAR a = [Ref year]
VAR actuel = CALCULATE ( [Emissions (t)], Years[Year] = a )
VAR precedent = CALCULATE ( [Emissions (t)], Years[Year] = a - 1 )
RETURN DIVIDE ( actuel - precedent, precedent )

Sites ref year =
VAR a = [Ref year]
RETURN CALCULATE ( [Site count], Years[Year] = a )

-- Share within the activities selected on the page
Activity share =
DIVIDE ( [Emissions (t)], CALCULATE ( [Emissions (t)], ALLSELECTED ( Sites[Activity] ) ) )

-- Share of all industrial CO₂ (removes sector and activity filters)
Share of total industrial CO2 =
DIVIDE (
    [Emissions (t)],
    CALCULATE ( [Emissions (t)], REMOVEFILTERS ( Sites[EPRTR_SectorName], Sites[Activity] ) )
)
```

Design choices worth noting:
- `YoY %` returns blank when a year is missing, instead of misleading values (−100%, +17,000%).
- `Ref year` uses `REMOVEFILTERS(Sites)`, otherwise each facility in the top 10 would use its own last reporting year (e.g. 2017 for Norwegian sites).

## Report pages

1. **Overview**: KPIs for the latest year, emissions by country, share by sector, facilities map (bubble size = emissions), top 10 facilities; slicers for country, sector and year.
2. **Mineral products**: the same view filtered on cement & lime, ceramics and mineral fibres, with breakdown by activity and the sector's share of total industrial CO₂.
3. **Data quality**: reporting facilities per country and year, all checks and limitations.

## Key findings (2024)

- **101 Mt CO₂** reported by **241 facilities**, **−6.1%** vs 2023.
- **Energy (~42%)** and **paper & wood (~33%)** dominate: the weight of the pulp and paper industry is a distinctive Nordic feature.
- **Mineral products account for ~6%** of industrial CO₂ (5.9%), and **99% of that is cement and lime**.
- **Two cement plants**, Aalborg Portland (Denmark, 1.44 Mt) and Slite (Sweden, 1.39 Mt), emit more than half of the sector's total.
- Mineral products emissions fell **14.5%** in 2024. The long-term decline is partly a coverage effect (Norway and Lithuania drop out of the data); on a constant scope (DK, SE, FI, EE, LV) the decline is [to complete].

## Data quality checks

| Check | Finding | Action |
|---|---|---|
| Facilities per country and year | Norway: no data after 2017 (stopped reporting to the EU registry). Lithuania: incomplete 2023–2024. Denmark: reporting scope expanded in 2018 (23 → 66 facilities) | Documented on the report; trends interpreted per country |
| Iceland | Erratic, incomplete data (e.g. 870 t in 2021 for a country with large aluminium smelters) | Excluded |
| Plausibility of trends | Estonia 2017 at 25 Mt (≈ the country's total GHG emissions). Two implausible single-year values: Gren Anne CHP (7.66 Mt vs ~260 kt usual) and Repo Vabrikud (1.48 Mt, single year) | Excluded in a named, documented step |
| Coordinates | Odense facility with latitude 5.33 (instead of ~55) plotted off Africa | Validation rule: out-of-region coordinates excluded from the map, kept in totals |
| Text fields | Facility names with embedded line breaks from the source CSV | Cleaned (*Clean*, *Trim*) |
| Sector | 1 facility with a blank sector | Hidden from sector visuals |
| Classification | The "Mineral industry" sector includes offshore oil & gas platforms (activity 3(a)), ~30% of the sector in 2015–2017 | Excluded from the Mineral products page |
| Duplicates | No facility reported twice for the same year | — |

## Validation against national totals

| Country & year | Dashboard | EEA national table | Gap |
|---|---|---|---|
| Sweden 2022 | 48,594,000 t | 48,594,000 t | 0% |

The national table is built from the same facility reports. This check confirms that the import is complete, but it cannot detect errors in the source itself, which is why the plausibility checks above were needed.

## Limitations

- The register only covers facilities above **100,000 t CO₂/year**: it is not a national inventory. Smaller plants, including most **glass and mineral-wool plants**, are not visible.
- Coverage breaks (Norway after 2017, Lithuania 2023–2024, Denmark 2018) make regional totals unsuitable for long-term trend analysis without a constant-scope check.
- Emissions only: no production volumes (not yet published by the EEA), so no carbon-intensity analysis.

## Next step: applying the approach to plant data

The same approach (cleaning, validation, plausibility checks, a clear star model) could be applied to a manufacturer's own **energy and production data per plant and per line**, to identify where AI or automation would bring the largest energy and emission savings.
