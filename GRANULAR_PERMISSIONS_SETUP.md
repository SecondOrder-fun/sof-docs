# Granular Permissions System - Setup Guide

## Overview

The granular permissions system provides flexible access control with:

- **Tier-based access levels** (0-4): Public, Connected, Allowlist, Beta, Admin
- **Group-based permissions**: Resource-specific access (e.g., "Season #5 VIP")
- **Route protection**: Per-route and per-resource access control
- **FID-first authentication**: Prioritizes Farcaster ID over wallet address
- **Admin UI**: Manage routes, groups, and user access

## Installation

### 1. Run Database Migration

```bash
cd sof-backend
chmod +x scripts/run-migration.sh
./scripts/run-migration.sh
```

Or manually:

```bash
psql $DATABASE_URL -f migrations/006_granular_access.sql
```

### 2. Restart Backend

```bash
npm run dev:backend
```

The backend will automatically load the new access control routes at `/api/access`.

### 3. Restart Frontend

```bash
npm run dev
```

## Configuration

### Default Access Level

New allowlist entries default to level 2 (Allowlist). To change:

```sql
UPDATE access_settings
SET value = '3'
WHERE key = 'default_access_level';
```

Or via API:

```bash
curl -X POST http://localhost:3000/api/access/set-default-level \
  -H "Content-Type: application/json" \
  -d '{"level": 3}'
```

### Default Route Configurations

The migration creates these default routes:

- `/` - Public (level 0)
- `/raffles` - Allowlist (level 2)
- `/raffles/:id` - Allowlist (level 2)
- `/markets` - Beta (level 3)
- `/account` - Connected (level 1)
- `/admin` - Admin (level 4)

## Usage

### Protecting Routes

Routes are automatically protected based on database configuration. The system checks:

1. **Public override** - If enabled, anyone can access
2. **Disabled state** - If enabled, shows maintenance page
3. **Access level** - User must meet minimum level
4. **Group membership** - User must be in required groups (if specified)

### Inline Access Gates

Use `AccessGate` for conditional rendering:

```jsx
import { AccessGate } from '@/components/access';
import { ACCESS_LEVELS } from '@/config/accessLevels';

// Level-based gate
<AccessGate requiredLevel={ACCESS_LEVELS.BETA}>
  <BetaFeature />
</AccessGate>

// Group-based gate
<AccessGate requiredGroups={['season-5-vip']}>
  <VIPContent />
</AccessGate>
```

### Checking Access in Components

```jsx
import { useAllowlist } from "@/hooks/useAllowlist";
import { ACCESS_LEVELS } from "@/config/accessLevels";

function MyComponent() {
  const { accessLevel, groups, hasLevel, hasGroup, isBeta, isAdmin } =
    useAllowlist();

  if (isBeta()) {
    return <BetaFeatures />;
  }

  if (hasGroup("season-5-vip")) {
    return <VIPFeatures />;
  }

  return <StandardFeatures />;
}
```

## Admin Panel

Access the admin panels at `/admin` (requires Admin level):

### Route Access Panel

Manage route configurations:

- Set required access level per route
- Assign required groups
- Toggle public override
- Enable maintenance mode

### Groups Panel

Manage access groups:

- Create/delete groups
- Add/remove users from groups
- View group members
- Set expiration dates

## API Reference

### Check User Access

```bash
GET /api/access/check?fid=12345
GET /api/access/check?wallet=0x123...
```

Response:

```json
{
  "isAllowlisted": true,
  "accessLevel": 3,
  "levelName": "beta",
  "groups": ["season-5-vip", "early-adopter"],
  "entry": { ... }
}
```

### Check Route Access

```bash
GET /api/access/check-access?fid=12345&route=/markets
```

Response:

```json
{
  "hasAccess": true,
  "reason": "level_met",
  "userLevel": 3,
  "requiredLevel": 3,
  "requiredGroups": [],
  "isPublicOverride": false
}
```

### Set User Access Level

```bash
POST /api/access/set-access-level
Content-Type: application/json

{
  "fid": 12345,
  "accessLevel": 3
}
```

### Create Group

```bash
POST /api/access/groups
Content-Type: application/json

{
  "slug": "season-5-vip",
  "name": "Season #5 VIP",
  "description": "VIP access to Season #5"
}
```

### Add User to Group

```bash
POST /api/access/groups/assign
Content-Type: application/json

{
  "fid": 12345,
  "groupSlug": "season-5-vip",
  "expiresAt": "2025-12-31T23:59:59Z"
}
```

## Database Schema

### Tables Created

- `access_groups` - Group definitions
- `user_access_groups` - User-group memberships
- `route_access_config` - Route access requirements
- `access_settings` - Global configuration

### Columns Added

- `allowlist_entries.access_level` - User access level (0-4)

## Access Levels

| Level     | Name        | Value | Description                 |
| --------- | ----------- | ----- | --------------------------- |
| Public    | `public`    | 0     | Anyone (no wallet required) |
| Connected | `connected` | 1     | Wallet connected            |
| Allowlist | `allowlist` | 2     | On allowlist                |
| Beta      | `beta`      | 3     | Beta testers                |
| Admin     | `admin`     | 4     | Full admin access           |

## Troubleshooting

### Routes not protected

1. Check route configuration exists in database
2. Verify backend restarted after migration
3. Check browser console for errors

### User can't access route

1. Check user's access level: `GET /api/access/check?fid=<fid>`
2. Check route requirements: `GET /api/access/route-config?route=<route>`
3. Verify group membership if required

### Admin panel not accessible

1. Ensure user has Admin level (4)
2. Check route protection is working
3. Verify `/admin` route configuration exists

## Security Notes

- Frontend checks are UX only - always enforce on backend
- FID must come from authenticated Farcaster context
- All admin endpoints should verify admin status
- Consider audit logging for access level changes

## Examples

### Make a route public

```sql
UPDATE route_access_config
SET is_public = true
WHERE route_pattern = '/raffles/1';
```

### Create VIP group for specific raffle

```sql
-- Create group
INSERT INTO access_groups (slug, name, description)
VALUES ('raffle-10-vip', 'Raffle #10 VIP', 'VIP access to Raffle #10');

-- Configure route
INSERT INTO route_access_config
  (route_pattern, resource_type, resource_id, required_level, required_groups, name)
VALUES
  ('/raffles/10', 'raffle', '10', 2, ARRAY['raffle-10-vip'], 'Raffle #10 VIP');

-- Add user to group
INSERT INTO user_access_groups (fid, group_id, granted_by)
SELECT 12345, id, 'admin' FROM access_groups WHERE slug = 'raffle-10-vip';
```

### Promote user to Beta

```sql
UPDATE allowlist_entries
SET access_level = 3
WHERE fid = 12345;
```

## Support

For issues or questions, check:

- Database logs for migration errors
- Backend logs for API errors
- Browser console for frontend errors
