# Integrated Bioreactor

A full-stack bioreactor control and monitoring system: ESP32 firmware for real-time control of pH, temperature, and stirring, with a Python data pipeline for telemetry logging and anomaly detection using statistical detectors and One-Class SVM.

## Project Structure

```
main/                   -- ESP32 Arduino firmware
  main.ino              -- WiFi/MQTT orchestrator
  PHSubsystem.*         -- pH sensor + acid/base pump control
  StirringSubsystem.*   -- DC motor + Hall effect RPM feedback
  heatingSubsystem.*    -- Thermistor + PWM heater control
  secrets.h             -- WiFi/MQTT credentials (not committed)

data-analysis/          -- Python telemetry and anomaly detection
  telemetry_logger.py   -- MQTT subscriber that logs live data to CSV
  anomaly_analysis.py   -- Offline/real-time anomaly detection pipeline
  detectors.py          -- Z-Score, Hysteresis, and Sliding Window detectors
  logs/                 -- CSV data files

anomalydetection/       -- SVM-based anomaly detection
  svm_train.py          -- Train One-Class SVM on fault-free MQTT data
  svm_test.py           -- Test SVM on live streams or CSV files
  detectors.py          -- Statistical detectors (standalone version)
```

## Embedded System (ESP32)

The firmware manages three subsystems with a modular architecture. Each subsystem exposes `setup`, `execute`, `getStatus`, and `handleAttributes` functions called by the main controller.

### Hardware

| Component | Type |
|-----------|------|
| Microcontroller | ESP32 |
| pH Sensor | Analog input + peristaltic acid/base pumps |
| Temperature | Thermistor + PWM-controlled heating element |
| Stirring | DC motor + Hall effect sensor for RPM measurement |

### Communication

All communication uses MQTT via ThingsBoard:

- **Telemetry** (device to cloud): JSON payload every 5 seconds with pH, RPM, temperature, and system state
- **Shared Attributes** (cloud to device): target setpoints (pH, RPM, temperature), tolerances, operational mode
- **RPC** (cloud to device): manual pump control, temperature override

## Anomaly Detection

### Statistical Detectors (`data-analysis/`)

Three complementary detectors trained on fault-free baseline data:

| Detector | Method | Use case |
|----------|--------|----------|
| **Z-Score** | Flags values exceeding N standard deviations from baseline mean | Sudden spikes |
| **Hysteresis** | Triggers when values leave a normal range, with margin to prevent toggling | Sustained drift |
| **Sliding Window** | Compares rolling average against baseline, detects gradual drift | Slow trends |

All detectors include a `ConfusionMatrix` class for evaluation with precision, recall, F1, and accuracy metrics.

### One-Class SVM (`anomalydetection/`)

A machine learning approach using scikit-learn's One-Class SVM with RBF kernel:

```bash
# Train on fault-free MQTT stream (collects 500 samples)
python svm_train.py --samples 500

# Test on faulty stream
python svm_test.py --model svm_model.pkl --stream three_faults

# Test on CSV file
python svm_test.py --model svm_model.pkl --csv test_data.csv
```

The training script exports both a Python pickle model and a C header file (`svm_model.h`) for deployment directly on the ESP32.

## Running

### Firmware

1. Copy `secrets.h.example` to `secrets.h` and fill in WiFi/MQTT credentials
2. Open `main/main.ino` in Arduino IDE
3. Select ESP32 board and upload

### Data Analysis

```bash
cd data-analysis
pip install -r requirements.txt

# Log live telemetry to CSV
python telemetry_logger.py

# Run anomaly detection on captured data
python anomaly_analysis.py --csv logs/bioreactor_data.csv
```

## Tech Stack

- **C++ / Arduino** for ESP32 firmware
- **Python 3** with NumPy, pandas, paho-mqtt, scikit-learn
- **ThingsBoard** for IoT dashboard and MQTT broker
- **ArduinoJson** for telemetry serialization
