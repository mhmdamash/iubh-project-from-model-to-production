# Real-Time Anomaly Detection Service (IoT / Stream Processing)

**Course:** DLBDSMTP01 – Project: From Model to Production
**Task 1:** Anomaly detection in an IoT setting (spotlight: stream processing)

A simple machine-learning model that scores factory sensor readings
(temperature, humidity, sound) for anomalies, served as a RESTful API and fed by a
continuous simulated sensor stream.

---

## What's in this project

| File                  | Purpose                                                        |
|-----------------------|----------------------------------------------------------------|
| `generate_data.py`    | Creates `sensor_data.csv` (synthetic factory sensor data).     |
| `create_model.py`     | Trains an Isolation Forest and saves it to `model.pkl`.        |
| `app.py`              | Flask REST API: `/predict` and `/health`.                      |
| `stream_simulator.py` | Continuously sends readings to the API and flags anomalies.    |
| `requirements.txt`    | Python dependencies.                                           |

## How to run it

```bash
# 1. Install dependencies (ideally in a virtual environment)
py -m pip install -r requirements.txt

# 2. Generate the data and train the model
py generate_data.py
py create_model.py          # creates model.pkl

# 3. Start the API (leave this terminal running)
py app.py                   # serves http://127.0.0.1:5000

# 4. In a SECOND terminal, start the stream
py stream_simulator.py      # one reading/sec; ALERT on anomalies
```

Test a single reading manually:

```bash
curl -X POST http://127.0.0.1:5000/predict \
     -H "Content-Type: application/json" \
     -d '{"temperature": 88, "humidity": 73, "sound": 95}'
```

## Architecture (data flow)

```
[Stream simulator] --reading--> [Flask /predict API] --uses--> [model.pkl]
        ^                               |
        |                               v
   raise ALERT  <----- anomaly score ---+----> [predictions.log  (monitoring)]
```

The model is trained offline and frozen to a file. The API loads it once at
start-up and serves predictions statelessly, so it can be replicated/scaled. The
stream feeds it continuously, and every prediction is logged for monitoring.

---

## Challenges and Concerns

**Challenges of integrating a model into a service.** Matching the data format the
model expects on every request; keeping the same preprocessing in training and
serving (solved here by bundling the scaler + detector in one pickled Pipeline);
loading the model once rather than per request; and handling malformed or missing
input gracefully.

**Constraints of a model-as-a-service.** The service must be *stateless* (each
request is independent), respect a fixed input/output contract (JSON schema),
return fast enough for a real-time stream, and be versioned so a new model can
replace the old one without breaking callers.

**Data acquisition / storage / processing.** Data arrives as a continuous stream of
small JSON messages. We process one (or a small batch) at a time rather than
loading a big file. Readings and scores are appended to a log; in production this
would be a time-series database or a message queue (e.g. Kafka) feeding the API.

**Monitoring components.** A `/health` endpoint for uptime checks; a prediction log
for every request (latency, inputs, scores); tracking the share of items flagged
over time to detect **data drift** (if the flag rate suddenly jumps, the sensors or
the process may have changed and the model should be retrained).

**System design.** See the architecture diagram. The model is *stored* as a pickle
file, *accessed* by loading it into the API at start-up, and *served* over the
`/predict` REST endpoint.

