# 💳 ACCOUNT - Accounts module documentation

**Versión**: 2026-01-28  
**Última actualización**: 2026-01-28  
**Propósito**: Servir como guía interna para Product Owners y equipos de ingeniería que necesitan crear User Stories y tareas referentes al módulo de cuentas. El documento reúne la visión de negocio, arquitectura live-code, reglas de validación y patrones concretos que existen en el frontend React + backend API actuales.

---

## 📋 Visión general

**Contexto de negocio**: El módulo de cuentas permite que personal de atención, back-office y administradores consulten y modifiquen datos financieros y personales asociados a cuentas de tarjeta de crédito. Ofrece una vista unificada (AccountView) y una edición segura (AccountUpdate) que preserva las validaciones de negocios y empuja los cambios de forma transaccional al backend (Account + Customer).

**Responsabilidades clave**:
- Búsqueda por `accountId` de 11 dígitos, validación de no ceros y protección de datos.
- Presentación de resumen financiero y cliente en tarjetas MUI con enmascaramiento configurable (SSN, tarjeta).
- Edición segura con `editMode` explícito, validaciones inline (números, ZIP, status) y confirmación antes del guardado.
- Sincronización atómica con APIs RESTful (`GET /account-view`, `PUT /accounts/{id}`) mediante `apiClient`.
- Prevención de estados inconsistentes con detección de cambios (`hasChanges`) y resets desde el hook `useAccountUpdate`.
- Protección de rutas mediante chequeo de `userRole` almacenado en `localStorage`.

---

## 🏗️ Arquitectura del módulo

### Frontend (React + MUI)
- **Páginas**:
  - `AccountViewPage.tsx`: monta `useAccountView`, verifica `userRole`, llama a `initializeScreen` y orquesta navegación.
  - `AccountUpdatePage.tsx`: monta `useAccountUpdate`, limpia datos al montar y expone callbacks para la pantalla de edición.
- **Screens**:
  - `AccountViewScreen.tsx`: formulario de búsqueda, test accounts, toggles de datos sensibles, alertas, tarjetas de información financiera y cliente, acciones de teclado (F3 = exit).
  - `AccountUpdateScreen.tsx`: búsqueda, modo edición, validaciones, control de cambios `hasChanges`, diálogo de confirmación con `Dialog`, shortcuts (F3, F5, F12) y botones Save/Reset.
- **Hooks**:
  - `useAccountView`: `useMutation` personalizado que llama a `apiClient.get('/account-view?accountId=...')`, maneja loading/error y `initializeScreen`.
  - `useAccountUpdate`: dos mutaciones de `useMutation` (`searchAccount` y `updateAccount`), detección de cambios (`JSON.stringify`), validaciones locales y resets.
- **Servicios comunes**:
  - `apiClient` (https://.../app/services/api.ts): encapsula headers, timeout, manejo de tokens y detección de respuestas reales vs. MSW. `account` explota `get`, `put`, parseo de `ApiResponse`.
  - `SystemHeader` y `LoadingSpinner` (layout compartido): consistencia visual de pantallas, transacción `COACTVWC` y `COACTUPC`.

### Backend (API)
- **Endpoints disponibles**:
  - `GET /account-view?accountId={id}`: Devuelve `AccountViewResponse` con valores financieros, cliente, mensajes de error/info y `inputValid`.
  - `GET /account-view/initialize`: Crea payload inicial con `currentDate`, `currentTime`, `transactionId` para evitar pantallas vacías en modo demo.
  - `GET /accounts/{accountId}`: Carga `AccountUpdateData` completa para el formulario de actualización.
  - `PUT /accounts/{accountId}`: Persiste datos actualizados (Account + Customer) en una transacción atómica; responde con `{ success, data }`.
- **Mecánica de autenticación**: Los endpoints se protegen con token (`Authorization: Bearer`) y el frontend lo setea automáticamente desde `localStorage`.

---

## 🔗 APIs documentadas

```text
GET  /account-view?accountId={11-digit}
  → parsea accountId (padStart a 11 dígitos), devuelve AccountViewResponse.

GET  /account-view/initialize
  → Metadata inicial (fecha, hora, transactionId) para pre-cargar la pantalla.

GET  /accounts/{accountId}
  → Obtiene los datos para editar AccountUpdateData (account + customer).

PUT  /accounts/{accountId}
  → Persiste AccountUpdateData; la respuesta incluye `success` y la data guardada.
```

`apiClient` reside en `/app/services/api.ts`, maneja timeouts, logging (🎯), y detecta respuestas directas del backend vs. mocks MSW.

---

## 📊 Modelos de datos clave

### `AccountViewResponse` (frontend `/app/types/account.ts`)
```typescript
interface AccountViewResponse {
  currentDate: string;
  currentTime: string;
  transactionId: string;
  programName: string;
  accountId?: number;
  accountStatus?: 'Y' | 'N';
  creditLimit?: number;
  currentBalance?: number;
  cashCreditLimit?: number;
  currentCycleCredit?: number;
  currentCycleDebit?: number;
  openDate?: string;
  expirationDate?: string;
  reissueDate?: string;
  groupId?: string;
  customerId?: number;
  customerSsn?: string;
  ficoScore?: number;
  firstName?: string;
  lastName?: string;
  addressLine1?: string;
  state?: string;
  zipCode?: string;
  country?: string;
  phoneNumber1?: string;
  infoMessage?: string;
  errorMessage?: string;
  inputValid: boolean;
}
```

### `AccountUpdateData` (frontend `/app/types/accountUpdate.ts`)
```typescript
interface AccountUpdateData {
  accountId: number;
  activeStatus: 'Y' | 'N';
  creditLimit: number;
  cashCreditLimit: number;
  openDate: string;
  expirationDate: string;
  reissueDate: string;
  currentCycleCredit: number;
  currentCycleDebit: number;
  groupId: string;
  customerId: number;
  firstName: string;
  middleName?: string;
  lastName: string;
  addressLine1: string;
  stateCode: string;
  countryCode: string;
  zipCode: string;
  phoneNumber1: string;
  ssn: string;
  governmentIssuedId: string;
  ficoScore: number;
  primaryCardIndicator: string;
}
```

### Valores adicionales:
- `hasChanges` (hook) compara `JSON.stringify` para detectar ediciones locales.
- `validationErrors` en `AccountUpdateScreen` cubre `creditLimit`, `zipCode` y `activeStatus`.

---

## 🔐 Reglas de negocio y validaciones

1. **Account ID**: exactamente 11 dígitos, no puede ser `00000000000`; el campo se valida en los formularios y el backend espera un entero (se parsea con `parseInt`).
2. **Status activo**: `activeStatus` solo acepta `Y` o `N`; se bloquea en el select y se muestra con chips de color (verde/rojo).
3. **Datos sensibles**: SSN (`customerSsn`) y número de tarjeta se enmascaran automáticamente (`***-**-XXXX` y `****-****-****-XXXX`). Solo se revela al activar `showSensitiveData`.
4. **Validaciones numéricas**: Límites y balances se validan localmente para evitar NaN antes de enviar (checksum en `useAccountUpdate`).
5. **ZIP code**: Debe cumplir regex `^\d{5}(-\d{4})?$`.
6. **Modo edición**: La pantalla `AccountUpdateScreen` solo acepta soluciones de guardado cuando `editMode` está activado, `hasChanges` es `true` y no hay errores de validación.
7. **Actualización transaccional**: PUT `/accounts/{id}` guarda `Account` + `Customer`; el hook reinicia `hasChanges` cuando la respuesta indica `success`.
8. **Keyboard shortcuts**: `F3` sale de la pantalla (llamando a `onExit`), `F5` guarda (si hay cambios) y `F12` resetea. Esto mantiene consistencia con terminales principales del sistema.
9. **Test accounts**: En entorno dev se listan IDs preconfigurados para acelerar QA.

---

## 🎯 Patrones de User Stories (domain-specific)

- **Visualizar cuenta**: "Como representante de servicio, quiero consultar el balance y los límites de una cuenta para responder consultas del titular."
- **Actualizar datos críticos**: "Como administrador de cuentas, quiero activar el modo edición y guardar cambios con diálogo de confirmación para mantener datos sincronizados."
- **Seguridad de datos**: "Como oficial de cumplimiento, quiero que los SSN y tarjetas estén enmascarados por defecto y solo se muestren tras mi consentimiento."
- **Inicio rápido**: "Como QA, quiero usar las cuentas de prueba (F3) para verificar flujos sin consultar la base real."

**Guía de complejidad**:
- *Simple (1-2 pts)*: Añadir un campo de solo lectura o un tip de validación en `AccountViewScreen`.
- *Medio (3-5 pts)*: Introducir nuevas reglas de validación (p. ej. nueva regex para `groupId`) o un nuevo campo editable con validaciones backend+frontend.
- *Complejo (5-8 pts)*: Integrar con un servicio externo de scoring o crear auditoría que registre cada actualización de `Account`.

---

## ⚡ Factores de aceleración

- **Material-UI (MUI)**: TextField, Card, Grid, Button, Dialog y Hooks `useTheme` ya están configurados en ambas pantallas; replicar estilos es directo.
- **Hooks compartidos**: `useAccountView` y `useAccountUpdate` encapsulan lógica de llamadas API, errores, loading y resets.
- **SystemHeader + LoadingSpinner**: Layout replicable para nuevas pantallas (transacción, títulos, minor details).
- **apiClient centralizada**: Admite log de respuestas, token automática y retry default sin reimplementar fetch.
- **Keyboard shortcuts y test accounts**: Proveen control rápido para QA sin crear nuevas vistas.

---

## 🧪 Pruebas y QA

- **Mock handlers (MSW)** (si se utiliza): `useAccountView` y `useAccountUpdate` detectan si la respuesta tiene `success`/`data` o atributos `currentDate` para adaptarse a mocks y backend real.
- **Datos de prueba**: extiende `testAccounts` en `AccountViewScreen` para cubrir distintos estados (activo, inactivo, alto balance).
- **Alertas + logging**: `console.error` en hooks y `Alert` en pantallas muestran errores; se pueden transformar en toasts si se integra `Snackbar`.

---

## 🚨 Riesgos y consideraciones

- **Sin i18n**: Todo el texto está hardcodeado en inglés; cualquier internacionalización debe hacerse file-by-file (MUI `TextField`).
- **Dependencia de localStorage**: `userRole` y `auth-token` deben estar presentes; el módulo redirige a login si faltan.
- **Validaciones comentadas**: Algunos chequeos en `AccountUpdateScreen` están desactivados (líneas comentadas en `handleAccountIdChange`), revisar antes de cambios de negocio.
- **Performance**: Búsqueda en backend no usa caché; la UI mantiene targets de <500ms y <3 queries por búsqueda (ver docs de DB).

---

## 📚 Referencias rápidas

- **Pantallas**: `/app/components/account/AccountViewScreen.tsx`, `/app/components/account/AccountUpdateScreen.tsx`
- **Hooks**: `/app/hooks/useAccountView.ts`, `/app/hooks/useAccountUpdate.ts`
- **Requests**: `/app/types/account.ts`, `/app/types/accountUpdate.ts`, `/app/services/api.ts`
- **Routing**: `/app/App.tsx` (paths `/accounts/view` y `/accounts/update`)
- **Modals y patrones**: `Dialog` + `FormControlLabel` + `Switch` en `AccountUpdateScreen` y `Collapse` en `AccountViewScreen`

---

**Mantenido por**: Equipo DS3A  
**Precisión del código**: 98% (datos extraídos del código actual)
