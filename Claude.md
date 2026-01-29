# APC Effectiveness Analyzer - Project Documentation

## Overview
This is a web-based visualization tool for analyzing the effectiveness of Automatic Precompile (APC) candidates in zkVM (Zero-Knowledge Virtual Machine) environments. The tool helps evaluate the performance impact of generating specialized precompiles for each basic block in a RISC-V program.

### Context
In zkVM systems, precompiles can significantly reduce the computational cost of proving program execution. This analyzer visualizes how effective each automatically generated precompile is at reducing various computational metrics. The tool is a web-based port of the original Python script `plot_effectiveness.py` from the powdr project.

## Project Structure
```
autoprecomile-analyzer/
├── index.html          # Single-page application with embedded JS/CSS
└── Claude.md          # This documentation file
```

## Key Features

### 1. Data Input Methods
- **File Upload**: Drag-and-drop or click to upload JSON files
- **URL Loading**: Paste URLs directly in the input field
  - Direct JSON URLs (e.g., `https://example.com/data.json`)
  - GitHub file links (automatically converted to raw URLs)
  - GitHub Gist links
- **URL Parameters**: Load data directly via GET parameter
  - `?url=<json-url>` or `?data=<json-url>`
  - Browser URL automatically updates when loading data via input field
  - Enables easy sharing of visualizations
- **Format**: Expects JSON array of APC candidate objects
- **Visual source indicator**: When data is loaded from a URL, the top navbar shows a monospace, clickable `Data: <link>` (HTTPS stripped, truncated with head/tail) so you can see exactly which source is being visualized.

### 2. Effectiveness Metrics
The tool calculates effectiveness as a ratio (before/after) for zkVM-specific metrics:
- **Cost**: Overall computational cost reduction in the zkVM prover
- **Main Columns**: Reduction in main column usage (polynomial commitments in the proof system)
- **Constraints**: Reduction in constraint count (arithmetic constraints that must be satisfied)
- **Bus Interactions**: Reduction in bus interaction overhead (communication between different parts of the zkVM)

### 3. Interactive Visualizations

#### Bar Chart
- X-axis: Cumulative instruction trace cells (software version)
- Y-axis: Effectiveness ratio
- Color coding: Log scale based on instruction count (green=few, red=many)
- Small blocks (<0.1% of total cells) grouped as "Other"
- Red dashed line shows weighted mean effectiveness

#### Value–Cost Plot (saved prover cost vs added verifier cost)
- Plots cumulative saved prover cost against added verifier cost (log scale), sorted by density.
- Hover shows guide lines and labels for:
  - Saved prover cost with percentage and implied reduction factor: `Saved prover cost: X (Y%, Zx reduction)` where `Z = 100 / (100 − Y)`.
  - Accelerated prover cost with percentage of software cost.
- Point click/hover stays in sync with the bar chart selection.

#### Code Panel
- Displays actual code statements for selected basic blocks
- Shows PC address and instruction count
- Updates on bar selection

### 4. User Interactions
- **Hover**: Highlights corresponding elements across all views
- **Click**: Selects a block and displays its code
- **Title Click**: Returns to upload screen
- **Metric Switch**: Dynamically recalculates all visualizations

## Data Format

The analyzer supports two JSON formats for backwards compatibility:

### New Format (with labels)

```json
{
  "apcs": [
    {
      "execution_frequency": 50000,      // How often this block is executed (usize)
      "original_block": {
        "start_pc": 12345,              // Program counter address in RISC-V program (u64)
        "statements": ["instruction1", "instruction2", ...]  // RISC-V assembly instructions (Vec<String>)
      },
      "stats": {
        "before": {                      // AirStats metrics without precompile
          "main_columns": 100,           // Number of main columns (usize)
          "constraints": 200,            // Number of polynomial constraints (usize)
          "bus_interactions": 50         // Number of bus interactions (usize)
        },
        "after": {                       // AirStats metrics with precompile
          "main_columns": 50,
          "constraints": 100,
          "bus_interactions": 25
        }
      },
      "width_before": 100,               // Width before optimisation, used for software version cells (usize)
      "value": 5000,                     // Value used in ranking of candidates (usize)
      "cost_before": 1000.0,             // Cost before optimisation (f64)
      "cost_after": 500.0,               // Cost after optimization (f64)
      "apc_candidate_file": "path/to/candidate.json"  // Path to the APC candidate file (String)
    }
  ],
  "labels": {
    "2099200": ["memset"],
    "2099448": ["memcpy"],
    "2100512": ["<&mut openvm::serde::deserializer::Deserializer<R> as serde::de::Deserializer>::deserialize_bytes::h2f21bf9a8767c624"],
    "2101092": ["<&mut openvm::serde::deserializer::Deserializer<R> as serde::de::Deserializer>::deserialize_struct::h0a34205bb4e64527"]
  }
}
```

### Old Format (backwards compatible)

```json
[
  {
    "execution_frequency": 50000,
    "original_block": {
      "start_pc": 12345,
      "statements": ["instruction1", "instruction2", ...]
    },
    "stats": {
      "before": {
        "main_columns": 100,
        "constraints": 200,
        "bus_interactions": 50
      },
      "after": {
        "main_columns": 50,
        "constraints": 100,
        "bus_interactions": 25
      }
    },
    "width_before": 100,
    "value": 5000,
    "cost_before": 1000.0,
    "cost_after": 500.0,
    "apc_candidate_file": "path/to/candidate.json"
  }
]
```

The analyzer automatically detects the format. If the JSON is an object with `apcs` and `labels` keys, it uses the new format. Otherwise, it treats the JSON as the old format (array of APCs directly) with an empty labels dictionary.

### Field Definitions

#### APC Candidates (from `ApcCandidateJsonExport` struct)

- **execution_frequency**: Number of times this basic block is executed during program run
- **original_block**: The `BasicBlock` containing:
  - **start_pc**: The program counter of the first instruction in this block
  - **statements**: Pretty-printed assembly instructions as strings
- **stats**: `EvaluationResult` containing before/after optimization metrics:
  - **before**: `AirStats` for software execution (sum of all AIRs that would be involved)
  - **after**: `AirStats` for the APC implementation
- **width_before**: Number of trace rows without precompile (used for calculating software version cells)
- **value**: Computed value used in the fractional knapsack algorithm for ranking candidates
- **cost_before**: zkVM proving cost without precompile
- **cost_after**: zkVM proving cost with precompile
- **apc_candidate_file**: Filesystem path to the detailed APC candidate file

#### Labels

The `labels` field is a dictionary mapping PC addresses (as strings) to arrays of label strings:
- **Key**: PC address in decimal format (as string, e.g., "2099200")
- **Value**: Array of label strings associated with that PC address
  - Labels typically represent function names or demangled Rust symbols
  - Multiple labels can be associated with a single PC address
  - Examples: `["memset"]`, `["memcpy"]`, or mangled Rust function names

Labels are currently stored but not yet displayed in the visualization. Future enhancements will integrate labels into the code panel and tooltips.

## Technical Implementation

### Libraries Used
- **Bootstrap 5.3.0**: UI framework for responsive design
- **D3.js v7**: Data visualization and chart rendering

### Key Functions

#### Data Processing
- `calculateEffectiveness()`: Computes effectiveness ratios based on selected metric
- `processData()`: Transforms raw data for visualization
- `formatCellCount()`: Human-readable formatting (K/M/B suffixes)

#### Visualization
- `createChart()`: Main chart rendering with D3.js
- `highlightElements()`: Synchronized hover effects
- `selectBlock()`: Handle block selection and code display
- `showCode()`: Display code for selected block
- `clearCode()`: Reset code panel

#### File Handling
- Drag-and-drop support  
- File validation (JSON only)
- Error handling for malformed data
- URL loading with automatic GitHub URL conversion
- URL parameter persistence in browser address bar
- Auto-load from URL parameters on page load

### Performance Optimizations
- Groups small blocks (<0.1% threshold) to reduce visual clutter
- Uses log scale for color mapping to handle wide instruction count ranges
- Efficient D3.js updates with proper enter/exit patterns
- Maintains selection state across metric changes

## Styling

### Color Scheme
- Primary: Bootstrap primary blue (#0d6efd)
- Highlight: Yellow (#ffc10755 for hover, #ffc107 for selection)
- Background: Light gray (#f8f9fa)
- Chart colors: Red-Yellow-Green gradient (log scale)
- Grid lines: Light gray (#e0e0e0)

### Responsive Design
- Bootstrap grid system for layout
- Tooltip positioning follows cursor
- Full-width chart display

## Usage Instructions

### Loading Data
1. **File Upload**: Drag and drop or click to select JSON file
2. **URL Input**: Paste a URL and click "Load from URL" (or press Enter)
3. **Direct Link**: Access with `?data=<url>` or `?url=<url>` parameter

### Interacting with Visualization
1. **Select Metric**: Choose effectiveness type from dropdown (Cost, Main Columns, Constraints, Bus Interactions)
2. **Explore Data**:
   - **Hover** over bars to see details in tooltip
   - **Click** to select and view assembly code
   - **Red dashed line** shows weighted mean effectiveness
   - **Color gradient** indicates instruction count (green=few, red=many)
3. **Navigation**:
   - Click title to return to upload screen
   - Browser URL updates automatically for easy sharing

## Browser Compatibility
Modern browsers with ES6+ support required for:
- Arrow functions
- Template literals
- Array methods (map, filter, reduce)
- D3.js v7 requirements

## Development Notes

### Adding New Metrics
To add a new effectiveness metric:
1. Add option to effectiveness type select dropdown
2. Update `calculateEffectiveness()` function with new case
3. Ensure data structure includes required fields

### Customizing Visualizations
- Chart dimensions: Modify margin, width, and height variables
- Color scales: Adjust interpolator or domain for instruction color mapping
- Grouping threshold: Change 0.1% threshold for "Other" grouping
- Grid appearance: Modify grid line styles and opacity

### State Management
- `currentData`: Stores loaded APC candidates array
- `currentLabels`: Stores PC-to-labels mapping (empty object if using old format)
- `selectedBlock`: Tracks currently selected block
- Selection persists across metric changes when block still exists

### Error Handling
- JSON parsing errors caught and displayed to user
- File type validation (only .json accepted)
- Graceful handling of missing data fields

## Relationship to Original Python Script

This web application is a port of `plot_effectiveness.py` from the powdr project, with the following enhancements:
- **Interactive visualization** instead of static matplotlib plots
- **Real-time metric switching** without regenerating plots
- **Synchronized hover/selection** across chart and code view
- **Code display** for selected basic blocks
- **URL loading** support for remote data files
- **No installation required** - runs entirely in the browser

The core logic (effectiveness calculation, data processing, grouping threshold) remains identical to ensure consistency with the original analysis tool.

## zkVM-Specific Concepts

### Basic Blocks
Contiguous sequences of RISC-V instructions with single entry/exit points. Each block becomes a candidate for precompile generation.

### Trace Cells
The product of `width_before * execution_frequency`, representing the total computational impact of a basic block in the program trace. High trace cell counts indicate hot paths that benefit most from optimization.

### Effectiveness Ratio
Values > 1.0 indicate improvement (reduction in cost/resources). For example, effectiveness of 2.0 means the precompile reduces the metric by 50%.

### Precompile Impact
The tool helps identify which basic blocks provide the best return on investment for precompile generation, focusing optimization efforts on the most impactful code paths.

## Deployment

### GitHub Pages Deployment

1. **Create GitHub Repository**:
   ```bash
   git init
   git add index.html Claude.md
   git commit -m "Initial commit - APC Effectiveness Analyzer"
   git branch -M main
   git remote add origin https://github.com/USERNAME/REPO.git
   git push -u origin main
   ```

2. **Enable GitHub Pages**:
   - Go to repository Settings → Pages
   - Source: Deploy from branch
   - Branch: main, folder: / (root)
   - Save

3. **Access Your Site**:
   - URL: `https://USERNAME.github.io/REPO/`
   - Share with data: `https://USERNAME.github.io/REPO/?data=<json-url>`

### Handling Large JSON Files

For files too large for direct upload (>8MB), use GitHub Gists:

1. **Upload via GitHub CLI**:
   ```bash
   gh gist create your-file.json --public --desc "APC Analysis Data"
   ```

2. **Convert to Raw URL**:
   - Gist URL: `https://gist.github.com/USERNAME/GIST_ID`
   - Raw URL: `https://gist.githubusercontent.com/USERNAME/GIST_ID/raw/your-file.json`

3. **Use in Analyzer**:
   - Paste the raw URL in the analyzer's URL input field
   - Or use directly: `https://USERNAME.github.io/REPO/?data=<raw-gist-url>`

## Potential Improvements
- Export functionality for charts/data (SVG, PNG, CSV)
- Multiple file comparison side-by-side
- Time-series analysis for effectiveness trends
- Additional chart types (scatter plot, heatmap)
- Filtering/search for specific PC ranges
- Save/load visualization state
- Keyboard shortcuts for navigation
- Detailed statistics panel with percentiles
- Performance profiling for large datasets
- Mobile-responsive chart sizing
- Integration with powdr toolchain for direct data loading
- Comparison between different precompile strategies
- Cost-benefit analysis (precompile generation cost vs. runtime savings)
