# Zapier Integration Guide for ICON Royalty Portal

## Overview
This guide explains how to properly integrate Zapier (or other external services) with the ICON Royalty Portal Firebase database to avoid creating duplicate athlete entries.

## The Problem
If you use Firebase's `.push()` method, it creates entries with auto-generated IDs like `-OecXb5yQCrzTrU7VzFS`, which creates duplicates and breaks the referral system.

## The Solution
Use sanitized names as database keys with `.set()` instead of `.push()`.

---

## Correct Firebase Structure

### Athletes should be stored like this:
```
athletes/
  ├── Test_Athlete/              ✅ Sanitized name as key
  │   ├── name: "Test Athlete"
  │   ├── sanitizedName: "Test_Athlete"
  │   ├── email: "test@example.com"
  │   ├── accessCode: "TA2024"
  │   └── referrals/
  │       └── -N1xkj3kd.../
  │
  └── John_Doe/                   ✅ Another athlete
      ├── name: "John Doe"
      └── ...
```

### NOT like this:
```
athletes/
  ├── -OecXb5yQCrzTrU7VzFS/      ❌ Push ID (WRONG!)
  │   ├── name: "Test Athlete"
  │   └── referrals/
  │
  └── Test_Athlete/              ❌ Creates duplicate
      └── name: "Test Athlete"
```

---

## Zapier Configuration

### Step 1: Sanitize the Athlete Name

Before sending data to Firebase, you need to sanitize the athlete name to create a valid Firebase key.

**In Zapier's Code step (JavaScript):**

```javascript
// Input: athlete name from your form/source
const athleteName = inputData.athleteName; // e.g., "Test Athlete"

// Sanitize function (matches the portal's sanitization)
function sanitizeFirebaseKey(name) {
    if (!name) return '';
    return name
        .replace(/\./g, '_')   // Replace periods
        .replace(/\$/g, '_')   // Replace dollar signs
        .replace(/\#/g, '_')   // Replace hashtags
        .replace(/\[/g, '_')   // Replace opening brackets
        .replace(/\]/g, '_')   // Replace closing brackets
        .replace(/\//g, '_')   // Replace forward slashes
        .replace(/\s+/g, '_')  // Replace spaces with underscores
        .trim();
}

const sanitizedName = sanitizeFirebaseKey(athleteName);

// Generate access code (optional - use your own logic)
function generateAccessCode(name) {
    const nameParts = name.trim().split(/\s+/);
    if (nameParts.length === 1) {
        return nameParts[0].substring(0, 2).toUpperCase() + '2024';
    }
    const initials = nameParts.map(part => part[0]).join('').toUpperCase();
    return initials + '2024';
}

const accessCode = generateAccessCode(athleteName);

// Output these for the next step
output = {
    sanitizedName: sanitizedName,
    athleteName: athleteName,
    accessCode: accessCode
};
```

### Step 2: Configure Firebase Action

**In Zapier's Firebase action:**

1. **Action**: "Update Data in Firebase"
2. **Reference**: Use the sanitized name as part of the path
   ```
   athletes/{{sanitizedName}}
   ```
3. **Data**: Set the athlete data structure

**Example Firebase Data (JSON):**
```json
{
  "name": "{{athleteName}}",
  "sanitizedName": "{{sanitizedName}}",
  "email": "{{athleteEmail}}",
  "accessCode": "{{accessCode}}",
  "createdAt": "{{currentDateTime}}",
  "createdFrom": "zapier",
  "asanaProjectId": "{{asanaProjectId}}",
  "hasAccessCode": true,
  "totalSales": 0
}
```

### Step 3: Add Access Code Mapping

**Add another Firebase action:**

1. **Reference**:
   ```
   accessCodes/{{sanitizedName}}
   ```
2. **Data**:
   ```json
   "{{accessCode}}"
   ```

### Step 4: Add Name Mapping

**Add another Firebase action:**

1. **Reference**:
   ```
   nameMapping/{{sanitizedName}}
   ```
2. **Data**:
   ```json
   "{{athleteName}}"
   ```

---

## Complete Zapier Workflow Example

### Trigger
- **App**: JotForm / Google Forms / Typeform / etc.
- **Event**: New Submission

### Actions

1. **Code by Zapier** (JavaScript)
   - Sanitize athlete name
   - Generate access code

2. **Firebase** - Update Data
   - Path: `athletes/{{sanitizedName}}`
   - Data: Full athlete object

3. **Firebase** - Update Data
   - Path: `accessCodes/{{sanitizedName}}`
   - Data: `"{{accessCode}}"`

4. **Firebase** - Update Data
   - Path: `nameMapping/{{sanitizedName}}`
   - Data: `"{{athleteName}}"`

5. **Firebase Auth** - Create User (Optional)
   - Email: Generated synthetic email (e.g., `test_athlete@iconroyalty.local`)
   - Password: `{{accessCode}}`

---

## Testing Your Integration

### After setting up Zapier:

1. Submit a test form with name "Test Athlete"
2. Check Firebase Database:
   - Look under `athletes/`
   - You should see `Test_Athlete` (NOT `-OecXb...`)
   - Check that `accessCodes/Test_Athlete` exists
   - Check that `nameMapping/Test_Athlete` exists

### If you see push IDs:
- Your Zapier is still using `.push()` instead of setting the path correctly
- Review Step 2 above - the path must include the sanitized name

---

## Cleaning Up Existing Duplicates

If you already have duplicates in your database:

1. Login to the admin portal
2. Scroll to **"⚙️ Database Maintenance"**
3. Click **"Scan for Duplicates"**
4. Review the list of duplicates
5. Click **"Consolidate Duplicates"**
6. Wait for completion and page reload

This will move all referrals from push ID entries to the correct sanitized name entries.

---

## Database Rules to Apply

Make sure your Firebase Realtime Database rules allow Zapier to write:

```json
{
  "rules": {
    "athletes": {
      ".read": true,
      ".write": true,  // Or restrict to service account
      "$athleteName": {
        ".indexOn": ["name", "organization", "uid"]
      }
    },
    "accessCodes": {
      ".read": false,
      ".write": true
    },
    "nameMapping": {
      ".read": true,
      ".write": true
    }
  }
}
```

**Security Note:** For production, use Firebase Service Account credentials in Zapier instead of open write access.

---

## Common Mistakes to Avoid

❌ **Using `.push()`** - Creates random IDs
```javascript
// DON'T DO THIS
firebase.database().ref('athletes').push(athleteData);
```

❌ **Not sanitizing names** - Creates invalid Firebase keys
```javascript
// DON'T DO THIS
firebase.database().ref('athletes/' + "Test Athlete").set(data); // Spaces are invalid
```

✅ **Correct approach**
```javascript
// DO THIS
const sanitizedName = sanitizeFirebaseKey(athleteName);
firebase.database().ref('athletes/' + sanitizedName).set(athleteData);
```

---

## Support

If you continue to see duplicate entries after following this guide:
1. Check the Zapier task history for the exact Firebase path being used
2. Verify the sanitization function is working correctly
3. Use the Database Maintenance tool in the admin portal to clean up

## Questions?

Contact your developer or refer to:
- Firebase Documentation: https://firebase.google.com/docs/database
- Zapier Firebase Integration: https://zapier.com/apps/firebase/integrations
