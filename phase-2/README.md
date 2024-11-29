# Phase 2: Session Management - Detailed Implementation Plan

## 1. Redis Cluster Implementation
- Deploy Redis to Kubernetes:
  ```yaml
  # Redis deployment with StatefulSet
  # Redis service
  # Redis config with:
    - Memory settings
    - Persistence config
    - Basic security
  ```
- Redis data structures:
  ```python
  # Active sessions: Hash
  sessions:{session_id} = {
      'player_id': str,
      'status': str,
      'rating': int,
      'timestamp': str
  }
  
  # Match queues: Sorted Set
  queue:matchmaking = {
      player_id: rating_score
  }
  
  # Player ratings: Hash
  player:{player_id}:stats = {
      'current_rating': int,
      'games_played': int,
      'win_rate': float
  }
  ```

## 2. Gaming Features Implementation

### Session Management:
```python
# New Endpoints:
POST /api/v1/sessions/create
    - Create new game session
    - Associate players
    - Set initial state

GET /api/v1/sessions/{session_id}/state
    - Get current session state
    - Include player info
    - Include timestamps

PUT /api/v1/sessions/{session_id}/update
    - Update session state
    - Handle state transitions

DELETE /api/v1/sessions/{session_id}
    - Clean up session data
    - Update player stats
```

### Player Authentication:
```python
# New Endpoints:
POST /api/v1/auth/register
    - Create new player account
    - Initial rating assignment

POST /api/v1/auth/login
    - Player authentication
    - Token generation

GET /api/v1/auth/profile
    - Get player profile
    - Include stats/history
```

### Basic Rating System:
```python
# Rating calculation:
class RatingSystem:
    def calculate_rating_change(
        player_rating: int,
        opponent_rating: int,
        outcome: float
    ) -> int:
        # Basic ELO implementation
        # Outcome: 1.0 win, 0.5 draw, 0.0 loss
        
# New Endpoints:
GET /api/v1/players/{player_id}/rating
    - Get current rating
    - Rating history

POST /api/v1/matches/result
    - Submit match result
    - Update ratings
```

### Match Queue System:
```python
# Queue management:
class MatchQueue:
    def add_to_queue(player_id: str, rating: int)
    def find_match(player_id: str) -> Optional[str]
    def remove_from_queue(player_id: str)
    
# New Endpoints:
POST /api/v1/queue/join
    - Add player to queue
    - Consider rating

DELETE /api/v1/queue/leave
    - Remove from queue

GET /api/v1/queue/status
    - Queue position
    - Estimated wait time
```

## 3. Enhanced Logging System

### Structured Logging:
```python
# Log format:
{
    "timestamp": "ISO-8601",
    "level": "INFO|ERROR|DEBUG",
    "service": "api|queue|auth",
    "event": "event_name",
    "player_id": "optional_player_id",
    "session_id": "optional_session_id",
    "details": {},
    "trace_id": "uuid"
}
```

### Debug Information:
- Queue performance metrics
- Matchmaking decisions
- Rating calculations
- Session state changes

## 4. Data Layer Implementation

### Redis Data:
- Active session management
- Queue management
- Real-time player states
- Rating caching

### MongoDB Collections:
```javascript
// Players collection
{
    _id: ObjectId,
    username: String,
    auth_data: {
        password_hash: String,
        last_login: DateTime
    },
    profile: {
        created_at: DateTime,
        games_played: Number,
        current_rating: Number
    },
    rating_history: [{
        timestamp: DateTime,
        rating: Number,
        change: Number
    }]
}

// Sessions collection
{
    _id: ObjectId,
    created_at: DateTime,
    players: [{
        player_id: ObjectId,
        rating: Number
    }],
    status: String,
    history: [{
        timestamp: DateTime,
        event: String,
        details: Object
    }]
}
```

## 5. Updated Directory Structure:
```
game-session-manager/
├── src/
│   ├── api/
│   │   ├── routes/
│   │   │   ├── auth.py
│   │   │   ├── sessions.py
│   │   │   ├── queue.py
│   │   │   └── ratings.py
│   │   └── models/
│   ├── services/
│   │   ├── auth_service.py
│   │   ├── queue_service.py
│   │   ├── rating_service.py
│   │   └── session_service.py
│   └── utils/
│       ├── redis_client.py
│       └── logging.py
├── deployment/
│   └── kubernetes/
│       ├── redis.yaml
│       └── updated-services.yaml
└── tests/
    └── phase2/
        ├── test_auth.py
        ├── test_queue.py
        └── test_ratings.py
```

## Success Criteria for Phase 2:
1. Redis cluster operational
2. All new endpoints responding correctly
3. Session management working:
   - Creation
   - State updates
   - Cleanup
4. Player authentication functional
5. Basic matchmaking working
6. Rating system operational
7. Logs showing clear system activity
8. Data persistence confirmed in both Redis and MongoDB

Ready to start any specific component of Phase 2?