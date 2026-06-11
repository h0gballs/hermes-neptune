# Neptune's Pride API (v0.1)

## Request
URL: `https://np.ironhelmet.com/api`
Method: `GET`

Query Parameters:
- `api_version`: `0.1`
- `game_number`: integer string
- `code`: 12-character API key

## Response (scanning_data)
The API returns a JSON object. If successful, the key `scanning_data` contains:
- `name`: Game name
- `tick`: Current game tick
- `paused`: Boolean
- `players`: Dict of UIDs -> Player objects
  - `alias`: Display name
  - `totalStars`: Number of stars owned
  - `totalStrength`: Total ships
  - `cash`: Current balance (visible only for 'me')
- `stars`: Dict of UIDs -> Star objects
- `fleets`: Dict of UIDs -> Fleet objects
- `puid`: The player UID belonging to the API key used
