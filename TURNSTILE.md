# Cloudflare Turnstile Authentication

Version: v1.2.0+

## Overview

This project includes optional Cloudflare Turnstile authentication to protect your map from bots. Turnstile is a user-friendly alternative to CAPTCHA that uses invisible challenge-response verification.

When enabled, unauthenticated users are redirected to a landing page where they must verify with Turnstile before accessing the live map.

## Setup

### 1. Get Turnstile Keys

1. Sign up for Cloudflare: https://dash.cloudflare.com
2. Navigate to **Turnstile** in your dashboard
3. Create a new site with:
   - **Domain**: your domain (e.g., `yourdomain.com`)
   - **Mode**: Managed (recommended)
4. Copy your **Site Key** and **Secret Key**

### 2. Configure Environment Variables

Add the following to your `.env` file:

```env
TURNSTILE_ENABLED=true
TURNSTILE_SITE_KEY=your_site_key_here
TURNSTILE_SECRET_KEY=your_secret_key_here
TURNSTILE_API_URL=https://challenges.cloudflare.com/turnstile/v0/siteverify
TURNSTILE_TOKEN_TTL_SECONDS=86400
```

**Optional variables:**
- `TURNSTILE_API_URL`: Custom Cloudflare API endpoint (default works for most users)
- `TURNSTILE_TOKEN_TTL_SECONDS`: How long auth tokens remain valid in seconds (default: 24 hours)

### 3. Restart Docker

```bash
docker compose up -d --build
```

Check logs to verify Turnstile is enabled:

```bash
docker compose logs -f meshmap-live | grep -i turnstile
```

You should see:
```
[startup] Turnstile authentication enabled
```

## How It Works

### Authentication Flow

1. **User visits map** → Unauthenticated
2. **Redirected to landing page** → Displays Turnstile widget with loading animation
3. **Widget loads** → User sees verification challenge or progress indicators
4. **User completes verification** → Shows 4-stage progress (init → widget load → verification → token submit)
5. **Backend verifies with Cloudflare** → Validates the token
6. **Auth token issued** → Stored in cookie + localStorage
7. **Redirected to map** → User is now authenticated
8. **Map loads** → User can interact with the live map

### Backend Architecture

**New files:**
- `backend/turnstile.py` - `TurnstileVerifier` class
  - Verifies tokens with Cloudflare API
  - Issues auth tokens to verified users
  - Manages token expiration and cleanup
  - Stores tokens in memory (per-instance)

**New endpoints:**
- `POST /api/verify-turnstile` - Verify Turnstile token and issue auth token
  - Sends request to Cloudflare with the challenge token
  - Returns JSON with auth token if successful
  - Sets auth cookie (httpOnly would be ideal, but JS-accessible for compatibility)

**Modified endpoints:**
- `GET /` - Root handler now:
  - Checks for valid auth cookie
  - Serves landing page if unauthenticated
  - Serves map if authenticated
  
- `GET /map` - Explicit map endpoint with auth check
  - Serves authenticated map
  - Redirects to `/` if unauthenticated

### Frontend Architecture

**New files:**
- `backend/static/landing.html` - Landing page with:
  - Turnstile widget container
  - Loading animation (spinner)
  - 4-stage progress tracker (detailed console logging)
  - Error and success message displays
  - Responsive mobile design

- `backend/static/turnstile.js` - Widget handler with:
  - Turnstile API initialization
  - Widget rendering with error handling
  - Token verification callback
  - Automatic token submission to backend
  - Token storage (cookie + localStorage)
  - Detailed step-by-step console logging
  - Error recovery with widget reset

**Modified files:**
- `backend/static/index.html` - Map page with:
  - Client-side auth check (only on landing page)
  - Turnstile enabled flag in body attributes
  - Logging for auth state

## Features

### Landing Page UI

- **Dark theme** matching your map
- **Loading animation** with spinner while Turnstile API loads
- **4-step progress tracker**:
  1. Initializing Turnstile
  2. Waiting for widget to load
  3. Waiting for user verification
  4. Submitting token to server
- **Real-time console output** showing each step
- **Error messages** if widget fails to load or verification fails
- **Success message** when verification completes
- **Mobile responsive** with touch-friendly interface
- **Graceful fallbacks** if widget loading takes too long

### Console Logging

Every step logs to browser console with timestamps and status indicators:

```
[11:23:45] Step 1: Initializing Turnstile authentication...
[11:23:46] Step 1: Turnstile API loaded
[11:23:47] Step 2: Widget rendered successfully
[11:23:50] Step 3: Verification received from Cloudflare
[11:23:51] Step 4: Token verified successfully
```

### Token Management

- **Automatic expiration** based on `TURNSTILE_TOKEN_TTL_SECONDS` (default 24 hours)
- **Cookie storage** - sent with every request to backend
- **Client-side storage** - sessionStorage + localStorage as backup
- **Secure cookie settings** - path `/`, SameSite=Lax

## Configuration Examples

### Development (Turnstile Optional)

```env
TURNSTILE_ENABLED=false
```

Map is accessible without Turnstile. Good for local testing.

### Production (Turnstile Required)

```env
TURNSTILE_ENABLED=true
TURNSTILE_SITE_KEY=0x4AAA...
TURNSTILE_SECRET_KEY=0x4AAA...
TURNSTILE_TOKEN_TTL_SECONDS=86400
```

Users must verify with Turnstile to access the map.

### Short Token Lifetime

```env
TURNSTILE_ENABLED=true
TURNSTILE_SITE_KEY=0x4AAA...
TURNSTILE_SECRET_KEY=0x4AAA...
TURNSTILE_TOKEN_TTL_SECONDS=3600
```

Tokens expire after 1 hour instead of 24 hours.

## Troubleshooting

### "Turnstile widget took too long to load"

**Cause:** Cloudflare API endpoint unreachable or very slow

**Solution:**
- Check internet connectivity
- Verify `TURNSTILE_API_URL` is correct (default is usually fine)
- Check Cloudflare service status

### Widget loads but verification fails

**Cause:** 
- Invalid site key/secret key
- Domain mismatch (site key configured for different domain)
- Network connectivity issue

**Solution:**
- Verify keys in Cloudflare dashboard
- Ensure your domain matches Turnstile configuration
- Check browser console for error details

### Users stuck in auth loop

**Cause:**
- Cookie not being set by browser
- Auth token not being recognized by backend
- Backend and frontend out of sync

**Solution:**
- Check browser cookies (DevTools → Application → Cookies)
- Verify `TURNSTILE_ENABLED=true` in logs
- Restart docker container: `docker compose restart meshmap-live`

### Token verified but redirects back to landing page

**Cause:** Auth cookie not being sent with request

**Solution:**
- Check browser console for cookie setting logs
- Verify cookies are enabled in browser
- Check browser DevTools → Application → Cookies for `meshmap_auth`

## Client-Side Implementation Details

### How the Frontend Knows If It's Authenticated

The frontend checks:
1. **If on landing page** (detects `#turnstile-container` element) → Stay on landing page
2. **If on map page** → Trust server authentication (server already verified before serving HTML)

### How the Backend Knows If User Is Authenticated

The backend checks:
1. **Is Turnstile enabled?** → If no, allow all users
2. **Does request have `meshmap_auth` cookie?** → Check if it's in the valid tokens list
3. **Is token still valid?** → Check expiration time
4. **Valid?** → Serve authenticated content
5. **Invalid?** → Redirect to landing page

### Token Lifecycle

```
1. User completes Turnstile verification
2. Frontend: POST /api/verify-turnstile with Turnstile token
3. Backend: Verify with Cloudflare API
4. Backend: Issue new auth token (random 32-byte URL-safe string)
5. Backend: Store in memory with expiration time
6. Backend: Set cookie in response
7. Frontend: Store in sessionStorage + localStorage
8. Browser: Auto-sends cookie with subsequent requests
9. Backend: Validates cookie on each protected page request
10. After TTL: Token expires and is auto-cleaned up
11. User redirected to landing page for new verification
```

## Security Considerations

### What's Protected

- Access to the live map
- WebSocket connection to real-time data
- API endpoints

### What's NOT Protected

- Static files (JS, CSS, images)
- Manifest file
- Preview image endpoint (by design, for sharable links)

### Token Storage

**Current approach (browser JavaScript can access):**
- Cookie: Sent automatically, can be read by JavaScript
- localStorage/sessionStorage: Can be read/modified by JavaScript

**More secure approach (optional future enhancement):**
- Use HttpOnly cookies (not accessible to JavaScript)
- Require token refresh for long-lived sessions
- Rate limit token verification endpoint
- Log verification attempts

## Disabling Turnstile

To disable Turnstile and allow unrestricted access:

```env
TURNSTILE_ENABLED=false
```

Rebuild and restart:

```bash
docker compose up -d --build
```

Users will bypass the landing page and access the map directly.

## Monitoring

Check Turnstile verification logs:

```bash
docker compose logs meshmap-live | grep turnstile
```

Expected output:
```
[startup] Turnstile authentication enabled
[turnstile] Verification successful, issued auth token
[turnstile] Verification failed: {...error details...}
```

## Additional Resources

- [Cloudflare Turnstile Docs](https://developers.cloudflare.com/turnstile/)
- [Turnstile Demo](https://demo.turnstile.dev/)
- [Turnstile Supported Platforms](https://developers.cloudflare.com/turnstile/faq/)

## Technical Stack

- **Frontend**: Vanilla JavaScript (no frameworks)
- **Backend**: FastAPI + Python
- **Verification**: Cloudflare Turnstile API (REST)
- **Storage**: In-memory token store (not persisted)
- **Auth Method**: HTTP cookies + localStorage

## Version History

- **v1.2.0** - Initial Turnstile authentication implementation
  - Landing page with widget
  - Backend token verification
  - Automatic cookie management
  - Step-by-step progress UI
  - Console logging for debugging
