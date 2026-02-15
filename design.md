# Design Document: Pollution-Aware Route Optimization Platform

## Overview

The Pollution-Aware Route Optimization Platform is a cloud-native, serverless system designed to minimize pollution exposure during travel across Indian cities. The platform integrates real-time air quality, weather, and traffic data to compute pollution exposure scores for routes and recommend cleaner alternatives.

The system architecture follows a microservices pattern deployed on AWS, leveraging serverless compute (Lambda), managed databases (DynamoDB), object storage (S3), and machine learning services (SageMaker) for spatio-temporal pollution prediction. The design prioritizes scalability, low latency, and cost-effectiveness while maintaining high availability.

Key design principles:
- **Event-driven architecture**: Data ingestion and processing triggered by scheduled events and API requests
- **Separation of concerns**: Clear boundaries between data ingestion, computation, prediction, and API layers
- **Caching strategy**: Multi-tier caching to optimize performance and reduce compute costs
- **Graceful degradation**: System continues operating with cached data when external APIs fail
- **Horizontal scalability**: Stateless services that scale automatically based on demand

## Architecture

### High-Level Architecture

The system consists of five primary layers:

1. **Data Ingestion Layer**: Fetches AQI, weather, and traffic data from third-party APIs every 15 minutes
2. **Data Storage Layer**: Stores raw and processed data in DynamoDB and S3
3. **Computation Layer**: Calculates pollution exposure scores and optimizes routes
4. **Prediction Layer**: Forecasts future pollution levels using ML models
5. **API Layer**: Exposes RESTful endpoints for external clients

**Architecture Flow**:
```
Third-Party APIs → Data Ingestion Service → Data Storage
                                                ↓
Client Request → API Gateway → Route Optimizer ← Exposure Calculator ← Current Data
                      ↓                              ↓
                 Response                    Prediction Engine ← Historical Data
```

**AWS Services Mapping**:
- **API Gateway**: AWS API Gateway with Lambda integration
- **Compute**: AWS Lambda functions for all business logic
- **Storage**: DynamoDB for operational data, S3 for historical data and ML artifacts
- **Caching**: ElastiCache (Redis) for route and data caching
- **ML**: SageMaker for training and hosting prediction models
- **Scheduling**: EventBridge for periodic data ingestion triggers
- **Monitoring**: CloudWatch for logs, metrics, and alarms
- **Notifications**: SNS for real-time alerts to users

### Data Ingestion Layer Design

**Components**:
- **Ingestion Orchestrator**: EventBridge rule triggering every 15 minutes
- **API Fetchers**: Separate Lambda functions for each data source (AQI, weather, traffic)
- **Data Validator**: Validates and cleanses incoming data
- **Data Transformer**: Normalizes data into common schema

**Data Sources**:
- AQI: Central Pollution Control Board (CPCB) API, private sensor networks
- Weather: India Meteorological Department (IMD) API, OpenWeatherMap
- Traffic: Google Maps Traffic API, TomTom Traffic API

**Ingestion Process**:
1. EventBridge triggers Ingestion Orchestrator every 15 minutes
2. Orchestrator invokes parallel Lambda functions for each data source
3. Each fetcher implements retry logic with exponential backoff (3 attempts)
4. Data Validator checks format, range, and consistency
5. Invalid data points are flagged; nearby sensor data used for interpolation
6. Transformed data written to DynamoDB with timestamp
7. Data quality metrics published to CloudWatch

**Error Handling**:
- Failed API calls logged to CloudWatch with alert thresholds
- System continues with cached data if all retries fail
- Data quality scores tracked per source; alerts triggered if score < 70%

**Data Schema** (DynamoDB):
```
Table: EnvironmentalData
Partition Key: CityId_GridCell (e.g., "DEL_28.6139_77.2090")
Sort Key: Timestamp
Attributes:
  - AQI: Number
  - PM25: Number
  - PM10: Number
  - Temperature: Number
  - Humidity: Number
  - WindSpeed: Number
  - TrafficLevel: String (LOW, MEDIUM, HIGH)
  - DataQuality: Number (0-100)
  - Source: String
```

### Pollution Scoring Engine Design

**Components**:
- **Exposure Calculator**: Computes pollution exposure for route segments
- **Route Segmenter**: Divides routes into geographic segments
- **Aggregator**: Combines segment scores into overall route exposure

**Exposure Calculation Algorithm**:
```
For each route:
  1. Divide route into segments (500m each)
  2. For each segment:
     - Identify nearest grid cell with AQI data
     - Estimate travel time through segment (distance / speed)
     - Calculate exposure = AQI × travel_time × traffic_multiplier
  3. Aggregate segment exposures:
     - Total_Exposure = Σ(segment_exposure)
     - Normalized_Score = (Total_Exposure / Max_Possible) × 100
  4. Return normalized score (0-100)
```

**Traffic Multiplier Logic**:
- LOW traffic: 1.0x (normal ventilation)
- MEDIUM traffic: 1.3x (reduced ventilation, more idling)
- HIGH traffic: 1.6x (significant idling, concentrated emissions)

**Performance Optimization**:
- Pre-compute grid cell to AQI mappings every 15 minutes
- Cache segment-level exposures for 15 minutes
- Use spatial indexing (geohash) for fast nearest-neighbor lookups

**Data Structures**:
```python
class RouteSegment:
    start_lat: float
    start_lon: float
    end_lat: float
    end_lon: float
    distance_meters: float
    estimated_time_seconds: float
    grid_cell_id: str
    aqi: float
    traffic_level: str
    exposure_score: float

class RouteExposure:
    route_id: str
    segments: List[RouteSegment]
    total_exposure: float
    normalized_score: float  # 0-100
    travel_time_seconds: float
    distance_meters: float
```

### Spatio-Temporal Prediction Model Approach

**Model Architecture**:
- **Type**: Gradient Boosting (XGBoost) with spatio-temporal features
- **Alternative**: LSTM for time-series if sufficient historical data available
- **Prediction Horizon**: 6 hours ahead, 1-hour intervals
- **Spatial Resolution**: Grid cells (1km × 1km)

**Features**:
- **Temporal**: Hour of day, day of week, month, season
- **Historical**: AQI values from past 24 hours (rolling window)
- **Spatial**: Latitude, longitude, distance to major roads, industrial zones
- **Weather**: Temperature, humidity, wind speed, wind direction, precipitation
- **Traffic**: Current and predicted traffic levels
- **Contextual**: Holidays, festivals, construction activity

**Training Process**:
1. Historical data aggregated from S3 (minimum 6 months)
2. Feature engineering pipeline in SageMaker Processing
3. Model training using SageMaker Training Jobs
4. Hyperparameter tuning with SageMaker Automatic Model Tuning
5. Model evaluation: MAE, RMSE, R² on held-out test set
6. Model deployment to SageMaker Endpoint if MAE < 15 AQI points

**Inference Process**:
1. Prediction Engine Lambda invoked every 15 minutes
2. Fetches current environmental data from DynamoDB
3. Calls SageMaker Endpoint with feature vectors for all grid cells
4. Receives predictions for next 6 hours (6 values per cell)
5. Stores predictions in DynamoDB with confidence scores
6. Flags predictions with confidence < 70% as uncertain

**Model Retraining**:
- Weekly retraining with latest data to capture seasonal patterns
- Automated pipeline triggered by EventBridge
- A/B testing of new models before full deployment

**Prediction Schema** (DynamoDB):
```
Table: PollutionPredictions
Partition Key: CityId_GridCell
Sort Key: PredictionTimestamp
Attributes:
  - PredictedAQI: Number
  - Confidence: Number (0-100)
  - ModelVersion: String
  - GeneratedAt: Timestamp
```

### Route Optimization Logic

**Components**:
- **Route Generator**: Generates candidate routes using routing engine
- **Route Ranker**: Ranks routes by exposure score and travel time
- **Trade-off Analyzer**: Evaluates time vs. exposure trade-offs

**Optimization Algorithm**:
```
Input: origin, destination, departure_time, user_preferences
Output: ranked list of routes with exposure scores

1. Generate candidate routes:
   - Use external routing API (Google Maps, Mapbox) for 3-5 alternatives
   - Include fastest route, shortest route, and alternatives

2. For each candidate route:
   - Calculate current exposure score using Exposure Calculator
   - If departure_time is future, use predicted AQI values
   - Calculate travel time considering current/predicted traffic

3. Rank routes:
   - Primary: exposure score (lower is better)
   - Secondary: travel time (lower is better)
   - Apply user preferences (max time increase threshold)

4. Filter routes:
   - Remove routes exceeding user's max time increase
   - Keep at least 2 routes (including fastest if necessary)

5. Return top routes with metadata:
   - Exposure score, travel time, distance
   - Exposure reduction % vs. fastest route
   - Time increase % vs. fastest route
   - High pollution segments highlighted
```

**User Preference Handling**:
```python
class UserPreferences:
    max_time_increase_percent: int = 30  # default
    alert_threshold_aqi: int = 150  # default
    prioritize_exposure: bool = True  # vs. time
```

**Route Comparison**:
- Calculate exposure reduction: `(fastest_exposure - route_exposure) / fastest_exposure × 100`
- Calculate time penalty: `(route_time - fastest_time) / fastest_time × 100`
- Present clear trade-offs to user

### API Design Overview

**RESTful Endpoints**:

1. **POST /routes/optimize**
   - Request body: `{origin, destination, departure_time, preferences}`
   - Response: List of ranked routes with exposure scores
   - Timeout: 3 seconds
   - Rate limit: 100 requests/minute per API key

2. **GET /routes/{route_id}/monitor**
   - Response: Current pollution levels along active route
   - Used for real-time monitoring
   - Rate limit: 20 requests/minute per route

3. **POST /routes/batch**
   - Request body: Array of up to 100 route requests
   - Response: Array of route recommendations
   - Timeout: 30 seconds
   - Rate limit: 10 requests/minute per API key

4. **GET /predictions/{city_id}**
   - Response: Predicted AQI for all grid cells in city for next 6 hours
   - Rate limit: 50 requests/minute per API key

5. **GET /reports/exposure**
   - Query params: `start_date, end_date, vehicle_id, format`
   - Response: Historical exposure report (JSON/CSV/PDF)
   - Rate limit: 10 requests/minute per API key

6. **POST /alerts/subscribe**
   - Request body: `{route_id, phone_number, alert_threshold}`
   - Response: Subscription confirmation
   - Enables real-time alerts via SNS

**Authentication**:
- API key-based authentication via API Gateway
- Keys stored in AWS Secrets Manager
- Usage tracked per key for billing

**Response Format**:
```json
{
  "request_id": "uuid",
  "routes": [
    {
      "route_id": "uuid",
      "exposure_score": 45,
      "travel_time_seconds": 1800,
      "distance_meters": 15000,
      "exposure_reduction_percent": 35,
      "time_increase_percent": 12,
      "segments": [
        {
          "start": {"lat": 28.6139, "lon": 77.2090},
          "end": {"lat": 28.6200, "lon": 77.2150},
          "aqi": 180,
          "traffic": "HIGH"
        }
      ]
    }
  ],
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Data Storage Strategy

**DynamoDB Tables**:

1. **EnvironmentalData**: Current AQI, weather, traffic (15-min retention in hot storage)
   - Partition by CityId_GridCell, sort by Timestamp
   - TTL: 7 days (then archived to S3)
   - Provisioned capacity: On-demand mode for variable load

2. **PollutionPredictions**: Forecasted AQI values
   - Partition by CityId_GridCell, sort by PredictionTimestamp
   - TTL: 24 hours
   - Provisioned capacity: On-demand mode

3. **RouteCache**: Cached route computations
   - Partition by OriginDestinationHash, sort by Timestamp
   - TTL: 15 minutes
   - Provisioned capacity: On-demand mode

4. **CompletedRoutes**: Historical route data for reporting
   - Partition by VehicleId, sort by CompletionTimestamp
   - No TTL (retained for 12 months, then archived)
   - Provisioned capacity: On-demand mode

5. **ApiKeys**: API key metadata and usage tracking
   - Partition by ApiKey
   - Attributes: OrgId, RateLimit, UsageCount, CreatedAt

**S3 Buckets**:

1. **historical-data**: Archived environmental data for ML training
   - Partitioned by year/month/day/city
   - Lifecycle policy: Transition to Glacier after 6 months

2. **ml-artifacts**: Model files, training data, evaluation metrics
   - Versioned for model lineage tracking

3. **reports**: Generated exposure reports (CSV/PDF)
   - Lifecycle policy: Delete after 90 days

**ElastiCache (Redis)**:
- **Route cache**: Frequently requested routes (15-min TTL)
- **Grid cell AQI**: Current AQI for all grid cells (15-min TTL)
- **Prediction cache**: Forecasted AQI (1-hour TTL)
- Cluster mode enabled for high availability

### Scalability Approach

**Horizontal Scaling**:
- Lambda functions scale automatically (up to 1000 concurrent executions per region)
- DynamoDB on-demand mode scales read/write capacity automatically
- ElastiCache cluster mode with multiple shards for distributed caching
- API Gateway handles up to 10,000 requests/second per region

**Performance Optimization**:
- **Caching strategy**: 3-tier (ElastiCache → DynamoDB → S3)
- **Batch processing**: Batch route requests processed in parallel
- **Async processing**: Long-running tasks (reports, predictions) processed asynchronously via SQS
- **CDN**: CloudFront for static content and API response caching

**Load Testing Targets**:
- 1000 concurrent route requests with <3s response time
- 10,000 daily route requests per city
- 100 batch requests per minute

**Auto-scaling Triggers**:
- Lambda: Automatic based on invocation rate
- DynamoDB: On-demand mode adjusts automatically
- ElastiCache: CloudWatch alarms trigger manual scaling (CPU > 80%)
- SageMaker Endpoint: Auto-scaling based on invocation rate

**Geographic Partitioning**:
- Data partitioned by city for query efficiency
- Separate DynamoDB tables per region if expanding beyond India
- Multi-region deployment for disaster recovery (future enhancement)

### Security & Compliance Considerations

**Authentication & Authorization**:
- API Gateway with API key authentication
- IAM roles for Lambda functions (least privilege principle)
- Secrets Manager for storing third-party API credentials
- API keys rotated every 90 days

**Data Protection**:
- Encryption at rest: DynamoDB and S3 use AWS KMS
- Encryption in transit: TLS 1.2+ for all API communications
- VPC: Lambda functions in private subnets with NAT Gateway for external API calls
- Security groups restrict access between services

**Compliance**:
- **Data Residency**: All data stored in AWS India regions (Mumbai, Hyderabad)
- **Privacy**: No personally identifiable information (PII) stored without consent
- **Audit Logging**: CloudTrail logs all API calls and data access
- **Data Retention**: Compliance with Indian data protection regulations

**Monitoring & Alerting**:
- CloudWatch dashboards for system health metrics
- Alarms for API latency, error rates, data quality scores
- SNS notifications to operations team for critical issues
- X-Ray for distributed tracing and performance analysis

**Disaster Recovery**:
- DynamoDB point-in-time recovery enabled
- S3 versioning and cross-region replication for critical data
- Lambda functions deployed via Infrastructure as Code (CloudFormation/Terraform)
- RTO: 4 hours, RPO: 15 minutes

### AWS Services Mapping

| Component | AWS Service | Purpose |
|-----------|-------------|---------|
| API Gateway | AWS API Gateway | RESTful API endpoints |
| Compute | AWS Lambda | Serverless business logic |
| Operational DB | DynamoDB | Real-time data storage |
| Archival Storage | S3 | Historical data, ML artifacts |
| Cache | ElastiCache (Redis) | Performance optimization |
| ML Training | SageMaker Training | Model training |
| ML Inference | SageMaker Endpoint | Real-time predictions |
| Scheduling | EventBridge | Periodic data ingestion |
| Notifications | SNS | Real-time alerts |
| Monitoring | CloudWatch | Logs, metrics, alarms |
| Tracing | X-Ray | Distributed tracing |
| Secrets | Secrets Manager | API credentials |
| Security | KMS | Encryption keys |
| Networking | VPC, NAT Gateway | Network isolation |
| Audit | CloudTrail | Compliance logging |

## Components and Interfaces

### Data Ingestion Service

**Interface**:
```python
class DataIngestionService:
    def fetch_aqi_data(city_id: str) -> List[AQIReading]:
        """Fetch AQI data from third-party APIs"""
        pass
    
    def fetch_weather_data(city_id: str) -> List[WeatherReading]:
        """Fetch weather data from third-party APIs"""
        pass
    
    def fetch_traffic_data(city_id: str) -> List[TrafficReading]:
        """Fetch traffic data from third-party APIs"""
        pass
    
    def validate_data(data: Any) -> ValidationResult:
        """Validate data format and ranges"""
        pass
    
    def transform_data(raw_data: Any) -> EnvironmentalData:
        """Transform data to common schema"""
        pass
    
    def store_data(data: EnvironmentalData) -> bool:
        """Store data in DynamoDB"""
        pass
```

**Dependencies**:
- Third-party API clients (CPCB, IMD, Google Maps)
- DynamoDB client
- CloudWatch client for metrics

### Exposure Calculator

**Interface**:
```python
class ExposureCalculator:
    def calculate_route_exposure(
        route: Route,
        departure_time: datetime,
        use_predictions: bool = False
    ) -> RouteExposure:
        """Calculate pollution exposure for a route"""
        pass
    
    def segment_route(route: Route) -> List[RouteSegment]:
        """Divide route into segments"""
        pass
    
    def get_segment_aqi(
        segment: RouteSegment,
        timestamp: datetime,
        use_predictions: bool
    ) -> float:
        """Get AQI for a segment at given time"""
        pass
    
    def calculate_segment_exposure(
        segment: RouteSegment,
        aqi: float,
        traffic_level: str
    ) -> float:
        """Calculate exposure for a single segment"""
        pass
    
    def normalize_exposure_score(total_exposure: float) -> float:
        """Normalize exposure to 0-100 scale"""
        pass
```

**Dependencies**:
- DynamoDB client (EnvironmentalData, PollutionPredictions)
- ElastiCache client for caching
- Geospatial utilities (geohash, distance calculations)

### Prediction Engine

**Interface**:
```python
class PredictionEngine:
    def generate_predictions(city_id: str) -> List[PollutionPrediction]:
        """Generate 6-hour AQI predictions for all grid cells"""
        pass
    
    def prepare_features(
        grid_cell: GridCell,
        historical_data: List[EnvironmentalData]
    ) -> FeatureVector:
        """Prepare features for ML model"""
        pass
    
    def invoke_model(features: List[FeatureVector]) -> List[Prediction]:
        """Call SageMaker endpoint for predictions"""
        pass
    
    def calculate_confidence(
        prediction: Prediction,
        historical_accuracy: float
    ) -> float:
        """Calculate confidence score for prediction"""
        pass
    
    def store_predictions(predictions: List[PollutionPrediction]) -> bool:
        """Store predictions in DynamoDB"""
        pass
```

**Dependencies**:
- SageMaker Runtime client
- DynamoDB client (EnvironmentalData, PollutionPredictions)
- S3 client for historical data

### Route Optimizer

**Interface**:
```python
class RouteOptimizer:
    def optimize_route(
        origin: Location,
        destination: Location,
        departure_time: datetime,
        preferences: UserPreferences
    ) -> List[RouteRecommendation]:
        """Generate and rank route recommendations"""
        pass
    
    def generate_candidate_routes(
        origin: Location,
        destination: Location
    ) -> List[Route]:
        """Generate alternative routes using routing engine"""
        pass
    
    def rank_routes(
        routes: List[Route],
        exposures: List[RouteExposure],
        preferences: UserPreferences
    ) -> List[RouteRecommendation]:
        """Rank routes by exposure and time"""
        pass
    
    def calculate_trade_offs(
        route: Route,
        fastest_route: Route
    ) -> TradeOffMetrics:
        """Calculate exposure reduction and time increase"""
        pass
    
    def check_cache(
        origin: Location,
        destination: Location,
        timestamp: datetime
    ) -> Optional[List[RouteRecommendation]]:
        """Check if route is cached"""
        pass
```

**Dependencies**:
- External routing API (Google Maps, Mapbox)
- ExposureCalculator
- ElastiCache client for caching
- DynamoDB client (RouteCache)

### Alert Service

**Interface**:
```python
class AlertService:
    def monitor_active_route(route_id: str) -> MonitoringResult:
        """Monitor pollution levels on active route"""
        pass
    
    def check_alert_conditions(
        current_aqi: float,
        baseline_aqi: float,
        threshold: float
    ) -> bool:
        """Check if alert should be triggered"""
        pass
    
    def send_alert(
        user_id: str,
        route_id: str,
        alert_type: str,
        alternative_route: Optional[Route]
    ) -> bool:
        """Send alert via SNS"""
        pass
    
    def subscribe_to_alerts(
        user_id: str,
        route_id: str,
        phone_number: str,
        threshold: float
    ) -> str:
        """Subscribe user to route alerts"""
        pass
```

**Dependencies**:
- SNS client for notifications
- DynamoDB client (EnvironmentalData)
- RouteOptimizer for alternative routes

## Data Models

### Core Data Models

```python
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional

@dataclass
class Location:
    latitude: float
    longitude: float

@dataclass
class AQIReading:
    grid_cell_id: str
    timestamp: datetime
    aqi: float
    pm25: float
    pm10: float
    source: str
    data_quality: float

@dataclass
class WeatherReading:
    grid_cell_id: str
    timestamp: datetime
    temperature: float
    humidity: float
    wind_speed: float
    wind_direction: float
    precipitation: float

@dataclass
class TrafficReading:
    grid_cell_id: str
    timestamp: datetime
    traffic_level: str  # LOW, MEDIUM, HIGH
    average_speed: float

@dataclass
class EnvironmentalData:
    city_id: str
    grid_cell_id: str
    timestamp: datetime
    aqi: float
    pm25: float
    pm10: float
    temperature: float
    humidity: float
    wind_speed: float
    traffic_level: str
    data_quality: float
    source: str

@dataclass
class RouteSegment:
    segment_id: str
    start: Location
    end: Location
    distance_meters: float
    estimated_time_seconds: float
    grid_cell_id: str
    aqi: float
    traffic_level: str
    exposure_score: float

@dataclass
class Route:
    route_id: str
    origin: Location
    destination: Location
    waypoints: List[Location]
    distance_meters: float
    estimated_time_seconds: float
    polyline: str  # Encoded polyline

@dataclass
class RouteExposure:
    route_id: str
    segments: List[RouteSegment]
    total_exposure: float
    normalized_score: float  # 0-100
    travel_time_seconds: float
    distance_meters: float
    calculated_at: datetime

@dataclass
class PollutionPrediction:
    grid_cell_id: str
    prediction_timestamp: datetime
    predicted_aqi: float
    confidence: float
    model_version: str
    generated_at: datetime

@dataclass
class UserPreferences:
    max_time_increase_percent: int = 30
    alert_threshold_aqi: int = 150
    prioritize_exposure: bool = True

@dataclass
class TradeOffMetrics:
    exposure_reduction_percent: float
    time_increase_percent: float
    distance_increase_percent: float

@dataclass
class RouteRecommendation:
    route: Route
    exposure: RouteExposure
    trade_offs: TradeOffMetrics
    rank: int
    is_fastest: bool
    is_cleanest: bool

@dataclass
class CompletedRoute:
    route_id: str
    vehicle_id: str
    origin: Location
    destination: Location
    departure_time: datetime
    arrival_time: datetime
    actual_exposure: float
    baseline_exposure: float  # Fastest route exposure
    exposure_reduction_percent: float
    distance_meters: float

@dataclass
class ExposureReport:
    organization_id: str
    start_date: datetime
    end_date: datetime
    total_routes: int
    total_exposure: float
    baseline_exposure: float
    exposure_reduction_percent: float
    routes: List[CompletedRoute]
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

The following properties are derived from the acceptance criteria in the requirements document and represent universal rules that must hold across all valid inputs and system states.

### Property 1: Retry Logic with Exponential Backoff

*For any* third-party API call that fails, the system should retry exactly 3 times with exponentially increasing delays between attempts.

**Validates: Requirements 1.2**

### Property 2: Graceful Degradation on API Failure

*For any* third-party API that fails after all retries, the system should log the failure and continue operating using cached data without throwing exceptions.

**Validates: Requirements 1.3**

### Property 3: Input Validation Rejects Invalid Data

*For any* incoming data (AQI, weather, traffic), the system should reject data that is malformed or outside valid ranges (e.g., AQI must be 0-500), and accept data within valid ranges.

**Validates: Requirements 1.5, 9.1**

### Property 4: Temporal Data Includes Timestamps

*For any* environmental data stored in the system, the stored record should include a valid timestamp indicating when the data was collected.

**Validates: Requirements 1.6**

### Property 5: Exposure Score Computed for All Routes

*For any* set of candidate routes between an origin and destination, the exposure calculator should compute an exposure score for each route.

**Validates: Requirements 2.1**

### Property 6: Exposure Aggregation Weighted by Time

*For any* route with multiple segments, the total exposure should be the sum of segment exposures, where each segment's exposure is proportional to the time spent in that segment and the AQI level.

**Validates: Requirements 2.2**

### Property 7: Exposure Calculation Uses All Data Sources

*For any* route exposure calculation, changing the AQI, weather, or traffic data should affect the computed exposure score (demonstrating all data sources are incorporated).

**Validates: Requirements 2.3**

### Property 8: Exposure Score Range Invariant

*For any* route, the normalized exposure score should always be between 0 and 100 inclusive.

**Validates: Requirements 2.4**

### Property 9: Routes Ranked by Exposure Score

*For any* set of routes between the same origin and destination, the routes should be ordered such that route[i].exposure_score ≤ route[i+1].exposure_score.

**Validates: Requirements 2.5**

### Property 10: Predictions Generated for All Grid Cells

*For any* city in the system, when predictions are generated, every grid cell in that city should have predictions for the next 6 hours.

**Validates: Requirements 3.1**

### Property 11: Predictions Use All Required Features

*For any* prediction, changing historical AQI data, weather forecasts, or traffic patterns should affect the predicted AQI value (demonstrating all features are used).

**Validates: Requirements 3.2**

### Property 12: Low Confidence Predictions Flagged

*For any* prediction with confidence below 70%, the prediction should be flagged as uncertain; predictions with confidence ≥ 70% should not be flagged.

**Validates: Requirements 3.4**

### Property 13: Prediction Accuracy Metrics Stored

*For any* batch of predictions generated, accuracy metrics (MAE, RMSE) should be stored in the system for monitoring purposes.

**Validates: Requirements 3.6**

### Property 14: Alternative Routes Include Comparison Metrics

*For any* alternative route presented to the user, the response should include time difference and exposure reduction percentage compared to the fastest route.

**Validates: Requirements 4.1, 4.2**

### Property 15: High Time-Increase Routes Still Presented

*For any* route that increases travel time by more than 30% but reduces exposure, the route should still be included in the results with trade-off information.

**Validates: Requirements 4.3**

### Property 16: Minimum Route Count When Alternatives Exist

*For any* origin-destination pair where multiple paths exist, the system should return at least 2 alternative routes.

**Validates: Requirements 4.4**

### Property 17: High Pollution Scenario Handling

*For any* route request where all available routes have exposure scores above 75 (high pollution), the system should recommend the route with the lowest exposure and issue a high-exposure alert.

**Validates: Requirements 4.5**

### Property 18: User Time Preferences Respected

*For any* route that exceeds the user's configured maximum time increase threshold, the route should be filtered from results unless it's the only available option.

**Validates: Requirements 4.6**

### Property 19: Alert Triggered on Significant Pollution Increase

*For any* active route being monitored, if the current AQI increases by more than 50 points compared to the baseline, an alert should be sent to the user.

**Validates: Requirements 5.1**

### Property 20: Alerts Include Alternative Routes When Available

*For any* alert triggered due to high pollution, if an alternative route exists, the alert should include that alternative route recommendation.

**Validates: Requirements 5.3**

### Property 21: All-Clear Notification on Pollution Decrease

*For any* active route where an alert was previously sent, if pollution levels return to normal (decrease by more than 50 points), an all-clear notification should be sent.

**Validates: Requirements 5.5**

### Property 22: Custom Alert Thresholds Respected

*For any* user with a custom alert threshold configured, alerts should be triggered based on that threshold rather than the default 50 AQI point increase.

**Validates: Requirements 5.6**

### Property 23: API Authentication Enforced

*For any* API request without a valid API key, the system should return an authentication error (401); requests with valid API keys should be processed.

**Validates: Requirements 6.2**

### Property 24: API Response Format Validity

*For any* successful route request, the response should be valid JSON that can be parsed without errors.

**Validates: Requirements 6.3**

### Property 25: Rate Limiting Enforced

*For any* API key that exceeds its rate limit, subsequent requests should return HTTP 429 status with a retry-after header.

**Validates: Requirements 6.4**

### Property 26: Batch Request Processing

*For any* batch route request with up to 100 origin-destination pairs, the system should process all pairs and return results for each.

**Validates: Requirements 6.5**

### Property 27: Request Logging for All API Calls

*For any* API request received, a corresponding log entry should be created with request details, timestamp, and API key.

**Validates: Requirements 6.6**

### Property 28: Report Generation with Aggregation Options

*For any* reporting request, the system should be able to generate reports aggregated by time period, vehicle ID, or route, based on the request parameters.

**Validates: Requirements 7.2**

### Property 29: Exposure Reduction Calculation

*For any* completed route, the system should calculate the exposure reduction percentage as: (baseline_exposure - actual_exposure) / baseline_exposure × 100.

**Validates: Requirements 7.3**

### Property 30: Report Export Format Support

*For any* exposure report, the system should be able to export it in both CSV and PDF formats, with both exports containing the same data.

**Validates: Requirements 7.4**

### Property 31: Historical Data Filtering

*For any* historical data query with filters (date range, vehicle ID, geographic region), the returned data should only include records matching all specified filters.

**Validates: Requirements 7.6**

### Property 32: Outlier Detection and Flagging

*For any* data point that deviates more than 3 standard deviations from recent historical values, the system should flag it as an outlier.

**Validates: Requirements 9.2**

### Property 33: Fallback to Interpolated Values

*For any* data validation failure, the system should use interpolated values from nearby sensors rather than leaving gaps in the data.

**Validates: Requirements 9.3**

### Property 34: Data Quality Score Maintenance

*For any* data source, the system should maintain and update a quality score based on validation success rate and data completeness.

**Validates: Requirements 9.4**

### Property 35: Quality Score Alert Threshold

*For any* data source whose quality score falls below 70%, an alert should be sent to administrators.

**Validates: Requirements 9.5**

### Property 36: Validation Failure Logging

*For any* data validation failure, a log entry should be created with details of the failure, the invalid data, and the timestamp.

**Validates: Requirements 9.6**

### Property 37: Route Caching Behavior

*For any* computed route, the route should be stored in cache with a TTL of 15 minutes, and subsequent requests for the same origin-destination pair should return the cached result if still valid.

**Validates: Requirements 10.1, 10.2**

### Property 38: Cache Invalidation on Data Changes

*For any* cached route, if the underlying pollution data changes by more than 20 AQI points, the cached route should be invalidated and recomputed on next request.

**Validates: Requirements 10.3**

### Property 39: LRU Cache Eviction

*For any* cache that reaches 90% capacity, adding a new entry should evict the least-recently-used entry to make space.

**Validates: Requirements 10.5**

## Error Handling

The system implements comprehensive error handling across all layers to ensure reliability and graceful degradation:

### Data Ingestion Layer Errors

**Third-Party API Failures**:
- **Retry Strategy**: Exponential backoff with 3 attempts (delays: 1s, 2s, 4s)
- **Fallback**: Use cached data from previous successful ingestion
- **Logging**: Log all failures with API endpoint, error message, and timestamp
- **Alerting**: Alert operations team if failure rate exceeds 20% over 1 hour

**Data Validation Errors**:
- **Invalid Format**: Reject data and log validation error
- **Out of Range**: Flag as invalid, use interpolated value from nearby sensors
- **Missing Fields**: Attempt to use default values or previous reading
- **Outliers**: Flag for review but include in dataset with quality score penalty

**Rate Limiting by Third-Party APIs**:
- **Detection**: Monitor for 429 status codes
- **Response**: Respect retry-after headers, adjust polling frequency
- **Mitigation**: Distribute requests across multiple API keys if available

### Computation Layer Errors

**Missing Environmental Data**:
- **Scenario**: No AQI data available for a grid cell
- **Handling**: Use interpolated value from nearest 3 grid cells with data
- **Fallback**: If no nearby data, use city-wide average
- **User Notification**: Flag route as "estimated" in response

**Route Generation Failures**:
- **Scenario**: External routing API fails or returns no routes
- **Handling**: Retry with alternative routing provider
- **Fallback**: Return error to user with suggestion to try again
- **Logging**: Log routing API failures for monitoring

**Exposure Calculation Errors**:
- **Division by Zero**: Handle routes with zero travel time (same origin/destination)
- **Null Values**: Validate all inputs before calculation, use defaults if needed
- **Overflow**: Cap exposure scores at maximum value (100)

### Prediction Layer Errors

**Model Inference Failures**:
- **Scenario**: SageMaker endpoint unavailable or times out
- **Handling**: Retry inference request up to 2 times
- **Fallback**: Use historical average AQI for the time period
- **User Impact**: Flag predictions as "unavailable" in route recommendations

**Low Confidence Predictions**:
- **Scenario**: Model returns confidence < 70%
- **Handling**: Flag prediction as uncertain, still use but with warning
- **User Notification**: Indicate "prediction uncertainty" in route response

**Feature Preparation Errors**:
- **Missing Features**: Use default values or historical averages
- **Invalid Features**: Log error and skip prediction for affected grid cells

### API Layer Errors

**Authentication Errors**:
- **Invalid API Key**: Return 401 Unauthorized with error message
- **Expired API Key**: Return 401 with message to renew key
- **Missing API Key**: Return 401 with instructions

**Rate Limiting Errors**:
- **Exceeded Limit**: Return 429 Too Many Requests with retry-after header
- **Calculation**: Track requests per API key in Redis with sliding window

**Validation Errors**:
- **Invalid Parameters**: Return 400 Bad Request with specific field errors
- **Missing Required Fields**: Return 400 with list of missing fields
- **Invalid Coordinates**: Return 400 with valid range information

**Timeout Errors**:
- **Scenario**: Request processing exceeds 3-second timeout
- **Handling**: Return 504 Gateway Timeout
- **Mitigation**: Use cached results when available, optimize slow queries

**Internal Server Errors**:
- **Scenario**: Unexpected exceptions in Lambda functions
- **Handling**: Catch all exceptions, log stack trace, return 500
- **Alerting**: Alert operations team for 5xx error rate > 1%

### Storage Layer Errors

**DynamoDB Errors**:
- **Throttling**: Implement exponential backoff, use on-demand mode to auto-scale
- **Item Not Found**: Return appropriate error to caller, don't expose internal details
- **Conditional Check Failures**: Retry with updated data

**S3 Errors**:
- **Access Denied**: Log error, alert operations team
- **Object Not Found**: Handle gracefully, use defaults or skip operation
- **Upload Failures**: Retry up to 3 times with exponential backoff

**ElastiCache Errors**:
- **Connection Failures**: Log error, continue without cache (direct DB access)
- **Cache Miss**: Compute value and store in cache for future requests
- **Eviction**: Normal operation, recompute on next request

### Monitoring and Alerting

**CloudWatch Alarms**:
- API error rate > 5% for 5 minutes
- Lambda function errors > 10 per minute
- DynamoDB throttling events > 0
- Data ingestion failures > 20% for 1 hour
- Cache hit rate < 30% for 1 hour
- Model inference latency > 2 seconds

**Error Metrics**:
- Error count by type (authentication, validation, timeout, internal)
- Error rate by API endpoint
- Failed API calls by third-party provider
- Data quality score by source

## Testing Strategy

The testing strategy employs a dual approach combining unit tests for specific scenarios and property-based tests for universal correctness guarantees.

### Property-Based Testing

**Framework**: Use `hypothesis` (Python) or `fast-check` (TypeScript/JavaScript) for property-based testing.

**Configuration**:
- Minimum 100 iterations per property test
- Each test tagged with: `Feature: pollution-aware-routing, Property {N}: {property_text}`
- Seed-based reproducibility for failed test cases

**Test Categories**:

1. **Data Validation Properties** (Properties 3, 4, 32, 36)
   - Generate random valid and invalid environmental data
   - Verify validation logic correctly accepts/rejects data
   - Test edge cases: boundary values, null values, extreme outliers

2. **Exposure Calculation Properties** (Properties 5, 6, 7, 8)
   - Generate random routes with varying segments
   - Verify exposure scores are computed correctly
   - Test invariants: score range 0-100, proportionality to time/AQI

3. **Route Optimization Properties** (Properties 9, 14, 15, 16, 17, 18)
   - Generate random origin-destination pairs and candidate routes
   - Verify ranking, filtering, and comparison logic
   - Test edge cases: single route, all high-pollution routes

4. **Prediction Properties** (Properties 10, 11, 12, 13)
   - Generate random grid cells and environmental data
   - Verify predictions are generated for all cells
   - Test confidence flagging and metric storage

5. **Alert Properties** (Properties 19, 20, 21, 22)
   - Generate random pollution level changes
   - Verify alert triggering logic and thresholds
   - Test custom user preferences

6. **API Properties** (Properties 23, 24, 25, 26, 27)
   - Generate random API requests (valid and invalid)
   - Verify authentication, rate limiting, response format
   - Test batch processing with varying sizes

7. **Caching Properties** (Properties 37, 38, 39)
   - Generate random cache operations
   - Verify TTL behavior, invalidation, and eviction
   - Test cache hit/miss scenarios

8. **Error Handling Properties** (Properties 2, 33)
   - Generate random failure scenarios
   - Verify graceful degradation and fallback behavior
   - Test retry logic with exponential backoff

### Unit Testing

Unit tests complement property tests by focusing on specific examples, integration points, and edge cases.

**Test Coverage Areas**:

1. **Data Ingestion**
   - Test specific API response formats from CPCB, IMD, Google Maps
   - Test retry logic with mocked API failures
   - Test data transformation for each data source

2. **Exposure Calculation**
   - Test specific routes with known AQI values
   - Test traffic multiplier application
   - Test segment division for various route lengths

3. **Prediction Engine**
   - Test feature preparation with sample data
   - Test SageMaker endpoint invocation (mocked)
   - Test confidence calculation logic

4. **Route Optimization**
   - Test specific origin-destination pairs in known cities
   - Test trade-off calculation with sample routes
   - Test user preference filtering

5. **API Layer**
   - Test each endpoint with sample requests
   - Test error responses for various failure scenarios
   - Test JSON serialization/deserialization

6. **Caching**
   - Test cache key generation
   - Test TTL expiration
   - Test cache invalidation triggers

**Integration Testing**:
- End-to-end tests for complete route optimization flow
- Test with real AWS services in staging environment
- Test data ingestion from actual third-party APIs (with test accounts)
- Test SageMaker model inference with deployed model

**Performance Testing**:
- Load test API endpoints with 1000 concurrent requests
- Measure response times under various load conditions
- Test auto-scaling behavior
- Verify cache hit rate under realistic traffic patterns

**Test Execution**:
- Unit tests: Run on every commit (CI/CD pipeline)
- Property tests: Run on every commit (may take longer)
- Integration tests: Run on every merge to main branch
- Performance tests: Run weekly and before major releases

**Test Data**:
- Use anonymized production data for realistic testing
- Generate synthetic data for edge cases
- Maintain test fixtures for consistent unit testing

### Example Property Test

```python
from hypothesis import given, strategies as st
import pytest

@given(
    aqi=st.floats(min_value=0, max_value=500),
    travel_time=st.floats(min_value=0, max_value=3600),
    traffic_level=st.sampled_from(['LOW', 'MEDIUM', 'HIGH'])
)
def test_exposure_score_range_invariant(aqi, travel_time, traffic_level):
    """
    Feature: pollution-aware-routing, Property 8: Exposure Score Range Invariant
    
    For any route, the normalized exposure score should always be between 0 and 100.
    """
    route = create_test_route(aqi, travel_time, traffic_level)
    exposure_calculator = ExposureCalculator()
    
    exposure = exposure_calculator.calculate_route_exposure(route)
    
    assert 0 <= exposure.normalized_score <= 100, \
        f"Exposure score {exposure.normalized_score} outside valid range [0, 100]"
```

## Future Enhancements

1. **Multi-Modal Routing**: Extend to include public transit, walking, cycling with pollution exposure
2. **Personal Health Integration**: Integrate with health apps to adjust recommendations based on individual health conditions
3. **Real-Time Sensor Network**: Deploy proprietary sensors to increase data coverage and accuracy
4. **Carbon Footprint Tracking**: Add CO2 emissions tracking alongside pollution exposure
5. **Predictive Maintenance**: Use pollution exposure data to predict vehicle maintenance needs
6. **Social Features**: Allow users to share cleaner routes and contribute to community air quality awareness
7. **Government Integration**: Partner with city governments to influence traffic management based on pollution patterns
8. **Advanced ML Models**: Implement deep learning models (LSTM, Transformers) for improved prediction accuracy
9. **Mobile SDK**: Provide native SDKs for iOS and Android for easier integration
10. **Gamification**: Reward users for choosing cleaner routes with points/badges for ESG reporting
