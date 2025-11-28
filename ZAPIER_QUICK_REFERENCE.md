# Zapier Quick Reference - Athlete Creation

## ⚡ Quick Setup

### 1. Code Step - Sanitize Name
```javascript
const athleteName = inputData.name;

// Sanitize name for Firebase key
const sanitizedName = athleteName
    .replace(/\./g, '_')
    .replace(/\$/g, '_')
    .replace(/\#/g, '_')
    .replace(/\[/g, '_')
    .replace(/\]/g, '_')
    .replace(/\//g, '_')
    .replace(/\s+/g, '_')
    .trim();

// Generate access code
const nameParts = athleteName.trim().split(/\s+/);
const initials = nameParts.map(part => part[0]).join('').toUpperCase();
const accessCode = initials + '2024';

output = {
    sanitizedName: sanitizedName,
    athleteName: athleteName,
    accessCode: accessCode
};
```

### 2. Firebase Action - Create Athlete
**Path:** `athletes/{{sanitizedName}}`

**Data:**
```json
{
  "name": "{{athleteName}}",
  "sanitizedName": "{{sanitizedName}}",
  "email": "{{email}}",
  "accessCode": "{{accessCode}}",
  "createdAt": "{{zap_meta_human_now}}",
  "createdFrom": "zapier",
  "hasAccessCode": true,
  "totalSales": 0
}
```

### 3. Firebase Action - Access Code
**Path:** `accessCodes/{{sanitizedName}}`

**Data:** `"{{accessCode}}"`

### 4. Firebase Action - Name Mapping
**Path:** `nameMapping/{{sanitizedName}}`

**Data:** `"{{athleteName}}"`

---

## ✅ Correct vs ❌ Incorrect

### ✅ CORRECT Path Format:
```
athletes/John_Doe
athletes/Test_Athlete
athletes/Mary_Jane_Smith
```

### ❌ WRONG Path Format:
```
athletes/          (using .push())
athletes/-OecXb... (auto-generated ID)
athletes/John Doe  (spaces not sanitized)
athletes/John.Doe  (periods not sanitized)
```

---

## 🔍 How to Verify

After running your Zap, check Firebase:
1. Go to Firebase Console → Realtime Database
2. Navigate to `athletes/`
3. You should see athlete names with underscores (e.g., `John_Doe`)
4. You should NOT see entries starting with `-O` or `-N`

---

## 🆘 If You See Duplicates

Use the Database Maintenance tool in admin portal:
1. Login as admin
2. Scroll to "⚙️ Database Maintenance"
3. Click "Scan for Duplicates"
4. Click "Consolidate Duplicates"

---

## 📋 Referral Integration (Optional)

If processing referrals via Zapier, use this path structure:

**Path:** `athletes/{{referrerSanitizedName}}/referrals/{{-auto-}}` or use `.push()` here

**Data:**
```json
{
  "referrerCode": "{{referrerName}}",
  "referrerType": "athlete",
  "friendName": "{{newAthleteName}}",
  "friendInstagram": "{{instagram}}",
  "dateReferred": "{{zap_meta_human_now}}",
  "status": "success"
}
```

Note: `.push()` is OK for referrals (children), but NOT for top-level athletes!
