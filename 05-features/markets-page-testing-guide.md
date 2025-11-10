# Markets Page Testing Guide

## Prerequisites

Before testing, ensure:
1. ✅ Backend server is running on `http://localhost:3000`
2. ✅ Frontend dev server is running on `http://localhost:5173`
3. ✅ Supabase connection is active
4. ✅ At least one market exists in the database

---

## Step 1: Restart Backend Server

The backend needs to be restarted to load the new InfoFi routes.

```bash
# From the project root
cd backend
npm run dev
```

**Expected Output:**
```
✅ Supabase configured and connected
Mounted /api/usernames
Mounted /api/users
Mounted /api/infofi  ← Look for this line!
Server listening at http://127.0.0.1:3000
```

---

## Step 2: Test Backend API Endpoints

### Test 1: Get All Markets
```bash
curl -s http://localhost:3000/api/infofi/markets | jq '.'
```

**Expected Response:**
```json
{
  "markets": {
    "1": [
      {
        "id": 22,
        "seasonId": 1,
        "player": "0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266",
        "market_type": "WINNER_PREDICTION",
        "current_probability_bps": 10000,
        "is_active": true,
        ...
      }
    ]
  },
  "total": 1
}
```

### Test 2: Get Markets for Season 1
```bash
curl -s http://localhost:3000/api/infofi/markets?seasonId=1 | jq '.'
```

### Test 3: Get Active Markets Only
```bash
curl -s http://localhost:3000/api/infofi/markets?isActive=true | jq '.'
```

### Test 4: Get Single Market
```bash
curl -s http://localhost:3000/api/infofi/markets/22 | jq '.'
```

### Test 5: Get Stats
```bash
curl -s http://localhost:3000/api/infofi/stats | jq '.'
```

**Expected Response:**
```json
{
  "totalMarkets": 1,
  "activeMarkets": 1,
  "settledMarkets": 0,
  "marketsByType": {
    "WINNER_PREDICTION": 1
  }
}
```

---

## Step 3: Test Frontend

### Navigate to Markets Page
1. Open browser to `http://localhost:5173/markets`
2. Page should load without errors

### Expected UI Elements

**Header Section:**
- ✅ "Markets" title
- ✅ "Browse active markets" subtitle
- ✅ Active season indicator (if season is active)

**Search & Filter Bar:**
- ✅ Search input with magnifying glass icon
- ✅ Three filter tabs: All, Active, Settled
- ✅ Icons on filter tabs

**Results Section:**
- ✅ Results count (e.g., "Showing 1 market")
- ✅ Season grouping (e.g., "Season #1")
- ✅ Market type sections (e.g., "Winner Prediction")
- ✅ Market cards in 3-column grid (desktop)

---

## Step 4: Test Search Functionality

### Test Search by Player Address

1. **Enter full address:**
   - Type: `0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266`
   - ✅ Markets should filter to show only matching player

2. **Enter partial address:**
   - Type: `0xf39f`
   - ✅ Should still match and show results

3. **Enter non-existent address:**
   - Type: `0x1234567890`
   - ✅ Should show empty state with "No markets match your search"
   - ✅ "Clear Search" button should appear

4. **Clear search:**
   - Click "Clear Search" button
   - ✅ All markets should reappear

---

## Step 5: Test Filter Tabs

### Test "All" Tab
1. Click "All" tab
2. ✅ Should show all markets (active + settled)

### Test "Active" Tab
1. Click "Active" tab
2. ✅ Should show only markets where `is_active = true`
3. ✅ Results count should update

### Test "Settled" Tab
1. Click "Settled" tab
2. ✅ Should show only markets where `is_settled = true`
3. ✅ If no settled markets, should show empty state

---

## Step 6: Test Responsive Design

### Desktop (1920x1080)
- ✅ 3-column grid layout
- ✅ Search bar and filters on same row
- ✅ Full market cards visible

### Tablet (768x1024)
- ✅ 2-column grid layout
- ✅ Search bar and filters stack vertically
- ✅ Market cards adapt to smaller width

### Mobile (375x667)
- ✅ Single column layout
- ✅ Search bar full width
- ✅ Filter tabs full width
- ✅ Market cards full width

---

## Step 7: Test Market Card Interactions

### Click Market Card
1. Click on any market card
2. ✅ Should navigate to `/markets/:marketId` detail page
3. ✅ URL should update

### Test YES/NO Buttons
1. On market card, click "YES" button
2. ✅ Button should highlight with green border
3. ✅ Payout calculation should update

4. Click "NO" button
5. ✅ Button should highlight with red border
6. ✅ Payout calculation should update

### Test Trade Input
1. Enter amount in trade input (e.g., "10")
2. ✅ Payout should update in real-time
3. ✅ Profit calculation should show

---

## Step 8: Test Real-Time Updates

Markets should update every 10 seconds via React Query polling.

1. Open browser console
2. Watch for network requests to `/api/infofi/markets`
3. ✅ Should see requests every 10 seconds
4. ✅ UI should update if data changes

---

## Step 9: Test Loading States

### Initial Load
1. Refresh page
2. ✅ Should show "Loading markets from backend..." message
3. ✅ Should transition to market grid when loaded

### Refetch After Error
1. Stop backend server
2. Wait for error state
3. ✅ Should show error message with "Retry" button
4. Restart backend
5. Click "Retry"
6. ✅ Should reload successfully

---

## Step 10: Test Empty States

### No Markets in Database
1. Clear all markets from database (if testing)
2. Refresh page
3. ✅ Should show empty state with emoji
4. ✅ Message: "No prediction markets available yet..."

### No Search Results
1. Search for non-existent address
2. ✅ Should show empty state
3. ✅ Message: "No markets match your search..."
4. ✅ "Clear Search" button visible

---

## Common Issues & Solutions

### Issue: "Not Found" Error from API

**Symptom:** API returns `{"error": "Not Found"}`

**Solution:**
1. Restart backend server
2. Check logs for "Mounted /api/infofi" message
3. Verify route file exists: `backend/fastify/routes/infoFiRoutes.js`

### Issue: Markets Not Displaying

**Symptom:** Page loads but no markets show

**Solution:**
1. Check browser console for errors
2. Verify API returns data: `curl http://localhost:3000/api/infofi/markets`
3. Check database has markets: Query Supabase `infofi_markets` table

### Issue: Search Not Working

**Symptom:** Search input doesn't filter results

**Solution:**
1. Check browser console for errors
2. Verify `searchQuery` state is updating
3. Check `groupedBySeason` memo includes search filter

### Issue: Filters Not Working

**Symptom:** Filter tabs don't change results

**Solution:**
1. Check `statusFilter` state is updating
2. Verify `groupedBySeason` memo includes status filter
3. Check markets have correct `is_active`/`is_settled` values

---

## Performance Benchmarks

### API Response Times
- ✅ `/api/infofi/markets` - < 100ms (1 market)
- ✅ `/api/infofi/markets` - < 500ms (100 markets)
- ✅ `/api/infofi/markets/:id` - < 50ms

### Frontend Render Times
- ✅ Initial load - < 1s
- ✅ Search filter - < 100ms
- ✅ Tab switch - < 100ms

### Memory Usage
- ✅ Frontend - < 50MB (100 markets)
- ✅ Backend - < 100MB

---

## Browser Console Commands

### Check React Query Cache
```javascript
// In browser console
window.__REACT_QUERY_DEVTOOLS_GLOBAL_HOOK__
```

### Force Refetch Markets
```javascript
// In browser console
queryClient.invalidateQueries({ queryKey: ['infofi', 'markets'] })
```

### Check Current Filter State
```javascript
// Add to MarketsIndex.jsx for debugging
console.log({ searchQuery, statusFilter, totalMarketsCount })
```

---

## Success Criteria Checklist

- [ ] Backend API returns markets correctly
- [ ] Frontend displays markets in grid layout
- [ ] Search by player address works
- [ ] Filter tabs work (All/Active/Settled)
- [ ] Results count updates correctly
- [ ] Empty states display properly
- [ ] Loading states work
- [ ] Error states work with retry
- [ ] Responsive design works on all screen sizes
- [ ] Market cards display correctly
- [ ] Real-time updates work (10s polling)
- [ ] Navigation to market detail works
- [ ] No console errors
- [ ] No network errors
- [ ] Performance is acceptable

---

## Next Steps After Testing

1. ✅ Verify all tests pass
2. ✅ Fix any issues found
3. ✅ Add unit tests for new components
4. ✅ Add E2E tests with Playwright
5. ✅ Update documentation
6. ✅ Create PR for review
7. ✅ Deploy to staging
8. ✅ Deploy to production

---

**Happy Testing! 🚀**
