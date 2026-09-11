# GeoAI DEM/GCP Calibration & Validation Pipeline

**Role: Person 2 — GeoAI**  
**Deliverables:** GeoTIFF handling, DEM/GCP calibration, slope/aspect geomorphometry, relative-to-metric calibration pipeline, and MAE/RMSE validation suite.

---

## 📌 Overview

In aerial and satellite GeoAI workflows, uncalibrated stereo photogrammetry, monocular depth models, and radar disparity products output **relative elevation** ($Z_{\text{rel}}$) that lacks metric datum scale and vertical translation.

This repository provides a modular, production-grade geospatial Python pipeline to:
1. **Handle GeoTIFFs**: Read, inspect, reproject, sample, and export rasters while preserving CRS, affine georeferencing, and NoData masks.
2. **Calibrate Relative DEM to Metric Elevation**: Estimate optimal scale $s$ and shift $t$ using Ground Control Points (GCPs) or reference rasters, utilizing **RANSAC** to automatically reject vegetation canopy or measurement blunders.
3. **Analyze Terrain Geomorphometry**: Compute Horn's slope ($0^\circ - 90^\circ$), compass aspect ($0^\circ - 360^\circ$), and 3D hillshade shaded relief.
4. **Validate Vertical Accuracy**: Compute industry-standard vertical accuracy metrics (MAE, RMSE, MedAE, MBE, LE90, LE95, Pearson $r$, $R^2$), export residual difference GeoTIFFs, and generate multi-panel diagnostic plots.

---

## 📐 Mathematical Formulation

### 1. Relative-to-Metric Calibration
For a relative DEM pixel $Z_{\text{rel}}$, the calibrated metric elevation $Z_{\text{metric}}$ (meters above datum) is given by:

$$Z_{\text{metric}} = s \cdot Z_{\text{rel}} + t$$

Where:
- $s$: Global vertical scale factor.
- $t$: Datum shift / translation (m).

For regions experiencing sensor tilt or geoid warping, the planar tilt model expands to:

$$Z_{\text{metric}} = s \cdot Z_{\text{rel}} + a \cdot X + b \cdot Y + c$$

Where $(X, Y)$ are projected spatial coordinates, and $(a, b)$ compensate for linear tilt.

### 2. Terrain Slope & Aspect (Horn's Algorithm)
Given a $3 \times 3$ elevation window around center pixel $e$:
```
[a  b  c]
[d  e  f]
[g  h  i]
```
The orthogonal spatial gradients are:
$$\frac{\partial z}{\partial x} = \frac{(c + 2f + i) - (a + 2d + g)}{8 \Delta x}, \quad \frac{\partial z}{\partial y} = \frac{(g + 2h + i) - (a + 2b + c)}{8 \Delta y}$$

$$\text{Slope (degrees)} = \arctan\left(\sqrt{\left(\frac{\partial z}{\partial x}\right)^2 + \left(\frac{\partial z}{\partial y}\right)^2}\right) \times \frac{180}{\pi}$$

$$\text{Aspect (compass bearing)} = \left(90^\circ - \arctan2\left(\frac{\partial z}{\partial y}, -\frac{\partial z}{\partial x}\right)\right) \pmod{360^\circ}$$

### 3. Accuracy Metrics
- **Mean Absolute Error (MAE)**: $\frac{1}{N}\sum |Z_{\text{pred}} - Z_{\text{true}}|$
- **Root Mean Square Error (RMSE)**: $\sqrt{\frac{1}{N}\sum (Z_{\text{pred}} - Z_{\text{true}})^2}$
- **Median Absolute Error (MedAE)**: $\text{median}(|Z_{\text{pred}} - Z_{\text{true}}|)$
- **Mean Bias Error (MBE)**: $\frac{1}{N}\sum (Z_{\text{pred}} - Z_{\text{true}})$
- **Linear Error at 90% (LE90)**: 90th percentile of absolute residuals $|Z_{\text{pred}} - Z_{\text{true}}|$.

---

## 📁 Repository Structure

```
geoai-dem-calibration/
├── README.md                          # Comprehensive project documentation
├── requirements.txt                   # Geospatial and machine learning dependencies
├── pyproject.toml                     # Package build configuration
├── .gitignore                         # Standard Python & geospatial gitignore
├── geoai/                             # Core Python package
│   ├── __init__.py                    # Public API exports
│   ├── geotiff_io.py                  # GeoTIFF I/O, coordinate sampling & reprojection
│   ├── calibration.py                 # DEMCalibrator (Linear, RANSAC, Huber, Planar Tilt)
│   ├── terrain.py                     # Horn's slope, compass aspect, and hillshade
│   └── validation.py                  # Evaluation metrics, residual maps, and plotting
├── scripts/
│   ├── generate_synthetic_data.py     # Procedural alpine DEM & noisy GCP generator
│   ├── run_calibration.py             # CLI to calibrate relative DEMs
│   └── run_validation.py              # CLI to compute metrics and export error maps
├── notebooks/
│   └── dem_calibration_validation.ipynb # Step-by-step interactive Jupyter notebook
└── tests/
    ├── test_geotiff_io.py             # Unit tests for raster reading/writing
    ├── test_calibration.py            # Unit tests for scale/shift recovery & RANSAC
    └── test_terrain.py                # Unit tests for slope and aspect calculation
```

---

## 🚀 Quickstart (Zero-Setup Demo)

### 1. Install Dependencies
```bash
pip install -r requirements.txt
```

### 2. Run Synthetic Demo in 3 Commands
Generate sample alpine terrain, calibrate with GCPs, and evaluate accuracy:

```bash
# Step A: Generate realistic synthetic relative DEM, reference DEM, and GCPs
python scripts/generate_synthetic_data.py --output_dir data

# Step B: Calibrate relative DEM using GCPs with robust RANSAC
python scripts/run_calibration.py --relative data/relative_dem.tif --gcps data/gcps.csv --output data/calibrated_dem.tif --method ransac

# Step C: Compute MAE/RMSE, export error GeoTIFF, and calculate slope/aspect
python scripts/run_validation.py --predicted data/calibrated_dem.tif --reference data/ground_truth_dem.tif --output_dir results --compute_terrain
```

Generated outputs will be saved in `results/`:
- `results/validation_metrics.json` (Full metric report)
- `results/error_map.tif` (Residual difference GeoTIFF)
- `results/validation_dashboard.png` (6-panel diagnostic figure)
- `results/slope.tif` (Terrain slope raster in degrees)
- `results/aspect.tif` (Terrain aspect raster in compass degrees)
- `results/hillshade.tif` (8-bit shaded relief raster)

### 3. Run Interactive Jupyter Notebook
```bash
jupyter notebook notebooks/dem_calibration_validation.ipynb
```

---

## 🗺️ How to Use with Real Project Data

### Preparing Your Input Data:
1. **Relative DEM (`.tif`)**: Put your uncalibrated relative DEM GeoTIFF into `data/your_relative_dem.tif`.
2. **Ground Control Points (`.csv`)**: Place your GCP survey file into `data/your_gcps.csv`.  
   Format required:
   ```csv
   x,y,elevation
   654210.5,5142380.2,852.4
   658930.1,5145120.8,1120.6
   ...
   ```
   *(The code automatically recognizes `x`, `y`, `lon`, `lat`, `easting`, `northing`, `z`, `elevation`).*

3. **Or Reference DEM (`.tif`)**: If you have a reference raster (e.g. Copernicus DEM 30m, SRTM, or airborne LiDAR), place it in `data/your_reference_dem.tif`.

### Running Calibration on Real Data:
```bash
python scripts/run_calibration.py \
    --relative data/your_relative_dem.tif \
    --gcps data/your_gcps.csv \
    --output results/real_calibrated_dem.tif \
    --method ransac
```

---

## 🧪 Running Automated Unit Tests

Run test suite with pytest:
```bash
pytest tests -v
```

---

## 📤 How to Push to GitHub

Follow these steps to upload this complete repository to your GitHub account:

### 1. Open Terminal in this folder:
```bash
cd geoai-dem-calibration
```

### 2. Initialize Git Repository:
```bash
git init
```

### 3. Add Files and Commit:
```bash
git add .
git commit -m "feat: complete Person 2 GeoAI DEM calibration and validation pipeline"
```

### 4. Create Repository on GitHub:
- Go to [GitHub New Repository](https://github.com/new).
- Name your repository (e.g., `geoai-dem-calibration`).
- Keep it empty (do **not** check "Add a README" or ".gitignore" as they are already provided here).

### 5. Link and Push:
```bash
git branch -M main
git remote add origin https://github.com/<YOUR-GITHUB-USERNAME>/geoai-dem-calibration.git
git push -u origin main
```
