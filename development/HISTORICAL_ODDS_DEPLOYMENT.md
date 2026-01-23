# Historical Odds System - Deployment Checklist

## Overview

This document provides a comprehensive checklist for deploying the Redis-based historical odds storage system to production.

## Pre-Deployment Checklist

### ✅ Code Review

- [x] All unit tests passing (18/18)
- [x] All integration tests passing (9/9)
- [x] Code reviewed and approved
- [x] No console.log statements in production code
- [x] Error handling implemented for all Redis operations
- [x] Non-blocking error handling for historical data recording

### ✅ Configuration

- [ ] Redis instance provisioned (local/Upstash)
- [ ] `REDIS_URL` configured in production `.env`
- [ ] `ODDS_RETENTION_DAYS` set (default: 90)
- [ ] `ODDS_DOWNSAMPLE_THRESHOLD` set (default: 500)
- [ ] `ODDS_CLEANUP_INTERVAL_HOURS` set (default: 24)
- [ ] Redis authentication configured (if using Upstash)
- [ ] Redis TLS enabled for production (rediss://)

### ✅ Infrastructure

- [ ] Redis memory limit configured (recommend: 2GB minimum)
- [ ] Redis persistence enabled (RDB or AOF)
- [ ] Redis backup strategy in place
- [ ] Monitoring alerts configured
- [ ] Log aggregation configured

### ✅ Testing

- [ ] Run verification script: `node scripts/test-historical-odds.js`
- [ ] Test API endpoint manually
- [ ] Verify frontend chart displays data
- [ ] Test with multiple markets
- [ ] Test downsampling with large datasets
- [ ] Verify cleanup job runs correctly

---

## Deployment Steps

### Phase 1: Staging Deployment

#### Step 1: Deploy Backend

```bash
# 1. Pull latest code
git pull origin main

# 2. Install dependencies
npm install

# 3. Set environment variables
cp .env.example .env
# Edit .env with staging Redis URL

# 4. Run verification
node scripts/test-historical-odds.js

# 5. Start backend
npm run dev:backend
```

#### Step 2: Verify Backend

```bash
# Test health endpoint
curl http://localhost:3000/api/health

# Test historical odds endpoint (should return empty initially)
curl http://localhost:3000/api/infofi/markets/0/history?range=1D

# Monitor logs
tail -f logs/backend.log
```

#### Step 3: Deploy Frontend

```bash
# 1. Build frontend
npm run build

# 2. Deploy to staging
# (deployment method depends on hosting provider)

# 3. Test in browser
# Navigate to market detail page
# Verify chart loads (may show "Cannot retrieve chart data" until data accumulates)
```

#### Step 4: Generate Test Data

```bash
# Place some bets or trigger price updates to generate historical data
# Wait a few minutes for data to accumulate
# Refresh market detail page to see chart populate
```

### Phase 2: Production Deployment

#### Step 1: Pre-Production Checks

- [ ] Staging tests passed for 24+ hours
- [ ] No memory leaks detected
- [ ] Redis memory usage stable
- [ ] API response times < 50ms (P95)
- [ ] No errors in logs
- [ ] Backup strategy tested

#### Step 2: Production Deployment

```bash
# 1. Set production environment variables
REDIS_URL=rediss://your-production-redis-url
ODDS_RETENTION_DAYS=90
ODDS_CLEANUP_INTERVAL_HOURS=24

# 2. Deploy backend
# (use your deployment pipeline)

# 3. Deploy frontend
# (use your deployment pipeline)

# 4. Monitor deployment
# Watch logs for errors
# Check Redis memory usage
# Verify API endpoints responding
```

#### Step 3: Post-Deployment Verification

```bash
# 1. Health check
curl https://api.secondorder.fun/api/health

# 2. Test historical odds endpoint
curl https://api.secondorder.fun/api/infofi/markets/0/history?range=1D

# 3. Monitor Redis
node scripts/monitor-redis-odds.js --watch

# 4. Check frontend
# Navigate to market detail page
# Verify chart displays correctly
```

---

## Monitoring & Maintenance

### Daily Monitoring

Run the monitoring script to check system health:

```bash
node scripts/monitor-redis-odds.js --watch
```

**Key Metrics to Watch:**

1. **Redis Memory Usage**
   - Alert if > 80% of max memory
   - Expected: ~19.4 MB per market (90 days)
   - 100 markets ≈ 1.94 GB

2. **API Response Time**
   - Target: < 50ms (P95)
   - Alert if > 100ms consistently

3. **Data Point Count**
   - Monitor growth rate
   - Expected: ~1,440 points/day per market (1 update/min)

4. **Cleanup Job**
   - Verify runs daily
   - Check logs for completion
   - Monitor removed entry count

### Weekly Tasks

- [ ] Review Redis memory trends
- [ ] Check for any error spikes in logs
- [ ] Verify cleanup job running successfully
- [ ] Review API latency metrics
- [ ] Check for any markets with unusually high data points

### Monthly Tasks

- [ ] Review retention policy (90 days appropriate?)
- [ ] Analyze downsampling effectiveness
- [ ] Review Redis backup strategy
- [ ] Update documentation if needed
- [ ] Performance optimization review

---

## Troubleshooting

### Issue: High Memory Usage

**Symptoms:**
- Redis memory > 80% of limit
- Memory alerts firing

**Solutions:**
1. Check data point counts per market
2. Reduce retention period (e.g., 60 days instead of 90)
3. Increase Redis memory limit
4. Run manual cleanup: `node scripts/test-historical-odds.js`

### Issue: Slow API Response

**Symptoms:**
- API response time > 100ms
- Chart takes long to load

**Solutions:**
1. Check Redis connection latency
2. Verify downsampling is working (should limit to 500 points)
3. Check if Redis is under memory pressure
4. Consider Redis connection pooling

### Issue: Missing Historical Data

**Symptoms:**
- Chart shows "Cannot retrieve chart data"
- API returns empty dataPoints array

**Solutions:**
1. Verify Redis is running: `redis-cli ping`
2. Check if data is being recorded: `redis-cli KEYS "odds:history:*"`
3. Verify pricingService integration is working
4. Check backend logs for errors
5. Manually trigger price update to test recording

### Issue: Cleanup Job Not Running

**Symptoms:**
- Old data not being removed
- Memory usage growing unbounded

**Solutions:**
1. Check backend logs for cleanup job execution
2. Verify `ODDS_CLEANUP_INTERVAL_HOURS` is set
3. Check if `db.getActiveInfoFiMarkets()` is returning markets
4. Manually run cleanup: `historicalOddsService.cleanupOldData(seasonId, marketId)`

---

## Rollback Plan

If critical issues arise, follow this rollback procedure:

### Step 1: Disable Historical Recording

Edit `backend/shared/pricingService.js`:

```javascript
// Comment out the historicalOddsService call
// await historicalOddsService.recordOddsUpdate(...)
```

Redeploy backend.

### Step 2: Frontend Fallback

The frontend already handles missing data gracefully:
- Shows "Cannot retrieve chart data" message
- No user-facing errors
- Other market features continue working

### Step 3: Clear Redis Data (if needed)

```bash
# List all historical odds keys
redis-cli KEYS "odds:history:*"

# Delete all historical odds data
redis-cli KEYS "odds:history:*" | xargs redis-cli DEL
```

### Step 4: Investigate and Fix

1. Review error logs
2. Identify root cause
3. Fix issue in development
4. Test thoroughly
5. Redeploy with fix

---

## Performance Benchmarks

### Expected Performance

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| API Response Time (P50) | < 20ms | > 50ms |
| API Response Time (P95) | < 50ms | > 100ms |
| Redis Memory per Market | ~19.4 MB | > 25 MB |
| Data Points per Market | ~130k (90 days) | > 150k |
| Cleanup Job Duration | < 5 min | > 10 min |

### Load Testing Results

Run load tests before production deployment:

```bash
# Install load testing tool
npm install -g autocannon

# Test API endpoint
autocannon -c 100 -d 30 http://localhost:3000/api/infofi/markets/0/history?range=1D
```

**Expected Results:**
- Throughput: > 1000 req/sec
- Latency (P95): < 50ms
- Error rate: 0%

---

## Security Considerations

### Redis Security

- [ ] Use TLS for Redis connections in production (rediss://)
- [ ] Enable Redis authentication (password)
- [ ] Restrict Redis network access (firewall rules)
- [ ] Use Redis ACLs to limit command access
- [ ] Regular security updates for Redis

### API Security

- [ ] Rate limiting enabled (100 req/min per IP)
- [ ] Input validation for marketId and range parameters
- [ ] CORS configured correctly
- [ ] No sensitive data in historical odds
- [ ] Proper error messages (no stack traces in production)

---

## Success Criteria

The deployment is considered successful when:

- ✅ All tests passing (27/27)
- ✅ Backend server starts without errors
- ✅ Redis connection established
- ✅ Historical data being recorded automatically
- ✅ API endpoint responding < 50ms (P95)
- ✅ Frontend chart displaying data correctly
- ✅ Cleanup job running on schedule
- ✅ No memory leaks after 24 hours
- ✅ Monitoring alerts configured
- ✅ Documentation updated

---

## Support & Escalation

### Contact Information

- **Development Team**: [Your team contact]
- **DevOps/Infrastructure**: [Your DevOps contact]
- **On-Call Engineer**: [Your on-call rotation]

### Escalation Path

1. **Level 1**: Check logs and monitoring dashboard
2. **Level 2**: Run diagnostic scripts
3. **Level 3**: Contact development team
4. **Level 4**: Implement rollback plan

---

## Appendix

### Useful Commands

```bash
# Check Redis memory
redis-cli INFO memory

# List all historical odds keys
redis-cli KEYS "odds:history:*"

# Get data point count for a market
redis-cli ZCARD odds:history:1:0

# Get oldest/newest timestamps
redis-cli ZRANGE odds:history:1:0 0 0 WITHSCORES
redis-cli ZRANGE odds:history:1:0 -1 -1 WITHSCORES

# Monitor Redis in real-time
redis-cli MONITOR

# Check API health
curl http://localhost:3000/api/health

# Test historical odds endpoint
curl http://localhost:3000/api/infofi/markets/0/history?range=1D | jq

# Run monitoring script
node scripts/monitor-redis-odds.js --watch

# Run verification script
node scripts/test-historical-odds.js
```

### Environment Variables Reference

```bash
# Redis Configuration
REDIS_URL=redis://localhost:6379              # Local development
REDIS_URL=rediss://user:pass@host:6379        # Production (TLS)

# Historical Odds Configuration
ODDS_RETENTION_DAYS=90                        # Data retention period
ODDS_DOWNSAMPLE_THRESHOLD=500                 # Max points before downsampling
ODDS_CLEANUP_INTERVAL_HOURS=24                # Cleanup frequency
```

---

**Document Version**: 1.0  
**Last Updated**: 2025-10-18  
**Next Review**: 2025-11-18  
**Owner**: Development Team
