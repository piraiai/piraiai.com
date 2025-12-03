# Google Apps Script Setup for File Upload

This guide will help you set up the backend for the Convert Now page using Google Apps Script.

## Step 1: Create a New Google Apps Script Project

1. Go to [Google Apps Script](https://script.google.com/)
2. Click **"New project"**
3. Name it something like "Pirai AI File Upload Handler"

## Step 2: Add the Script Code

Replace the default `Code.gs` content with the following code:

```javascript
/**
   * Pirai AI - Academic Document Conversion Request Handler
   * This script receives file uploads from the convert-now.html page
   * saves files to Google Drive, and emails the link to admin@piraiai.com
   */

  function doPost(e) {
    try {
      // Parse the incoming data
      const data = e.parameter;

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

      // Extract form fields (for regular file upload submissions)
      const name = data.name || 'Not provided';
      const email = data.email || 'Not provided';
      const platform = data.platform || 'Not specified';
      const role = data.role || 'Not provided';
      const institution = data.institution || 'Not provided';
      const fileName = data.fileName || 'document';
      const timestamp = data.timestamp || new Date().toISOString();
      const fileContent = data.fileContent;
      const mimeType = data.mimeType || 'application/octet-stream';

      // Create or get the folder in Google Drive
      const folderName = 'Pirai AI - Academic Conversions';
      let folder;
      const folders = DriveApp.getFoldersByName(folderName);
      if (folders.hasNext()) {
        folder = folders.next();
      } else {
        folder = DriveApp.createFolder(folderName);
      }

      // Save file to Google Drive
      let fileUrl = 'No file uploaded';
      let driveFile = null;

      if (fileContent) {
        try {
          // Decode base64 file content
          const decodedFile = Utilities.base64Decode(fileContent);
          const blob = Utilities.newBlob(decodedFile, mimeType, fileName);

          // Create file in Google Drive
          driveFile = folder.createFile(blob);

          // Make file accessible to anyone with link
          driveFile.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);

          fileUrl = driveFile.getUrl();
          Logger.log('File uploaded successfully: ' + fileUrl);
        } catch (fileError) {
          Logger.log('File upload error: ' + fileError.toString());
          fileUrl = 'Error uploading file: ' + fileError.toString();
        }
      }

      // Compose email to admin
      const adminEmail = 'admin@piraiai.com';
      const subject = `🎓 Academic Conversion Request - ${name}`;

      const emailBody = `
  New Academic Document Conversion Request

  ═══════════════════════════════════════

  REQUESTER INFORMATION:
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Name:        ${name}
  Email:       ${email}
  Role:        ${role}
  Institution: ${institution}

  CONVERSION REQUEST:
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Platform:    ${platform}
  File Name:   ${fileName}
  Submitted:   ${new Date(timestamp).toLocaleString()}

  📎 DOWNLOAD DOCUMENT:
  ${fileUrl}

  NEXT STEPS:
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  1. Click the link above to download the document
  2. Convert to ${platform} format using AI agent
  3. Reply to: ${email}
  4. Attach the converted file

  ═══════════════════════════════════════

  This is an early access request from the academic program.
  Expected turnaround: Within few hours during business hours.
      `.trim();

      // Send email to admin
      GmailApp.sendEmail(adminEmail, subject, emailBody, {
        name: 'Pirai AI Convert Now',
        replyTo: email
      });

      // Send confirmation email to the user
      const userSubject = 'Your Pirai AI Conversion Request Received';
      const userBody = `
  Dear ${name},

  Thank you for submitting your document for conversion to Qualtrics QSF format!

  We've received your request and our AI team will begin processing it shortly.

  WHAT YOU SUBMITTED:
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Document:    ${fileName}
  Submitted:   ${new Date(timestamp).toLocaleString()}

  WHAT HAPPENS NEXT:
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  1. Our team receives your document
  2. Our AI agent will convert the doc
  3. Quality assurance check ensures accuracy
  4. You receive the .qsf file via email

  ⏱️ EXPECTED TIMEFRAME:
  We typically complete conversions within few hours during business hours.
  Depending on your timezone, it may take a bit longer.

  You'll receive another email at ${email} with your converted file attached.

  Please check your spam folder if you don't see it in your inbox.

  Questions? Just reply to this email or contact us at admin@piraiai.com

  Best regards,
  The Pirai AI Team

  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Pirai AI | Survey Automation Platform
  https://piraiai.com
      `.trim();

      GmailApp.sendEmail(email, userSubject, userBody, {
        name: 'Pirai AI'
      });

      // Log the submission
      logSubmission(name, email, platform, role, institution, fileName, timestamp, fileUrl);

      // Return success response
      return ContentService.createTextOutput(JSON.stringify({
        status: 'success',
        message: 'File received and emails sent'
      })).setMimeType(ContentService.MimeType.JSON);

    } catch (error) {
      Logger.log('Error: ' + error.toString());

      // Return error response
      return ContentService.createTextOutput(JSON.stringify({
        status: 'error',
        message: error.toString()
      })).setMimeType(ContentService.MimeType.JSON);
    }
  }

  /**
   * Log submission to Google Sheets
   * This automatically logs all leads to a spreadsheet with newest entries at the top
   */
  function logSubmission(name, email, platform, role, institution, fileName, timestamp, fileUrl) {
    try {
      // IMPORTANT: Replace with your Google Sheet ID
      const sheetId = 'YOUR_GOOGLE_SHEET_ID_HERE';
      const ss = SpreadsheetApp.openById(sheetId);
      const sheet = ss.getSheetByName('Leads') || ss.insertSheet('Leads');

      // Add headers if sheet is empty
      if (sheet.getLastRow() === 0) {
        sheet.appendRow(['Date/Time', 'Name', 'Email', 'Platform', 'Role', 'Institution', 'File Name', 'File URL']);
        // Format header row
        const headerRange = sheet.getRange(1, 1, 1, 8);
        headerRange.setFontWeight('bold');
        headerRange.setBackground('#9B6BFF');
        headerRange.setFontColor('#FFFFFF');
      }

      // Insert new row at position 2 (right after header) - this puts newest leads at the top
      sheet.insertRowBefore(2);

      // Add submission data to the new row
      const newRow = [
        new Date(timestamp),
        name,
        email,
        platform,
        role,
        institution,
        fileName,
        fileUrl
      ];

      sheet.getRange(2, 1, 1, 8).setValues([newRow]);

      // Auto-resize columns for better readability
      sheet.autoResizeColumns(1, 8);

      Logger.log('Successfully logged submission to sheet: ' + email);
    } catch (error) {
      Logger.log('Error logging to sheet: ' + error.toString());
      // Don't throw error - continue with email sending even if sheet logging fails
    }
  }

  /**
   * Log referral source to Google Sheets
   * This tracks how users found out about Pirai AI (exit intent modal data)
   */
  function logReferralSource(email, source, timestamp) {
    try {
      // IMPORTANT: Replace with your Google Sheet ID (same sheet as above)
      const sheetId = 'YOUR_GOOGLE_SHEET_ID_HERE';
      const ss = SpreadsheetApp.openById(sheetId);
      const sheet = ss.getSheetByName('Referral Sources') || ss.insertSheet('Referral Sources');

      // Add headers if sheet is empty
      if (sheet.getLastRow() === 0) {
        sheet.appendRow(['Date/Time', 'Email', 'Source']);
        // Format header row
        const headerRange = sheet.getRange(1, 1, 1, 3);
        headerRange.setFontWeight('bold');
        headerRange.setBackground('#9B6BFF');
        headerRange.setFontColor('#FFFFFF');
      }

      // Insert new row at position 2 (right after header) - newest at the top
      sheet.insertRowBefore(2);

      // Add referral data to the new row
      const newRow = [
        new Date(timestamp),
        email,
        source
      ];

      sheet.getRange(2, 1, 1, 3).setValues([newRow]);

      // Auto-resize columns for better readability
      sheet.autoResizeColumns(1, 3);

      Logger.log('Successfully logged referral source: ' + source + ' for ' + email);
    } catch (error) {
      Logger.log('Error logging referral source: ' + error.toString());
      // Don't throw error - just log it
    }
  }

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

  /**
   * Test function to verify the script works
   */
  function testEmail() {
    GmailApp.sendEmail(
      'admin@piraiai.com',
      'Test Email from Pirai AI Script',
      'If you receive this, the script is working correctly!'
    );
  }
```

## Step 3: Deploy as Web App

1. Click **"Deploy"** > **"New deployment"**
2. Click the gear icon ⚙️ next to "Select type"
3. Select **"Web app"**
4. Configure the deployment:
   - **Description**: "Pirai AI File Upload Handler v1"
   - **Execute as**: "Me"
   - **Who has access**: "Anyone" (this allows the public form to submit)
5. Click **"Deploy"**
6. **Authorize the app** (you'll need to grant permissions)
7. **Copy the Web App URL** - it will look like:
   ```
   https://script.google.com/macros/s/AKfycbz.../exec
   ```

## Step 4: Update the HTML File

Open `/convert-now.html` and replace this line (around line 497):

```javascript
const scriptURL = 'YOUR_GOOGLE_APPS_SCRIPT_URL_HERE';
```

With your actual script URL:

```javascript
const scriptURL = 'https://script.google.com/macros/s/AKfycbz.../exec';
```


## Step 5: Test the Setup

1. Run the `testEmail()` function in the script editor to verify email sending works
2. Submit a test conversion through your website
3. Check that you receive the email at admin@piraiai.com

## Step 6: Setup Google Sheets Lead Tracking

This step configures automatic logging of all submissions to a Google Sheet with newest leads at the top.

### 6.1 Create the Google Sheet

1. Go to [Google Sheets](https://sheets.google.com/)
2. Click **"Blank"** to create a new spreadsheet
3. Name it **"Pirai AI - Conversion Leads"**
4. The sheet will be automatically named "Leads" by the script

### 6.2 Get the Sheet ID

1. Look at the URL of your Google Sheet
2. Copy the Sheet ID from the URL:
   ```
   https://docs.google.com/spreadsheets/d/SHEET_ID_HERE/edit
   ```
3. The Sheet ID is the long string between `/d/` and `/edit`

Example:
```
https://docs.google.com/spreadsheets/d/1abc123XYZ456-defGHI789/edit
```
The Sheet ID is: `1abc123XYZ456-defGHI789`

### 6.3 Update the Script

1. Go back to your Google Apps Script editor
2. Find line 184 in the `logSubmission` function:
   ```javascript
   const sheetId = 'YOUR_GOOGLE_SHEET_ID_HERE';
   ```
3. Replace `YOUR_GOOGLE_SHEET_ID_HERE` with your actual Sheet ID:
   ```javascript
   const sheetId = '1abc123XYZ456-defGHI789';
   ```
4. Click **"Save"** (💾 icon)

### 6.4 Re-deploy the Script

After making changes, you need to create a new version:

1. Click **"Deploy"** > **"Manage deployments"**
2. Click the **Edit** icon (✏️) on your existing deployment
3. Under "Version", select **"New version"**
4. Add description: "Added Google Sheets logging"
5. Click **"Deploy"**

### 6.5 What Gets Logged

The spreadsheet will automatically track:
- **Date/Time** - When the submission was made
- **Name** - Lead's full name
- **Email** - Lead's email address
- **Platform** - Target conversion platform (e.g., Qualtrics, SurveyMonkey)
- **Role** - Their academic role
- **Institution** - Their institution name
- **File Name** - Name of the uploaded document
- **File URL** - Google Drive link to the document

### 6.6 Features

✅ **Newest leads at the top** - New submissions are inserted at row 2 (right after header)

✅ **Automatic formatting** - Header row is styled with purple background (#9B6BFF) and white text

✅ **Auto-resize columns** - Columns automatically adjust to fit content

✅ **No duplicates** - Each submission creates a new row

✅ **Direct file access** - Click the File URL to open the document in Google Drive

## Troubleshooting

### Common Issues:

1. **CORS errors**: Make sure deployment is set to "Anyone" can access
2. **File upload fails**: Google Apps Script has file size limits (50MB). Consider using Google Drive approach.
3. **Emails not sending**: Check Gmail sending limits (100-500 emails/day depending on account type)

### Alternative Solution: Use Formspree

If Google Apps Script is too complex, use Formspree:

1. Sign up at [formspree.io](https://formspree.io)
2. Create a new form
3. Enable file uploads in settings
4. Replace the fetch URL in convert-now.html with your Formspree endpoint

## Production Checklist

- [ ] Google Apps Script deployed
- [ ] Script URL added to convert-now.html
- [ ] Google Sheet created for lead tracking
- [ ] Sheet ID added to script (lines 219, 268, 311)
- [ ] Script re-deployed with new version
- [ ] Test submission successful
- [ ] Email received at admin@piraiai.com
- [ ] User confirmation email working
- [ ] File upload working correctly
- [ ] Lead logged to Google Sheet "Leads" tab (check row 2)
- [ ] Form validation working
- [ ] GTM tracking events firing
- [ ] Exit intent modal appears after successful submission
- [ ] Referral source question appears first in exit modal
- [ ] Pricing question appears after selecting referral source
- [ ] Submit button disabled until both questions answered
- [ ] Referral source data logged to "Referral Sources" sheet
- [ ] Pricing willingness data logged to "Pricing Willingness" sheet
- [ ] Both referral and pricing GTM events tracking correctly with parameters
- [ ] Modal displays properly on mobile devices
- [ ] Pricing options layout correctly in 2x2 grid

## Security Notes

- The script runs under your Google account
- Anyone can submit to the web app (it's public)
- Consider adding rate limiting if you get spam
- Academic email validation happens on client-side (add server-side check if needed)

## Step 7: Exit Intent Modal - Referral Source & Pricing Willingness Tracking

The convert-now page now includes an exit-intent modal that appears after successful form submission. This collects two types of valuable data:
1. **Referral Source** - How users found your service
2. **Pricing Willingness** - Whether users would pay for the service

### 7.1 How It Works

- **Trigger**: Shows when user moves mouse to close tab/window after successful submission, or automatically after 1.2 seconds
- **Two-Step Process**:
  1. First, users select how they heard about Pirai AI (Word of Mouth, Google Search, AI Search, Social Media, Other)
  2. Then, a pricing question appears asking if they would pay for the service
- **Data Collected**: User email, selected source, pricing willingness, timestamp
- **Separate Sheets**: Creates both "Referral Sources" and "Pricing Willingness" sheets in the same Google Spreadsheet

### 7.2 Referral Source Data

The "Referral Sources" sheet will automatically track:
- **Date/Time** - When the user submitted the referral source
- **Email** - The user's email (matched to their conversion submission)
- **Source** - How they heard about Pirai AI (Word of Mouth, Google Search, AI Search, Social Media, Other - [custom text])

### 7.3 Pricing Willingness Data

The "Pricing Willingness" sheet will automatically track:
- **Date/Time** - When the user submitted their pricing preference
- **Email** - The user's email (matched to their conversion submission)
- **Pricing Willingness** - What they'd be willing to pay ($1, $2, $2-5, or $0 for "Not interested")

### 7.4 Features

✅ **Automatic sheet creation** - Creates both "Referral Sources" and "Pricing Willingness" sheets if they don't exist

✅ **Newest entries at top** - New submissions inserted at row 2

✅ **Formatted headers** - Purple background (#9B6BFF) with white text

✅ **Email matching** - You can match this data with the "Leads" sheet by email to get complete user profiles

✅ **Mobile responsive** - Works seamlessly on mobile devices

✅ **Polished messaging** - Professional, startup-friendly copy

### 7.5 Analytics Tracking

The exit intent modal tracks events in Google Tag Manager:
- `referral_source_submitted` - When user completes both referral source and pricing questions
- Includes both `source` and `pricing` parameters

### 7.6 No Additional Setup Required

Both referral source and pricing tracking use the same Google Sheet ID you configured in Step 6.3. The script will automatically create two new sheet tabs:
1. "Referral Sources" - For tracking user acquisition channels
2. "Pricing Willingness" - For tracking pricing sensitivity

## Support

For issues with Google Apps Script, refer to:
- [Google Apps Script Documentation](https://developers.google.com/apps-script)
- [Web Apps Guide](https://developers.google.com/apps-script/guides/web)
