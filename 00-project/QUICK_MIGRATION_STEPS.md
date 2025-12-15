# Quick Migration Steps

## 1. Create Backend Repository

```bash
mkdir sof-backend && cd sof-backend
git init
```

## 2. Copy Backend Files

From `sof-alpha` root:

```bash
# Backend code
cp -r backend/* ../sof-backend/

# Backend tests
mkdir -p ../sof-backend/tests
cp -r tests/backend ../sof-backend/tests/
cp -r tests/api ../sof-backend/tests/

# Backend scripts
mkdir -p ../sof-backend/scripts
cp scripts/reset-local-db.js ../sof-backend/scripts/
cp scripts/scan-historical-events.js ../sof-backend/scripts/
```

## 3. Setup Backend Configuration

Copy `package.json` from `BACKEND_PACKAGE_JSON.md`
Copy `.env.example` from `BACKEND_ENV_EXAMPLE.md`

## 4. Update Frontend

Remove backend:

```bash
cd sof-alpha
rm -rf backend/
```

Update `package.json` using `FRONTEND_PACKAGE_JSON.md`
Update `.env.example` using `FRONTEND_ENV_EXAMPLE.md`

## 5. Install Dependencies

```bash
# Backend
cd sof-backend
npm install

# Frontend
cd sof-alpha
npm install
```

## 6. Test Locally

Terminal 1 - Backend:

```bash
cd sof-backend
npm run dev
```

Terminal 2 - Frontend:

```bash
cd sof-alpha
npm run dev
```

## 7. Deploy

- Backend: Railway/Render
- Frontend: Vercel/Netlify
- Set environment variables in each platform
