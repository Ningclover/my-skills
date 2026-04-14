---
name: bee_url_check
description: Given a BEE upload log file (beefile.txt), extract all URLs, check each one for the presence of apa0, apa1, apa2, and apa3, and write valid URLs (those containing all four) to beefile_allapa.txt in the same folder.
---

# BEE APA Filter

## Context

BEE (Browser Event Explorer) upload log files contain lines of the form:
```
https://www.phy.bnl.gov/twister/bee/set/<uuid>/event/list/
```
interspersed with Django authentication and upload status lines. Each URL points to a BEE event set. A "valid" set is one that contains all four APA views: apa0, apa1, apa2, and apa3.

The site uses a self-signed certificate, so SSL verification must be bypassed (`curl -sk`).

---

## Procedure

### Step 1 — Get the input file path

Ask the user for the path to the beefile if not already provided. The file is typically named `beefile.txt` or `beefile_tot.txt`.

### Step 2 — Extract URLs

Extract all lines matching the BEE URL pattern:
```
https://www.phy.bnl.gov/twister/bee/set/<uuid>/event/list/
```

Use grep:
```bash
grep -oE 'https://www\.phy\.bnl\.gov/twister/bee/set/[a-f0-9-]+/event/list/' <input_file>
```

### Step 3 — Check each URL for all four APAs in parallel

For each URL, fetch the page and check whether it contains apa0, apa1, apa2, and apa3. Run all checks in parallel using bash background jobs:

```bash
urls=( ... )  # array of extracted URLs

for url in "${urls[@]}"; do
  (
    result=$(curl -sk "$url" | grep -oE "apa[0-3]" | sort -u | tr '\n' ' ')
    echo "$url | $result"
  ) &
done
wait
```

A URL is **valid** if the result contains all four: `apa0 apa1 apa2 apa3`.

### Step 4 — Write valid URLs to output file

Write all valid URLs (one per line) to `beefile_allapa.txt` in the **same directory** as the input file.

```bash
output_dir=$(dirname <input_file>)
output_file="$output_dir/beefile_allapa.txt"
```

### Step 5 — Report results

Tell the user:
- Total URLs checked
- How many passed (all 4 APAs present)
- How many failed
- The path to the output file

---

## Notes

- The BEE server uses a self-signed TLS certificate — always use `curl -sk` (skip SSL verification).
- Running all curl requests in parallel (`&` + `wait`) is important for speed; there can be 50+ URLs per file.
- The output file is always named `beefile_allapa.txt` and placed next to the input file.
- A URL with an empty APA result (page returned nothing) is treated as invalid.
