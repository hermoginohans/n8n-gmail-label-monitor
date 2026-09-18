1 file changed
+110
-0
README.workflow.prepared.md
# Gmail Label Monitor to Sheets & Drive

An n8n workflow that checks unread Gmail messages in selected categories, logs email details to Google Sheets, and passes attachments to a separate workflow for Google Drive storage.

The template checks every **15 minutes** and monitors **Primary, Promotions, and Social** by default. Account credentials and destination IDs use placeholders that you must configure before running it.

## Workflow

```
flowchart TD
    A[Every 15 minutes] --> B[Build Gmail category query]
    B --> C[Fetch unread messages and attachments]
    C --> D[Extract sender and message details]
    D --> E[Append to Google Sheets]
    E --> F[Mark message as read]
    D --> G{Has attachments?}
    G -->|Yes| H[Add Drive root folder ID]
    H --> I[Call attachment subworkflow]
    D --> J[Create or resolve category labels]
    J --> K[Apply matching Gmail label]
```

The attachment subworkflow is **not included in this JSON**. Google Drive uploads require that separate workflow and its own configured credentials.

## Requirements

- An n8n instance that supports the node versions in the exported workflow.
- A Gmail account to monitor, connected through an n8n Gmail OAuth2 credential.
- A Google spreadsheet and a Google Sheets OAuth2 credential with edit access to it.
- For attachments: a destination Google Drive folder and a subworkflow that receives message data and binary attachments and uploads them to Drive.

## Setup

### 1. Import the workflow

Download [Gmail Label Monitor to Sheets & Drive.json](<Gmail Label Monitor to Sheets & Drive.json>). In the n8n workflow editor, use **Import from File** and select the downloaded JSON. See the [n8n import documentation](https://docs.n8n.io/workflows/export-import/).

Keep the workflow inactive while configuring and testing it.

### 2. Connect your accounts

Select your Gmail credential in all eight Gmail nodes. Use the account whose inbox you want to monitor. In **Log to Google Sheet**, select a Google Sheets credential belonging to an account with edit access to your spreadsheet; it can be the same Google account.

The credential placeholders are n8n credential references, not email addresses or passwords. Select actual credentials in the editor instead of entering passwords in the JSON.

| Placeholder or setting | Where to configure it | Required value |
| --- | --- | --- |
| `YOUR_GMAIL_CREDENTIAL_ID` | All Gmail nodes | Your connected Gmail credential |
| `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | Log to Google Sheet | Your connected Google Sheets credential |
| `YOUR_GOOGLE_SPREADSHEET_ID` | Log to Google Sheet → Document | Destination spreadsheet ID |
| `Sheet1` | Log to Google Sheet → Sheet | Your spreadsheet tab name |
| `<__PLACEHOLDER_VALUE__Google Drive root folder ID for attachment storage__>` | Add Root Folder → rootFolderId | Destination Drive folder ID |
| `YOUR_PROCESS_ATTACHMENTS_WORKFLOW_ID` | Process Attachments → Workflow | Your attachment subworkflow ID |

### 3. Prepare the spreadsheet

Create a tab named `Sheet1`, or change the sheet name in the workflow. Add these column headers to the first row:

```text
messageId | senderName | senderEmail | subject | matchedLabel | categoryName | dateReceived | hasAttachments
```

Each name above is a separate column. The Sheets node is configured to map incoming fields automatically; confirm its column mapping after selecting your spreadsheet.

For a spreadsheet URL such as `https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit`, use the `SPREADSHEET_ID` portion as the document ID.

### 4. Configure attachment processing

Set `rootFolderId` in **Add Root Folder** to your destination folder ID. In **Process Attachments**, select an imported or newly created attachment subworkflow.

The parent is configured to call the subworkflow once per item and wait for it to finish. Your subworkflow should accept the extracted message fields, `rootFolderId`, and the binary attachment data. Configure Google Drive credentials and upload logic in that subworkflow. The intended folder organization is label, date, then sender; this parent workflow does not implement that storage logic itself.

For a logging-only setup, disconnect the branch from **Extract Sender Info** to **Has Attachments?** before running the workflow.

### 5. Choose categories and schedule

Edit **Config - Labels to Monitor** to choose the monitored categories:

```javascript
const CATEGORIES = ['primary', 'promotions', 'social'];
return [{ json: { categories: CATEGORIES } }];
```

The extraction and labeling logic currently handles these three categories. Supporting Updates or Forums also requires changes to **Extract Sender Info**, the label creation nodes, and **Build Apply List**.

Change **Every 15 Minutes** to adjust the polling interval.

### 6. Test and enable

Use a small set of test messages and run the workflow manually. Confirm that rows appear in the sheet, messages are marked as read, and the expected labels are applied. If you configured attachments, verify that the subworkflow receives the files and uploads them successfully.

Once the checks pass, publish or activate the workflow using the control available in your n8n version to enable scheduled runs. See [n8n publishing documentation](https://docs.n8n.io/workflows/publish/).

## Current behavior and limitations

- The query filters by Gmail categories, while the labeling branch creates and applies user labels named Primary, Promotions, and Social.
- `matchedLabel` currently falls back to `Unlabeled` because **Build Query** supplies an empty label map. `categoryName` is calculated separately.
- Rows are appended without deduplication. Retrying a partially completed run or marking a processed message unread may create duplicate rows.
- Marking messages as read is on the Sheets branch. It is not gated on successful attachment storage, so an attachment failure can leave a message read and excluded from future unread searches.
- The date extraction falls back to the current date when it cannot convert the incoming date to a number. Verify `dateReceived` against your Gmail output if exact received dates matter.
- The exported workflow is inactive. End-to-end execution requires your own credentials, destinations, and attachment subworkflow; it has not been verified against your connected accounts as part of preparing this README.

## Files

| File | Purpose |
| --- | --- |
| `Gmail Label Monitor to Sheets & Drive.json` | Parent n8n workflow template |
| `README.md` | Configuration and usage instructions |

Upload both files to the root of your GitHub repository. Include the attachment subworkflow export separately if you want others to reproduce the full Drive storage setup.
