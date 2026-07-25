# AAA Project – Chicago Taxi Demand Analytics

**Course:** Advanced Analytics and Applications (SS 2026)  
**University of Cologne**

## Project Team

| Name | Student ID |
|-------|------------|
| Larissa Theilmann | 7432976 |
| Emre Uyar | 7443638 |
| Tjorge Orlitz | 7443662 |
| Nils Hornstein | 7369566 |


## Project Overview

This project analyzes historical taxi trip data from Chicago to investigate spatial and temporal demand patterns for urban mobility. The overall objective is to derive insights that support the operation of an electric ride-hailing fleet.

The project combines
- data preparation,
- descriptive analytics,
- predictive machine learning,
- reinforcement learning,
- and a final scientific report generated with Quarto.


## Repository Structure
.
├── assets/                    
├── data/                      Input datasets and processed data
├── docs/
│   ├── maps/                  Generated maps
│   ├── plots/                 Generated figures
│   ├── report.pdf             Final report
├── models_predictions/        Saved prediction models
├── notebooks/                 Complete analysis pipeline
├── partials/                  Custom LaTeX templates
├── sections/                  Sections of the report
├── README.md                  
├── references.bib             Literature references
├── pyproject.toml             Python dependencies
├── report.qmd                 Main Quarto report
├── uv.lock                    Locked dependency versions
└── _quarto.yml                Quarto configuration


### 1. Prerequisites
Ensure you have the following installed:
- **[Quarto](https://quarto.org/docs/get-started/)**: The publishing system used to render the report.
- **[uv](https://github.com/astral-sh/uv)**: A fast Python package manager.
- **LaTeX**: A TeX distribution (like [TinyTeX](https://yihui.org/tinytex/)) to generate the PDF.
  ```bash
  quarto install tinytex
  ```

### 2. Setup the Environment
We use `uv` to manage dependencies. Run the following commands in the root directory:
```bash
uv sync
uv run python -m ipykernel install --user --name team-project-template --display-name "Python (Team Project Template)"
```
This will create a `.venv` directory and register the Python kernel so Quarto can find it.

### 3. Place required data
Place the required Chicago Taxi Trips dataset inside the `data\`directory before running the notebook.

### 4. Execute notebooks
The notebooks are designed to be executed in the following order.

| Order | Notebook | Description |
|------:|----------|-------------|
| 1 | [`1.1 TaxiDataPrep.ipynb`](notebooks/1.1%20TaxiDataPrep.ipynb) | Cleans and preprocesses the raw Chicago taxi dataset, removes invalid records and creates the two base datasets used throughout the project. |
| 2 | [`1.2 WeatherDataPrep.ipynb`](notebooks/1.2%20WeatherDataPrep.ipynb) | Cleans and prepares the weather data for later integration with the taxi dataset. |
| 3 | [`1.3 POIDataPrep.ipynb`](notebooks/1.3%20POIDataPrep.ipynb) | Processes Point-of-Interest (POI) data and aggregates it spatially. |
| 4 | [`1.4 TemporalSpatialDiscretization.ipynb`](notebooks/1.4%20TemporalSpatialDiscretization.ipynb) | Generates temporal features and merges taxi, weather and POI data into unified datasets. |
| 5 | [`2.1 DescriptiveAnalysisSpatial.ipynb`](notebooks/2.1%20DescriptiveAnalysisSpatial.ipynb) | Performs the spatial descriptive analysis, compares spatial resolutions and identifies demand hotspots. |
| 6 | [`2.2 DescriptiveAnalysisTemporal.ipynb`](notebooks/2.2%20DescriptiveAnalysisTemporal.ipynb) | Analyzes temporal demand patterns and trip characteristics. |
| 7 | [`2.4 DescriptiveAnalyticsWeather.ipynb`](notebooks/2.4%20DescriptiveAnalyticsWeather.ipynb) | Investigates the relationship between weather conditions and taxi demand. |
| 8 | [`3.1 FeatureEngineering.ipynb`](notebooks/3.1%20FeatureEngineering.ipynb) | Creates the feature sets used by all predictive models. |
| 9 | [`3.2 PredictionCommonGround.ipynb`](notebooks/3.2%20PredictionCommonGround.ipynb) | Defines the common preprocessing pipeline, train/test split and evaluation setup shared by all prediction models. |
| 10 | [`3.2a PredictionSVMStandard.ipynb`](notebooks/3.2a%20PredictionSVMStandard.ipynb) | Trains and evaluates the Support Vector Machine demand prediction model. |
| 11 | [`3.2b PredictionSVMResidual.ipynb`](notebooks/3.2b%20PredictionSVMResidual.ipynb) | Implements a residual-based SVM approach to improve prediction performance. |
| 12 | [`3.3 PredictionDeepLearning.ipynb`](notebooks/3.3%20PredictionDeepLearning.ipynb) | Implements and evaluates the feed-forward neural network for taxi demand prediction. |
| 13 | [`3.4 ModelPerformanceVisualization.ipynb`](notebooks/3.4%20ModelPerformanceVisualization.ipynb) | Generates comparison plots and evaluation figures for all predictive models. |
| 14 | [`4.1 ReinforcementLearning.ipynb`](notebooks/4.1%20ReinforcementLearning.ipynb) | Develops and evaluates a reinforcement learning agent for smart charging of an electric taxi fleet. |

### 5. Render the Report
To generate the final report as a pdf, use `uv run` to ensure Quarto uses the correct environment:

  ```bash
  uv run quarto render report.qmd --to pdf
  ```

## Reproducibility

All required Python packages are managed using `uv`.
The project can be reproduced by

1. cloning the repository,
2. running `uv sync`,
3. placing the required datasets into `data/`,
4. executing the notebooks in the order described above,
5. rendering the report with Quarto.


