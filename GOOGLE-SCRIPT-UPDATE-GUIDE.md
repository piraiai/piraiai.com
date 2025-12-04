# Google Apps Script Update Guide - Pricing Question Addition

## What Changed?
You're adding a pricing willingness question to the exit intent modal. This requires updating your Google Apps Script to save that data to a new sheet called "Pricing Willingness".

---

## Step-by-Step Update Instructions

### Step 1: Open Your Google Apps Script
1. Go to https://script.google.com/
2. Open your existing "Pirai AI File Upload Handler" project
3. You should see your `Code.gs` file

---

### Step 2: Update the `doPost()` Function

**Find this section** (around line 28):
```javascript
// Check if this is a referral source submission
if (data.type === 'referral_source') {
```

**Replace the entire `if` block** with this:

```javascript
// Check if this is a referral source AND pricing submission
if (data.type === 'referral_and_pricing') {
  const email = data.email || 'Not provided';
  const source = data.source || 'Not provided';
  const pricing = data.pricing || 'Not provided';
  const timestamp = data.timestamp || new Date().toISOString();

  // Log to referral sources sheet
  logReferralSource(email, source, timestamp);

  // Log to pricing willingness sheet
  logPricingWillingness(email, pricing, timestamp);

  return ContentService.createTextOutput(JSON.stringify({
    status: 'success',
    message: 'Referral source and pricing recorded'
  })).setMimeType(ContentService.MimeType.JSON);
}

// Legacy support: Check if this is a referral source submission only
if (data.type === 'referral_source') {
  const email = data.email || 'Not provided';
  const source = data.source || 'Not provided';
  const timestamp = data.timestamp || new Date().toISOString();

  // Log to referral sources sheet
  logReferralSource(email, source, timestamp);

  return ContentService.createTextOutput(JSON.stringify({
    status: 'success',
    message: 'Referral source recorded'
  })).setMimeType(ContentService.MimeType.JSON);
}
```

**What this does:**
- Handles the new `referral_and_pricing` type from the frontend
- Extracts both source and pricing data
- Calls two functions to log to two separate sheets
- Keeps the old code for backward compatibility

---

### Step 3: Add the New Function

**Scroll to the bottom of your script** (after the `logReferralSource` function and before the `testEmail` function).

**Add this entire new function:**

```javascript
/**
 * Log pricing willingness to Google Sheets
 * This tracks user willingness to pay for the service (pricing question data)
 */
function logPricingWillingness(email, pricing, timestamp) {
  try {
    // IMPORTANT: Replace with your Google Sheet ID (same sheet as above)
    const sheetId = 'YOUR_GOOGLE_SHEET_ID_HERE';
    const ss = SpreadsheetApp.openById(sheetId);
    const sheet = ss.getSheetByName('Pricing Willingness') || ss.insertSheet('Pricing Willingness');

    // Add headers if sheet is empty
    if (sheet.getLastRow() === 0) {
      sheet.appendRow(['Date/Time', 'Email', 'Pricing Willingness']);
      // Format header row
      const headerRange = sheet.getRange(1, 1, 1, 3);
      headerRange.setFontWeight('bold');
      headerRange.setBackground('#9B6BFF');
      headerRange.setFontColor('#FFFFFF');
    }

    // Insert new row at position 2 (right after header) - newest at the top
    sheet.insertRowBefore(2);

    // Add pricing data to the new row
    const newRow = [
      new Date(timestamp),
      email,
      pricing
    ];

    sheet.getRange(2, 1, 1, 3).setValues([newRow]);

    // Auto-resize columns for better readability
    sheet.autoResizeColumns(1, 3);

    Logger.log('Successfully logged pricing willingness: ' + pricing + ' for ' + email);
  } catch (error) {
    Logger.log('Error logging pricing willingness: ' + error.toString());
    // Don't throw error - just log it
  }
}
```

**What this does:**
- Creates a new sheet called "Pricing Willingness" if it doesn't exist
- Logs: Date/Time, Email, and Pricing choice ($1, $2, $2-5, or $0)
- Formats the header with purple background
- Puts newest entries at the top (row 2)

---

### Step 4: Update the Sheet ID

**In the new function you just added**, find this line:
```javascript
const sheetId = 'YOUR_GOOGLE_SHEET_ID_HERE';
```

**Replace it with your actual Google Sheet ID** (the same ID you used in the other functions):
```javascript
const sheetId = '1abc123XYZ456-defGHI789';  // Your actual sheet ID
```

💡 **Tip:** Look at your existing `logSubmission` or `logReferralSource` functions - copy the `sheetId` from there.

---

### Step 5: Save and Deploy

1. Click the **Save** icon (💾) or press `Ctrl+S` / `Cmd+S`
2. Click **Deploy** → **Manage deployments**
3. Click the **Edit** icon (✏️) next to your existing deployment
4. Under "Version", select **New version**
5. Add description: "Added pricing willingness tracking"
6. Click **Deploy**
7. You'll see "Deployment successfully updated"

---

## What You'll See After Deployment

### In Your Google Sheet:
You'll now have **3 sheets** (tabs at the bottom):
1. **Leads** - Original form submissions
2. **Referral Sources** - How users found you
3. **Pricing Willingness** - NEW! What users would pay

### The Pricing Willingness Sheet Will Show:
| Date/Time | Email | Pricing Willingness |
|-----------|-------|-------------------|
| 12/3/2025 10:30 AM | user@example.com | $2 |
| 12/3/2025 10:15 AM | test@school.edu | $2-5 |
| 12/3/2025 10:00 AM | prof@uni.edu | $0 |

---

## Testing

After deploying, test it:
1. Go to your convert-now page
2. Submit a conversion
3. When the exit modal appears, answer both questions
4. Check your Google Sheet - you should see new entries in both:
   - "Referral Sources" sheet
   - "Pricing Willingness" sheet (NEW!)

---

## Summary of Changes

✅ **Added:** 1 new check for `referral_and_pricing` type
✅ **Added:** 1 new function `logPricingWillingness()` (40 lines)
✅ **Kept:** All existing functionality intact
✅ **Result:** New "Pricing Willingness" sheet with user pricing preferences

**Total code addition:** ~60 lines
**Complexity:** Low (just copying the existing pattern)
**Risk:** Minimal (doesn't touch existing flows)

---

## Troubleshooting

**Problem:** Data not appearing in the new sheet
**Solution:** Check that you updated the `sheetId` in the `logPricingWillingness` function

**Problem:** Old submissions still work but new ones don't
**Solution:** Make sure you deployed a "New version" (not just saved)

**Problem:** Script says "logPricingWillingness is not defined"
**Solution:** Make sure you added the entire new function and saved it

---

Need help? The changes follow the exact same pattern as your existing `logReferralSource()` function - if that works, this will too!
