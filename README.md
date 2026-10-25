===============================================================================
WEALTH DATA SOURCES REPORT: OPERATIONAL & INTEGRATION ANALYSIS
===============================================================================

1. EXECUTIVE OVERVIEW
-------------------------------------------------------------------------------
This report provides an operational and technical evaluation of four core data 
repositories utilized within wealth management technology ecosystems:
  1. SEC Form 13F (Institutional Equity Holdings)
  2. SEC Form ADV Part 1A (RIA Regulatory & Disciplinary Disclosures)
  3. Redtail CRM (Client Relationship Management)
  4. SS&C Black Diamond (Portfolio Accounting & Performance Reporting)

The analysis establishes a unified analytical schema across public regulatory 
filings and proprietary client databases, defining freshness parameters, data 
gaps, relational join logic, and sample business queries while maintaining strict 
compliance with zero-PII data privacy standards.


2. SOURCE PROFILES SUMMARY
-------------------------------------------------------------------------------
+--------------------+--------------------------------------------------------+
| Data Source        | SEC Form 13F                                           |
| Contents           | Equity holdings (> $100M AUM institutional managers)  |
| Key Fields         | CUSIP, Ticker, Issuer Name, Value ($K), Shares,        |
|                    | Investment Discretion, Voting Authority                |
| Access Protocol    | SEC EDGAR REST API / Quarterly Bulk TSV Data Sets      |
| Update Frequency   | Quarterly (filed within 45 days post calendar quarter) |
| Volume Estimate    | ~5,000+ filers; ~1,000,000+ holding rows per quarter   |
+--------------------+--------------------------------------------------------+
| Data Source        | SEC Form ADV (Part 1A)                                 |
| Contents           | Registered Investment Adviser disclosures & AUM        |
| Key Fields         | Firm CRD, CIK, Legal Name, Total AUM, Account Counts,  |
|                    | Disciplinary Reporting Page (DRP) Flags                |
| Access Protocol    | Investment Adviser Public Disclosure (IAPD) / SEC Bulk |
| Update Frequency   | Annual Updating Amendment (within 90 days FYE) +       |
|                    | Interim amendments for material changes                |
| Volume Estimate    | ~15,000+ SEC-registered RIAs; ~35,000+ State RIAs      |
+--------------------+--------------------------------------------------------+
| Data Source        | Redtail CRM                                            |
| Contents           | Client demographics, contact profiles, households      |
| Key Fields         | Contact ID, Household ID, Category, Tax ID (SSN/EIN),  |
|                    | User-Defined Fields (UDFs), Status                     |
| Access Protocol    | Redtail REST API / Direct Database Sync                |
| Update Frequency   | Real-time / Intraday API updates                       |
| Volume Estimate    | 1,000s to 100,000s contact records per advisory firm   |
+--------------------+--------------------------------------------------------+
| Data Source        | SS&C Black Diamond                                     |
| Contents           | Reconciled portfolio holdings, performance, valuations |
| Key Fields         | Account ID, Household ID, Custodian, Ticker, CUSIP,    |
|                    | Market Value, Asset Class, Quantity                    |
| Access Protocol    | Black Diamond REST API / Direct Custodial Feeds        |
| Update Frequency   | Daily overnight batch sync post custodial reconcile    |
| Volume Estimate    | 10,000s of accounts and position records per firm      |
+--------------------+--------------------------------------------------------+


3. FRESHNESS & DATA GAPS ANALYSIS
-------------------------------------------------------------------------------
* SEC Form 13F:
  - Reporting Lags: 45-day delay following quarter-end creates backward-looking 
    position views.
  - Omissions: Excludes short positions, non-US listed equities, cash, and 
    holdings below SEC reporting thresholds.

* SEC Form ADV Part 1A:
  - Refresh Cycle: Annual update within 90 days of FYE; interim amendments 
    occur as needed for material firm changes.
  - Coverage Gaps: DRP disclosures require cross-verification against state 
    regulatory feeds for real-time disciplinary tracking.

* Redtail CRM & SS&C Black Diamond:
  - CRM Gaps: Frequently unpopulated User-Defined Fields (UDFs) and stale client 
    risk tolerance metrics.
  - Reconciliation Gaps: Cash and money market positions in Black Diamond 
    require explicit ticker re-mapping rules (e.g., mapping "$CASH" or "CASH" 
    to standardized asset codes) for seamless downstream CRM aggregation.


4. RELATIONAL JOIN ARCHITECTURE & MATCHING RULES
-------------------------------------------------------------------------------
1. Black Diamond Holdings -> SEC Form 13F Holdings
   - Join Pathway: Position.CUSIP / Ticker -> 13F.CUSIP / Ticker
   - Classification: Clean Key
   - Rule / Logic: Exact deterministic match on 9-character CUSIP or ticker.

2. 13F Filer -> ADV Adviser
   - Join Pathway: 13F.CIK / CRD -> FormADV.CRD / CIK
   - Classification: Clean Key
   - Rule / Logic: Exact match using official SEC Central Index Key (CIK) or 
     Firm CRD Identifier.

3. Redtail Contact -> Black Diamond Account
   - Join Pathway: Contact.TaxID / HouseholdID -> Account.TaxID / HouseholdID
   - Classification: Needs a Rule
   - Multi-tier Rule Logic:
     * Tier 1 (Primary): Exact match on Integration_External_ID.
     * Tier 2 (Secondary): Exact match on Tax_ID (SSN or EIN).
     * Tier 3 (Fallback): Probabilistic match on Normalized_LastName + 
       Normalized_FirstName + ZipCode. Unresolved ties route to an exception 
       review queue.


5. SAMPLE ANALYTICAL BUSINESS QUERIES
-------------------------------------------------------------------------------
Query 1: Institutional Co-Investment Benchmarking
  - Question: Which institutional 13F managers hold the top 5 equity positions 
    present in our firm's aggregate Black Diamond portfolios?
  - Business Purpose: Validates internal asset allocation strategies against 
    broad institutional consensus.

Query 2: Adviser Disciplinary & Risk Screening
  - Question: Are any 13F filers or ADV RIAs holding our core positions 
    flagged for recent Disciplinary Reporting Events (DRPs) in Form ADV Part 1A?
  - Business Purpose: Automated counterparty and sub-adviser due diligence.

Query 3: Unallocated Cash & Portfolio Rebalancing Targeting
  - Question: Which client accounts in Black Diamond hold >10% cash where the 
    linked Redtail Contact record indicates a "Growth" profile?
  - Business Purpose: Identifies immediate cash-deployment and yield-optimization 
    opportunities.


6. GOVERNANCE & OPEN ISSUES
-------------------------------------------------------------------------------
- PII Policy: Strict zero-PII enforcement across all reporting files. Client 
  identifiers utilize synthetic UUIDs and sanitized structures.
- Unresolved Discrepancies: Symbol mapping conflicts (e.g., unmapped cash tickers) 
  and orphan CRM records are escalated to Data Operations for manual review.
===============================================================================
