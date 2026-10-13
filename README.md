# Indonesia GeoJSON for Metabase Region Maps

Ready-to-use GeoJSON boundaries of Indonesia's **38 provinces** and **513 regencies/cities (kabupaten/kota)**, keyed by the official Kemendagri region code. They're built to be used as **custom region maps in Metabase**, but they work in any tool that reads GeoJSON (Leaflet, QGIS, Kepler.gl, Superset, Power BI's shape map, etc.).

> **Bahasa Indonesia:** Batas wilayah provinsi dan kabupaten/kota Indonesia dalam format GeoJSON, menggunakan kode wilayah Kemendagri (misal `32.73` = Kota Bandung). Siap dipakai sebagai *custom region map* di Metabase.

---

## Files

| File | Features | Size | Key property | Example |
|---|---|---|---|---|
| `indonesia_regencies.geojson` | 513 kabupaten/kota | ~1.6 MB | `code` | `"32.73"` → Kota Bandung |
| `indonesia_provinces.geojson` | 38 provinces | ~0.7 MB | `code` | `"32"` → Jawa Barat |
| `regency_coordinates.csv` | 514 kabupaten/kota | ~15 KB | `code` | `"32.73", -6.910656, 107.60987` |

Each GeoJSON feature has two properties:

```json
{ "code": "32.73", "name": "Kota Bandung" }
```

`regency_coordinates.csv` contains one point per regency (the location of its capital town, **not** the geographic centroid), for Metabase **Pin maps**.

---

## Use it in Metabase

### 1. Get the raw URL

Metabase loads custom maps **from a URL**, not from a file upload. If you're using this repo directly:

1. Click `indonesia_regencies.geojson` (or the provinces file) in this repo.
2. Click **Raw** (or **View raw**).
3. Copy the URL. It must start with `https://raw.githubusercontent.com/...`

A `github.com/.../blob/...` URL is a web page, not the file, so it won't work.

### 2. Add the map

1. **Admin settings → Settings → Maps → Add a map**
2. Fill in:
   - **Name:** `Indonesia Kabupaten/Kota`
   - **URL:** the raw URL, then click **Load**
   - **Region's identifier:** `code`
   - **Region's display name:** `name`
3. Click **Add map**.

### 3. Build the visualization

Your query needs:
- a **region code column** matching the GeoJSON `code` (text such as `'32.73'`, or `'32'` for provinces)
- a **numeric metric** to shade by

```sql
SELECT r.code, r.name, COUNT(o.id) AS total_orders
FROM regencies r
LEFT JOIN orders o ON o.regency_code = r.code
GROUP BY r.code, r.name
```

Then choose **Visualization → Map → Region map**, select **Indonesia Kabupaten/Kota**, set **Region field** to `code`, and set **Metric** to your number.

---

## Common pitfalls

**Match on `code`, not `name`.** Names vary in spelling ("Kota Bau Bau" vs "Kota Baubau"), and some databases store them with trailing spaces. A mismatch leaves the region blank without any error, while codes match exactly.

**Store codes as text, not numbers.** If your `code` column is FLOAT or DECIMAL, `32.10` becomes `32.1` and won't match `"32.10"`. Use `VARCHAR`. The same applies when you open the CSV in Excel or Google Sheets: they convert `32.10` to `32.1`. Import the column as **Text**.

**Province codes may need a cast.** If `province_code` is an integer, cast it to text if regions show up blank:

```sql
CAST(province_code AS CHAR)   -- MySQL
province_code::text            -- PostgreSQL
```

**The Metabase server must reach GitHub.** Metabase's server downloads the file, not your browser. If Metabase runs in a network without internet access, **Load** will time out. Host the file on an internal server or bucket instead.

**Pin maps need numeric lat/long.** If you use `regency_coordinates.csv` for a Pin map, keep latitude and longitude as signed decimal numbers (south and west negative). Text values like `6.91°S` won't plot. In Metabase, set the columns' semantic type to **Latitude** and **Longitude** (Admin → Table Metadata, or a model's **Edit metadata**).

---

## Known limitations

- **Kota Banjar (`32.79`) has no boundary** in the source data, so it appears blank on region maps. It *is* included in the CSV.
- **The boundaries are simplified** (about 500 m tolerance for regencies, about 1 km for provinces) to keep the files light for browser rendering. That's fine for choropleths, but not for precise spatial analysis. Very small islands may lose detail.
- **Coordinates in the CSV mark the capital town, not the centroid.** For large regencies (e.g. Merauke, Kapuas Hulu), the capital can be 100+ km from the geographic centre.
- **Administrative boundaries change.** New regencies are created periodically. This data reflects Kepmendagri No. 300.2.2-2430 Tahun 2025 (see Source).

## Data corrections applied

| Code | Regency | Issue in source | Fix |
|---|---|---|---|
| `74.07` | Kabupaten Wakatobi | Longitude stored as `23.5389`, which places it in Congo | Corrected to `123.538901` |

**Validation:** every CSV point was checked to lie inside Indonesia's bounding box, and checked against its own regency polygon. The 11 points that fall slightly outside their (simplified) polygon were verified manually against the known capital town.

---

## Source & license

The boundaries, coordinates and codes are derived from **[cahyadsn/wilayah](https://github.com/cahyadsn/wilayah)** (`db/wilayah_level_1_2.sql`), which follows **Kepmendagri No. 300.2.2-2430 Tahun 2025**.

Changes made here:
- Converted from `[lat, lng]` arrays in SQL to standard GeoJSON `[lng, lat]`
- Repaired invalid rings and simplified the geometry
- Rounded coordinates to 4 decimals (about 11 m)
- Fixed the Wakatobi longitude

The original data is © 2017–2025 Cahya DSN, released under the **MIT License**. This derived dataset is released under the same license; see [`LICENSE`](LICENSE). Please keep the attribution if you redistribute.

**No warranty.** Boundaries are for visualization only and are not an official legal reference. For authoritative boundaries, consult [Badan Informasi Geospasial](https://tanahair.indonesia.go.id).

## Contributing

Found a wrong boundary, a missing region, or an outdated code? Open an issue with the region `code`, what's wrong, and a source if you have one.
