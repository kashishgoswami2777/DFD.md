# RIROP — Sequence Diagram-Style Data Flow Diagrams (Module-wise)

> Platform: Risk-Aware Intelligent Route Optimization Platform (RIROP)
> Diagram Style: Sequence Diagram-based Data Flow Diagrams (DFDs)
> Coverage: All 9 functional modules + full system-level overview

---

## FULL SYSTEM-LEVEL SEQUENCE DFD (End-to-End)

```
sequenceDiagram
    actor User
    participant FE as Frontend UI
    participant BE as Backend API Server
    participant ORS as OpenRouteService API
    participant WE as Weather Service
    participant NE as News Service
    participant DB as Database
    participant Cache as Redis Cache
    participant RE as Risk Engine
    participant RR as Route Ranker

    User->>FE: Enter Source & Destination
    FE->>BE: POST /route-optimize {source, destination}

    BE->>ORS: GET /routes {source, destination, alternatives=true}
    ORS-->>BE: [Route A, Route B, Route C] (GeoJSON + waypoints)

    BE->>BE: Extract cities/waypoints per route

    loop For each city in all routes
        BE->>Cache: GET weather_score:{city}
        alt Cache HIT
            Cache-->>BE: weather_score
        else Cache MISS
            BE->>WE: GET /weather {city}
            WE-->>BE: {temp, rain, wind, alerts, visibility}
            BE->>BE: Compute weather_risk_score
            BE->>Cache: SET weather_score:{city} TTL=1hr
        end

        BE->>Cache: GET news_score:{city}
        alt Cache HIT
            Cache-->>BE: news_score
        else Cache MISS
            BE->>NE: GET /news {city, keywords, last72h}
            NE-->>BE: [articles...]
            BE->>BE: NLP keyword scoring -> news_risk_score
            BE->>Cache: SET news_score:{city} TTL=1hr
        end

        BE->>DB: SELECT logistics_data WHERE city_id=?
        DB-->>BE: {traffic_density, road_closure, accident_index}
        BE->>BE: Compute logistics_risk_score

        BE->>DB: SELECT economic_indicators WHERE country=?
        DB-->>BE: {fuel_price_index, inflation_rate, gdp_growth}
        BE->>BE: Compute economic_risk_score
    end

    BE->>RE: Compute final_risk per route
    RE-->>BE: {routeA_risk, routeB_risk, routeC_risk}

    BE->>RR: Rank routes by [risk, duration, distance]
    RR-->>BE: best_route + ranked_list

    BE-->>FE: {best_route, risk_breakdown, alternatives}
    FE-->>User: Display map + risk scores + recommendation
```

---

## MODULE 1: Route Generation

Purpose: Accept user input and fetch multiple route alternatives from OpenRouteService API.

```
sequenceDiagram
    actor User
    participant FE as Frontend UI
    participant BE as Backend API Server
    participant Validator as Input Validator
    participant ORS as OpenRouteService API

    User->>FE: Enter Source City / Lat-Long
    User->>FE: Enter Destination City / Lat-Long
    FE->>FE: Basic form validation (non-empty)
    FE->>BE: POST /api/optimize { source, destination }

    BE->>Validator: Validate & geocode input
    Validator-->>BE: { src_lat, src_lon, dst_lat, dst_lon }

    BE->>ORS: GET /v2/directions/driving-car
              ?start={src_lon,src_lat}&end={dst_lon,dst_lat}&alternatives=true

    alt ORS responds successfully
        ORS-->>BE: GeoJSON FeatureCollection
                   [Route A, Route B, Route C]
                   (geometry, distance, duration, bbox)
        BE->>BE: Parse GeoJSON -> extract polylines, distances, durations
        BE-->>FE: { routes: [RouteA, RouteB, RouteC] }
    else ORS timeout / error
        ORS-->>BE: Error / Timeout
        BE->>BE: Retry (up to 3 times)
        BE-->>FE: { error: "Routing service unavailable" }
    end
```

Data Flows:
  User        -> Frontend   : source, destination (text/coordinates)
  Frontend    -> Backend    : POST body {source, destination}
  Backend     -> ORS API    : Query params: start/end coordinates, alternatives=true
  ORS API     -> Backend    : GeoJSON routes with geometry, distance, duration
  Backend     -> Frontend   : Parsed route array

---

## MODULE 2: Waypoint Extraction

Purpose: Break each route polyline into meaningful city/region waypoints for downstream risk analysis.

```
sequenceDiagram
    participant BE as Backend API Server
    participant GeoParser as GeoJSON Parser
    participant Sampler as Polyline Sampler
    participant GeoLookup as Reverse Geocoder
    participant DB as Database (cities table)

    BE->>GeoParser: Pass route GeoJSON geometry (coordinates array)
    GeoParser->>GeoParser: Decode polyline -> list of [lat, lon] points

    GeoParser->>Sampler: Send full coordinate list
    Sampler->>Sampler: Sample every Nth point (reduce to ~10-20 key waypoints)
    Sampler-->>BE: Sampled waypoints [ {lat, lon}, ... ]

    loop For each sampled waypoint
        BE->>GeoLookup: Reverse geocode {lat, lon}
        GeoLookup-->>BE: City name, region, country

        BE->>DB: SELECT id, name FROM cities WHERE name LIKE '%{city_name}%'
        alt City found in DB
            DB-->>BE: city_id, city_name
        else City not in DB
            DB-->>BE: No match
            BE->>DB: INSERT INTO cities (name, lat, lon)
            DB-->>BE: new city_id
        end
    end

    BE->>BE: Deduplicate city list per route
    BE-->>BE: Final city list per route [{city_id, name, lat, lon}, ...]
```

Data Flows:
  Backend         -> GeoJSON Parser    : Raw route geometry (coordinate array)
  Polyline Sampler -> Backend          : Sampled waypoints (lat/lon pairs)
  Backend         -> Reverse Geocoder  : Lat/lon -> city name lookup
  Backend         -> DB (cities)       : City name lookup / insert
  DB              -> Backend           : city_id, city_name, coordinates

---

## MODULE 3: Weather Risk Module

Purpose: Fetch real-time weather for each city and compute a normalized weather risk score (0-1).

```
sequenceDiagram
    participant BE as Backend API Server
    participant Cache as Redis Cache
    participant WAPI as WeatherAPI.com
    participant WScorer as Weather Risk Scorer

    loop For each city in route
        BE->>Cache: GET cached_weather:{city_id}

        alt Cache HIT (< 1 hour old)
            Cache-->>BE: { weather_score, timestamp }
        else Cache MISS or Expired
            BE->>WAPI: GET /v1/current.json?key=API_KEY&q={city_name}

            alt WeatherAPI responds
                WAPI-->>BE: { temp_c, precip_mm, wind_kph, vis_km,
                              condition: {text, code}, alerts: [...] }

                BE->>WScorer: Pass weather data
                WScorer->>WScorer: Score computation:
                                   Heavy rain (precip>10mm)  -> +0.3
                                   Storm alert present       -> +0.4
                                   Wind > 40 kph             -> +0.2
                                   Low visibility (<2km)     -> +0.2
                                   Extreme temp              -> +0.1
                                   Clamp score to [0.0, 1.0]
                WScorer-->>BE: weather_risk_score (float)

                BE->>Cache: SET cached_weather:{city_id} { score, raw_data, timestamp } TTL=3600s
            else WeatherAPI down / timeout
                WAPI-->>BE: Error
                BE->>BE: Use fallback score = 0.3 (medium risk)
            end
        end
    end

    BE->>BE: Aggregate per route: route_weather_risk = AVG(city_weather_scores)
    BE-->>BE: route_weather_risk_score
```

Risk Scoring Table:
  Heavy rain (precip > 10mm)  -> +0.30
  Storm alert present         -> +0.40
  Wind speed > 40 kph         -> +0.20
  Visibility < 2 km           -> +0.20
  Extreme temperature         -> +0.10
  Clear/sunny conditions      -> +0.00

---

## MODULE 4: News Risk Module

Purpose: Fetch recent news articles for each city, apply NLP keyword scoring, and compute a news risk score.

```
sequenceDiagram
    participant BE as Backend API Server
    participant Cache as Redis Cache
    participant NAPI as NewsAPI
    participant NLP as NLP Keyword Scorer
    participant Classifier as Risk Classifier

    loop For each city in route
        BE->>Cache: GET cached_news:{city_id}

        alt Cache HIT (< 1 hour old)
            Cache-->>BE: { news_score, timestamp }
        else Cache MISS or Expired
            BE->>NAPI: GET /v2/everything
                       ?q={city}+(protest OR riot OR accident OR strike OR blockade OR flood)
                       &from={72h_ago}&language=en&sortBy=relevancy

            alt NewsAPI responds
                NAPI-->>BE: { articles: [{ title, description, source, publishedAt }, ...] }

                BE->>NLP: Pass articles list

                loop For each article
                    NLP->>NLP: Scan title + description for risk keywords:
                               "protest/riot"       -> HIGH
                               "accident/crash"     -> HIGH
                               "strike/blockade"    -> MEDIUM
                               "flood/storm"        -> HIGH
                               "construction/delay" -> LOW
                    NLP-->>NLP: article_risk_level
                end

                NLP->>Classifier: Aggregate article scores
                Classifier->>Classifier: Map to numeric:
                                         HIGH   -> 0.8
                                         MEDIUM -> 0.5
                                         LOW    -> 0.2
                                         None   -> 0.0
                                         city_news_score = max(article_scores)
                Classifier-->>BE: news_risk_score (float)

                BE->>Cache: SET cached_news:{city_id} { score, article_count, timestamp } TTL=3600s
            else NewsAPI down / no results
                NAPI-->>BE: Error or empty
                BE->>BE: news_risk_score = 0.0 (no news = no risk signal)
            end
        end
    end

    BE->>BE: Aggregate per route: route_news_risk = MAX(city_news_scores)
    BE-->>BE: route_news_risk_score
```

NLP Keyword Risk Mapping:
  Violence/Unrest  (protest, riot, curfew)          -> HIGH   -> 0.8
  Accidents        (crash, accident, collision)      -> HIGH   -> 0.8
  Natural Events   (flood, storm, earthquake)        -> HIGH   -> 0.8
  Labor Disruption (strike, blockade, shutdown)      -> MEDIUM -> 0.5
  Infrastructure   (construction, closure, detour)   -> LOW    -> 0.2
  No relevant news                                   -> NONE   -> 0.0

---

## MODULE 5: Logistics Data Module

Purpose: Retrieve semi-static logistics risk data from the database for each city along the route.

```
sequenceDiagram
    participant BE as Backend API Server
    participant DB as PostgreSQL Database
    participant LScorer as Logistics Risk Scorer

    loop For each city in route
        BE->>DB: SELECT traffic_density, road_closure_flag, accident_index, construction_flag
                 FROM logistics_data WHERE city_id = {city_id}

        alt Record found
            DB-->>BE: { traffic_density: 0-10, road_closure_flag: 0/1,
                        accident_index: 0-10, construction_flag: 0/1 }

            BE->>LScorer: Pass logistics fields
            LScorer->>LScorer: Score computation:
                               road_closure_flag = 1  -> +0.4
                               traffic_density > 7    -> +0.3
                               accident_index > 6     -> +0.25
                               construction_flag = 1  -> +0.1
                               Normalize & clamp to [0.0, 1.0]
            LScorer-->>BE: logistics_risk_score
        else No record found
            DB-->>BE: Empty result
            BE->>BE: logistics_risk_score = 0.2 (default medium-low)
        end
    end

    BE->>BE: Aggregate per route: route_logistics_risk = AVG(city_logistics_scores)
    BE-->>BE: route_logistics_risk_score
```

Logistics Scoring Logic:
  road_closure_flag = 1 (closed)   -> +0.40
  traffic_density > 7 (scale 0-10) -> +0.30
  accident_index > 6 (scale 0-10)  -> +0.25
  construction_flag = 1 (active)   -> +0.10

---

## MODULE 6: Economic Indicator Module

Purpose: Retrieve macro-economic indicators for strategic risk context (not real-time path weighting).

```
sequenceDiagram
    participant BE as Backend API Server
    participant DB as PostgreSQL Database
    participant EScorer as Economic Risk Scorer

    Note over BE: Economic data fetched once per route (country-level, not per city)

    BE->>DB: SELECT fuel_price_index, inflation_rate, gdp_growth, last_updated
             FROM economic_indicators WHERE country = {route_country}

    alt Record found & recent (< 7 days)
        DB-->>BE: { fuel_price_index: float, inflation_rate: float(%),
                    gdp_growth: float(%), last_updated: timestamp }

        BE->>EScorer: Pass economic fields
        EScorer->>EScorer: Score computation:
                           fuel_price_index > 120  -> +0.3
                           inflation_rate > 8%     -> +0.2
                           gdp_growth < 0%         -> +0.2
                           Combined instability    -> +0.1
                           Normalize to [0.0, 1.0]
        EScorer-->>BE: economic_risk_score
    else Stale or missing data
        DB-->>BE: Old/empty record
        BE->>BE: economic_risk_score = 0.1 (low default - strategic only)
    end

    BE-->>BE: economic_risk_score (applied once to entire route)
```

Economic Scoring Logic:
  fuel_price_index > 120 (high)   -> +0.30
  inflation_rate > 8%             -> +0.20
  gdp_growth < 0% (recession)     -> +0.20
  Combined instability            -> +0.10

---

## MODULE 7: Risk Scoring Engine

Purpose: Combine all four risk scores using the weighted formula to compute a final risk score per route.

```
sequenceDiagram
    participant BE as Backend API Server
    participant RE as Risk Engine
    participant Validator as Score Validator

    BE->>RE: Submit per-route score bundle:
             {
               route_id: "A",
               weather_risk: 0.65,
               news_risk: 0.40,
               logistics_risk: 0.30,
               economic_risk: 0.15,
               city_breakdown: [...]
             }

    RE->>Validator: Validate all scores in [0.0, 1.0]
    Validator-->>RE: Validation OK / clamp outliers

    RE->>RE: Apply weighted formula:
             final_risk =
               (0.4 x weather_risk)
             + (0.3 x news_risk)
             + (0.2 x logistics_risk)
             + (0.1 x economic_risk)

             Example:
             = (0.4x0.65) + (0.3x0.40) + (0.2x0.30) + (0.1x0.15)
             = 0.26 + 0.12 + 0.06 + 0.015
             = 0.455

    RE->>RE: Classify risk level:
             0.0 - 0.3  -> LOW
             0.3 - 0.6  -> MEDIUM
             0.6 - 0.8  -> HIGH
             0.8 - 1.0  -> CRITICAL

    RE-->>BE: {
               route_id: "A",
               final_risk: 0.455,
               risk_level: "MEDIUM",
               breakdown: {
                 weather: 0.26,
                 news: 0.12,
                 logistics: 0.06,
                 economic: 0.015
               }
             }
```

Risk Formula:
  final_risk = (0.4 x weather_risk) + (0.3 x news_risk) + (0.2 x logistics_risk) + (0.1 x economic_risk)

Risk Level Classification:
  0.0 - 0.30  -> LOW      -> Recommend freely
  0.30 - 0.60 -> MEDIUM   -> Recommend with caution note
  0.60 - 0.80 -> HIGH     -> Warn user, show alternatives
  0.80 - 1.00 -> CRITICAL -> Block / strongly advise against

---

## MODULE 8: Route Ranking & Recommendation

Purpose: Filter, rank, and select the best route based on risk score, duration, and distance.

```
sequenceDiagram
    participant BE as Backend API Server
    participant Filter as Risk Filter
    participant Ranker as Route Ranker
    participant Selector as Best Route Selector

    BE->>Filter: Submit all routes with final_risk scores:
                 [
                   {id:"A", risk:0.455, duration:180, distance:320},
                   {id:"B", risk:0.720, duration:150, distance:290},
                   {id:"C", risk:0.280, duration:210, distance:380}
                 ]

    Filter->>Filter: Apply risk threshold filter:
                     Remove routes where risk > 0.75 (configurable)
    Filter-->>BE: Filtered routes:
                  [ {id:"A", risk:0.455}, {id:"C", risk:0.280} ]
                  Removed: Route B (risk=0.72 > threshold)

    BE->>Ranker: Send filtered routes
    Ranker->>Ranker: Sort by:
                     1. risk ASC (primary)
                     2. duration ASC (secondary)
                     3. distance ASC (tertiary)

    Ranker-->>BE: Ranked list:
                  [ {rank:1, id:"C", risk:0.280}, {rank:2, id:"A", risk:0.455} ]

    BE->>Selector: Select top-ranked route
    Selector->>Selector: Validate route C is viable (not CRITICAL risk, has geometry)
    Selector-->>BE: {
                      best_route: RouteC,
                      reason: "Lowest risk score",
                      alternatives: [RouteA]
                    }

    alt No routes pass filter (all high risk)
        BE->>Selector: All routes exceed threshold
        Selector-->>BE: {
                          best_route: lowest_risk_route,
                          warning: "All routes have elevated risk",
                          forced_selection: true
                        }
    end
```

Ranking Priority:
  1st -> final_risk  (Ascending - lower is better)
  2nd -> duration    (Ascending - faster is better)
  3rd -> distance    (Ascending - shorter is better)

---

## MODULE 9: Frontend Display

Purpose: Present the recommended route, risk breakdown, and map visualization to the user.

```
sequenceDiagram
    actor User
    participant FE as Frontend UI
    participant MapLib as Map Library (Leaflet/MapBox)
    participant BE as Backend API Server

    BE-->>FE: API Response:
              {
                best_route: {
                  geometry: GeoJSON,
                  distance: 380km,
                  duration: 210min,
                  risk_score: 0.280,
                  risk_level: "LOW"
                },
                risk_breakdown: {
                  weather: 0.12,
                  news: 0.08,
                  logistics: 0.06,
                  economic: 0.02
                },
                alternatives: [RouteA],
                city_details: [...]
              }

    FE->>FE: Parse API response
    FE->>FE: Validate data integrity

    FE->>MapLib: Render map with:
                 - Route polyline (best route)
                 - City markers with risk colors
                 - Source/destination pins
    MapLib-->>FE: Rendered map layer

    FE->>FE: Render Risk Dashboard:
             - Overall risk badge (LOW/MEDIUM/HIGH)
             - Risk breakdown bar chart
             - Weather / News / Logistics / Economic scores

    FE->>FE: Render Route Details Panel:
             - Distance, Duration
             - Waypoints list
             - Risk score per city

    FE->>FE: Render Alternative Routes:
             - Collapsed accordion
             - Risk comparison table

    FE-->>User: Display complete UI:
                Map | Risk Breakdown | Route Details | Risk Warnings

    User->>FE: Click alternative route
    FE->>MapLib: Update polyline to Route A
    FE->>FE: Update risk dashboard for Route A
    MapLib-->>FE: Updated map
    FE-->>User: Show Route A details

    User->>FE: Click city marker
    FE-->>User: Popup: city name, weather conditions, news headlines, logistics flags
```

UI Components & Data:
  Map Layer      <- Route GeoJSON geometry     -> Polyline, markers, pins
  Risk Badge     <- final_risk + risk_level    -> Color-coded severity
  Risk Chart     <- risk_breakdown object      -> Weighted bar chart
  Route Panel    <- distance, duration, waypoints -> Trip summary
  City Popups    <- city_details array         -> Per-city risk info
  Alternatives   <- alternatives array         -> Comparison table

---

## MODULE INTERACTION SUMMARY

```
sequenceDiagram
    participant M1 as Module 1 - Route Generation
    participant M2 as Module 2 - Waypoint Extraction
    participant M3 as Module 3 - Weather Risk
    participant M4 as Module 4 - News Risk
    participant M5 as Module 5 - Logistics Data
    participant M6 as Module 6 - Economic Indicators
    participant M7 as Module 7 - Risk Scoring Engine
    participant M8 as Module 8 - Route Ranking
    participant M9 as Module 9 - Frontend Display

    M1->>M2: Route GeoJSON + coordinates
    M2->>M3: City list per route
    M2->>M4: City list per route
    M2->>M5: City list per route
    M2->>M6: Country name
    M3-->>M7: weather_risk_score per route
    M4-->>M7: news_risk_score per route
    M5-->>M7: logistics_risk_score per route
    M6-->>M7: economic_risk_score per route
    M7->>M8: final_risk per route + breakdown
    M8->>M9: best_route + alternatives + reasoning
```

---

## DATA DICTIONARY

  source                 | string  | -         | Origin city name or lat/lon
  destination            | string  | -         | Destination city name or lat/lon
  route_geometry         | GeoJSON | -         | Polyline coordinates of route
  distance               | float   | km        | Total route distance
  duration               | float   | minutes   | Estimated travel time
  city_id                | integer | -         | DB primary key for city
  weather_risk_score     | float   | 0.0-1.0   | Normalized weather risk per city
  news_risk_score        | float   | 0.0-1.0   | Normalized news risk per city
  logistics_risk_score   | float   | 0.0-1.0   | Normalized logistics risk per city
  economic_risk_score    | float   | 0.0-1.0   | Country-level economic risk
  final_risk             | float   | 0.0-1.0   | Weighted composite risk score
  risk_level             | enum    | LOW/MEDIUM/HIGH/CRITICAL | Human-readable classification
  traffic_density        | integer | 0-10      | Road congestion level
  road_closure_flag      | boolean | 0/1       | Active road closure indicator
  accident_index         | integer | 0-10      | Historical accident frequency
  fuel_price_index       | float   | -         | Relative fuel price (100 = baseline)
  inflation_rate         | float   | %         | Annual inflation rate
  gdp_growth             | float   | %         | GDP growth rate (negative = recession)
