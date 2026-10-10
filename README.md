# ResQGrid

> A real-time emergency response and resource allocation platform connecting Citizens, Dispatchers, Ambulances, and Hospitals.

---

## 1. What is ResQGrid?

In traditional emergency response systems:
- Call operators manually decide which ambulance to send without knowing live road conditions.
- Up to 20-30% of emergency calls are duplicates or pranks, tying up dispatchers.
- Ambulances often rush patients to the nearest hospital only to find **no available ICU beds**, forcing dangerous diversions.
- Citizens have no visibility into where their ambulance is or when it will arrive.

**ResQGrid** connects all four stakeholders in a single real-time platform:
1. **Citizens** — report emergencies with one tap and track their assigned ambulance live.
2. **Dispatchers** — monitor a city-wide radar map, review AI triage recommendations, and manage fleet dispatches.
3. **Ambulance Drivers** — receive dispatches on a mobile console with turn-by-turn traffic navigation.
4. **Hospitals** — track incoming patients and manage bed/ICU capacity in real-time.

---

## 2. System Architecture

```
[Citizen App] ──(SOS Report)──► [Node.js Express Backend] ◄──► [PostgreSQL / PostGIS]
                                       │
                         ┌─────────────┴─────────────┐
                         ▼                           ▼
               [Python ML Service]         [GeoAgent Decision Engine]
               - Severity (1 to 5)         - Distance + Traffic
               - False Report Check        - ICU Bed Headroom Match
               - Travel ETA Regressor      - Vehicle Equipment Match
                         │                           │
                         └─────────────┬─────────────┘
                                       ▼
                       [Socket.IO Real-time Stream]
                                       │
                 ┌─────────────────────┴─────────────────────┐
                 ▼                                           ▼
      [Dispatcher Command Center]                 [Ambulance Navigation]
```

---

## 3. Implemented Modules

### 1. Citizen Application (`frontend/src/pages/citizen`)
- **Emergency Reporting:** Quick category buttons (*Cardiac*, *Road Accident*, *Severe Trauma*, *Respiratory*, *Fire*) with auto-detected GPS coordinates.
- **Live Tracking:** Real-time map displaying ambulance movement, distance, and ETA countdown.
- **Emergency Contacts:** One-click SOS calling to preset emergency contacts.

### 2. Ambulance Driver Console (`frontend/src/pages/ambulance`)
- **Dispatch Alert:** Audible incoming dispatch notification with incident details and patient notes.
- **One-Tap Actions:** Accept/reject dispatch, update status (*En Route*, *On Scene*, *Transporting*, *Completed*).
- **Turn-by-Turn Navigation:** Live telemetry broadcast (latitude, longitude, speed, heading) and traffic-cleared routing.

### 3. Hospital Operations (`frontend/src/pages/hospital`)
- **Live Bed Tracker:** Tracks total beds, available regular beds, ICU beds, and on-duty emergency doctors.
- **Pre-Arrival Alerts:** Prepares trauma teams before the ambulance arrives with estimated arrival time and patient type.
- **Resource Management:** Real-time updates directly influence where the dispatch engine routes future patients.

### 4. Dispatcher Command Center (`frontend/src/pages/dispatcher`)
- **Tactical Radar Map:** Leaflet map showing active emergencies, moving ambulances, and hospital capacity pins.
- **Active Incident Stream:** Table with ML severity badges, verification scores, and quick filters (*Critical*, *En Route*).
- **GeoAgent Decision Trace:** Explains exactly why a specific hospital and ambulance were chosen over others.
- **Green Corridor Overrides:** Option to trigger traffic signal bypasses for high-priority cases.

### 5. Backend & Database (`backend/src`)
- **REST API:** Express 5 + TypeScript handling authentication, incident lifecycles, and fleet telemetry.
- **Role-Based Access Control (RBAC):** Secure JWT auth across 6 roles (`citizen`, `dispatcher`, `ambulance_driver`, `hospital_staff`, `hospital_admin`, `system_admin`).
- **PostGIS Spatial Queries:** Detects duplicate calls within 500 meters reported inside a 30-minute window.
- **Resilience Engine:** Automatically falls back to in-memory caching if PostgreSQL or external APIs are offline.

---

## 4. Machine Learning (ML) Implementation

The ML service is a lightweight Python **FastAPI** service running on port `8000`. It uses three trained **Random Forest** models built with Scikit-Learn.

```
   [Incoming Incident]
           │
           ├──► Model 1: Severity Classifier ──► Level 1 to 5 + Clinical Reason
           ├──► Model 2: Fraud Detector       ──► Real vs Fake Score (0-100%)
           └──► Model 3: Response ETA         ──► Arrival Time in Minutes
```

---

### Model 1: Severity Prediction (`severity_model.joblib`)
- **What it does:** Reads the report details and predicts how critical the situation is on a scale of 1 to 5.
- **Algorithm:** `RandomForestClassifier` (100 trees) + Clinical symptom keyword extraction.
- **Inputs:**
  - `emergency_type`: Category (*cardiac*, *stroke*, *accident*, *fire*, etc.).
  - `description`: Text written by the user (searched for critical terms like *unconscious*, *chest pain*, *massive bleeding*).
  - `people_affected`: Number of injured people.
  - `has_address`: Boolean indicating whether a specific address was given.
  - `hour_of_day`: Time of report (0–23).
- **Outputs:**
  - `predicted_severity`: Integer from `1` (Minor) to `5` (Catastrophic).
  - `severity_label`: e.g., `"Critical"`.
  - `confidence`: e.g., `0.85` (85%).
  - `explanation`: e.g., `"Emergency type 'cardiac' indicates critical priority. Acute symptoms detected."`

---

### Model 2: False Report / Fraud Detection (`fraud_model.joblib`)
- **What it does:** Spots fake reports, bots, and pranks to keep ambulances available for real emergencies.
- **Algorithm:** `RandomForestClassifier` (150 trees) with asymmetric class weighting (missing a fake is penalized 2x).
- **Inputs:**
  - `description`: Length and quality of text (very short or gibberish text is flagged).
  - `has_address`: Presence of physical address.
  - `is_within_india`: Validates geographic boundaries.
  - `report_hour`: Time of submission (prank calls peak late at night).
  - `reporter_history_score`: Trust rating of the citizen's account (0 to 1).
  - `nearby_incident_count`: Corroborating reports nearby.
  - `severity_claimed`: Flags inconsistency (e.g. claiming catastrophic severity with a 2-word description).
- **Outputs:**
  - `is_suspicious`: `true` / `false`.
  - `fraud_probability`: 0.0 to 1.0.
  - `confidence_score`: 0–100 credibility rating (used in the verification state machine).
  - `flags`: List of human-readable reasons (e.g., *["Very short description", "Low trust score"]*).

---

### Model 3: Response Time Regressor (`eta_model.joblib`)
- **What it does:** Estimates realistic ambulance travel time by factoring in real-world traffic patterns.
- **Algorithm:** `RandomForestRegressor` (100 trees).
- **Inputs:**
  - `distance_km`: Straight-line or road distance.
  - `hour_of_day`: Identifies morning (8–10 AM) and evening (5–8 PM) rush hours.
  - `day_of_week`: Weekday vs weekend patterns.
  - `emergency_type`: High-urgency types receive faster routing priority.
  - `ambulance_type`: Advanced Life Support (ALS with sirens) vs Basic.
  - `congestion_level`: Road traffic (0: Clear, 1: Moderate, 2: Heavy).
- **Outputs:**
  - `predicted_eta_minutes`: Estimated arrival time (e.g., `8.5 minutes`).
  - `eta_range`: Confidence window (e.g., `7.0 to 10.0 minutes`).
  - `factors`: List of impact factors (e.g., *["Distance: 5.4 km", "Moderate traffic", "ALS siren priority"]*).

---

## 5. GeoAgent Decision Matching Engine

Instead of blindly picking the nearest facility, our **GeoAgent** engine calculates a multi-factor score:

$$\text{Score} = \text{Proximity} + \text{ICU Bed Availability} + \text{Trauma Level Match}$$

- **Hard Rules:** If an emergency requires an ICU and a hospital has 0 ICU beds left, that hospital is immediately disqualified.
- **Decision Trace:** Every dispatch produces a clear explanation for the dispatcher (e.g., *"Selected Victoria Hospital due to 14 available ICU beds and Level-1 Trauma readiness, despite Apollo Hospital being 1.2 km closer."*).

---

## 6. How to Run the Project Locally

### Prerequisites
- Node.js (v20+)
- Python (3.10 or 3.11)
- npm

---

### 1. Start the Python ML Service
```bash
cd ml-service

# Create virtual environment (optional)
python -m venv .venv
# Windows: .venv\Scripts\activate
# Mac/Linux: source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run the FastAPI server
python main.py
```
> Runs at `http://localhost:8000`. Swagger API docs available at `http://localhost:8000/docs`.

---

### 2. Start the Backend
Open a second terminal:
```bash
cd backend
npm install
npm run dev
```
> Runs at `http://localhost:5000`. Health check endpoint: `http://localhost:5000/api/health`.

---

### 3. Start the Frontend
Open a third terminal:
```bash
cd frontend
npm install
npm run dev
```
> Runs at `http://localhost:5173`.

---

## 7. Demo Accounts for Testing

All test accounts use the password: `Emergency@123`

| Role | Email | Dashboard |
| :--- | :--- | :--- |
| **Dispatcher** | `dispatcher@emergency.gov.in` | Command Center radar map, incident queue, decision trace |
| **Citizen** | `citizen@emergency.gov.in` | Emergency SOS intake and live tracking |
| **Ambulance Driver** | `ambulance@emergency.gov.in` | Vehicle navigation and telemetry console |
| **Hospital Admin** | `hospital@emergency.gov.in` | ICU beds, total beds, and doctor availability manager |
| **System Admin** | `admin@emergency.gov.in` | Administrative settings and system audit |

---

*ResQGrid Emergency Coordination Platform*
