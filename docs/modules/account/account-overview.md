# 💳 ACCOUNT - Accounts Module Overview

**Module ID**: account
**Version**: 1.0
**Last updated**: 2026-01-26
**Purpose**: Provide detailed customer account inquiry and update capabilities so back-office teams can view balances, validate credit usage, and keep personal data current within a PCI-aware workflow.

---

## 📋 Module Summary

- **Business Context**: This module sits behind the authentication wall and gives back-office representatives, analysts, and administrators a single place to view every account metric (balances, limits, cycles) and to adjust customer data in real time.
- **Primary Responsibilities**:
  - Query accounts by the canonical 11-digit `accountId` and resolve related customer/card data via `CardXrefRecord`.
  - Surface key financial KPIs (balance, credit limit, available credit, status) with masked sensitive values (SSN, card numbers).
  - Allow controlled updates on account metadata and customer demographics while keeping operations transactional.
  - Enforce data quality (account ID format, ZIP code pattern, FICO range, mandatory fields).
  - Power data for downstream modules (Credit Card, Transactions, Bill Payment) via the shared `Account` entity.

---

## 🏗️ Key Components & Patterns

| Component | Responsibility | Reuse Potential |
|-----------|----------------|-----------------|
| `AccountViewScreen.tsx` | Presents search controls, status cards, and a card grid for account/customer detail | Template for other inquiry screens.
| `AccountUpdateScreen.tsx` | Toggle-driven edit mode with inline validation and save confirmation | Pattern reused for transactional forms in other modules.
| `useAccountView.ts` | Hook that orchestrates loading, error handling, and validation for the lookup flow | Blueprint for any feature that fetches keyed data.
| `useAccountUpdate.ts` | Hook that compares original vs current payload to avoid dirty submissions | Hook pattern duplicated in other update-heavy flows.
| `AccountViewService.java` | Backend service that joins `CardXref → Account → Customer` to build the view model | Example of multi-entity aggregation.
| `AccountUpdateService.java` | Transactional service (`@Transactional`) that updates `Account` + `Customer` together | Gold standard for atomic updates.
| `AccountValidationService.java` | Centralizes field validators (`isValidAccountId`, `validateYesNo`, FICO ranges, etc.) | Should be reused before calling other services.


**UI Patterns**: Direct implementation per feature (no shared BaseForm/MUI wrappers). Forms live in full pages using Material-UI components (`TextField`, `Card`, `Grid`, `Button`, `Dialog`). Validation is manual and relies on HTML5 attributes plus helper functions. Notifications are currently handled inline (errors displayed in the form) and there is no global snackbar system yet.

**i18n Status**: Text is hard-coded in English. Future internationalization should follow a `app/i18n/locales/{en,es,pt}` structure with keys grouped by page and form (e.g., `account.view`, `account.form`).

---

## 🌐 Public Interfaces (APIs)

### `GET /api/account/acccount?accountId={id}`
**Description**: Returns the fully enriched account view (account, customer, masked cards) for the given `accountId`.
**Response (200)**:
```json
{
  "accountId": "11111111111",
  "status": "Y",
  "balance": 1250.75,
  "creditLimit": 5000.00,
  "availableCredit": 3749.25,
  "groupId": "PREMIUM",
  "customer": {
    "customerId": "1000000001",
    "firstName": "JOHN",
    "middleName": "MICHAEL",
    "lastName": "SMITH",
    "ssn": "123-45-6789",
    "ficoScore": 750,
    "address": {
      "addressLine1": "123 MAIN STREET",
      "addressLine2": "APT 4B",
      "city": "NEW YORK",
      "state": "NY",
      "zipCode": "10001",
      "country": "USA"
    },
    "phones": [
      {
        "phoneType": "HOME",
        "phoneNumber": "(555) 123-4567"
      }
    ]
  },
  "cards": [
    { "cardNumber": "4111-1111-1111-1111", "status": "ACTIVE" }
  ]
}
```

### `PUT /api/account/update`
**Description**: Persists updates against `Account` and the associated `Customer` in a transactional scope.
**Request Body**:
```json
{
  "accountId": "11111111111",
  "customer": {
    "firstName": "JOHN",
    "middleName": "MICHAEL",
    "lastName": "SMITH",
    "address": {
      "addressLine1": "456 NEW STREET",
      "city": "NEW YORK",
      "state": "NY",
      "zipCode": "10002"
    }
  }
}
```
**Response (200)**:
```json
{ "success": true, "message": "Account updated successfully" }
```

---

## 📦 Data Models (TypeScript + Java)

```typescript
interface Account {
  accountId: string;
  balance: number;
  creditLimit: number;
  availableCredit: number;
  status: string;
  groupId: string;
  customer: Customer;
  cards: CreditCard[];
}

interface Customer {
  customerId: string;
  firstName: string;
  middleName: string;
  lastName: string;
  ssn: string;
  ficoScore: number;
  address: Address;
  phones: Phone[];
}

interface Address {
  addressLine1: string;
  addressLine2?: string;
  city: string;
  state: string;
  zipCode: string;
  country: string;
}

interface Phone {
  phoneType: string;
  phoneNumber: string;
}

interface CreditCard {
  cardNumber: string;
  accountId: string;
  embossedName: string;
  expirationDate: string;
  status: 'ACTIVE' | 'INACTIVE' | 'EXPIRED' | 'BLOCKED';
  cvv: string;
  cardType: string;
}
```

> **Java Entities** (simplified)
>
> `Account`: `@Entity` with 11-digit `accountId`, financial fields (`creditLimit`, `currentBalance`), status (`Y/N`).
>
> `Customer`: 9-digit `customerId`, SSN, FICO (300-850), address, state, zip.
>
> `CardXrefRecord`: Join table linking Account ↔ Customer ↔ CreditCard.

---

## 🧩 Business Rules

1. `accountId` must be exactly 11 digits and not all zeros.
2. Only `status='Y'` accounts are eligible for updates or transaction-related lookups.
3. `balance` may be negative (overdraft scenarios); available credit = `creditLimit - balance`.
4. Every account has at least one associated customer and one masked card in the view payload.
5. SSN is displayed as `***-**-XXXX`; card numbers are masked as `****-****-****-XXXX`.
6. FICO score must be between 300 and 850; ZIP accepts `^\d{5}(-\d{4})?$`.
7. Updates run within a transactional boundary (`@Transactional`) and acquire a pessimistic read lock (`READ FOR UPDATE`).
8. Account update validation is centralized in `AccountValidationService` (FICO range, yes/no flags, ZIP, names).
9. The module must log attempted changes (auditing planned) and show friendly error messages when validation fails.

---

## 🎯 User Story Templates and Complexity

1. **Inquiry**: “Como representante de servicio, quiero buscar una cuenta por su ID para mostrar saldo, límite y tarjetas, de modo que pueda responder dudas del cliente.” *(Simple: 1-2 pts)*
2. **Update**: “Como administrador, quiero actualizar el límite de crédito y la dirección del cliente para reflejar mejoras crediticias en el sistema.” *(Medio: 3-5 pts)*
3. **Compliance**: “Como oficial de cumplimiento, quiero que los datos sensibles se presenten enmascarados para cumplir con PCI-DSS.” *(Simple: 1-2 pts)*
4. **Complex Scenario**: “Como supervisor, quiero incorporar aprobación adicional cada vez que el crédito disponible supere $10,000 y registrar auditoría para cada cambio.” *(Complejo: 5-8 pts)*

**Complexity Guidelines**
- Simple (1-2 pts): Adjust UI text, add read-only field, or reuse existing API with minimal logic.
- Medium (3-5 pts): Add business validation, new editable field with server round-trip.
- Complex (5-8 pts): Integrate scoring engines, orchestrate approvals, or add cross-module side effects.

---

## ✅ Acceptance Criteria Patterns

- Authentication: Every screen requires an authenticated session managed by `authSlice`.
- Validation: Account ID requires 11 digits; updates fail fast if ZIP or FICO range invalid.
- Performance: Queries must return in <500ms (P95); show loading indicator on slower calls.
- Error handling: Display contextual messages for “Account not found” or validation failures.
- Security: Mask SSN and card numbers; never log CVV.
- Transactionality: PUT operations must succeed or rollback both account and customer.

---

## ⚡ Performance & Readiness Notes

- **Response budgets**: GETs < 500ms; PUTs < 1s (P95).
- **Concurrency**: Use DB indexes on `accountId`, `customerId`, `cardNumber`; avoid full table scans.
- **Mocks**: MSW provides consistent delays (300-800ms) and ensures UI works offline.
- **Risks**: Heavy lookups across three tables can degrade; plan caching or read replicas if load spikes.
- **Tech Debt**: Notifications system missing (snackbar recommended), validations handled manually => future refactor to shared validation helpers may reduce bugs.

---

## 🔗 Supporting Material

- Interactive guide: `docs/site/modules/account/index.html` (includes code patterns, UI details, performance metrics, risks). 
- System view: `docs/system-overview.md` for the full module catalog, API catalog, and acceptance criteria matrix.

**Get started**: Review `docs/site/modules/account/index.html` for actionable user-story templates, technical patterns, and risk mitigations before writing tickets.
