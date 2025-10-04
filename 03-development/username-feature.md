# Username Feature Implementation

## Overview

The username feature allows users to set personalized usernames for their wallet addresses, replacing truncated addresses throughout the application with human-readable names. The system uses Redis for fast key-value storage and supports both local development (Homebrew Redis) and production (Upstash Redis).

## Architecture

### Backend

#### Redis Client (`backend/shared/redisClient.js`)
- Singleton pattern for connection management
- Supports local development (`redis://localhost:6379`) and production (Upstash with TLS)
- Automatic reconnection with exponential backoff
- Graceful shutdown integration with Fastify server

#### Username Service (`backend/shared/usernameService.js`)
- Business logic for username operations
- Key schema:
  - Primary: `wallet:{address}` → `{username}`
  - Reverse lookup: `username:{username}` → `{address}`
- Validation rules:
  - Length: 3-20 characters
  - Characters: Alphanumeric + underscore only (`/^[a-zA-Z0-9_]{3,20}$/`)
  - Reserved words: `admin`, `system`, `null`, `undefined`, `root`, `moderator`
- Case-insensitive uniqueness checking

#### API Routes (`backend/fastify/routes/usernameRoutes.js`)

**Endpoints:**

```
GET /api/usernames/:address
```
Get username for a wallet address.

**Response:**
```json
{
  "address": "0x123...",
  "username": "alice" | null
}
```

---

```
POST /api/usernames
```
Set username for a wallet address.

**Request:**
```json
{
  "address": "0x123...",
  "username": "alice"
}
```

**Response:**
```json
{
  "success": true,
  "address": "0x123...",
  "username": "alice"
}
```

**Error Codes:**
- `400` - Invalid address or username format
- `409` - Username already taken

---

```
GET /api/usernames/check/:username
```
Check if username is available.

**Response:**
```json
{
  "available": true,
  "username": "alice"
}
```

---

```
GET /api/usernames/batch?addresses=0x123,0x456
```
Get usernames for multiple addresses (batch lookup).

**Response:**
```json
{
  "0x123...": "alice",
  "0x456...": null
}
```

---

```
GET /api/usernames/all
```
Get all username mappings (admin/debug endpoint).

**Response:**
```json
{
  "count": 42,
  "usernames": [
    {"address": "0x123...", "username": "alice"},
    {"address": "0x456...", "username": "bob"}
  ]
}
```

### Frontend

#### Hooks

**`src/hooks/useUsername.js`**
- `useUsername(address)` - Fetch username for an address
- `useSetUsername()` - Mutation to set username
- `useCheckUsername(username)` - Check username availability
- React Query caching with 5-minute stale time

**`src/hooks/useBatchUsernames.js`**
- `useBatchUsernames(addresses)` - Efficient batch lookup
- Automatically filters invalid addresses
- Used for user lists and market cards

#### Context

**`src/context/UsernameContext.jsx`**
- Global state for username management
- Automatically triggers username dialog on first wallet connection
- Provides `showDialog`, `setShowDialog`, `username`, `isLoading`, `hasCheckedUsername`

#### Components

**`src/components/user/UsernameDialog.jsx`**
- Modal dialog for setting/editing username
- Real-time availability checking with 500ms debounce
- Character counter (3-20 chars)
- Validation feedback with color-coded messages
- "Skip for now" option
- Full i18n support

**`src/components/user/UsernameDisplay.jsx`**
- Reusable component for displaying usernames
- Props:
  - `address` (required) - Wallet address
  - `linkTo` (optional) - Link destination
  - `showBadge` (optional) - Show "You" badge for current user
  - `className` (optional) - Additional CSS classes
- Falls back to formatted address if no username
- Handles loading states

## UI Integration

### Locations Updated

1. **Header/Wallet Connection** (`src/components/wallet/WalletConnection.jsx`)
   - Shows username instead of truncated address
   - Format: "alice.eth" vs "0x1234...5678"

2. **User List** (`src/routes/UsersIndex.jsx`)
   - Displays usernames with "You" badge for current user
   - Falls back to formatted addresses

3. **InfoFi Market Cards** (`src/components/infofi/InfoFiMarketCard.jsx`)
   - Market titles show usernames: "Will alice.eth win Raffle Season #1?"
   - Clickable username links to user profile

4. **Account/Profile Page** (`src/routes/UserProfile.jsx`)
   - Username displayed prominently as page title
   - Wallet address shown as subtitle
   - "Edit Username" / "Set Username" button for own account
   - "You" badge when viewing own profile

## Internationalization (i18n)

All UI text is translatable via `public/locales/en/common.json`:

```json
{
  "username": "Username",
  "setUsername": "Set Username",
  "editUsername": "Edit Username",
  "changeUsername": "Change Username",
  "usernameDialogDescription": "Choose a unique username...",
  "usernamePlaceholder": "Enter username (3-20 characters)",
  "skipForNow": "Skip for now",
  "usernameSet": "Username set to {{username}}",
  "usernameTooShort": "Username must be at least 3 characters",
  "usernameTooLong": "Username must be 20 characters or less",
  "usernameInvalidChars": "Username can only contain letters, numbers, and underscores",
  "usernameAvailable": "Username is available!",
  "usernameNotAvailable": "Username is not available",
  "checkingAvailability": "Checking availability...",
  "usernameError": {
    "USERNAME_REQUIRED": "Username is required",
    "USERNAME_TOO_SHORT": "Username must be at least 3 characters",
    "USERNAME_TOO_LONG": "Username must be 20 characters or less",
    "USERNAME_INVALID_CHARS": "Username can only contain letters, numbers, and underscores",
    "USERNAME_RESERVED": "This username is reserved",
    "USERNAME_TAKEN": "Username is already taken",
    "INTERNAL_ERROR": "An error occurred. Please try again.",
    "UNKNOWN_ERROR": "An unknown error occurred"
  }
}
```

**Note:** Username values themselves are NOT translated - only UI labels and messages.

## Environment Configuration

### Local Development

```env
# .env
REDIS_URL=redis://localhost:6379
```

**Prerequisites:**
```bash
# Install Redis via Homebrew (macOS)
brew install redis

# Start Redis
brew services start redis

# Verify
redis-cli ping  # Should return PONG
```

### Production (Upstash)

```env
# .env.production
REDIS_URL=rediss://default:YOUR_PASSWORD@YOUR_ENDPOINT.upstash.io:6379
REDIS_TOKEN=YOUR_UPSTASH_REST_TOKEN  # Optional, for REST SDK
```

**Setup:**
1. Create Upstash Redis database at [console.upstash.com](https://console.upstash.com)
2. Copy connection URL (use `rediss://` with double 's' for TLS)
3. Add to environment variables in hosting platform (Vercel, Netlify, etc.)

See [Redis Setup Guide](redis-setup-guide.md) for detailed instructions.

## Testing

### Backend API Tests

Location: `tests/api/usernameRoutes.test.js`

**Test Coverage:**
- GET username for non-existent address (returns null)
- GET username with invalid address format (400 error)
- POST valid username (success)
- POST username too short (400 error)
- POST username too long (400 error)
- POST username with invalid characters (400 error)
- Check availability for new username (available)
- Check availability for invalid username (not available)
- Batch lookup for multiple addresses
- Batch lookup with invalid addresses (400 error)

**Run tests:**
```bash
npm test tests/api/usernameRoutes.test.js
```

### Manual Testing

1. **Connect wallet** - Dialog should appear if no username set
2. **Set username** - Enter valid username, verify availability check works
3. **View in header** - Username should appear in wallet connection display
4. **View in user list** - Navigate to /users, verify usernames display
5. **View in market cards** - Check InfoFi markets show usernames
6. **Edit username** - Go to /account, click "Edit Username", change username
7. **Skip username** - Connect new wallet, click "Skip for now", verify no errors

## Performance Considerations

### Caching Strategy
- React Query caches usernames for 5 minutes
- Batch lookups reduce API calls for user lists
- Redis provides sub-millisecond response times

### Optimization Tips
- Use `useBatchUsernames` for lists with multiple addresses
- Avoid calling `useUsername` in tight loops
- Consider implementing username prefetching for known addresses

## Security Considerations

### Validation
- All usernames validated server-side (never trust client)
- SQL injection not applicable (Redis key-value store)
- XSS prevention through React's automatic escaping

### Rate Limiting
- Fastify rate limiting applies to all username endpoints
- Default: 100 requests per minute per IP
- Adjust in `backend/fastify/server.js` if needed

### Reserved Usernames
- Prevents impersonation of system accounts
- List maintained in `usernameService.js`
- Add more reserved words as needed

## Troubleshooting

### Redis Connection Issues

**Local Development:**
```bash
# Check if Redis is running
brew services list | grep redis

# Restart Redis
brew services restart redis

# Check logs
tail -f /usr/local/var/log/redis.log
```

**Production (Upstash):**
- Verify connection URL uses `rediss://` (with double 's')
- Check password has no trailing spaces
- Verify network/firewall allows connections
- Test connection from hosting platform's shell

### Username Not Appearing

1. Check browser console for API errors
2. Verify Redis connection in backend logs
3. Check React Query DevTools for cache status
4. Ensure `UsernameProvider` is in provider tree
5. Verify API endpoint is accessible (`/api/usernames/:address`)

### Dialog Not Showing

1. Check `UsernameContext` is properly initialized
2. Verify wallet is connected
3. Check if username already exists for address
4. Look for JavaScript errors in console

## Future Enhancements

See "Optional Username Enhancements (Future Work)" section in `instructions/project-tasks.md`:

- Username history tracking
- Username search functionality
- Verification badges
- Username expiry/reclamation
- ENS integration
- Username analytics

## Related Documentation

- [Redis Setup Guide](redis-setup-guide.md) - Local and production Redis configuration
- [Frontend Guidelines](frontend-guidelines.md) - React/UI development standards
- [Project Tasks](../instructions/project-tasks.md) - Implementation tracking

---

**Last Updated:** 2025-10-04
