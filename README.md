# 🌳 Tree Doctor

**Agentic AI + Signal Processing for tree-health monitoring**
Ingénieux Hackathon 2026 · Track 2 · Mahindra University

**Team Sunami Star:** Siddartha Varma, Ram Sameer Akshay, Dharmikh Sai, Vinay

## Research question
Can an agent that fuses cleaned, anomaly-checked sensor time series with on-demand leaf-image evidence diagnose tree health better than a single-sensor threshold, and recognise when it is too uncertain to decide?

## Problem and motivation
Tree stress (drought, overwatering, heat, disease) is usually noticed late. Cheap field sensors are noisy, drop out and fail silently, so simple threshold alerts give wrong advice. Tree Doctor cleans the signals, detects sensor faults, and uses an AI agent that decides what to check next, such as a leaf photo. It gives a diagnosis, or asks for a human check when unsure.

## How it works
```
Sensors (soil, temp, humidity) → Signal processing → Features ─┐
                                                               ├→ Agent loop → Diagnosis + recommendations
Leaf photo (on request) → Segmentation → Symptom evidence ─────┘
         observe → reason → decide → act (resample / ask for photo) → adapt (memory)
```
- **Signal processing** (`signal_processing/`): gap filling, Hampel spike removal, zero-phase Butterworth low-pass; detection of spikes, flat-lined sensors and sudden drops; drying rate, irrigation events and a data-quality score.
- **Agentic AI** (`agentic_ai/`): a Bayesian belief over 9 conditions. Each step the agent picks the missing observation with the highest expected information gain, or finalizes. It remembers whether its last advice worked (for example, watered but soil did not rise) and flags a human check when confidence is below 70%.
- **Computer vision** (`computer_vision/`): leaf segmentation and yellow / brown / lesion fractions; an optional RandomForest trainer for PlantVillage.

## Results (all on SIMULATED data or team-written scenarios)
| Test | Result |
|---|---|
| Noise reduction (RMSE vs ground truth) | raw 1.61, best moving average 0.55, **ours 0.44** (73% below raw) |
| Anomaly detection (event F1, mean of 5 seeds) | **0.98** (z-score baseline 0.92) |
| Agent scenarios | **7/7** correct (soil-threshold baseline 3/7) |
| Compute | 49 ms to process 7 days of data on one CPU |

These check the signal chain and agent logic. They are **not** field accuracy. Leaf-classifier accuracy and sensor accuracy against a reference were **not measured**.

## Status
| Feature | Status |
|---|---|
| Filtering and anomaly detection | Working (simulated data) |
| Agent loop, terminal demo | Working |
| Flask backend and Streamlit dashboard | Working (tested on simulated data) |
| Leaf analysis (rule-based) | Partially working (synthetic leaves only) |
| ESP32 firmware | Partially working: written, not yet flashed or validated |
| Real-data demo (Open-Meteo) | Available via `datasets/fetch_real_data.py` (model-derived data, not in-situ sensors) |
| RandomForest leaf classifier | Planned: code written, not trained |
| Field trial with reference readings | Planned |

## Run it
```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

python demo.py                                  # full workflow in the terminal, no hardware
python demo.py --sample spotted                 # also: healthy, scorched
python demo.py --leaf path/to/leaf.jpg          # your own leaf photo
python results/run_experiments.py               # regenerate all metrics and graphs
streamlit run frontend/dashboard.py             # dashboard (choose "Demo (simulated)")

# optional: real environmental data (needs internet), then run the agent on it
python datasets/fetch_real_data.py
python demo.py --csv datasets/real_openmeteo.csv --no-leaf

# optional: leaf classifier (download PlantVillage into datasets/plantvillage/<class>/ first)
python computer_vision/disease_classification.py train --data datasets/plantvillage
```
Hardware: flash `hardware/esp32_code.ino` (set Wi-Fi, backend IP and calibration; see `hardware/sensor_configuration.md`), then run `python backend/app.py`.

## Repository layout
`hardware/` firmware · `signal_processing/` filtering, anomaly detection, time-series features · `computer_vision/` leaf analysis · `agentic_ai/` agent, decision engine, recommendations · `backend/` Flask API and pipeline · `frontend/` dashboard · `datasets/` data scripts · `results/` metrics, graphs, screenshots, evidence logs.

## Limitations
- Signal results use simulated Gaussian-noise data; real probes drift with temperature, salinity and soil type.
- The agent's probabilities are expert-set assumptions, not learned. Scenarios were written by the team (a logic test, not field validation).
- Naive Bayes assumes independent evidence.
- Leaf thresholds depend on lighting; PlantVillage images are lab-style.
- Soil "%" is a calibrated wetness index, not true volumetric water content.

## Next research question
Can likelihoods learned from labelled field data, and an agent that schedules its own sampling, beat the expert-set model at lower energy cost?

## Disclosure
- **Datasets:** simulated sensor data generated by `signal_processing/synthetic_data.py`; optional PlantVillage (Hughes & Salathé, 2015, arXiv:1511.08060); optional Open-Meteo weather data (model-derived).
- **Third-party libraries:** NumPy, pandas, SciPy, scikit-learn, OpenCV, Matplotlib, Flask, Streamlit, Adafruit DHT library. No pre-existing project code was reused.
- **Generative AI:** Claude (Anthropic) was used to draft the code, report and slides. The team ran the demo and experiment scripts to check that they work.
- **Assumptions:** priors, likelihoods and HSV thresholds are heuristic; see Limitations.
