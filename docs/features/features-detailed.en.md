# Detailed Features & Walkthrough

In this section, we provide detailed step-by-step functionality breakdowns alongside visual examples to demonstrate ease of use.

## 1. Advanced Search and Highlights
When searching for a keyword, the system dives deep—not just querying metadata, but dynamically parsing the actual internal text of OCR-scanned attachments. The results are shown with precise text highlighting context, pinpointing exactly where the word occurs.

![Search Highlights](assets/placeholder.png)
*(Placeholder for search highlighting screenshot)*

## 2. Disposal Management and Certificates
Documents are never randomly deleted. There's a strict lifecycle algorithm handling expiries, followed by a managerial disposal review (Disposal Management), concluding with a generated formal PDF certificate listing all deleted records safely.

![Disposal Certificate](assets/placeholder.png)
*(Placeholder for actual disposal certificate output)*

## 3. Legal Hold Integration
If a subpoena or an ad-hoc legal issue blocks document deletion, management can toggle an immediate "Legal Hold." This explicitly guarantees immunity from any automated deletion bots and logs the exact timestamp into the rigorous `Audit Log`.

![Legal Hold Area](assets/placeholder.png)
*(Placeholder for Document Modal showcasing Legal Hold badge)*
