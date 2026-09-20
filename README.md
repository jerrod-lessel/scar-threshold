# Scar Threshold

**How hard it has to rain before a burned canyon lets go.**

### [Open the map](https://scar-threshold.pages.dev/)

[![Scar Threshold, debris flow hazard for the 2024 Bridge Fire](web/screenshot.png)](https://scar-threshold.pages.dev/)

---

After a wildfire, a short burst of rain can turn a burned hillside into a fast slurry of mud and rock. It usually happens in the first winter, and often from a storm that would be unremarkable on unburned ground.

This pipeline takes satellite imagery, elevation and soil data for a burn scar, splits the terrain into drainage basins, and reports for each one the rainfall intensity that gives it a 50% chance of producing a debris flow. It's been run on four fires and 756 basins, all from public data, running end to end in a browser from a single configuration block.

Three of those fires have a published field survey to check the answer against. The fourth was run while it was still burning, before any field data existed, which is a different exercise and is kept separate throughout.

The model is not the contribution. USGS publishes both the equations and a reference implementation. What this project offers is the ingest, the validation, and the delivery, plus an uncertainty analysis that operational assessments do not typically publish.

## Four results

**The implementation is exact.** Given identical inputs, this project's model matches `pfdf.models.staley2017`, the official USGS package, bit for bit. The check runs on every fire and has returned a maximum difference of zero on all four, across 4,536 forward evaluations and 756 inverse solves. It also reproduces the published USGS assessment for the Bridge Fire to floating point precision on all 703 pieces of burned ground. The comparison was then deliberately broken to confirm it was capable of failing.

**The disagreement with USGS is traceable.** On the Bridge Fire, the two assessments agree on where the hazard is: 99.8% of the ground USGS rates high is rated high here. They differ on magnitude by 7.7 mm/hr (area-weighted), and that gap decomposes into terrain, soil and burn severity, with each contribution measured. Delineation scale was tested across a 25-fold range and ruled out.

**It over-warns, and does not meaningfully under-warn.** Rerunning with the field-validated BAER soil burn severity map instead of satellite severity pushes basins outside the six-run parameter envelope in one direction only: toward less hazard. That happens in 87 of 237 basins on Bridge, 24 of 171 on Line and 40 of 239 on Borel. The only basin that moves the other way, on Line, does so by 0.4 mm/hr at a threshold of 66 mm/hr. On all three fires, every basin the pipeline rates never-High is also not High when rerun with the BAER map: 296 basins, no exceptions.

**One notebook, any fire, including one that is still burning.** Everything fire-specific lives in one configuration block: the name, rough centre and acreage from the incident page, and two date windows. The notebook suggests the satellite scenes, runs through to the web map, and checks itself against earlier validated runs. Rerunning Bridge and Line through it reproduces the originals basin for basin, to floating point precision.

## The fires (so far...)

| | Bridge | Line | Borel |
|---|---|---|---|
| Ignited | 8 September 2024 | 5 September 2024 | 24 July 2024 |
| Size | 56,281 acres | 43,978 acres | 59,288 acres |
| Where | San Gabriel Mountains | San Bernardino Mountains | southern Sierra foothills |
| Basins | 237 | 171 | 239 |
| Burn area 23 degrees or steeper | 83.0% | 71.0% | 51.7% |
| Moderate or high severity, satellite vs BAER | 76.5% vs 58% | 85% vs 71% | 70.6% vs 46.6% |
| Median basin threshold | **16.4 mm/hr** | **19.0 mm/hr** | **26.3 mm/hr** |

The median threshold is the 15-minute rainfall that gives the typical basin a coin-flip chance of a debris flow. On Bridge and Line, neither number is a remarkable storm in southern California.

Borel is the outlier, and the reason is terrain. Barely half its burn area is steep enough to matter to the model, against five sixths on Bridge, so its basins need substantially more rain to reach the same likelihood. It was added as a third fire specifically to break the pattern the first two share: a July fire in the southern Sierra foothills, through oak woodland and grass rather than Transverse Range chaparral, and matched to Bridge in size so that basin counts stay comparable.

Satellite severity rates more ground badly burned than the BAER field teams do on all three fires, which is why the pipeline runs conservative. The size of that gap does not line up with the size of the over-warning, though: Borel has the largest severity gap and a middling over-warning share. Three fires establish the direction, not the magnitude.

## The fourth fire, and why it's different

The three above are a validation study: each has a published BAER soil burn severity map, and the question they answer is whether the pipeline is right.

The **Plaskett Fire**, 29,993 acres in the Santa Lucia Mountains south of Big Sur, asks something else. It was run on 20 September 2026 at 97% containment, with no BAER assessment in existence and none likely for weeks. That's the window where a rainfall threshold could actually change a decision, and a tool that needs the field survey first arrives after the decisions have been made.

109 basins, median 24.3 mm/hr, median envelope spread 5.21 mm/hr. **None of it is validated.** The parameter envelope is the uncertainty of the estimate, not its accuracy. All the track record says is that on the three fires with an anchor this pipeline over-warned and didn't meaningfully under-warn, so these numbers are more likely cautious than optimistic. The map labels the fire as unchecked and as still burning, and the manifest carries both flags.

Running it exposed three failure modes the inland retrospective fires couldn't:

- **CAL FIRE FRAP has no current fires.** It's a historic layer on a lag, and it returned three older fires named Plaskett instead. NIFC WFIGS year-to-date is now queried for current-season fires, and a name match in the wrong year is rejected rather than accepted.
- **The Pacific Ocean broke terrain conditioning.** Flat water made the fill step invent drainage across 28.7% of the DEM window, and ocean inside a basin diluted both T and F, reporting four coastal basins above Highway 1 as safer than they are. Basins are now clipped to land first. This is the only systematic under-warning found anywhere in the project.
- **`basin_attributes` assumed basin ids with no gaps.** Clipping to land leaves gaps. It crashed rather than returning wrong numbers, which is the good version of that bug.

Full detail in [METHODS.md](METHODS.md).

## What is here

```
src/debrisflow/                  the pipeline: severity, terrain, basins, soils, model, sensitivity
tests/                           184 tests
05_generalized_pipeline.ipynb    any fire, perimeter to web map, from one config block
00 to 04 .ipynb                  the original Bridge notebooks: ingest, USGS comparison, sensitivity
web/                             the map
```

Data sources: Sentinel-2 via the Planetary Computer, USGS 3DEP 10 m elevation, USDA STATSGO soils, CAL FIRE FRAP and NIFC WFIGS perimeters, USDA Forest Service BAER soil burn severity. No API keys, no paid services.

## Run it

```bash
pip install -r requirements.txt
python -m pytest -q          # 184 passed
```

Or open `00_model_driver.ipynb` in Colab, which clones this repository and runs everything. To run a fire, open `05_generalized_pipeline.ipynb`, fill in the configuration block at the top, and run it top to bottom (with a quick loop back after cell 4 to select the desired pre-/post-burn dates). It can take a bit of time, since it reads satellite imagery and queries four external services.

## More

- **[METHODS.md](METHODS.md)** for the full method, every design decision, the validation in detail and the known limitations
- Staley, D.M. and others (2017), *Prediction of spatially explicit rainfall intensity-duration thresholds for post-fire debris-flow generation in the western United States*, Geomorphology 278, 149-162
- [USGS `pfdf`](https://code.usgs.gov/ghsc/lhp/pfdf), the authoritative implementation. This project's model is an independently tested reimplementation for cross-checking, not a replacement.

---

**Not an operational warning product.** This is a portfolio project. It reproduces a published USGS model from public data and has not been reviewed or endorsed by USGS, CAL FIRE, the US Forest Service, or any other agency. Nothing here predicts whether a particular canyon will produce a debris flow, and none of it should be used for evacuation, access, or any other safety decision. A basin shown as not high hazard has not been judged safe: it means the model did not rate it high under any of six parameter choices. For official post-fire hazard information see the [USGS Landslide Hazards Program](https://landslides.usgs.gov/hazards/postfire_debrisflow/) and your county emergency management agency.
