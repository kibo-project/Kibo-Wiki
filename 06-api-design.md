# 🔌 Kibo - Documentación Técnica de APIs

## Tabla Resumen de Endpoints

| # | Método | Ruta | Descripción | Contexto | Roles | Auth | Notas |
|---|---------|------|-------------|----------|-------|------|-------|
| 1 | POST | `/api/auth/connect` | Conectar wallet y crear/actualizar usuario | Autenticación | Todos | No | Privy signature required |
| 2 | GET | `/api/auth/profile` | Obtener perfil del usuario autenticado | Autenticación | Todos | Sí | Datos varían por rol |
| 3 | POST | `/api/auth/logout` | Cerrar sesión e invalidar token | Autenticación | Todos | Sí | - |
| 4 | POST | `/api/auth/refresh` | Renovar JWT token antes de expirar | Autenticación | Todos | Sí | - |
| 5 | POST | `/api/quote` | Calcular quote USDT/BOB para QR | Quotes | Usuario, Admin | Sí | Expira en 3 minutos |
| 6 | POST | `/api/quote/refresh` | Actualizar quote expirado | Quotes | Usuario, Admin | Sí | Genera nuevo quote |
| 7 | GET | `/api/quotes/current` | Obtener quotes actuales del mercado | Quotes | Todos | No | Datos públicos |
| 8 | GET | `/api/quotes/history` | Histórico de quotes | Quotes | Admin | Sí | Paginación, filtros |
| 9 | POST | `/api/orders` | Crear orden después de confirmar quote | Órdenes | Usuario, Admin | Sí | Requiere quote válida |
| 10 | GET | `/api/orders` | Listar órdenes del usuario | Órdenes | Todos | Sí | Filtros por rol |
| 11 | GET | `/api/orders/:id` | Detalles de orden específica | Órdenes | Todos | Sí | Permisos por propietario |
| 12 | GET | `/api/orders/available` | Órdenes disponibles para tomar | Órdenes | Aliado, Admin | Sí | Solo status=AVAILABLE |
| 13 | POST | `/api/orders/:id/take` | Tomar orden como aliado | Órdenes | Aliado, Admin | Sí | Concurrencia controlada |
| 14 | POST | `/api/orders/:id/proof` | Subir comprobante de pago | Órdenes | Aliado, Admin | Sí | Multipart form, auto-aprobación |
| 15 | POST | `/api/upload/qr` | Subir imagen QR como fallback | Upload | Usuario, Admin | Sí | Max 5MB, OCR parsing |
| 16 | POST | `/api/upload/proof` | Subir comprobante independiente | Upload | Aliado, Admin | Sí | Max 5MB, validación |
| 17 | DELETE | `/api/upload/:filename` | Eliminar archivo subido | Upload | Propietario, Admin | Sí | Solo propios archivos |
| 18 | GET | `/api/admin/dashboard` | Métricas y estado del sistema | Admin | Admin | Sí | Tiempo real, auto-refresh |
| 19 | GET | `/api/admin/orders` | Todas las órdenes con filtros avanzados | Admin | Admin | Sí | Paginación, múltiples filtros |
| 20 | GET | `/api/admin/users` | Gestión de usuarios y aliados | Admin | Admin | Sí | Estadísticas, penalizaciones |
| 21 | GET | `/api/admin/config` | Obtener configuración del sistema | Admin | Admin | Sí | Parámetros dinámicos |
| 22 | PUT | `/api/admin/config` | Actualizar configuración del sistema | Admin | Admin | Sí | Validaciones, historial |
| 23 | GET | `/api/admin/penalties` | Ver penalizaciones de aliados | Admin | Admin | Sí | Filtros, estadísticas |
| 24 | POST | `/api/admin/penalties` | Aplicar penalización manual | Admin | Admin | Sí | Override automático |
| 25 | POST | `/api/cron/check-timeouts` | Procesar órdenes expiradas | Sistema | Sistema | Cron | Vercel Cron Job |
| 26 | POST | `/api/cron/update-quotes` | Actualizar quotes del mercado | Sistema | Sistema | Cron | binance_api/coingecko API |
| 27 | POST | `/api/cron/cleanup` | Limpieza de datos antiguos | Sistema | Sistema | Cron | Job diario |
| 28 | POST | `/api/cron/process-refunds` | Procesar refunds pendientes | Sistema | Sistema | Cron | Queue processing |

---

## 🔐 Autenticación

### **#1 - POST /api/auth/connect**

**Descripción**: Conecta wallet del usuario via Privy y crea/actualiza perfil en base de datos.

**Parámetros de Entrada**:
- `walletAddress` (body, string, requerido): Dirección de wallet (0x...)
- `signature` (body, string, requerido): Firma generada por Privy para verificar propiedad
- `preferredRole` (body, string, opcional): "user" o "ally" para nuevos usuarios
- `country` (body, string, opcional): País del usuario, default "BO"

**Ejemplo Request**:
```json
{
  "walletAddress": "0x742d35Cc6634C0532925a3b8D4014DfF0c1e1c87",
  "signature": "0x...",
  "preferredRole": "ally",
  "country": "BO"
}
```

**Response Exitosa (200)**:
```json
{
  "success": true,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "walletAddress": "0x742d35Cc6634C0532925a3b8D4014DfF0c1e1c87",
    "role": "ally",
    "country": "BO",
    "verified": false,
    "successfulOrders": 0,
    "createdAt": "2025-07-29T10:30:00Z"
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 604800
}
```

**Autenticación**: No requerida (endpoint de login)

**Errores Posibles**:
- `400 INVALID_SIGNATURE`: Firma Privy inválida
- `400 WALLET_ALREADY_EXISTS`: Wallet ya registrada con rol diferente
- `400 INVALID_COUNTRY`: País no soportado
- `500 DATABASE_ERROR`: Error al crear usuario

**Lógica de Negocio**:
- Si wallet ya existe, actualiza `lastActive` y retorna datos existentes
- Si es nueva, crea usuario con `preferredRole` o default "user"
- Solo Bolivia soportada inicialmente en `country`
- Token JWT válido por 7 días

---

### **#2 - GET /api/auth/profile**

**Descripción**: Obtiene perfil completo del usuario autenticado con datos específicos según su rol.

**Parámetros de Entrada**: Ninguno (datos vienen del JWT token)

**Response Exitosa (200)**:
```json
{
  "success": true,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "walletAddress": "0x742d35Cc6634C0532925a3b8D4014DfF0c1e1c87",
    "role": "ally",
    "country": "BO",
    "verified": false,
    "successfulOrders": 15,
    "reputation": 95,
    "lastActive": "2025-07-29T10:30:00Z",
    "createdAt": "2025-07-15T08:15:00Z",
    "allyStats": {
      "totalEarned": 450.75,
      "avgCompletionTime": 8.5,
      "activePenalties": 0
    }
  }
}
```

**Autenticación**: JWT token requerido

**Diferencias por Rol**:
- **Usuario**: `allyStats` no incluido
- **Aliado**: Incluye `allyStats` con ganancias y performance
- **Admin**: Incluye métricas adicionales del sistema

**Errores Posibles**:
- `401 UNAUTHORIZED`: Token inválido o expirado
- `404 USER_NOT_FOUND`: Usuario eliminado de BD

---

### **#3 - POST /api/auth/logout**

**Descripción**: Cierra sesión del usuario e invalida el JWT token.

**Parámetros de Entrada**: Ninguno

**Response Exitosa (200)**:
```json
{
  "success": true,
  "message": "Session closed successfully"
}
```

**Autenticación**: JWT token requerido

**Lógica de Negocio**:
- Actualiza `lastActive` del usuario
- Invalida token agregándolo a blacklist (Redis futuro)
- Frontend debe eliminar token del storage

---

### **#4 - POST /api/auth/refresh**

**Descripción**: Renueva JWT token antes de que expire para mantener sesión activa.

**Parámetros de Entrada**: Ninguno (token actual en header)

**Response Exitosa (200)**:
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 604800
}
```

**Autenticación**: JWT token requerido (puede estar próximo a expirar)

**Errores Posibles**:
- `401 TOKEN_EXPIRED`: Token ya expiró completamente
- `401 INVALID_TOKEN`: Token malformado

---

## 💱 Quotes

### **#5 - POST /api/quote**

**Descripción**: Calcula quote USDT/BOB para un QR bancario específico con monto en bolivianos.

**Parámetros de Entrada**:
- `qrData` (body, string, requerido): Datos extraídos del QR bancario
- `amountFiat` (body, number, requerido): Monto en BOB (10-10000)
- `fiatCurrency` (body, string, opcional): Default "BOB"
- `cryptoToken` (body, string, opcional): Default "USDT"
- `network` (body, string, opcional): Default "mantle"

**Ejemplo Request**:
```json
{
  "qrData": "BANCO_UNION|123456789|REF_12345",
  "amountFiat": 300,
  "fiatCurrency": "BOB",
  "cryptoToken": "USDT"
}
```

**Response Exitosa (200)**:
```json
{
  "success": true,
  "quote": {
    "id": "quote_550e8400-e29b-41d4",
    "amountFiat": 300.00,
    "amountCrypto": 43.06,
    "rate": 6.97,
    "networkFee": 0.10,
    "kiboFee": 0.00,
    "totalAmount": 43.16,
    "expiresAt": "2025-07-29T10:33:00Z",
    "escrowAddress": "0x1234...abcd",
    "rateSource": "binance_api/coingecko",
    "rateTimestamp": "2025-07-29T10:30:00Z"
  }
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Usuario, Admin

**Validaciones**:
- `amountFiat` entre 10 y 10,000 BOB
- QR debe ser formato bancario boliviano válido
- Quote debe ser menor a 30 segundos de antigüedad

**Errores Posibles**:
- `400 INVALID_QR`: QR no es formato bancario reconocido
- `400 AMOUNT_OUT_OF_RANGE`: Monto fuera de límites permitidos
- `503 QUOTE_UNAVAILABLE`: No hay quote disponible

**Lógica de Negocio**:
- Quote válido por exactamente 3 minutos
- Incluye fee de red mantle estimado
- MVP: Kibo fee = 0
- Crea pre-orden en estado "quote" que expira

---

### **#6 - POST /api/quote/refresh**

**Descripción**: Actualiza un quote expirado generando nuevos cálculos con precio actual.

**Parámetros de Entrada**:
- `quoteId` (body, string, requerido): ID del quote a renovar

**Ejemplo Request**:
```json
{
  "quoteId": "quote_550e8400-e29b-41d4"
}
```

**Response Exitosa (200)**:
```json
{
  "success": true,
  "quote": {
    "id": "quote_660f9511-f39c-42e5",
    "amountFiat": 300.00,
    "amountCrypto": 43.28,
    "rate": 6.93,
    "networkFee": 0.10,
    "totalAmount": 43.38,
    "expiresAt": "2025-07-29T10:36:00Z",
    "escrowAddress": "0x1234...abcd",
    "priceChange": 0.57
  }
}
```

**Autenticación**: JWT token requerido

**Validaciones**:
- Quote original debe existir y estar expirada
- Solo el usuario que creó el quote puede refreshearlo
- QR original debe seguir siendo válido

**Lógica de Negocio**:
- Genera nuevo ID de quote
- Recalcula con precio de mercado actual
- Muestra cambio porcentual vs quote anterior

---

### **#7 - GET /api/quotes/current**

**Descripción**: Obtiene las cotizaciones actuales de mercado para todos los pares soportados.

**Parámetros de Entrada**: Ninguno

**Response Exitosa (200)**:
```json
{
  "success": true,
  "quotes": {
    "USDT/BOB": {
      "rate": 6.97,
      "timestamp": "2025-07-29T10:30:00Z",
      "source": "binance_api/coingecko",
      "isActive": true,
      "change24h": 0.15
    }
  },
  "lastUpdated": "2025-07-29T10:30:00Z",
  "nextUpdate": "2025-07-29T10:30:30Z"
}
```

**Autenticación**: No requerida (datos públicos)

**Roles con Acceso**: Todos (incluso sin login)

**Lógica de Negocio**:
- Datos actualizados cada 30 segundos por cron job
- Incluye cambio porcentual últimas 24 horas
- Fuente de datos y timestamp visible

---

### **#8 - GET /api/quotes/history**

**Descripción**: Histórico de cotizaciones para análisis y gráficos (solo admin).

**Parámetros de Entrada**:
- `token` (query, string, opcional): Default "USDT"
- `fiat` (query, string, opcional): Default "BOB"
- `from` (query, string, opcional): Fecha inicio ISO
- `to` (query, string, opcional): Fecha fin ISO
- `interval` (query, string, opcional): "1h", "1d", default "1h"
- `limit` (query, number, opcional): Default 100, max 1000

**Ejemplo Request**: `GET /api/quotes/history?token=USDT&fiat=BOB&from=2025-07-28T00:00:00Z&interval=1h&limit=24`

**Response Exitosa (200)**:
```json
{
  "success": true,
  "data": [
    {
      "timestamp": "2025-07-29T09:00:00Z",
      "rate": 6.95,
      "source": "binance_api/coingecko"
    },
    {
      "timestamp": "2025-07-29T10:00:00Z",
      "rate": 6.97,
      "source": "binance_api/coingecko"
    }
  ],
  "pagination": {
    "total": 24,
    "from": "2025-07-28T00:00:00Z",
    "to": "2025-07-29T10:00:00Z"
  }
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Admin

---

## 📋 Gestión de Órdenes

### **#9 - POST /api/orders**

**Descripción**: Crea orden definitiva después de que usuario confirme quote y esté listo para pagar.

**Parámetros de Entrada**:
- `quoteId` (body, string, requerido): ID de quote válido
- `qrImageUrl` (body, string, opcional): URL de imagen QR para mostrar al aliado

**Ejemplo Request**:
```json
{
  "quoteId": "quote_550e8400-e29b-41d4",
  "qrImageUrl": "https://storage.supabase.co/qrs/qr_123.jpg"
}
```

**Response Exitosa (201)**:
```json
{
  "success": true,
  "order": {
    "id": "order_770f8611-f49d-52f6",
    "status": "PENDING_PAYMENT",
    "amountFiat": 300.00,
    "amountCrypto": 43.06,
    "fiatCurrency": "BOB",
    "cryptoToken": "USDT",
    "network": "mantle",
    "escrowAddress": "0x1234...abcd",
    "expiresAt": "2025-07-29T10:33:00Z",
    "createdAt": "2025-07-29T10:30:00Z",
    "qrImageUrl": "https://storage.supabase.co/qrs/qr_123.jpg"
  }
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Usuario, Admin

**Validaciones**:
- Quote debe existir y no estar expirada
- Usuario debe ser propietario del quote
- Usuario no puede tener más de 3 órdenes activas simultáneamente

**Errores Posibles**:
- `400 QUOTE_EXPIRED`: Quote ya expiró
- `400 QUOTE_NOT_FOUND`: Quote ID no existe
- `429 TOO_MANY_ACTIVE_ORDERS`: Usuario con demasiadas órdenes activas

**Lógica de Negocio**:
- Crea orden en estado PENDING_PAYMENT
- Expira en 3 minutos (mismo tiempo que quote original)
- Crea registro de escrow asociado
- Log automático de creación

---

### **#10 - GET /api/orders**

**Descripción**: Lista órdenes del usuario autenticado con filtros y paginación.

**Parámetros de Entrada**:
- `status` (query, string, opcional): Filtro por estado, múltiples separados por coma
- `limit` (query, number, opcional): Default 20, max 100
- `offset` (query, number, opcional): Default 0
- `sortBy` (query, string, opcional): "createdAt", "completedAt", default "createdAt"
- `order` (query, string, opcional): "asc", "desc", default "desc"
- `from` (query, string, opcional): Fecha inicio ISO
- `to` (query, string, opcional): Fecha fin ISO

**Ejemplo Request**: `GET /api/orders?status=AVAILABLE,TAKEN&limit=10&sortBy=createdAt&order=desc`

**Response Exitosa (200)**:
```json
{
  "success": true,
  "orders": [
    {
      "id": "order_770f8611-f49d-52f6",
      "status": "TAKEN",
      "amountFiat": 300.00,
      "amountCrypto": 43.06,
      "fiatCurrency": "BOB",
      "cryptoToken": "USDT",
      "createdAt": "2025-07-29T10:30:00Z",
      "takenAt": "2025-07-29T10:32:00Z",
      "expiresAt": "2025-07-29T10:37:00Z",
      "secondsRemaining": 180,
      "ally": {
        "id": "ally_123",
        "walletAddress": "0x5678...efgh"
      }
    }
  ],
  "pagination": {
    "total": 45,
    "limit": 10,
    "offset": 0,
    "hasMore": true
  }
}
```

**Autenticación**: JWT token requerido

**Diferencias por Rol**:
- **Usuario**: Solo sus propias órdenes como creator
- **Aliado**: Sus órdenes como creator + órdenes que procesó como ally
- **Admin**: Todas las órdenes del sistema con filtros avanzados

**Lógica de Negocio**:
- `secondsRemaining` calculado en tiempo real
- Incluye datos de aliado si la orden fue tomada
- Paginación para performance con muchas órdenes

---

### **#11 - GET /api/orders/:id**

**Descripción**: Obtiene detalles completos de una orden específica.

**Parámetros de Entrada**:
- `id` (path, string, requerido): UUID de la orden

**Ejemplo Request**: `GET /api/orders/order_770f8611-f49d-52f6`

**Response Exitosa (200)**:
```json
{
  "success": true,
  "order": {
    "id": "order_770f8611-f49d-52f6",
    "status": "COMPLETED",
    "amountFiat": 300.00,
    "amountCrypto": 43.06,
    "fiatCurrency": "BOB",
    "cryptoToken": "USDT",
    "network": "mantle",
    "qrData": "BANCO_UNION|123456789|REF_12345",
    "qrImageUrl": "https://storage.supabase.co/qrs/qr_123.jpg",
    "proofUrl": "https://storage.supabase.co/proofs/proof_456.jpg",
    "createdAt": "2025-07-29T10:30:00Z",
    "takenAt": "2025-07-29T10:32:00Z",
    "completedAt": "2025-07-29T10:35:00Z",
    "escrowTxHash": "0xabc123...",
    "releaseTxHash": "0xdef456...",
    "user": {
      "id": "user_123",
      "walletAddress": "0x1234...abcd"
    },
    "ally": {
      "id": "ally_456",
      "walletAddress": "0x5678...efgh",
      "successfulOrders": 15
    },
    "timeline": [
      {
        "status": "PENDING_PAYMENT",
        "timestamp": "2025-07-29T10:30:00Z"
      },
      {
        "status": "AVAILABLE",
        "timestamp": "2025-07-29T10:31:00Z"
      },
      {
        "status": "TAKEN",
        "timestamp": "2025-07-29T10:32:00Z"
      },
      {
        "status": "COMPLETED",
        "timestamp": "2025-07-29T10:35:00Z"
      }
    ]
  }
}
```

**Autenticación**: JWT token requerido

**Permisos**:
- **Usuario**: Solo órdenes donde es creator o ally
- **Aliado**: Solo órdenes donde es creator o ally
- **Admin**: Cualquier orden del sistema

**Errores Posibles**:
- `404 ORDER_NOT_FOUND`: Orden no existe
- `403 FORBIDDEN`: Sin permisos para ver esta orden

**Lógica de Negocio**:
- Timeline muestra historial completo de cambios de estado
- Incluye hashes de transacciones blockchain
- URLs de imágenes (QR y comprobante) si están disponibles

---

### **#12 - GET /api/orders/available**

**Descripción**: Lista órdenes disponibles para que aliados puedan tomar (solo status=AVAILABLE).

**Parámetros de Entrada**:
- `country` (query, string, opcional): Filtrar por país, default basado en aliado
- `minAmount` (query, number, opcional): Monto mínimo en fiat
- `maxAmount` (query, number, opcional): Monto máximo en fiat
- `sortBy` (query, string, opcional): "createdAt", "expiresAt", "amount", default "expiresAt"
- `limit` (query, number, opcional): Default 50, max 200

**Ejemplo Request**: `GET /api/orders/available?country=BO&minAmount=100&sortBy=expiresAt`

**Response Exitosa (200)**:
```json
{
  "success": true,
  "orders": [
    {
      "id": "order_880g9722-g50e-63g7",
      "amountFiat": 300.00,
      "amountCrypto": 43.06,
      "fiatCurrency": "BOB",
      "cryptoToken": "USDT",
      "createdAt": "2025-07-29T10:32:00Z",
      "expiresAt": "2025-07-29T10:37:00Z",
      "secondsRemaining": 180,
      "userCountry": "BO",
      "estimatedGain": 43.06,
      "bankingInfo": {
        "bank": "Banco Unión",
        "reference": "KBO025"
      }
    }
  ],
  "metadata": {
    "totalAvailable": 12,
    "avgWaitTime": 245,
    "yourActiveOrders": 0
  }
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Aliado, Admin

**Validaciones**:
- Solo aliados no penalizados pueden acceder
- Solo órdenes del mismo país del aliado
- Excluye órdenes que el aliado dejó expirar antes

**Lógica de Negocio**:
- Auto-refresh cada 10 segundos en frontend
- `secondsRemaining` calculado en tiempo real
- `estimatedGain` = monto USDT que recibirá el aliado
- Metadata incluye estadísticas útiles

**Filtros Automáticos**:
- Solo status = "AVAILABLE"
- Solo expiresAt > NOW()
- Solo country matching aliado
- Excluir si aliado tiene penalizaciones activas

---

### **#13 - POST /api/orders/:id/take**

**Descripción**: Aliado toma orden disponible de forma exclusiva con control de concurrencia.

**Parámetros de Entrada**:
- `id` (path, string, requerido): UUID de la orden a tomar

**Ejemplo Request**: `POST /api/orders/order_880g9722-g50e-63g7/take`

**Response Exitosa (200)**:
```json
{
  "success": true,
  "order": {
    "id": "order_880g9722-g50e-63g7",
    "status": "TAKEN",
    "amountFiat": 300.00,
    "amountCrypto": 43.06,
    "qrData": "BANCO_UNION|123456789|REF_12345",
    "qrImageUrl": "https://storage.supabase.co/qrs/qr_567.jpg",
    "takenAt": "2025-07-29T10:35:00Z",
    "expiresAt": "2025-07-29T10:40:00Z",
    "bankingDetails": {
      "bank": "Banco Unión",
      "accountNumber": "123456789",
      "beneficiary": "Juan Pérez",
      "reference": "KBO025",
      "exactAmount": "300.00 BOB"
    }
  }
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Aliado, Admin

**Validaciones Críticas**:
- Orden debe estar en status="AVAILABLE"
- Orden no debe estar expirada
- Aliado no debe tener penalizaciones activas
- Aliado no debe tener otra orden activa
- Aliado no debe haber dejado expirar esta orden antes

**Errores Posibles**:
- `400 ORDER_ALREADY_TAKEN`: Otro aliado la tomó simultáneamente
- `400 ORDER_EXPIRED`: Orden expiró mientras aliado decidía
- `403 ALLY_PENALIZED`: Aliado tiene penalizaciones activas
- `403 ALLY_HAS_ACTIVE_ORDER`: Aliado ya tiene orden en proceso
- `403 ALLY_PREVIOUSLY_EXPIRED`: Aliado dejó expirar esta orden antes

**Lógica de Negocio**:
- Transacción atómica con SELECT FOR UPDATE
- Actualiza expires_at a NOW() + 5 minutos
- Crea log de ORDER_TAKEN
- QR y banking details solo visibles después de tomar

---

### **#14 - POST /api/orders/:id/proof**

**Descripción**: Aliado sube comprobante de pago bancario, triggering auto-aprobación MVP.

**Parámetros de Entrada**:
- `id` (path, string, requerido): UUID de la orden
- `proof` (body, File, requerido): Imagen del comprobante (multipart/form-data)
- `bankTransactionId` (body, string, opcional): ID de transacción del banco
- `notes` (body, string, opcional): Notas adicionales del aliado

**Ejemplo Request**:
```bash
curl -X POST \
  -H "Authorization: Bearer <token>" \
  -F "proof=@comprobante.jpg" \
  -F "bankTransactionId=TXN123456" \
  -F "notes=Pago realizado via app móvil" \
  /api/orders/order_880g9722-g50e-63g7/proof
```

**Response Exitosa (200)**:
```json
{
  "success": true,
  "order": {
    "id": "order_880g9722-g50e-63g7",
    "status": "COMPLETED",
    "proofUrl": "https://storage.supabase.co/proofs/proof_789.jpg",
    "completedAt": "2025-07-29T10:38:00Z",
    "releaseTransactionHash": "0x789abc...",
    "bankTransactionId": "TXN123456"
  },
  "payment": {
    "amountReleased": 43.06,
    "recipientWallet": "0x5678...efgh",
    "networkFee": 0.008
  },
  "message": "Payment proof uploaded and approved. 43.06 USDT released to your wallet."
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Aliado que tomó la orden, Admin

**Validaciones**:
- Orden debe estar en status="TAKEN"
- Aliado debe ser quien tomó la orden
- Archivo máximo 5MB
- Formatos permitidos: JPG, PNG, HEIC
- Orden no debe estar expirada

**Errores Posibles**:
- `400 FILE_TOO_LARGE`: Archivo > 5MB
- `400 INVALID_FILE_TYPE`: Formato no permitido
- `403 ORDER_NOT_TAKEN_BY_USER`: Aliado no tomó esta orden
- `400 ORDER_EXPIRED`: Orden expiró antes de subir comprobante
- `409 PROOF_ALREADY_UPLOADED`: Ya se subió comprobante

**Lógica de Negocio MVP**:
- **Auto-aprobación**: Comprobante subido = fondos liberados inmediatamente
- Actualiza orden a status="COMPLETED"
- Transfiere USDT del escrow al wallet del aliado
- Incrementa `successfulOrders` del aliado
- Envía notificaciones push a usuario y aliado
- Logs completos para auditoría futura

---

## 📤 Gestión de Archivos

### **#15 - POST /api/upload/qr**

**Descripción**: Fallback para subir imagen de QR cuando cámara nativa no funciona.

**Parámetros de Entrada**:
- `qr` (body, File, requerido): Imagen conteniendo QR bancario

**Response Exitosa (200)**:
```json
{
  "success": true,
  "qrData": "BANCO_UNION|123456789|REF_12345",
  "imageUrl": "https://storage.supabase.co/qrs/qr_890.jpg",
  "bankingInfo": {
    "bank": "Banco Unión",
    "account": "123456789",
    "detectedAmount": null,
    "reference": "REF_12345"
  },
  "confidence": 0.95
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Usuario, Admin

**Validaciones**:
- Imagen máximo 5MB
- Formatos: JPG, PNG
- Debe contener QR code legible

**Errores Posibles**:
- `400 QR_PARSE_FAILED`: No se pudo extraer QR de la imagen
- `400 INVALID_QR_FORMAT`: QR no es formato bancario boliviano

**Lógica de Negocio**:
- OCR/parsing para extraer datos del QR
- Validación de formato bancario boliviano
- Confidence score del parsing
- Imagen almacenada para mostrar al aliado

---

### **#16 - POST /api/upload/proof**

**Descripción**: Subir comprobante como endpoint independiente (alternativo a #14).

**Parámetros de Entrada**:
- `proof` (body, File, requerido): Imagen del comprobante
- `orderId` (body, string, requerido): UUID de la orden asociada

**Response Exitosa (200)**:
```json
{
  "success": true,
  "fileUrl": "https://storage.supabase.co/proofs/proof_901.jpg",
  "fileName": "proof_901.jpg",
  "fileSize": 2048576,
  "uploadedAt": "2025-07-29T10:40:00Z"
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Aliado, Admin

**Validaciones**:
- Mismas que endpoint #14
- Orden debe existir y estar en status="TAKEN"

**Lógica de Negocio**:
- Solo sube archivo, NO actualiza estado de orden
- Útil para pre-upload antes de confirmar
- Requiere llamada separada para marcar orden como completada

---

### **#17 - DELETE /api/upload/:filename**

**Descripción**: Elimina archivo subido (solo propietario o admin).

**Parámetros de Entrada**:
- `filename` (path, string, requerido): Nombre del archivo a eliminar

**Response Exitosa (200)**:
```json
{
  "success": true,
  "message": "File deleted successfully",
  "deletedFile": "proof_901.jpg"
}
```

**Autenticación**: JWT token requerido

**Permisos**:
- Usuario puede eliminar solo archivos que subió
- Admin puede eliminar cualquier archivo

**Errores Posibles**:
- `404 FILE_NOT_FOUND`: Archivo no existe
- `403 FORBIDDEN`: Sin permisos para eliminar
- `409 FILE_IN_USE`: Archivo está siendo usado en orden activa

---

## 👨‍💼 Administración

### **#18 - GET /api/admin/dashboard**

**Descripción**: Dashboard completo con métricas en tiempo real del sistema.

**Parámetros de Entrada**:
- `timeframe` (query, string, opcional): "today", "week", "month", default "today"
- `refresh` (query, boolean, opcional): Forzar recálculo de métricas

**Response Exitosa (200)**:
```json
{
  "success": true,
  "metrics": {
    "timeframe": "today",
    "orders": {
      "total": 45,
      "completed": 38,
      "expired": 7,
      "active": 3,
      "successRate": 84.4
    },
    "volume": {
      "fiat": 12500.00,
      "crypto": 1793.25,
      "avgOrderSize": 277.78
    },
    "performance": {
      "avgCompletionTime": 9.2,
      "fastestCompletion": 4.5,
      "slowestCompletion": 15.8
    },
    "allies": {
      "total": 18,
      "active": 12,
      "penalized": 2,
      "topPerformer": {
        "walletAddress": "0x1234...abcd",
        "completedToday": 8,
        "avgTime": 6.8
      }
    },
    "system": {
      "uptime": 99.8,
      "apiStatus": {
        "binance_api/coingecko": "operational",
        "mantle": "operational",
        "supabase": "operational"
      },
      "escrowBalance": 2450.75,
      "pendingRefunds": 0
    }
  },
  "recentActivity": [
    {
      "type": "order_completed",
      "orderId": "order_123",
      "timestamp": "2025-07-29T10:35:00Z",
      "user": "0x1234...abcd",
      "ally": "0x5678...efgh",
      "amount": 300.00
    }
  ],
  "alerts": [
    {
      "level": "warning",
      "message": "Ally 0x9abc...def has 2 timeouts in last 2 hours",
      "timestamp": "2025-07-29T10:30:00Z"
    }
  ]
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Admin

**Actualización**: Datos actualizados cada 30 segundos

---

### **#19 - GET /api/admin/orders**

**Descripción**: Lista completa de órdenes con filtros avanzados para administración.

**Parámetros de Entrada**:
- `status` (query, string, opcional): Filtro por estado, múltiples separados por coma
- `user` (query, string, opcional): Filtrar por wallet address del usuario
- `ally` (query, string, opcional): Filtrar por wallet address del aliado
- `from` (query, string, opcional): Fecha inicio ISO
- `to` (query, string, opcional): Fecha fin ISO
- `minAmount` (query, number, opcional): Monto mínimo fiat
- `maxAmount` (query, number, opcional): Monto máximo fiat
- `sortBy` (query, string, opcional): Campo ordenamiento
- `order` (query, string, opcional): "asc" o "desc"
- `limit` (query, number, opcional): Default 50, max 500
- `offset` (query, number, opcional): Default 0

**Ejemplo Request**: `GET /api/admin/orders?status=EXPIRED&from=2025-07-28T00:00:00Z&sortBy=createdAt&limit=20`

**Response Exitosa (200)**:
```json
{
  "success": true,
  "orders": [
    {
      "id": "order_abc123",
      "status": "EXPIRED",
      "amountFiat": 200.00,
      "amountCrypto": 28.70,
      "createdAt": "2025-07-29T09:15:00Z",
      "expiredAt": "2025-07-29T09:20:00Z",
      "user": {
        "id": "user_123",
        "walletAddress": "0x1234...abcd",
        "successfulOrders": 5
      },
      "ally": null,
      "refundStatus": "completed",
      "refundTxHash": "0xdef789...",
      "logs": [
        {
          "action": "ORDER_CREATED",
          "timestamp": "2025-07-29T09:15:00Z"
        },
        {
          "action": "ORDER_EXPIRED",
          "timestamp": "2025-07-29T09:20:00Z"
        }
      ]
    }
  ],
  "pagination": {
    "total": 156,
    "limit": 20,
    "offset": 0,
    "hasMore": true
  },
  "summary": {
    "totalVolume": 45230.00,
    "avgCompletionTime": 8.5,
    "successRate": 87.3
  }
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Admin

**Lógica de Negocio**:
- Incluye logs de cambios de estado
- Datos de refund si aplicable
- Summary estadístico del filtro aplicado

---

### **#20 - GET /api/admin/users**

**Descripción**: Gestión de usuarios con estadísticas detalladas y control de penalizaciones.

**Parámetros de Entrada**:
- `role` (query, string, opcional): "user", "ally", "admin"
- `country` (query, string, opcional): Filtrar por país
- `verified` (query, boolean, opcional): Solo usuarios verificados
- `minOrders` (query, number, opcional): Mínimo órdenes completadas
- `penalized` (query, boolean, opcional): Solo usuarios con penalizaciones activas
- `sortBy` (query, string, opcional): "createdAt", "successfulOrders", "lastActive"
- `limit` (query, number, opcional): Default 50, max 200

**Ejemplo Request**: `GET /api/admin/users?role=ally&minOrders=10&sortBy=successfulOrders`

**Response Exitosa (200)**:
```json
{
  "success": true,
  "users": [
    {
      "id": "user_456",
      "walletAddress": "0x5678...efgh",
      "role": "ally",
      "country": "BO",
      "verified": true,
      "successfulOrders": 42,
      "reputation": 98,
      "createdAt": "2025-07-15T08:00:00Z",
      "lastActive": "2025-07-29T10:30:00Z",
      "stats": {
        "totalEarned": 1250.75,
        "avgCompletionTime": 7.2,
        "timeouts": 2,
        "successRate": 95.5
      },
      "penalties": {
        "active": 0,
        "total": 3,
        "lastPenalty": "2025-07-20T15:30:00Z"
      }
    }
  ],
  "pagination": {
    "total": 18,
    "limit": 50,
    "offset": 0
  }
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Admin

---

### **#21 - GET /api/admin/config**

**Descripción**: Obtiene configuración actual del sistema.

**Response Exitosa (200)**:
```json
{
  "success": true,
  "config": {
    "timeouts": {
      "PENDING_PAYMENT_TIMEOUT_MINUTES": 3,
      "AVAILABLE_TIMEOUT_MINUTES": 5,
      "TAKEN_TIMEOUT_MINUTES": 5,
      "ALLY_PENALTY_MINUTES": 30
    },
    "limits": {
      "MIN_ORDER_AMOUNT_BOB": 10,
      "MAX_ORDER_AMOUNT_BOB": 10000,
      "MAX_DAILY_ORDERS_PER_USER": null,
      "MAX_FILE_SIZE_MB": 5
    },
    "quotes": {
      "QUOTE_UPDATE_INTERVAL_SECONDS": 30,
      "PRICE_CHANGE_ALERT_THRESHOLD": 5.0,
      "PRIMARY_QUOTE_SOURCE": "binance_api/coingecko"
    },
    "system": {
      "ESCROW_WALLET_ADDRESS": "0x1234...abcd",
      "SUPPORTED_COUNTRIES": ["BO"],
      "SUPPORTED_TOKENS": ["USDT"],
      "SUPPORTED_NETWORKS": ["mantle"]
    }
  },
  "lastUpdated": "2025-07-29T08:00:00Z"
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Admin

---

### **#22 - PUT /api/admin/config**

**Descripción**: Actualiza configuración del sistema dinámicamente.

**Parámetros de Entrada**:
```json
{
  "config": {
    "PENDING_PAYMENT_TIMEOUT_MINUTES": 5,
    "ALLY_PENALTY_MINUTES": 60,
    "MAX_ORDER_AMOUNT_BOB": 15000
  },
  "reason": "Aumentar timeouts basado en feedback usuarios"
}
```

**Response Exitosa (200)**:
```json
{
  "success": true,
  "updated": [
    "PENDING_PAYMENT_TIMEOUT_MINUTES",
    "ALLY_PENALTY_MINUTES", 
    "MAX_ORDER_AMOUNT_BOB"
  ],
  "changes": {
    "PENDING_PAYMENT_TIMEOUT_MINUTES": {
      "old": 3,
      "new": 5
    },
    "ALLY_PENALTY_MINUTES": {
      "old": 30,
      "new": 60
    }
  },
  "message": "Configuration updated successfully"
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Admin

**Validaciones**:
- Timeouts entre 1-60 minutos
- Límites coherentes (min < max)
- Valores numéricos positivos

**Lógica de Negocio**:
- Cambios aplican inmediatamente
- Log completo de cambios con timestamp y admin
- Validaciones para evitar configuraciones problemáticas

---

### **#23 - GET /api/admin/penalties**

**Descripción**: Ver todas las penalizaciones de aliados con filtros.

**Parámetros de Entrada**:
- `ally` (query, string, opcional): Wallet address del aliado
- `reason` (query, string, opcional): "TIMEOUT", "FAKE_PROOF", "NO_PAYMENT"
- `active` (query, boolean, opcional): Solo penalizaciones activas
- `from` (query, string, opcional): Fecha inicio
- `limit` (query, number, opcional): Default 50

**Response Exitosa (200)**:
```json
{
  "success": true,
  "penalties": [
    {
      "id": "penalty_123",
      "allyId": "ally_456",
      "allyWallet": "0x5678...efgh",
      "orderId": "order_789",
      "reason": "TIMEOUT",
      "penaltyMinutes": 30,
      "penaltyUntil": "2025-07-29T11:30:00Z",
      "isActive": true,
      "createdAt": "2025-07-29T11:00:00Z"
    }
  ],
  "summary": {
    "totalPenalties": 45,
    "activePenalties": 3,
    "byReason": {
      "TIMEOUT": 38,
      "FAKE_PROOF": 5,
      "NO_PAYMENT": 2
    }
  }
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Admin

---

### **#24 - POST /api/admin/penalties**

**Descripción**: Aplicar penalización manual a aliado.

**Parámetros de Entrada**:
```json
{
  "allyId": "ally_456",
  "reason": "FAKE_PROOF",
  "penaltyMinutes": 120,
  "orderId": "order_789",
  "notes": "Comprobante claramente editado"
}
```

**Response Exitosa (200)**:
```json
{
  "success": true,
  "penalty": {
    "id": "penalty_124",
    "allyId": "ally_456",
    "reason": "FAKE_PROOF",
    "penaltyUntil": "2025-07-29T13:00:00Z",
    "createdAt": "2025-07-29T11:00:00Z"
  },
  "message": "Manual penalty applied successfully"
}
```

**Autenticación**: JWT token requerido

**Roles con Acceso**: Solo Admin

**Validaciones**:
- Aliado debe existir y tener rol="ally"
- Penalty minutes entre 1-1440 (24 horas max)
- Reason debe ser válido

---

## ⏰ Sistema y Cron Jobs

### **#25 - POST /api/cron/check-timeouts**

**Descripción**: Job automático que procesa órdenes expiradas cada 30 segundos.

**Autenticación**: Vercel Cron secret

**Response Exitosa (200)**:
```json
{
  "success": true,
  "processed": {
    "expiredPendingPayment": 2,
    "expiredAvailable": 1,
    "expiredTaken": 3,
    "refundsInitiated": 4,
    "penaltiesApplied": 3
  },
  "executionTime": 1.25,
  "timestamp": "2025-07-29T11:00:30Z"
}
```

**Lógica de Negocio**:
- Ejecuta cada 30 segundos via Vercel Cron
- Procesa todos los tipos de timeout según estado
- Inicia refunds automáticos donde corresponde
- Aplica penalizaciones a aliados

---

### **#26 - POST /api/cron/update-quotes**

**Descripción**: Actualiza cotizaciones desde APIs externas cada 30 segundos.

**Response Exitosa (200)**:
```json
{
  "success": true,
  "updated": {
    "USDT/BOB": {
      "oldRate": 6.95,
      "newRate": 6.97,
      "change": 0.29,
      "source": "binance_api/coingecko"
    }
  },
  "errors": [],
  "timestamp": "2025-07-29T11:00:30Z"
}
```

**Lógica de Negocio**:
- Consulta binance_api/coingecko API para USDT/USD
- Calcula tasa BOB usando fuente local
- Marca quote anterior como inactive
- Alertas si cambio > 5%

---

### **#27 - POST /api/cron/cleanup**

**Descripción**: Limpieza diaria de datos antiguos (3:00 AM).

**Response Exitosa (200)**:
```json
{
  "success": true,
  "cleaned": {
    "oldQuotes": 145,
    "expiredRefundJobs": 3,
    "inactivePenalties": 8,
    "oldLogs": 234
  },
  "retainedDays": 30,
  "timestamp": "2025-07-29T03:00:00Z"
}
```

**Lógica de Negocio**:
- Elimina quotes > 30 días
- Limpia penalizaciones expiradas
- Archiva logs antiguos
- Mantiene órdenes indefinidamente

---

### **#28 - POST /api/cron/process-refunds**

**Descripción**: Procesa cola de refunds pendientes.

**Response Exitosa (200)**:
```json
{
  "success": true,
  "processed": {
    "completed": 5,
    "failed": 0,
    "skipped": 2
  },
  "totalAmount": 215.75,
  "timestamp": "2025-07-29T11:00:30Z"
}
```

**Lógica de Negocio**:
- Procesa refund_jobs con status="PENDING"
- Envía transacciones a través del escrow wallet
- Actualiza estados según resultado
- Reintentos automáticos en caso de fallo temporal

---

## 🔒 Autenticación y Seguridad

### **JWT Token Structure**
```json
{
  "userId": "user_123",
  "walletAddress": "0x1234...abcd",
  "role": "ally",
  "country": "BO",
  "iat": 1690627200,
  "exp": 1691232000
}
```

### **Rate Limiting**

| Categoría | Límite | Ventana | Scope |
|-----------|--------|---------|-------|
| Authentication | 5 req/min | Per IP | Global |
| Quote APIs | 10 req/min | Per user | Individual |
| Order APIs | 20 req/min | Per user | Individual |
| Upload APIs | 5 req/min | Per user | Individual |
| Admin APIs | 100 req/min | Per admin | Individual |
| Cron Jobs | Unlimited | - | System |

### **Headers de Seguridad**
```http
Authorization: Bearer <jwt_token>
X-API-Version: v1
X-Request-ID: <uuid>
Content-Type: application/json
```

---

**🎯 Beneficios de esta Documentación:**

- **Desarrollo paralelo**: Frontend y backend pueden trabajar con contratos claros
- **Testing sistemático**: Cada endpoint tiene casos de éxito y error definidos
- **Seguridad clara**: Permisos y validaciones explícitas
- **Performance**: Rate limiting y paginación para escalabilidad
- **Debugging**: Request/response examples para troubleshooting
- **Mantenimiento**: Documentación viva que evoluciona con el código

**🔑 Próximos pasos**: Implementar estos endpoints en Next.js API routes siguiendo exactamente estas especificaciones, incluyendo validaciones, permisos y manejo de errores.