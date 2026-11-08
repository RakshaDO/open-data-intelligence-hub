# Phase 1 Implementation: Rule Registry & Deterministic Calculation Engine

## 1. Architectural Summary

Phase 1 establishes the deterministic mathematical foundation and authoritative tax rule management layer for the **Resilient, Multi-Agent Tax Rule Engine & Data Reconciler**.

### Non-Negotiable Invariants Upheld
1. **Zero-Hallucination Arithmetic**: AI models (OpenAI/Ollama) are strictly forbidden from generating tax percentages or performing authoritative tax math. All calculations are executed by pure Python deterministic algorithms utilizing arbitrary-precision `decimal.Decimal`.
2. **Cryptographic Rule Immutability**: All statutory rules carry canonical SHA-256 checksums. Any runtime payload drift or unauthorized modification triggers immediate execution abortion.
3. **Verified vs. Synthetic Segregation**: Synthetic rules for automated unit testing and edge cases are explicitly labeled (`is_synthetic: true`) and cannot be executed in production workflows without explicit caller opt-in (`allow_synthetic: true`).
4. **Relational Persistence**: Uses SQLAlchemy 2.0 with an SQLite backend (WAL mode enabled) designed with full dialect-agnostic PostgreSQL compatibility.

---

## 2. Component Design

### 2.1 Tax Rule Registry (`backend/app/engine/rules/registry.py`)
- **Authority**: Loads canonical IRS Publication 946 MACRS rules (`verified_macrs.json`) and clearly labeled test rules (`synthetic_rules.json`).
- **Integrity**: Enforces SHA-256 checksum validation upon loading and registration via `compute_rule_checksum()`.
- **Conflict Prevention**: Rejects duplicate rule registrations when rule code and version collide with different canonical parameters (`RuleVersionConflictError`).
- **Strict Retrieval**: Looks up rules by jurisdiction, asset category, in-service date, and tax year. Flags missing rules with `TaxRuleNotFoundError(requires_human_review=True)`.

### 2.2 Deterministic Calculation Engine (`backend/app/engine/calculator.py`)
- **Arbitrary-Precision Arithmetic**: Uses `decimal.Decimal` with `ROUND_HALF_UP` for all monetary quantizations (2 decimal places) and rate fractions.
- **Straight-Line Engine (`straight_line.py`)**: Computes multi-year schedules under Half-Year and Full-Year conventions.
- **Declining Balance Engine (`declining_balance.py`)**: Executes MACRS 200% DB and 150% DB table-based or formulaic schedules with statutory straight-line switch logic.
- **Basis Adjustments**:
  - Prorates original cost basis by `business_use_percentage`.
  - Deducts elected `section_179_deduction` before depreciation calculations.
  - Computes `bonus_depreciation_rate` on post-179 basis.
  - Applies statutory luxury automobile/equipment caps (`annual_limit`) prorated to business use.
  - Guarantees ending book value reaches exact salvage value with zero penny-leakage.

### 2.3 Database Layer (`backend/app/db/models.py`)
- `DBTaxRule`: Master table for verified and synthetic tax rules.
- `DBRuleVersion`: Immutable historical snapshot of rule versions.
- `DBCalculationRecord`: Calculation header record capturing asset metadata, basis adjustments, and totals.
- `DBCalculationInput`: Normalized input parameters snapshot.
- `DBCalculationResultItem`: Year-by-year schedule line items (beginning basis, rate, expense, accumulated, ending basis).

### 2.4 API Layer (`backend/app/api/v1/`)
- `GET /api/v1/health`: Readiness probe reporting engine status and rule registry inventory.
- `POST /api/v1/rules`: Register new tax rules with SHA-256 validation.
- `GET /api/v1/rules`: Search and filter tax rules by jurisdiction, category, year, and verification status.
- `GET /api/v1/rules/{rule_id}`: Fetch single rule metadata.
- `POST /api/v1/rules/validate`: Offline checksum computation and schema validator.
- `POST /api/v1/calculate/depreciation`: Execute deterministic calculation and persist full schedule to database.
- `GET /api/v1/calculate/history`: List recent calculation runs.
- `GET /api/v1/calculate/history/{calculation_id}`: Retrieve full calculation results and complete schedule.

---

## 3. Calculation Examples

### Example 1: 5-Year MACRS 200% Declining Balance (IRS Table A-1)
- **Asset**: Enterprise Database Server
- **Acquisition Cost**: \$10,000.00
- **Placed in Service**: 2024-03-15 (Tax Year 2024)
- **Business Use**: 100.00%
- **Rule Applied**: `IRS-MACRS-GDS-05YR-200DB-HY`

| Period | Tax Year | Beginning Book Value | Depreciation Rate | Depreciation Expense | Accumulated Depreciation | Ending Book Value |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 1 | 2024 | \$10,000.00 | 20.00% | \$2,000.00 | \$2,000.00 | \$8,000.00 |
| 2 | 2025 | \$8,000.00 | 32.00% | \$3,200.00 | \$5,200.00 | \$4,800.00 |
| 3 | 2026 | \$4,800.00 | 19.20% | \$1,920.00 | \$7,120.00 | \$2,880.00 |
| 4 | 2027 | \$2,880.00 | 11.52% | \$1,152.00 | \$8,272.00 | \$1,728.00 |
| 5 | 2028 | \$1,728.00 | 11.52% | \$1,152.00 | \$9,424.00 | \$576.00 |
| 6 | 2029 | \$576.00 | 5.76% | \$576.00 | \$10,000.00 | \$0.00 |

### Example 2: Partial Business-Use Adjustment with Section 179
- **Asset**: Mobile Workstation
- **Acquisition Cost**: \$12,000.00
- **Business Use**: 80.00% $\rightarrow$ Business Basis = \$9,600.00
- **Section 179 Deduction**: \$2,000.00 $\rightarrow$ Depreciable Basis = \$7,600.00
- **Rule Applied**: `IRS-MACRS-GDS-05YR-200DB-HY`
- **Year 1 Total Deduction**: \$2,000.00 (Sec 179) + \$1,520.00 (20% of \$7,600.00) = **\$3,520.00**

---

## 4. API Usage Examples

### 4.1 Execute Depreciation Calculation
```http
POST /api/v1/calculate/depreciation
Content-Type: application/json

{
  "asset_name": "Dell PowerEdge Server",
  "acquisition_cost": "25000.00",
  "salvage_value": "0.00",
  "placed_in_service_date": "2024-04-10",
  "tax_year": 2024,
  "jurisdiction": "US-FEDERAL-IRS",
  "asset_category": "Computer Equipment",
  "business_use_percentage": "100.00",
  "section_179_deduction": "5000.00",
  "bonus_depreciation_rate": "0.00"
}
```

### 4.2 Query Calculation History
```http
GET /api/v1/calculate/history?tax_year=2024&limit=10
```

### 4.3 Validate Custom Tax Rule Offline
```http
POST /api/v1/rules/validate
Content-Type: application/json

{
  "rule_id": "CUSTOM-SL-3YR",
  "rule_code": "CUSTOM_SL_3YR",
  "title": "Custom 3-Year Straight Line Rule",
  "jurisdiction": "US-FEDERAL-TEST",
  "statutory_authority": "INTERNAL TEST POLICY",
  "asset_category": "Special Tools",
  "recovery_period_years": 3,
  "method": "STRAIGHT_LINE",
  "convention": "HALF_YEAR",
  "is_synthetic": true,
  "version": "1.0",
  "effective_from": "2024-01-01",
  "rates": ["0.1667", "0.3333", "0.3333", "0.1667"]
}
```

