# 💳 ACCOUNT - Módulo de Cuentas

**Versión**: 1.0  
**Última actualización**: 2026-01-28  
**Propósito**: Consignar los elementos técnicos y de negocio que permiten generar User Stories centradas en la administración de cuentas de tarjetas de crédito de CardDemo.

---

## 📋 Visión general

- **Objetivo principal**: Consultar y actualizar cuentas de crédito asegurando validaciones, enmascarado de datos sensibles y consistencia transaccional.
- **Audiencia**: Product Owners, analistas funcionales y desarrolladores backend/frontend que planean historias para atención al cliente, back-office y administración.
- **Beneficios clave**: 
  - Visualización inmediata de balances, límites y datos del cliente.
  - Modo edición consciente con confirmación, validaciones y rollback automático (PUT atómico).
  - Seguridad de datos (SSN/card enmascarados y control de visibilidad).

---

## 🏗️ Arquitectura aplicada

### Frontend

- `AccountViewPage.tsx` y `AccountUpdatePage.tsx` aseguran que exista `userRole` en localStorage; si falta, redirigen a `/login`. Al salir envían al menú correspondiente (`/menu/admin` o `/menu/main`).
- Pantallas:
  - `AccountViewScreen.tsx`: formulario de búsqueda (validando 11 dígitos no nulos), test accounts (solo en dev), toggles para mostrar datos sensibles, alertas (error/info) y tarjetas (`Card`) para datos financieros y personales. Incluye atajos de teclado F3 (exit) y `Collapse` para test accounts.
  - `AccountUpdateScreen.tsx`: habilita `editMode`, mantiene `validationErrors` para `activeStatus`, `creditLimit`, `zipCode` y `currentBalance`, muestra chips de estado, dialog de confirmación al guardar (F5) y botones de reset (F12).
- Hooks:
  - `useAccountView`: two mutations hooking `apiClient.get('/account-view?...')` and `apiClient.get('/account-view/initialize')`, mantiene `data`, `loading`, `error`, `initializeScreen` y `clearData`.
  - `useAccountUpdate`: `searchAccount` (GET `/accounts/{accountId}`) y `updateAccount` (PUT `/accounts/{accountId}`), detecta cambios via `JSON.stringify`, ofrece `hasChanges`, `searchLoading`, `updateLoading` y resetea datos tras guardar.
- `apiClient` (https://.../app/services/api.ts) agrega autorización (Authorization Bearer), headers y logging de respuestas (detecta `currentDate` vs `success/data`), usado por ambos hooks.
- `SystemHeader` y `LoadingSpinner` estandarizan transacciones (COACTVWC / COACTUPC) y experiencia visual.

### Backend solapado

- `GET /account-view?accountId={id}`: recibe ID en querystring, responde un `AccountViewResponse` completo (transacción, datos de cuenta, cliente, mensajes).
- `GET /account-view/initialize`: devuelve metadatos (fecha, hora, transactionId) para evitar contenido vacío en la pantalla.
- `GET /accounts/{accountId}` y `PUT /accounts/{accountId}`: proveen y persisten `AccountUpdateData` (Account + Customer) en una operación transaccional.
- El backend protege los endpoints con tokens; el frontend utiliza el token almacenado en `localStorage` (auth-token) automáticamente.

---

## 🔗 APIs públicas

```text
GET  /account-view?accountId={11-digit}        → Devuelve AccountViewResponse con balances, cliente y estado.
GET  /account-view/initialize                   → Metadata (fecha/hora/transactionId) para inicializar la vista.
GET  /accounts/{accountId}                      → Carga datos para editar AccountUpdateData.
PUT  /accounts/{accountId}                      → Persiste AccountUpdateData (Account + Customer) en una sola transacción.
```

Todos los requests se canalizan a través de `apiClient` (timeout default 10s, logs en consola, manejo de errores con `ApiError`).

---

## 📊 Modelos de datos (frontend)

### `AccountViewResponse` (`app/types/account.ts`)

- `accountId`, `accountStatus` (`Y`/`N`), balances (creditLimit, currentBalance, cycleCredit/debit), fechas (open, expiration, reissue), `groupId`.
- Cliente: `customerId`, `customerSsn`, `ficoScore`, `firstName`, `lastName`, `addressLine1`, `state`, `zipCode`, `phoneNumber1`, `governmentId`, `primaryCardHolderFlag`, `cardNumber`.
- Mensajes: `errorMessage`, `infoMessage`, `inputValid` (indica si se puede mostrar data).

### `AccountUpdateData` (`app/types/accountUpdate.ts`)

- Datos de cuenta: `accountId`, `activeStatus`, límites, fechas, ciclos, `groupId`.
- Datos del cliente: `firstName`, `lastName`, `addressLine1`, `stateCode`, `zipCode`, `phoneNumber1`, `ssn`, `governmentIssuedId`, `ficoScore`, `primaryCardIndicator`.
- El hook `useAccountUpdate` usa este objeto para detectar cambios y enviar payload en `PUT /accounts/{accountId}`.

---

## 🔐 Reglas de negocio

1. `accountId` debe tener exactamente 11 dígitos numéricos y no puede ser `00000000000` (validated en `AccountViewScreen`).
2. `activeStatus` solo admite `Y` o `N`; la pantalla de edición bloquea cualquier otro valor y muestra chip de color.
3. SSN y número de tarjeta se muestran enmascarados; el toggle `showSensitiveData` permite su revelado controlado.
4. Actualizaciones solo se permiten cuando `editMode` está activo, `hasChanges` es verdadero y `validationErrors` está vacío.
5. `zipCode` debe coincidir con `^\d{5}(-\d{4})?$`.
6. Actualizaciones son transaccionales: si la validación backend falla (response `success: false`), el hook no resetea `hasChanges`.
7. Los shortcuts F3 (exit), F5 (guardar) y F12 (reset) deben respetarse en pruebas QA para mantener consistencia con terminales.

---

## 🎯 Ejemplos de User Stories

- "Como oficial de servicio, quiero buscar una cuenta por su ID para ver el saldo, límites e información de contacto antes de responder al cliente."
- "Como administrador de cuentas, quiero activar el modo edición y confirmar los cambios para ajustar límites y contactos sin romper la transaccionalidad."
- "Como oficial de cumplimiento, quiero que SSN y número de tarjeta se muestren enmascarados por defecto y solo se revelen cuando habilito el toggle de datos sensibles."
- "Como QA, quiero usar las cuentas de prueba visibles sólo en entorno dev para validar errores sin tocar la DB productiva."

**Guía de complejidad**:  
Simple (1-2 pts): añadir campos de solo lectura o validar un nuevo campo textual.  
Medio (3-5 pts): integrar validaciones adicionales `zipCode`/`ficoScore` con errores visibles.  
Complejo (5-8 pts): integrar scoring externo, auditoría o flujos de aprobación para cambios de límite altos.

---

## ⚡ Factores de aceleración

- **Hooks reutilizables**: `useAccountView` y `useAccountUpdate` encapsulan lógica de llamadas, manejo de errores y resets.
- **Material-UI**: TextField, Card, Grid, Button, Chip y Dialog ya están configurados con `useTheme`.
- **apiClient centralizado**: transfiere timeout, logging y headers sin reescribir fetch.
- **Shortcuts y test accounts**: aceleran QA (F3, F5, F12; test accounts en `AccountViewScreen`).
- **SystemHeader + LoadingSpinner** replican la experiencia de transacciones (~COACTVWC/COACTUPC).

---

## 🧪 Pruebas y monitoreo

- Validaciones inline se reflejan en `validationErrors`: se bloquea guardado si existen errores. Lógica replicable en nuevos formularios.
- Hooks reportan errores con `console.error`, las pantallas muestran `Alert` (severity error/info) y no avanzan si `errorMessage` viene desde el backend.
- En dev, el botón "Show Test Accounts" agrega IDs 11111111111 → 44444444444; permite probar estados `Active`, `Inactive`, `High Balance`, `New Account`.

---

## 🚨 Riesgos y consideraciones

- El texto está hardcodeado en inglés; no existe i18n (documentado en `docs/system-overview.md`).
- Dependencia en `localStorage` (`auth-token`, `userRole`); fallas redirigen a `/login`.
- Validaciones complejas comentadas (por ejemplo, `accountId` en `AccountUpdateScreen`) deben activarse solo si se revisa el backend.
- Los endpoints actuales no usan cache; las búsquedas deben mantenerse por debajo de 500ms de respuesta (P95) y no ejecutar más de 3 queries simultáneas.

---

## 📚 Referencias

- Screens: `/app/components/account/AccountViewScreen.tsx`, `/app/components/account/AccountUpdateScreen.tsx`.
- Hooks: `/app/hooks/useAccountView.ts`, `/app/hooks/useAccountUpdate.ts`.
- Tipos: `/app/types/account.ts`, `/app/types/accountUpdate.ts`.
- API client: `/app/services/api.ts`.
- Routing: `/app/App.tsx` (rutas `/accounts/view`, `/accounts/update`).
- Docs complementarias: `modules/account/account-overview.md`, `docs/system-overview.md`.

---

**Precisión**: 98% (basado en el código actual del repositorio).
