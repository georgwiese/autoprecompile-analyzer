# APC Effectiveness Analyzer

Single-page web app for visualizing Automatic Precompile (APC) candidate effectiveness in zkVM systems. Port of `plot_effectiveness.py` from powdr.

## Project Structure
```
index.html          # SPA with embedded JS/CSS (~2000 lines)
CLAUDE.md           # This file
```

## Data Format

Two formats supported (auto-detected):

**New format** (object with `apcs` and `labels`):
```json
{
  "apcs": [{
    "execution_frequency": 50000,
    "original_block": { "start_pc": 12345, "statements": ["instr1", "instr2"] },
    "stats": {
      "before": { "main_columns": 100, "constraints": 200, "bus_interactions": 50 },
      "after": { "main_columns": 50, "constraints": 100, "bus_interactions": 25 }
    },
    "width_before": 100,
    "value": 5000,
    "cost_before": 1000.0,
    "cost_after": 500.0,
    "apc_candidate_file": "path/to/candidate.json"
  }],
  "labels": { "2099200": ["memset"], "2099448": ["memcpy"] }
}
```

**Old format**: Same structure as `apcs` array above, without wrapper object. Labels will be empty.

## Testing

Start server:
```bash
python3 -m http.server 8000 &
```

Test URL with real data (~11,300 APCs):
```
http://localhost:8000/?data=https%3A%2F%2Fgithub.com%2Fpowdr-labs%2Fbench-results%2Fblob%2Fgh-pages%2Fresults%2F2026-01-27-0453%2Freth%2Fapc_candidates.json
```

Verify:
- Data loads (GitHub URLs auto-convert to raw)
- Bar chart shows ~3.28x mean effectiveness
- Value-cost plot reaches ~80% savings at 1000 APCs
- Labels table expands with function names
- Block selection syncs across all views

Cache-bust: append `&_t=1` to URL.

## URL Parameters

```
?data=<url>           # Data source (required to load data)
&plot=value-cost      # Show value-cost plot (omit for default bar chart)
&block=0x2008f8       # Select block by PC address (hex)
```

Example - jump directly to value-cost plot with a block selected:
```
http://localhost:8000/?data=<url>&plot=value-cost&block=0x200af0
```

URL updates automatically as you interact with the app, enabling easy sharing of specific views.

## Development Notes

**D3.js chart redraw**: Charts are fully recreated on metric switch. Ensure `.remove()` is called on exit selections to prevent memory leaks.

**State persistence**: `selectedBlock` must survive metric changes. Check selection still exists in new processed data.

**GitHub URL conversion**: `loadFromUrl()` has regex converting blob URLs to raw URLs. Brittle - test after GitHub URL format changes.

**Grouping threshold**: Blocks <0.1% of total cells grouped as "Other". Hardcoded in `createChart()`.

**Weighted mean**: `sum(effectiveness * traceCells) / sum(traceCells)` - weights by trace cells, not block count.

### Common Errors
- **CORS**: GitHub blob URLs must convert to raw URLs
- **D3 selections**: Use enter/update/exit patterns; don't forget `.remove()`
- **Event handlers**: Remove old handlers when recreating charts
- **Test with full dataset**: ~11K items, not small test data
