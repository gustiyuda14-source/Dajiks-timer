# 🔧 Real-Time Sync Debugging Guide

## ✅ What I Fixed

All 5 critical bugs fixed:
1. ✅ Duplicate Firebase listeners on re-login
2. ✅ Pause/resume elapsed time calculation 
3. ✅ Logout not cleaning Firebase listeners
4. ✅ Credentials ref listener duplicates
5. ✅ Sync frequency increased from 15s → 5s

## 🔍 Test Real-Time Sync

Open **2 browser windows side-by-side**:

### Step 1: Check Console Logs
Open **Chrome DevTools** (F12) → **Console** tab on BOTH devices

### Step 2: Look for these messages when you START a timer:

**Device A (starts timer):**
```
[ACTION] Starting table 1
[SYNC] Table 1 synced to Firebase: running=true, elapsed=0
```

**Device B (should auto-update):**
```
[REALTIME] Tables data received: {m1: {running: true, ...}}
[REALTIME] Table 1 state changed: running=true
[REALTIME] Started interval for table 1
```

### Step 3: If you DON'T see those messages:

#### Problem A: Firebase Connection Failed
Look for this error on startup:
```
❌ [FIREBASE] Init error: ...
```

**Solution:**
1. Open `index.html` and check lines 483-491 (Firebase config)
2. Verify your Firebase config:
   - `apiKey` ✓
   - `databaseURL` ✓ (must be correct database URL)
   - `projectId` ✓

If unsure, go to [Firebase Console](https://console.firebase.google.com) and copy the config again.

#### Problem B: WebSocket Not Connected
Look for:
```
[FIREBASE] ✗ Disconnected from WebSocket
```

**Causes:**
- Network blocked (corporate firewall)
- Firebase database not deployed yet
- Wrong database URL

**Solutions:**
```javascript
// Temporary: Force HTTP instead of WebSocket
firebase.database().useEmulator('localhost', 9000); // if using emulator
// OR
fetch('https://your-project.firebaseio.com/tables.json') 
// Test if database is reachable
```

#### Problem C: Listeners Not Firing
If you see messages starting with `[SYNC]` but NOT `[REALTIME]`:
- Listener attached but not receiving updates
- Check Firebase Security Rules (go to Firebase Console → Realtime Database → Rules)

Rules must allow reads/writes:
```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

#### Problem D: Sync Every 5 Seconds Not Working
If timer shows on one device but not syncing every 5 seconds:
- Timer might not be in `tick()` function
- Check console for `[REALTIME] Tables data received` every 5 seconds

## 📋 Full Sync Flow

When you **start a timer on Device A**:

```
Device A: startT(1)
   ↓
Device A: fbSetT(table)
   ↓
Device A: [SYNC] Table 1 synced to Firebase
   ↓
Firebase: Writes to tables/m1 = {running:true, ...}
   ↓
Device A Listener: Fires immediately (local origin)
   ↓
Device B Listener: Fires within 1-2 seconds
   ↓
Device B: [REALTIME] Tables data received
   ↓
Device B: UI updates, interval starts
   ↓
Device B: Timer ticks and shows elapsed time
```

## 🎯 Testing Checklist

- [ ] **Both devices show REALTIME badge (green dot)** in header
- [ ] **Start timer on Device A** → Appears on Device B within 1 second
- [ ] **Check customer name** → Changes sync in real-time
- [ ] **Pause/Resume** → Works on both devices
- [ ] **Console shows [REALTIME] messages** every time data changes
- [ ] **No errors in console**
- [ ] **Close and reopen Device B** → Timer continues correctly

## 🚀 Advanced Debug

Enable ultra-verbose logging:

```javascript
// Paste this in browser console to see ALL Firebase reads/writes:
window._db.ref().on('value', function(snap) {
  console.log('[DEBUG] Full database:', snap.val());
});
```

## ❓ Still Not Working?

1. **Check Firebase is actually initialized:**
   ```javascript
   console.log(window._db); // Should NOT be undefined
   console.log(firebase.apps.length); // Should be 1
   ```

2. **Verify data is in Firebase:**
   - Go to [Firebase Console](https://console.firebase.google.com)
   - Realtime Database → Look for `tables` → `m1`, `m2`, etc.
   - If empty, data isn't being written

3. **Check network in DevTools:**
   - F12 → Network tab
   - Look for requests to `firebasedatabase.googleapis.com`
   - Should see WebSocket connection (green, not red)

4. **Try with this minimal test:**
   ```javascript
   // Paste in console of Device A:
   window._db.ref('test').set({msg:'hello'});
   
   // Then on Device B console:
   window._db.ref('test').on('value', snap => console.log(snap.val()));
   // Should print {msg: 'hello'}
   ```

---

**Open both console tabs and send me:**
1. Screenshot of browser console on Device B when you start timer on Device A
2. All error messages (if any)
3. Your Firebase `databaseURL` (safe to share)
