# SC/ST Portal — Boundary & Village GeoJSON Files

Generated from the official Uttarakhand GIS shapefiles + your census 2011 data
(`UK-dist.xlsx`, `UK-rules.xlsx`) and village dataset (`villages_matched.json`).

## Files

| File | Features | Contents |
|---|---|---|
| `uttarakhand_state.geojson` | 1 | State outline, with total population / SC / ST / literacy stats in `properties` |
| `uttarakhand_districts.geojson` | 13 | District polygons, joined to census stats by `district_code` |
| `uttarakhand_subdistricts.geojson` | 111 | Sub-district (tehsil) polygons, joined to census stats where possible |
| `villages.geojson` | 2,969 | Village points — **rebuilt**: population/SC/ST/literacy stats come from `UK-dist.xlsx`'s official VILLAGE-level census records (16,793 total), location (lat/long) comes from `villages_matched.json` only where verified |
| `uttarakhand_district_hq.geojson` | 13 | District headquarters points (name + lat/long) |
| `uttarakhand_subdistrict_hq.geojson` | 111 | Sub-district headquarters points |
| `uttarakhand_major_towns.geojson` | 168 | Major town points, for optional map labels/context |

All geometry is in **WGS84 (EPSG:4326)** — the raw shapefiles were actually in a
custom Lambert Conformal Conic projection (meters), which I reprojected to
lat/long so they line up correctly with Leaflet/D3/your village coordinates.

## District & sub-district properties

Each district/sub-district feature carries:
`district_code`, `district_name` (or `subdistrict_code` / `subdistrict_name`),
`households`, `total_population`, `male_population`, `female_population`,
`sc_population`, `st_population`, `literate_population`,
`sc_percent`, `st_percent`, `literacy_percent`.

Sub-district features also carry:
- `subdistrict_local_code` — the shapefile's own code (unique per polygon, includes newer tehsil splits)
- `subdistrict_code` — the matching **2011 census** code (used to join with `villages.geojson`)
- `census_2011_data_available` — `false` for 10 sub-districts that didn't exist as separate units in 2011

## How villages.geojson was built (updated)

Rather than trusting `villages_matched.json` end-to-end (which had the ~20-25%
coordinate mismatch described below), the final village dataset is built as:

1. **Population/SC/ST/literacy data** — pulled from `UK-dist.xlsx`'s official
   VILLAGE-level rows (16,793 villages total across Uttarakhand), using the
   column layout documented in `UK-rules.xlsx`. This is the authoritative,
   government-coded source.
2. **Location (lat/long)** — pulled from `villages_matched.json` **only** where
   the village's `(district_code, subdistrict_code, village_code)` key has a
   match there.
3. **Verification** — each candidate location was checked with a point-in-
   polygon test against `uttarakhand_subdistricts.geojson` (with a ~1km
   tolerance). Villages whose coordinates didn't actually fall inside their
   labeled sub-district were dropped rather than kept with a wrong pin.

**Result: 2,969 villages** made it through with both correct census data and a
verified, trustworthy location — down from 16,793 total villages and 5,643 in
the original matched file. Breakdown:
- 12,700 villages had no location at all in `villages_matched.json` — left out
- 1,124 villages had a location that failed the polygon check — left out
- 2,969 villages had a verified, correct location — **kept, used for Page 3**

This is a smaller dataset than before, but every village in it will render
correctly inside its own sub-district on the map — which was the actual
requirement. If you want, the ~13,824 excluded villages could be added back
later via re-geocoding or manual correction, but that's a separate effort from
this planning phase.

## Other caveats

1. **10 newer tehsils have no direct 2011 census stats.** Uttarakhand created ~10
   new tehsils after 2011 (e.g. Doiwala, Rudrapur, Nandprayag, Lamgara...). Their
   shapefile polygons exist, but the 2011 census only has data for their old
   parent tehsil, so `census_2011_data_available: false` on those.

2. File sizes are on the larger side for web (`uttarakhand_subdistricts.geojson`
   is ~7MB) since they retain full shapefile precision. These can be simplified
   (e.g. via `mapshaper`) to cut size significantly with negligible visual loss —
   worth doing as a build step later, not necessary for planning.
