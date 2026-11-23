# The Survey123 Connect rich-text editor deletes your HTML

*Worked on: October 2026*

I was finishing a public Survey123 form and wanted the thank-you screen to show the organization's logo. The logo was already hosted, so this should have been one `<img>` tag in the submission message. It kept disappearing.

## What went wrong

I would add the image and it would later be gone again. The pattern turned out to be simple: every time I opened that cell in Connect's rich-text view, the `<img src>` was quietly removed. No warning, no error. The editor just rewrote the HTML into the subset it understands and saved that.

It isn't only images. The same editor strips other styling it doesn't support, including:

- `background-color`
- `border`
- `padding`

So a nicely formatted note or hint can lose its look just because someone opened it to fix a typo.

## The workaround

I stopped editing HTML-bearing labels and hints in Connect at all. Instead I edit them directly in the XLSForm spreadsheet, and I treat the rich-text editor as off-limits for those cells. Connect still publishes the form; I just never open those particular cells in its editor.

For repeatable edits I used openpyxl, so the HTML goes into the spreadsheet exactly as written. A simplified version, with placeholder sheet, column, and file names:

```python
from openpyxl import load_workbook

XLSX = "public_survey.xlsx"
LOGO_HTML = (
    '<p style="text-align:center">'
    '<img src="https://example.com/logo.png" alt="Logo" width="200">'
    '</p>'
    '<p>Thank you for your response.</p>'
)

wb = load_workbook(XLSX)
ws = wb["settings"]

# Find the column by header instead of hard-coding a letter
headers = {cell.value: cell.column for cell in ws[1]}
col = headers["submission_message"]  # placeholder column name
ws.cell(row=2, column=col, value=LOGO_HTML)

wb.save(XLSX)
```

After writing the file, I re-open the form in Connect to publish, without touching that cell, and check the result in the live form.

## A related gotcha: HTML in choice labels

While I was in there, I hit a second HTML surprise. If you put HTML in the labels of a `select_one` question, that markup doesn't stay in the form. It ends up as literal text in the layer's domain, so pop-ups and anything else that reads the domain show raw tags like `<b>Yes</b>` instead of formatted text.

`select_multiple` behaves differently: it only stores the choice names, so the label HTML doesn't leak into the data the same way. But that difference is easy to forget, and it means formatting that looks fine in the form can make the data look broken everywhere else.

My rule now is to keep choice labels plain text and put any formatting in notes or hints instead.

## Takeaway

Survey123 Connect's rich-text editor isn't a safe place for HTML it doesn't support. Opening a cell in it can silently strip images and styling, and there's no prompt telling you it happened. Edit HTML-bearing labels and hints directly in the XLSForm (a script like openpyxl makes this repeatable), keep those cells out of the rich-text editor, and keep HTML out of `select_one` choice labels unless you want the tags showing up in your pop-ups.
