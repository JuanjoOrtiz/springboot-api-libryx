---
paths:
  - "src/main/java/**/export/**"
---

# Export rules

- PDF with iText, XLSX with Apache POI. One `ReportExporter` implementation per format, selected by
  format (Strategy). Adding a format is adding a class.
- Files are generated **on the fly** and streamed back; nothing is written to disk.
- The response is 200 with the format's `Content-Type` and `Content-Disposition: attachment`.
- Exports are rate limited (10 per hour per user) and record `libryx.exports.generated`.
- Adding a report type requires a migration that updates the `CHECK` on `export_logs.report_type`.

## Reports and access

| Report | Who | Personal data |
|---|---|---|
| `USERS` | `ADMIN` only | yes |
| `LOANS`, `LOAN_REQUESTS`, `SANCTIONS` | `LIBRARIAN` and `ADMIN` | no: the user appears by member number (`LBX-000123`), never by name or contact data |
| `CATALOG`, `INVENTORY` | `LIBRARIAN` and `ADMIN` | no |
| `MY_LOANS` | `USER`, their own loans only | no |

- An export takes the same criteria record and validation as the equivalent list endpoint, so it
  contains exactly what that list shows.

## Size

- At most `libryx.export.max-rows` rows. Above that, 422 `EXPORT_TOO_LARGE`, asking to narrow the filters.
- XLSX is written in streaming mode (`SXSSFWorkbook`) so a large report does not fill the memory.

## File content

- File name `libryx-<report>-<yyyyMMdd-HHmm>.<ext>`, never with personal data.
- Column headers come from `messages.properties` (Spanish); dates in the library time zone.

## Audit

- Every export inserts a row in `export_logs`: who, which report, format, filters, row count and
  whether it contains personal data (`contains_pii`). This is the GDPR audit trail.
- The row is written once the data has been queried and **before streaming starts**, so it is kept
  even if the client cancels the download.
