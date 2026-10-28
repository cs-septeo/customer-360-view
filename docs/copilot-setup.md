# Copilot setup – 360° Customer View

How the semantic model of **360° Customer View - Copy** is prepared for Copilot in Power BI.
Steps 1–3 are done in Power BI Desktop: **Home → Prep data for AI**. Repeat them if the model is rebuilt.

**Prerequisites (IT / Power BI admin):**
- The workspace must run on a Fabric capacity (F SKU) or Premium capacity (P SKU).
- Copilot must be enabled in the tenant settings.

Without these, the Copilot button is not shown.

Already in the model (no action needed):
- **Descriptions** on 18 tables, 54 key columns and all new measures.
- **135 German/English synonyms** in the linguistic schema (`definition/cultures/en-US.tmdl`).

---

## 1. Simplify the data schema

Select only these tables, with all their visible fields. Leave everything else unselected: the 45 unused tables and all `LocalDateTable_…` tables.

| Area | Tables |
|---|---|
| Customer | `fact account`, `fact contacts`, `dim bcbi_companies` |
| CRM sales | `fact opportunity`, `fact lead (4)`, `fact quote`, `fact salesorder` |
| CRM activities | `CRM Activities`, `CRM Contact Activities` |
| Business Central | `ERP ContractHeader`, `ERP ContractLine`, `ERP Debitor`, `ERP Invoices`, `ERP Credit Memos`, `ERP Open Items` |
| Support & projects | `Tickets`, `Organizations`, `EMS projects` |

Hidden key columns (`DebitorKey`, `Account ID`, `Contact ID`, `… Key`) do not need to be selected.

## 2. AI instructions

Paste this text into **Add AI instructions**:

```text
This model is a 360° customer view for STP / Septeo legal software customers (law firms,
insolvency administrators). It combines Dynamics 365 CRM data with Business Central (BC) data.

Customers
- "fact account" is the central customer table (Kunde, Mandant, account). Always answer
  customer questions by filtering on 'fact account'[Account Name].
- A CRM account links to CRM tables by Account ID and to BC tables by BC ID
  (format Entity|CustomerNo, e.g. 1STP|12007). BC company codes (Entity) are e.g. 1STP, ADHO,
  STCH.
- "fact contacts" are the contact persons (Ansprechpartner) of an account.

Business Central (ERP)
- Contracts (Vertraege) = ERP ContractHeader; contract lines (Vertragszeilen) = ERP ContractLine.
  Active contracts have ContractStatus = "ACTIVE". Use the measure [Active Contracts] and
  [Run Rate (Active CLs)] for recurring revenue.
- Invoices (Rechnungen) = ERP Invoices, last 3 years plus all invoices still open.
  Use [Invoiced Amount], [Invoiced Amount (12M)], [Invoice Count], [Overdue Invoices].
- Credit memos (Gutschriften) = ERP Credit Memos, last 3 years. Use [Credit Memo Amount].
- Open items (offene Posten) = ERP Open Items: open customer ledger entries. [Open Amount] is the
  net open balance (payments and credits are negative). [Overdue Amount] only counts positive
  amounts past their due date. "Mahnung" / "Mahnstufe" / reminder / dunning level =
  'ERP Open Items'[Reminder Level]; use [Max Reminder Level] per customer (0 = no reminder).
  Days Overdue and Aging Bucket are calculated at the last data refresh.
- Amounts are in the document currency of each BC company (mostly EUR; STCH is in CHF).
  Do not add amounts of different currencies without saying so.

CRM
- Quotes (Angebote) = fact quote. Open quotes have statecodename "Entwurf" or "Aktiv"
  ([Open Quotes], [Open Quotes Amount]). Sales orders (Auftraege) = fact salesorder.
- Opportunities = fact opportunity (ACV, sales forecast, closing date). Leads = fact lead (4).
- Activities (Aktivitaeten: emails, calls, tasks, appointments) = CRM Activities for the
  account and CRM Contact Activities for a contact, last 2 years only. "Letzter Kontakt" /
  last contact = [Last Activity Date]; [Days Since Last Activity].

Support and projects
- Support tickets = Tickets ([# Tickets]); projects = EMS projects.

Report navigation
- When a user asks about one specific customer, answer and suggest opening the
  "Account Detail" page (right-click the account in the Overview Accounts table ->
  Drill through -> Account Detail). For one contact, suggest "Contact Detail".
- Users may ask in German or English; answer in the language of the question.
```

## 3. Verified answers

Set these up on the **Account Detail** or **Contact Detail** page:
1. Right-click the visual and choose **Set up a verified answer**.
2. Add the trigger phrases.
3. Add **`fact account[Account Name]`** as a filter. For the contact visuals, use **`fact contacts[fullname]`** instead.

This lets users ask for any customer by name.

| # | Visual (page) | Trigger phrases |
|---|---|---|
| 1 | Offene Posten & Mahnungen (Account Detail) | Welche offenen Posten hat der Kunde? · Hat der Kunde Mahnungen? · Which open items does the customer have? · Show overdue items and reminder level |
| 2 | Receivables card (Account Detail) | Wie hoch ist der offene Betrag? · Was ist überfällig? · How much does the customer owe? · Overdue amount of the customer |
| 3 | Rechnungen (Account Detail) | Rechnungen des Kunden · Zeige alle Rechnungen · Show invoices for the customer · Unpaid invoices |
| 4 | Gutschriften (Account Detail) | Gutschriften des Kunden · Show credit memos for the customer |
| 5 | Verträge (Account Detail) | Welche Verträge hat der Kunde? · Aktive Verträge · Which contracts does the customer have? |
| 6 | Vertragszeilen (Account Detail) | Welche Produkte hat der Kunde? · Vertragszeilen · Which products / contract lines does the customer have? |
| 7 | Angebote (Account Detail) | Offene Angebote des Kunden · Show open quotes for the customer |
| 8 | Aufträge (Account Detail) | Aufträge des Kunden · Show orders for the customer |
| 9 | Aktivitäten (Account Detail) | Wann war der letzte Kontakt? · Letzte Aktivitäten · When was the last contact with the customer? · Recent activities |
| 10 | Activity Timeline (Contact Detail) | Aktivitäten des Ansprechpartners · Activities of the contact |

## 4. Test questions after publishing

- "Welche offenen Posten und Mahnstufe hat Wellensiek Rechtsanwälte?" (expected: Mahnstufe 2, 1 overdue item)
- "How many active contracts does PLUTA Rechtsanwalts GmbH have?" (expected: 11)
- "Wann war der letzte Kontakt mit Thorsten Holzmann?" (expected: 30.09.2026)
- "Show invoices of Heuking Kühn Lüer Wojtek" (expected: 129 invoices, open amount 100,328.41)

## Known limitations

- The model is a composite model: it uses DirectQuery to the shared model *Sales & Marketing Reporting – Reporting Playground*. Some *Prep data for AI* options may behave differently there, so test after publishing.
- BC data and CRM activities only cover the periods above (3 and 2 years). Copilot cannot answer questions about older data.
