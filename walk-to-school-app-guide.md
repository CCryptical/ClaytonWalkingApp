# SafeWalk NC-13: Walk-to-School Route Safety App
### A complete beginner's build guide for the Congressional App Challenge

---

## 1. What You're Building (Plain English)

A web app where a student (or parent) types in their home address and their school, and the app shows:

1. **The fastest walking route** (shortest time/distance)
2. **The safest walking route** (avoids hazards, better sidewalk coverage, lower crime areas)
3. **A comparison** — how much longer is "safe" vs "fast," and why
4. **A Walk Safety Score** (e.g., 0–100) for the recommended route, with a breakdown of what hurt or helped the score (missing sidewalks, high-speed roads, crossing a highway with no crosswalk, crime density nearby, etc.)
5. **A map** showing both routes and flagged hazard points

This is a genuinely strong Congressional App Challenge idea because it's hyper-local (rural/small-town NC-13 has real sidewalk gaps — unlike a big city app), solves a real safety problem, and uses real government/open data, which judges like.

---

## 2. The Tech Stack, Explained for a Beginner

| Layer | Tool | What it does | Why this one |
|---|---|---|---|
| Backend logic | **Python + Flask** | Runs your scoring algorithm, talks to APIs, serves data to the frontend | You already know Python; Flask is the simplest way to turn Python into a website |
| Map data | **OpenStreetMap (OSM)** | Free, crowd-sourced map data — roads, sidewalks, crosswalks | Free, no API key needed for basic use, very detailed in most areas |
| Getting OSM data into Python | **OSMnx** (Python library) | Downloads OSM road networks and turns them into a graph Python can do math on | Purpose-built for exactly this; used in real urban planning research |
| Routing math | **NetworkX** (Python library) | Finds shortest/best paths through a graph of roads | Pairs naturally with OSMnx; both are pure Python, no server to run |
| Frontend map display | **Leaflet.js** (JavaScript library) | Draws the interactive map in the browser using OSM tiles | The standard free library for OSM-based web maps; simple to learn |
| Frontend structure | **HTML/CSS** (+ small amount of JS) | The page itself, forms, buttons | What you already planned to use |
| Crime data | NC-specific open data (see Section 5) | Feeds the safety score | Real local data = judges love it |
| Hazard data | Extracted from OSM tags | Missing sidewalks, missing crosswalks, high-speed roads | Free, no separate API |

**Key beginner concept:** You do NOT need a paid routing API (like Google Maps) or a separate routing server (like Valhalla) if you're comfortable with a bit of extra Python. OSMnx + NetworkX can do everything Valhalla does for a walking app — download the walk network, build a graph, and run shortest-path algorithms on it — and it runs entirely inside your Flask app. This is dramatically simpler to set up and deploy than running a Valhalla server, and it's still deeply respected by CAC judges since you're doing the routing logic yourself rather than calling a black-box API. If you and your partner still want Valhalla later for speed, you can swap it in — but start with OSMnx/NetworkX.

---

## 3. Environment Setup (Do This First)

### 3.1 Install Python tools
Open a terminal and run:

```bash
pip install flask osmnx networkx geopy pandas requests
```

What each one is:
- `flask` — the web server framework
- `osmnx` — downloads and processes OpenStreetMap road/sidewalk networks
- `networkx` — the graph math library that finds routes
- `geopy` — turns addresses into latitude/longitude ("geocoding")
- `pandas` — handles tabular data (crime stats, hazard lists) cleanly
- `requests` — makes API calls (e.g., to crime data portals)

### 3.2 Set up your project folder
```
safewalk/
├── app.py                  # Main Flask app
├── routing.py               # Route-finding + scoring logic
├── hazards.py                # Hazard detection from OSM data
├── crime_data.py             # Crime data loading/scoring
├── static/
│   ├── style.css
│   └── script.js            # Leaflet map logic
├── templates/
│   └── index.html            # Main page
├── data/
│   └── crime_stats.csv       # Downloaded local crime data
└── requirements.txt
```

### 3.3 Version control
Set up a GitHub repo immediately (CAC requires a public GitHub repo with your source code). Since you have a partner, this also makes collaboration possible.

---

## 4. How the Routing Actually Works (Core Logic)

This is the heart of the app. Here's the step-by-step beginner explanation:

### Step 1: Turn the town into a graph
OSMnx can download every walkable path (roads, sidewalks, footpaths) in a bounding area and represent it as a **graph** — a network of nodes (intersections/points) connected by edges (street segments).

```python
import osmnx as ox

# Download the walk network for a town
G = ox.graph_from_place("Clayton, NC, USA", network_type="walk")
```

For rural areas, `network_type="walk"` includes road shoulders even without sidewalks, which is exactly the messy reality you're trying to model.

### Step 2: Geocode the addresses
Convert "123 Main St, Clayton NC" and "Clayton High School" into lat/lon points, then snap them to the nearest node in the graph.

```python
from geopy.geocoders import Nominatim
geolocator = Nominatim(user_agent="safewalk_app")
location = geolocator.geocode("123 Main St, Clayton, NC")
```

Nominatim is OpenStreetMap's own free geocoding service — no API key required, but be polite about request rate (1 request/second max, per their usage policy).

### Step 3: Assign a "cost" to every street segment
This is where your app becomes smart instead of just a basic map. Every edge (street segment) in the graph gets:
- **A time cost** (distance ÷ walking speed) — used for the "fastest" route
- **A safety cost** — a weighted penalty based on:
  - Missing sidewalk (OSM tag `sidewalk=no` or absent) → penalty
  - Road speed limit (OSM tag `maxspeed`) — higher speed = more dangerous to walk beside → penalty scales with speed
  - No marked crossing at an intersection where one is needed → penalty
  - Crime rate in the surrounding area (from your crime dataset) → penalty scaled by local statistics
  - Street lighting presence if available in OSM (`lit=yes/no`) → minor penalty if unlit

```python
for u, v, data in G.edges(data=True):
    speed = data.get('maxspeed', 30)  # default assumption if missing
    has_sidewalk = data.get('sidewalk', 'no') != 'no'
    safety_cost = data['length']  # base cost = distance
    if not has_sidewalk:
        safety_cost *= 1.8
    if isinstance(speed, (int, float)) and speed > 35:
        safety_cost *= 1.5
    data['safety_weight'] = safety_cost
```

### Step 4: Run shortest-path twice
```python
import networkx as nx

fastest_route = nx.shortest_path(G, orig_node, dest_node, weight='length')
safest_route = nx.shortest_path(G, orig_node, dest_node, weight='safety_weight')
```

This gives you two different paths through the same graph — one optimized purely for distance, one optimized for your safety-weighted cost. That's your "difference between safest and quickest" feature, built with two lines of actual pathfinding.

### Step 5: Calculate the final Walk Safety Score (0–100)
Once you have the safest route, walk along its edges and total up the penalties, then normalize to a 0–100 scale. Something like:

```python
def safety_score(route, G):
    total_penalty = 0
    total_length = 0
    for u, v in zip(route[:-1], route[1:]):
        edge = G.get_edge_data(u, v)[0]
        total_penalty += edge['safety_weight'] - edge['length']
        total_length += edge['length']
    penalty_ratio = total_penalty / total_length
    score = max(0, 100 - (penalty_ratio * 100))
    return round(score, 1)
```

This is a simplified version — you and your partner should tune the actual weighting (this is a great thing to document in your CAC submission as your "methodology").

---

## 5. Data Sources (What to Actually Plug In)

| Data | Source | How to get it |
|---|---|---|
| Roads, sidewalks, crosswalks, speed limits | **OpenStreetMap** via OSMnx | Free, built into OSMnx, no signup |
| Sidewalk/bike path coverage (more complete than OSM in NC) | **NC OneMap** | You already identified this — download shapefiles, load with `geopandas`, cross-reference against OSM edges to correct/supplement missing sidewalk tags |
| Crime statistics | County sheriff's office open data, NC DOJ **Uniform Crime Reporting** data, or town police department public logs | Rural NC towns often don't have a nice API — you may need to manually download CSVs from county GIS portals or FOIA-style public records requests. Start with whichever of Clayton/Smithfield/Sanford/Lillington/Wake Forest publishes usable data, and note in your submission which towns have data gaps (that's honest and judges respect it) |
| Street lighting (optional) | OSM `lit` tag, sparse coverage | Nice-to-have, not essential for v1 |

**Beginner tip:** Don't try to get live/real-time crime data. A static CSV of geocoded incidents (address or block + category + date) from the last 1–3 years is completely sufficient and much easier to work with. Load it with `pandas`, then use `geopy`/simple distance math to count incidents within e.g. 200 meters of each route segment.

---

## 6. The Flask Backend Structure

```python
# app.py
from flask import Flask, render_template, request, jsonify
from routing import get_routes_and_score

app = Flask(__name__)

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/api/route", methods=["POST"])
def route():
    data = request.json
    home = data["home_address"]
    school = data["school_address"]
    result = get_routes_and_score(home, school)
    return jsonify(result)

if __name__ == "__main__":
    app.run(debug=True)
```

`get_routes_and_score()` (in `routing.py`) does everything from Section 4 and returns something like:

```python
{
  "fastest_route": [[lat, lon], [lat, lon], ...],
  "safest_route": [[lat, lon], [lat, lon], ...],
  "fastest_time_min": 14,
  "safest_time_min": 19,
  "safety_score": 72.5,
  "hazards": [
    {"lat": 35.65, "lon": -78.45, "type": "no_sidewalk_stretch"},
    {"lat": 35.66, "lon": -78.46, "type": "unmarked_highway_crossing"}
  ]
}
```

Your frontend JavaScript then draws this on the map.

---

## 7. The Frontend (HTML + Leaflet.js)

`templates/index.html` skeleton:

```html
<!DOCTYPE html>
<html>
<head>
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <link rel="stylesheet" href="/static/style.css" />
</head>
<body>
  <h1>SafeWalk NC-13</h1>
  <form id="route-form">
    <input type="text" id="home" placeholder="Home address" />
    <input type="text" id="school" placeholder="School address" />
    <button type="submit">Find Routes</button>
  </form>
  <div id="map" style="height: 500px;"></div>
  <div id="score-panel"></div>

  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script src="/static/script.js"></script>
</body>
</html>
```

`static/script.js` (core idea):

```javascript
const map = L.map('map').setView([35.65, -78.45], 13); // default view, your NC-13 area
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
  attribution: '© OpenStreetMap contributors'
}).addTo(map);

document.getElementById("route-form").addEventListener("submit", async (e) => {
  e.preventDefault();
  const home = document.getElementById("home").value;
  const school = document.getElementById("school").value;

  const res = await fetch("/api/route", {
    method: "POST",
    headers: {"Content-Type": "application/json"},
    body: JSON.stringify({home_address: home, school_address: school})
  });
  const data = await res.json();

  L.polyline(data.fastest_route, {color: 'blue'}).addTo(map);
  L.polyline(data.safest_route, {color: 'green'}).addTo(map);
  data.hazards.forEach(h => {
    L.marker([h.lat, h.lon]).addTo(map).bindPopup(h.type);
  });

  document.getElementById("score-panel").innerText =
    `Safety Score: ${data.safety_score}/100 — Fastest: ${data.fastest_time_min} min, Safest: ${data.safest_time_min} min`;
});
```

**Important:** OpenStreetMap's tile server (`tile.openstreetmap.org`) has a strict usage policy — it's meant for light development/testing, not a production app with real traffic. For your CAC submission (a demo, low traffic) this is completely fine. Just make sure to keep the attribution line, which is required by OSM's license.

---

## 8. Suggested Build Order (Fits Your Aug 10 – Oct 26 Timeline)

| Phase | Weeks | What to build |
|---|---|---|
| 1. Foundation | 1–2 | Flask app running, OSMnx pulling a graph for one town (Clayton), basic Leaflet map showing on a webpage |
| 2. Basic routing | 3–4 | Geocode two addresses, get shortest path, draw it on the map |
| 3. Hazard tagging | 5–6 | Pull sidewalk/speed/crossing data from OSM tags, cross-reference with NC OneMap, mark hazards on map |
| 4. Crime data integration | 7–8 | Load a crime CSV for at least one town, build the "incidents near this route" function |
| 5. Safety score + safest-route algorithm | 9–10 | Build the weighted graph, generate the safest route, compute the score |
| 6. Multi-town support + polish | 11–12 | Extend to Smithfield/Sanford/Lillington/Wake Forest, fix UI, error handling for bad addresses |
| 7. Documentation + video + submission | 13–14 | GitHub README, methodology write-up, ~1 min demo video, CAC submission form |

---

## 9. Full List of Installs/Tools

```bash
pip install flask osmnx networkx geopy pandas requests geopandas
```

- `geopandas` — for reading NC OneMap shapefiles (sidewalk/bike path coverage)
- Leaflet.js — loaded via CDN in your HTML, no install needed
- No API keys required for OSM data, Nominatim geocoding, or OSM tiles at this scale
- GitHub — for your public repo (CAC requirement)
- Optional later: `flask-caching` if repeated requests to Nominatim get slow (cache geocoding results locally so you're not re-querying the same addresses)

---

## 10. Notes for Your CAC Submission

- **Methodology matters as much as the demo.** Judges want to see *why* your safety score is weighted the way it is — document your reasoning (sidewalk absence, speed limits, crime density, crossing safety) clearly in your README and video.
- **Be upfront about data gaps.** Rural areas have incomplete OSM sidewalk tagging and incomplete public crime data — acknowledging this and explaining your workaround (e.g., cross-referencing NC OneMap) shows real technical maturity.
- **Local relevance is your strongest card.** Explicitly tie the problem to NC-13's mix of small towns/rural areas lacking sidewalk infrastructure compared to bigger cities — this is exactly the kind of local-impact framing CAC judges reward.
