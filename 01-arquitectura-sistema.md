# 🏗️ Kibo - Arquitectura del Sistema

## Vista de Alto Nivel

```mermaid
graph TB
    subgraph "Cliente/Frontend"
        WEB[Web App Next.js]
        WALLET[Wallet Integration<br/>Privy]
    end

    subgraph "Backend Services"
        API[Next.js API Routes]
        AUTH[Authentication Service]
        ORDER[Orders Service]
        PRICING[Pricing Service]
        ESCROW[Escrow Service]
        CRON[Serverless Cron Jobs]
    end

    subgraph "Almacenamiento"
        DB[(Supabase PostgreSQL)]
        STORAGE[Supabase Storage]
        REALTIME[Supabase Realtime]
    end

    subgraph "Servicios Externos"
        PRICE_API[Price API<br/>CoinGecko]
        QR_SCAN[QR Scanner<br/>HTML5]
        BLOCKCHAIN[Polygon Network<br/>USDT]
    end

    subgraph "Usuarios"
        USER[👤 Usuario]
        ALLY[🤝 Aliado]
        ADMIN[👨‍💼 Admin]
    end

    %% Conexiones principales
    USER --> WEB
    ALLY --> WEB
    ADMIN --> WEB
    
    WEB --> API
    WALLET --> WEB
    
    API --> AUTH
    API --> ORDER
    API --> PRICING
    API --> ESCROW
    
    CRON --> ORDER
    CRON --> PRICING
    
    ORDER --> DB
    PRICING --> DB
    ESCROW --> DB
    AUTH --> DB
    
    ORDER --> STORAGE
    ORDER --> REALTIME
    
    PRICING --> PRICE_API
    WEB --> QR_SCAN
    ESCROW --> BLOCKCHAIN
    
    %% Estilos
    classDef frontend fill:#e1f5fe
    classDef backend fill:#f3e5f5
    classDef storage fill:#e8f5e8
    classDef external fill:#fff3e0
    classDef users fill:#fce4ec
    
    class WEB,WALLET frontend
    class API,AUTH,ORDER,PRICING,ESCROW,CRON backend
    class DB,STORAGE,REALTIME storage
    class PRICE_API,QR_SCAN,BLOCKCHAIN external
    class USER,ALLY,ADMIN users
```

## Arquitectura de Componentes

```mermaid
graph TB
    subgraph "🖥️ Frontend Layer (Next.js)"
        HOME[🏠 Landing Page]
        USER_DASH[👤 User Dashboard] 
        ALLY_DASH[🤝 Ally Dashboard]
        ADMIN_DASH[👨‍💼 Admin Panel]
        QR_COMP[📱 QR Scanner Component]
        WALLET_COMP[🔗 Wallet Connect Component]
    end

    subgraph "⚙️ Backend API (Next.js API Routes)"
        AUTH_API[🔐 /api/auth/*]
        ORDER_API[📋 /api/orders/*] 
        QUOTE_API[💱 /api/quote]
        ADMIN_API[⚙️ /api/admin/*]
        UPLOAD_API[📤 /api/upload]
        CRON_API[⏰ /api/cron/*]
    end

    subgraph "🏗️ Services Layer"
        ORDER_SVC[📋 Order Service<br/>• Create/Update Orders<br/>• State Management<br/>• Timeout Handling]
        PRICING_SVC[💱 Pricing Service<br/>• Rate Calculation<br/>• Quote Management<br/>• Price Updates]
        ESCROW_SVC[🏦 Escrow Service<br/>• Fund Management<br/>• Release/Refund<br/>• Blockchain Integration]
        USER_SVC[👤 User Service<br/>• Authentication<br/>• Role Management<br/>• Profile Management]
        NOTIFY_SVC[🔔 Notification Service<br/>• Real-time Updates<br/>• Status Changes<br/>• Push Notifications]
    end

    subgraph "🗄️ Data Layer (Supabase)"
        ORDERS_DB[(📋 orders)]
        USERS_DB[(👤 users)]
        QUOTES_DB[(💱 quotes)]
        ESCROW_DB[(🏦 escrow_accounts)]
        LOGS_DB[(📝 logs)]
        FILES_STORAGE[📁 File Storage<br/>QR Images + Proofs]
        REALTIME_DB[⚡ Realtime Subscriptions]
    end

    %% Conexiones Frontend → API
    HOME --> AUTH_API
    USER_DASH --> ORDER_API
    ALLY_DASH --> ORDER_API
    ADMIN_DASH --> ADMIN_API
    QR_COMP --> QUOTE_API
    WALLET_COMP --> AUTH_API
    
    %% Conexiones API → Services
    AUTH_API --> USER_SVC
    ORDER_API --> ORDER_SVC
    QUOTE_API --> PRICING_SVC
    ADMIN_API --> ORDER_SVC
    UPLOAD_API --> FILES_STORAGE
    CRON_API --> ORDER_SVC
    CRON_API --> PRICING_SVC
    
    %% Conexiones Services → Data
    ORDER_SVC --> ORDERS_DB
    ORDER_SVC --> ESCROW_SVC
    ORDER_SVC --> NOTIFY_SVC
    PRICING_SVC --> QUOTES_DB
    USER_SVC --> USERS_DB
    ESCROW_SVC --> ESCROW_DB
    NOTIFY_SVC --> REALTIME_DB
    
    ORDER_SVC --> LOGS_DB
    ESCROW_SVC --> LOGS_DB
    
    %% Estilos
    classDef frontend fill:#e3f2fd
    classDef api fill:#fff8e1
    classDef services fill:#fce4ec
    classDef data fill:#e8f5e8
    
    class HOME,USER_DASH,ALLY_DASH,ADMIN_DASH,QR_COMP,WALLET_COMP frontend
    class AUTH_API,ORDER_API,QUOTE_API,ADMIN_API,UPLOAD_API,CRON_API api
    class ORDER_SVC,PRICING_SVC,ESCROW_SVC,USER_SVC,NOTIFY_SVC services
    class ORDERS_DB,USERS_DB,QUOTES_DB,ESCROW_DB,LOGS_DB,FILES_STORAGE,REALTIME_DB data
```

## Stack Tecnológico Detallado

```mermaid
graph TB
    subgraph "Frontend Stack"
        NEXTJS[Next.js 14<br/>App Router]
        TAILWIND[Tailwind CSS<br/>Responsive Design]
        PRIVY[Privy SDK<br/>Wallet Authentication]
        SUPABASE_CLIENT[Supabase Client<br/>Realtime + Storage]
    end
    
    subgraph "Backend Stack"
        NEXTJS_API[Next.js API Routes<br/>Serverless Functions]
        TYPESCRIPT[TypeScript<br/>Type Safety]
        VERCEL_CRON[Vercel Cron Jobs<br/>Automated Tasks]
    end
    
    subgraph "Database & Storage"
        SUPABASE[Supabase Platform]
        POSTGRESQL[PostgreSQL 15<br/>Relational Database]
        REALTIME_SUB[Realtime Subscriptions<br/>Live Updates]
        STORAGE_SUB[Storage Buckets<br/>QR + Proof Images]
        ROW_SECURITY[Row Level Security<br/>Data Protection]
    end
    
    subgraph "External Integrations"
        COINGECKO[CoinGecko API<br/>Price Feeds]
        POLYGON[Polygon Network<br/>USDT Transactions]
        PRIVY_AUTH[Privy Authentication<br/>Wallet Management]
    end
    
    subgraph "DevOps & Deployment"
        VERCEL[Vercel Platform<br/>Frontend + API]
        GITHUB[GitHub Actions<br/>CI/CD Pipeline]
        MONITORING[Vercel Analytics<br/>Performance Monitoring]
    end

    %% Relaciones
    NEXTJS --> NEXTJS_API
    TAILWIND --> NEXTJS
    PRIVY --> NEXTJS
    SUPABASE_CLIENT --> NEXTJS
    
    NEXTJS_API --> TYPESCRIPT
    VERCEL_CRON --> NEXTJS_API
    
    SUPABASE --> POSTGRESQL
    SUPABASE --> REALTIME_SUB
    SUPABASE --> STORAGE_SUB
    SUPABASE --> ROW_SECURITY
    
    NEXTJS_API --> COINGECKO
    NEXTJS_API --> POLYGON
    PRIVY --> PRIVY_AUTH
    
    VERCEL --> NEXTJS
    VERCEL --> NEXTJS_API
    GITHUB --> VERCEL
    VERCEL --> MONITORING
    
    %% Estilos
    classDef frontend fill:#e1f5fe
    classDef backend fill:#f3e5f5  
    classDef database fill:#e8f5e8
    classDef external fill:#fff3e0
    classDef devops fill:#fce4ec
    
    class NEXTJS,TAILWIND,PRIVY,SUPABASE_CLIENT frontend
    class NEXTJS_API,TYPESCRIPT,VERCEL_CRON backend
    class SUPABASE,POSTGRESQL,REALTIME_SUB,STORAGE_SUB,ROW_SECURITY database
    class COINGECKO,POLYGON,PRIVY_AUTH external
    class VERCEL,GITHUB,MONITORING devops
```

## Flujo de Datos Principal

```mermaid
sequenceDiagram
    participant U as 👤 Usuario
    participant F as 🖥️ Frontend
    participant API as ⚙️ Next.js API
    participant P as 💱 Pricing Service
    participant E as 🏦 Escrow Service
    participant DB as 🗄️ Supabase
    participant BC as ⛓️ Polygon

    Note over U,BC: FLUJO PRINCIPAL: Usuario Crea y Paga Orden

    U->>F: Escanea QR + ingresa monto BOB
    F->>API: POST /api/quote
    Note right of API: {qrData, amountBOB}
    
    API->>P: Calcular equivalente USDT
    P->>DB: Obtener cotización actual
    DB-->>P: Rate USDT/BOB
    P-->>API: Monto USDT calculado
    
    API->>DB: Crear orden (PENDING_PAYMENT)
    Note right of DB: Cotización fija por 3 min
    DB-->>API: Orden creada con ID
    
    API-->>F: {orderId, amountUSDT, escrowAddress, expiresAt}
    F-->>U: "Paga X USDT en 3 minutos"
    
    Note over F: Countdown timer 3:00
    U->>F: Confirma pago
    F->>U: Privy wallet prompt
    U->>BC: Aprueba transacción USDT
    
    BC->>E: Webhook: Fondos recibidos
    E->>API: Notificar pago confirmado
    API->>DB: Actualizar estado → AVAILABLE
    DB->>F: Realtime: Orden disponible
    F-->>U: "✅ Pago confirmado. Buscando aliado..."
```

## Arquitectura de APIs (Next.js Routes)

```mermaid
graph TB
    subgraph "🔐 Authentication APIs"
        AUTH_CONNECT[POST /api/auth/connect\nConnect wallet via Privy]
        AUTH_PROFILE[GET /api/auth/profile\nGet user profile]
        AUTH_LOGOUT[POST /api/auth/logout\nDisconnect wallet]
    end
    
    subgraph "💱 Quote & Pricing APIs"
        QUOTE_CREATE[POST /api/quote\nCalculate USDT amount]
        QUOTE_REFRESH[GET /api/quote/:id/refresh\nUpdate expired quote]
        PRICING_CURRENT[GET /api/pricing/current\nGet current rates]
    end
    
    subgraph "📋 Order Management APIs"
        ORDER_CREATE[POST /api/orders\nCreate new order]
        ORDER_LIST[GET /api/orders\nList user orders]
        ORDER_DETAIL[GET /api/orders/:id\nOrder details]
        ORDER_TAKE[POST /api/orders/:id/take\nAlly takes order]
        ORDER_PROOF[POST /api/orders/:id/proof\nUpload payment proof]
        ORDER_AVAILABLE[GET /api/orders/available\nAvailable for allies]
    end
    
    subgraph "📤 Upload APIs"
        UPLOAD_QR[POST /api/upload/qr\nUpload QR image]
        UPLOAD_PROOF[POST /api/upload/proof\nUpload payment proof]
    end
    
    subgraph "👨‍💼 Admin APIs"
        ADMIN_DASHBOARD[GET /api/admin/dashboard\nSystem metrics]
        ADMIN_ORDERS[GET /api/admin/orders\nAll orders with filters]
        ADMIN_USERS[GET /api/admin/users\nUser management]
        ADMIN_CONFIG[GET/PUT /api/admin/config\nSystem configuration]
    end
    
    subgraph "⏰ Automated Cron APIs"
        CRON_TIMEOUTS[POST /api/cron/check-timeouts\nProcess expired orders]
        CRON_PRICING[POST /api/cron/update-pricing\nUpdate exchange rates]
        CRON_CLEANUP[POST /api/cron/cleanup\nClean old data]
    end

    %% Estilos por categoría
    classDef auth fill:#ffebee
    classDef quote fill:#e8f5e8
    classDef order fill:#e3f2fd
    classDef upload fill:#fff3e0
    classDef admin fill:#fce4ec
    classDef cron fill:#f3e5f5
    
    class AUTH_CONNECT,AUTH_PROFILE,AUTH_LOGOUT auth
    class QUOTE_CREATE,QUOTE_REFRESH,PRICING_CURRENT quote
    class ORDER_CREATE,ORDER_LIST,ORDER_DETAIL,ORDER_TAKE,ORDER_PROOF,ORDER_AVAILABLE order
    class UPLOAD_QR,UPLOAD_PROOF upload
    class ADMIN_DASHBOARD,ADMIN_ORDERS,ADMIN_USERS,ADMIN_CONFIG admin
    class CRON_TIMEOUTS,CRON_PRICING,CRON_CLEANUP cron

```

## Seguridad y Configuración

### 🔒 **Seguridad MVP**

```mermaid
graph TB
    subgraph "🛡️ Frontend Security"
        WALLET_AUTH[Wallet-based Authentication<br/>No passwords needed]
        HTTPS_ONLY[HTTPS Only<br/>All communications encrypted]
        CSP[Content Security Policy<br/>XSS protection]
    end
    
    subgraph "🔐 API Security"
        JWT_TOKENS[JWT Tokens<br/>Stateless authentication]
        RATE_LIMITING[Rate Limiting<br/>Prevent abuse]
        CORS[CORS Configuration<br/>Restricted origins]
        INPUT_VALIDATION[Input Validation<br/>Schema validation]
    end
    
    subgraph "🗄️ Database Security"
        RLS[Row Level Security<br/>User data isolation]
        ENCRYPTED_STORAGE[Encrypted Storage<br/>Sensitive data protection]
        BACKUP_ENCRYPTION[Encrypted Backups<br/>Data persistence security]
    end
    
    subgraph "⛓️ Blockchain Security"
        ESCROW_WALLET[Controlled Escrow Wallet<br/>Centralized fund management]
        TX_VERIFICATION[Transaction Verification<br/>Blockchain confirmations]
        PRIVATE_KEY_MGMT[Private Key Management<br/>Secure key storage]
    end

    %% Conexiones de seguridad
    WALLET_AUTH --> JWT_TOKENS
    JWT_TOKENS --> RLS
    ESCROW_WALLET --> TX_VERIFICATION
    INPUT_VALIDATION --> ENCRYPTED_STORAGE
```

### ⚙️ **Configuración del Sistema**

| Componente | Configuración | Valor MVP |
|------------|---------------|-----------|
| **Timeouts** | PENDING_PAYMENT | 3 minutos |
| | AVAILABLE | 5 minutos |
| | TAKEN | 5 minutos |
| **Limits** | Min order amount | 10 BOB |
| | Max order amount | 10,000 BOB |
| **Pricing** | Update interval | 30 segundos |
| **Penalties** | Ally timeout block | 30 minutos |
| **Blockchain** | Network | Polygon |
| | Token | USDT |
| **Storage** | Max file size | 5 MB |
| | Allowed formats | JPG, PNG |

## Principios Arquitectónicos

### 🎯 **Separación de Responsabilidades**
- **Frontend**: Solo UI/UX y presentación
- **API Routes**: Validación, autenticación y orquestación
- **Services**: Lógica de negocio pura y reglas
- **Database**: Persistencia y consultas optimizadas

### 🔄 **Escalabilidad Serverless**
- **Next.js API Routes**: Auto-scaling según demanda
- **Vercel Functions**: Execución bajo demanda
- **Supabase**: Managed database con auto-scaling
- **Edge-ready**: Preparado para distribución global

### 🛡️ **Seguridad por Capas**
- **Wallet Authentication**: Sin contraseñas tradicionales
- **Row Level Security**: Aislamiento de datos por usuario
- **Escrow Centralizado**: Control total de fondos en MVP
- **Audit Logs**: Trazabilidad completa de acciones

### ⚡ **Performance Optimizado**
- **Static Generation**: Páginas estáticas donde sea posible
- **Realtime Updates**: Solo datos que cambian frecuentemente
- **Image Optimization**: Next.js Image component
- **Edge Caching**: CDN para assets estáticos

---

**📝 Notas para el Equipo:**

- ✅ **Stack unificado**: Next.js tanto para frontend como backend
- ✅ **Serverless-first**: Escalable automáticamente sin gestión de servidores
- ✅ **TypeScript everywhere**: Type safety en toda la aplicación
- ✅ **Supabase como backbone**: Database, Storage, Realtime en una plataforma
- ✅ **Privy para crypto**: Abstrae complejidad de wallet management
- ✅ **Vercel deployment**: Deploy automático desde Git

**🔑 Decisiones Técnicas Clave:**
- **Escrow centralizado**: Backend controla wallet, no smart contracts
- **Auto-aprobación**: Comprobantes se aprueban automáticamente en MVP
- **Timeouts agresivos**: UX rápida, previene fondos bloqueados
- **Mobile-first**: Optimizado para uso móvil principalmente
