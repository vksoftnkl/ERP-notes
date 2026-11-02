# Company Master Table Documentation (public.companys)

## 1. Overview
The `companys` table is a master configuration table used to store core company/profile information for each business entity configured in the ERP system. Each record represents one company and controls statutory, billing, taxation, and system behavior settings for that company.

This table is critical for multi-company ERP environments and is referenced across billing, GST, e-invoicing, e-way bill, and system configuration modules.

---
## 2. Business Purpose
The purpose of the `companys` table is to:

- Store company identity and branding information
- Maintain statutory compliance data (GST, PAN, FSSAI, etc.)
- Control tax and regulatory applicability flags
- Store billing and invoice configuration
- Control ERP feature enablement at company level
- Support multi-company operations in a single database

---
## 3. Functional Needs
This table supports the following ERP functional needs:

- Company profile management
- GST & tax configuration per company
- E-invoice & E-way bill enablement
- Billing header/footer configuration
- Bank linkage for accounting
- Multi-branch / multi-company setups
- Feature toggles (SMS, negative stock, data export, etc.)

---
## 4. Column-Level Documentation

### 4.1 Primary Identity
| Column | Type | Purpose |
|--------|------|---------|
| comp_id | integer (PK) | Unique company identifier |
| comp_name | text | Registered company name |
| comp_short | text | Short name / alias |

### 4.2 Address Information
| Column | Purpose |
|--------|---------|
| comp_address1 | Primary address line 1 |
| comp_address2 | Address line 2 |
| comp_address3 | Address line 3 |
| comp_pin | Postal PIN code |
| state_name | State name |
| state_code | State GST code (2 char) |

### 4.3 Transport / Temporary Address
| Column | Purpose |
|--------|---------|
| comp_tname | Temporary / transport name |
| comp_taddress1 | Temp address line 1 |
| comp_taddress2 | Temp address line 2 |
| comp_taddress3 | Temp address line 3 |

### 4.4 Contact Information
| Column | Purpose |
|--------|---------|
| comp_tel | Telephone |
| comp_phone | Mobile number |
| comp_mail | Email |
| website_name | Website URL |

### 4.5 Statutory & Compliance
| Column | Purpose |
|--------|---------|
| comp_gst_no | GST Registration Number |
| pan_no | PAN Number |
| fssai_no | FSSAI License |

### 4.6 Tax & Regulatory Controls
| Column | Purpose |
|--------|---------|
| gst_applicable | GST enabled |
| tcs_applicable | TCS enabled |
| einvoice_applicable | E-Invoice enabled |
| einvoice_date | E-Invoice start date |
| eway_applicable | E-Way Bill enabled |
| eway_date | E-Way Bill start date |
| eway_inter_limit | Inter-state EWB limit |
| eway_intra_apl | Intra-state EWB enabled |
| eway_intra_limit | Intra-state EWB limit |
| einvoice_incl_eway | Include EWB in E-Invoice |

### 4.7 System & ERP Controls
| Column | Purpose |
|--------|---------|
| comp_default | Default company |
| is_active | Active / inactive company |
| stylesheet_id | UI theme/stylesheet |
| price_fixing | Price fixing policy |
| sms_applicable | SMS enabled |
| data_exp_apl | Data export enabled |
| negstk_apl | Allow negative stock |
| nrm_user_apl | Normal user access allowed |

### 4.8 Accounting & Banking
| Column | Purpose |
|--------|---------|
| bank_id | Linked default bank |

### 4.9 Billing & Messages
| Column | Purpose |
|--------|---------|
| bill_greeting | Invoice greeting message |
| comp_remarks | Company remarks |
| prefix_code | Bill/document prefix |

### 4.10 Audit Columns
| Column | Purpose |
|--------|---------|
| created_on | Record creation timestamp |
| modified_on | Last modified timestamp |

---
## 5. Data Relationships

Typical relationships:

- companys.comp_id → sales, purchase, ledger, stock, invoice tables
- companys.bank_id → banks.bank_id

---
## 6. NestJS CRUD API Design

### 6.1 API Module Structure

```
company/
 ├── company.controller.ts
 ├── company.service.ts
 ├── company.module.ts
 ├── dto/
 │   ├── create-company.dto.ts
 │   └── update-company.dto.ts
```

---
## 6.2 REST Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | /api/company | Create new company |
| GET | /api/company | Get all companies |
| GET | /api/company/:id | Get single company |
| PUT | /api/company/:id | Update company |
| DELETE | /api/company/:id | Soft delete / deactivate |

---
## 6.3 Sample DTO (Create)

```ts
export class CreateCompanyDto {
  compName: string;
  compShort: string;
  stateCode: string;
  compGstNo?: string;
  panNo?: string;
  isActive?: boolean;
  gstApplicable?: boolean;
  einvoiceApplicable?: boolean;
  ewayApplicable?: boolean;
}
```

---
## 6.4 Service Layer (Business Logic)

Responsibilities:

- Validate GST & State code combinations
- Ensure only one comp_default = true
- Handle compliance flags based on dates
- Enforce negative stock & pricing policies

---
## 6.5 Soft Delete Strategy (Recommended)

Instead of physical DELETE:

```sql
UPDATE companys
SET is_active = false
WHERE comp_id = :id;
```

---
## 7. Security & Validation

Recommended validations:

- Unique GST number
- Valid PAN format
- Valid State Code
- Only one default company

---
## 8. Performance & Indexing

Existing:
- Primary key index on comp_id

Recommended:

```sql
CREATE UNIQUE INDEX idx_company_gst
ON companys (comp_gst_no)
WHERE comp_gst_no IS NOT NULL;
```

---
## 9. Usage in VKSoftwares ERP

This table is central to:

- Multi-company support
- GST return generation
- Invoice printing headers
- E-invoice / E-way bill API integrations
- Feature enablement per customer

---
## 10. Future Enhancements

- Company logo storage
- Digital signature support
- API credential storage for GST/EWB portals
- Multi-branch mapping

---
## 11. Summary

The `companys` table acts as the backbone configuration entity for company-level behavior in the ERP. Proper API validation and business rules are critical to ensure statutory compliance and system consistency.

---

Prepared for: VKSoftwares ERP System

