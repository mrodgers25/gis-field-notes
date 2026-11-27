# "The document must not be an Office file"

*Worked on: October 2026*

I was setting up a Lambda that reads a task-tracking Google Sheet through a Google Cloud service account. The pattern was familiar: share the sheet with the service account's email, store the sheet ID in configuration, call the Sheets API. The first run failed with a 400 error from the Sheets API: "The document must not be an Office file."

## What went wrong

The file looked like a Google Sheet in the browser, but it wasn't one. It was an .xlsx that had been uploaded to Drive and was being displayed in Google's Office-compatibility mode. You can edit it in the browser, so it's easy to miss, but the Sheets API won't read it. It only works with native Google Sheets.

The fix is File → Save as Google Sheets. The catch is that this doesn't convert the file in place. It creates a **new file with a new ID**, and sharing doesn't carry over. So there were two follow-up steps:

1. Update the sheet ID in the Lambda's configuration to point at the new file.
2. Re-share the new file with the service account's email.

Skip the second step and you just trade one error for a permissions error.

```python
import os
from googleapiclient.discovery import build

sheets = build("sheets", "v4", credentials=creds)

# This ID must belong to a native Google Sheet, not an .xlsx in Drive,
# and that file must be shared with the service account.
result = sheets.spreadsheets().values().get(
    spreadsheetId=os.environ["SHEET_ID"],
    range="Tasks!A:Z",
).execute()
```

## Other setup snags along the way

The Office-file error was the headline, but a few AWS setup problems ate time too.

**Region mismatches.** CloudShell and the console were pointed at different regions. Commands in CloudShell returned "not found" for a function I could plainly see in the console. Nothing was missing. I was just looking in two different regions. Now I pass `--region` explicitly instead of relying on whatever the session defaults to:

```bash
aws lambda get-function-configuration \
  --function-name my-task-function \
  --region us-east-1
```

**A missing environment variable after a region move.** When the function ended up in a different region, one environment variable didn't come with it. The code had a fallback default, so instead of failing, it quietly used an old value. Nothing errored. It just pointed at the wrong thing. A required setting shouldn't have a silent default, and after any move I compare the environment variables against the original function.

**Mangled double dashes.** AWS CLI flags pasted into CloudShell came through with their `--` turned into a single dash, so the CLI didn't see the flags I thought I'd given it. When a pasted command misbehaves, it's worth retyping the flags by hand before assuming the command itself is wrong.

## Takeaway

"Looks like a Google Sheet" isn't the same as "is a Google Sheet." If the Sheets API says the document is an Office file, convert it, then remember that conversion means a new file ID and fresh sharing. More generally, most of this setup pain came from things that *looked* right: a file that opened in Sheets, a function I could see in the console, a variable with a fallback, a flag that looked like two dashes. When something says "not found" or quietly does the wrong thing, check the region, the ID, and the actual characters before you debug the code.
