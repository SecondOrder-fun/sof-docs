# Username Feature Testing Guide

## Quick Start Testing

### Prerequisites

1. **Redis Installed** (via Homebrew)

   ```bash
   brew install redis  # If not already installed
   ```

   > **Note:** The backend script will automatically start Redis if it's not running.

2. **Environment Configured**

   ```bash
   # Verify .env has Redis URL
   grep REDIS_URL .env
   # Should show: REDIS_URL=redis://localhost:6379
   ```

3. **Dependencies Installed**

   ```bash
   npm install  # Ensures axios and ioredis are installed
   ```

---

## Manual Testing Workflow

### Test 1: Backend API (Without Frontend)

**Start Backend (Automated Redis Startup):**
```bash
npm run dev:backend
# The script will:
# 1. Check if Redis is running
# 2. Start Redis via Homebrew if needed
# 3. Start the backend server
# Look for: "✅ Redis connection verified" and "HTTP server listening on port 3000"
```

**Run Test Script:**
```bash
# In another terminal
node scripts/test-username-api.js
```

**Expected Output:**
```
🧪 Testing Username API...

1️⃣ GET username for address (should be null)
   Response: { address: '0x742...', username: null }
   ✅ GET username works

2️⃣ Check username availability
   Response: { available: true, username: 'alice_test_...' }
   ✅ Check availability works

3️⃣ POST set username
   Response: { success: true, address: '0x742...', username: 'alice_test_...' }
   ✅ Set username works

4️⃣ GET username again (should return set username)
   Response: { address: '0x742...', username: 'alice_test_...' }
   ✅ Username persisted correctly

5️⃣ Try to set duplicate username (should fail)
   Response: { error: 'USERNAME_TAKEN' }
   ✅ Duplicate username rejected correctly

6️⃣ Batch username lookup
   Response: { '0x742...': 'alice_test_...', '0x123...': null }
   ✅ Batch lookup works

🎉 All tests passed!
```

### Test 2: Automated Backend Tests

```bash
npm test tests/api/usernameRoutes.test.js -- --run
```

**Expected:** ✅ 10/10 tests passing

### Test 3: Frontend Component Tests

```bash
npm test tests/components/UsernameDialog.test.jsx -- --run
npm test tests/components/UsernameDisplay.test.jsx -- --run
```

### Test 4: Full Stack Integration

**1. Start Backend:**
```bash
npm run dev:backend
# Redis will start automatically if needed
```

**2. Start Frontend:**
```bash
npm run dev:frontend
# Or just: npm run dev
```

**3. Test User Flow:**
- Open `http://localhost:5173`
- Click "Connect" wallet
- Username dialog should appear automatically
- Enter username (e.g., "alice")
- Watch real-time availability checking
- Submit username
- See username in header (top right)
- Navigate to `/users` - see username in list
- Navigate to `/account` - see username as title with Edit button

---

## Using Redis MCP (Windsurf)

### Verify Data Storage

**In Windsurf Cascade, ask:**

```
Get all keys matching pattern "wallet:*"
```

**Expected Response:**
```
wallet:0x742d35cc6634c0532925a3b844bc9e7595f0beb
```

**Get specific username:**
```
Get value for key "wallet:0x742d35cc6634c0532925a3b844bc9e7595f0beb"
```

**Expected Response:**
```
alice_test_1234567890
```

**Check reverse lookup:**
```
Get value for key "username:alice_test_1234567890"
```

**Expected Response:**
```
0x742d35cc6634c0532925a3b844bc9e7595f0beb
```

---

## Testing Checklist

### Backend Functionality
- [ ] Redis connection establishes on server start
- [ ] GET endpoint returns null for new address
- [ ] POST endpoint accepts valid username
- [ ] POST endpoint rejects invalid usernames
- [ ] Duplicate usernames are rejected
- [ ] Batch endpoint returns multiple usernames
- [ ] Redis disconnects gracefully on shutdown

### Frontend Functionality
- [ ] Dialog appears on first wallet connection
- [ ] Character counter updates in real-time
- [ ] Validation messages appear correctly
- [ ] Availability checking works (green checkmark)
- [ ] Submit button disabled for invalid usernames
- [ ] "Skip for now" closes dialog without error
- [ ] Username appears in header after setting
- [ ] Username appears in user list
- [ ] Username appears in InfoFi market titles
- [ ] Edit button works on account page
- [ ] Dialog reopens when clicking Edit Username

### Internationalization
- [ ] English translations work
- [ ] Japanese translations work
- [ ] Spanish translations work
- [ ] German translations work
- [ ] Username value is NOT translated

### Error Handling
- [ ] Invalid address format rejected (400)
- [ ] Username too short rejected (400)
- [ ] Username too long rejected (400)
- [ ] Invalid characters rejected (400)
- [ ] Duplicate username rejected (409)
- [ ] Redis connection failure handled gracefully
- [ ] Frontend shows formatted address if Redis unavailable

---

## Troubleshooting

### Issue: Dialog Doesn't Appear

**Check:**
1. Is wallet connected? (Check header)
2. Does address already have username? (Check Redis)
3. Is UsernameProvider in provider tree? (Check `main.jsx`)
4. Any console errors? (Check browser DevTools)

**Fix:**
```bash
# Clear username for testing
redis-cli DEL wallet:0x742d35cc6634c0532925a3b844bc9e7595f0beb
redis-cli DEL username:alice
```

### Issue: Username Not Appearing in UI

**Check:**
1. Is backend running? (`http://localhost:3000/api/health`)
2. Is Redis running? (`redis-cli ping`)
3. Check browser Network tab for API errors
4. Check React Query DevTools for cache status

**Debug:**
```bash
# Check if username is stored
redis-cli GET wallet:0x742d35cc6634c0532925a3b844bc9e7595f0beb

# Check backend logs
# Look for: "Redis connection established and verified"
```

### Issue: Tests Failing

**Check:**
1. Is Redis running? (`redis-cli ping`)
2. Are mocks properly set up? (Check test file)
3. Is `.env` configured? (`grep REDIS_URL .env`)

**Fix:**
```bash
# Ensure Redis is running
brew services start redis

# Run tests with verbose output
npm test tests/api/usernameRoutes.test.js -- --run --reporter=verbose
```

---

## Performance Validation

### Expected Response Times

- **GET username**: < 10ms (Redis lookup)
- **POST username**: < 20ms (Redis write + reverse lookup)
- **Check availability**: < 15ms (Redis lookup)
- **Batch lookup (10 addresses)**: < 30ms (Redis MGET)

### Load Testing

```bash
# Install Apache Bench (if not installed)
brew install httpd

# Test GET endpoint
ab -n 1000 -c 10 http://localhost:3000/api/usernames/0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb

# Expected: > 500 req/sec
```

---

## Production Deployment Checklist

### Before Deploying

- [ ] Create Upstash Redis database
- [ ] Copy connection URL (use `rediss://` with TLS)
- [ ] Add `REDIS_URL` to production environment variables
- [ ] Test connection from production environment
- [ ] Update Windsurf MCP config for production (optional)
- [ ] Run full test suite locally
- [ ] Verify all translations are complete

### After Deploying

- [ ] Verify backend connects to Upstash (check logs)
- [ ] Test username setting in production
- [ ] Verify usernames persist across sessions
- [ ] Monitor Redis memory usage (Upstash dashboard)
- [ ] Set up alerts for Redis connection failures
- [ ] Document production Redis credentials securely

---

## Monitoring

### Key Metrics to Track

1. **Username Adoption Rate**
   - % of connected wallets with usernames
   - Track via `GET /api/usernames/all` endpoint

2. **Redis Performance**
   - Connection uptime
   - Response times
   - Memory usage

3. **Error Rates**
   - Duplicate username attempts
   - Validation failures
   - Redis connection errors

### Upstash Dashboard

Monitor in production:
- Request count
- Latency (p50, p95, p99)
- Memory usage
- Connection count

---

## Security Considerations

### Implemented Protections

- ✅ Server-side validation (never trust client)
- ✅ Case-insensitive uniqueness checking
- ✅ Reserved username blocking
- ✅ Rate limiting (Fastify default: 100 req/min)
- ✅ Input sanitization (alphanumeric + underscore only)
- ✅ XSS prevention (React automatic escaping)

### Future Enhancements

- [ ] Signature verification for username changes
- [ ] Profanity filter
- [ ] Username moderation queue
- [ ] Abuse reporting system

---

## Quick Commands Reference

```bash
# Redis Management
redis-cli ping                     # Test connection
redis-cli KEYS wallet:*           # List all usernames
redis-cli GET wallet:0x123...     # Get specific username
redis-cli DEL wallet:0x123...     # Delete username
brew services stop redis           # Stop Redis (if needed)

# Testing
npm test tests/api/usernameRoutes.test.js -- --run
node scripts/test-username-api.js

# Backend (with automatic Redis startup)
npm run dev:backend                # Starts Redis + Backend
npm run dev:backend:simple         # Backend only (no Redis check)

# Frontend
npm run dev                        # or npm run dev:frontend
```

---

**Last Updated:** 2025-10-04
