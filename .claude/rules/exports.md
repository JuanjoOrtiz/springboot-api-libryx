---
paths:
  - "src/main/java/**/export/**"
---

# Export rules

- PDF with iText, XLSX with Apache POI. One `ReportExporter` implementation per format, selected by
  format (Strategy). Adding a format is adding a class.
- Files are generated **on the fly** and streamed back; nothing is written to disk.
- The response is 200 with the format's `Content-Type` and `Content-Disposition: attachment`.
- Every export inserts a row in `export_logs` in the same request: who, which report, format,
  filters, row count and whether it contains personal data (`contains_pii`). This is the GDPR audit trail.
- An export with personal data follows the same authorization as the equivalent endpoint: only `ADMIN`.
- Exports are rate limited (10 per hour per user) and record `libryx.exports.generated`.
- Adding a report type requires a migration that updates the `CHECK` on `export_logs.report_type`.
