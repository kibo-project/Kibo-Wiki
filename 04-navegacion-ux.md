# 🎨 Kibo - Navegación y Experiencia de Usuario

## Mapa de Navegación General

```mermaid
graph TD
    START[🌐 Landing Page<br/>kibo.app] --> AUTH{👤 Conectar Wallet?}
    
    AUTH -->|Wallet conectada| ROLE{🎭 ¿Qué rol tiene?}
    AUTH -->|Rechaza conexión| START
    
    ROLE -->|role: 'user'| USER_HOME[👤 User Dashboard]
    ROLE -->|role: 'ally'| ALLY_HOME[🤝 Ally Dashboard] 
    ROLE -->|role: 'admin'| ADMIN_HOME[👨‍💼 Admin Panel]
    
    subgraph "🔵 Flujo Usuario - Realizar Pagos"
        USER_HOME --> SCAN[📱 Escanear QR]
        USER_HOME --> HISTORY_U[📋 Mis Órdenes]
        
        SCAN --> QUOTE[💱 Ver Cotización<br/>⏱️ 3:00 countdown]
        QUOTE -->|Confirmar| PAY[💳 Pagar USDT<br/>Privy Wallet]
        QUOTE -->|Cancelar| USER_HOME
        
        PAY -->|Éxito| WAITING[⏳ Esperando Aliado<br/>Estado: AVAILABLE]
        PAY -->|Error| ERROR_PAY[❌ Error de Pago]
        
        WAITING --> TRACKING[👀 Rastrear Orden<br/>Estado: TAKEN]
        TRACKING --> SUCCESS_U[✅ Pago Completado<br/>Estado: COMPLETED]
        
        HISTORY_U --> DETAIL_U[🔍 Detalle de Orden]
        ERROR_PAY --> USER_HOME
    end
    
    subgraph "🟢 Flujo Aliado - Procesar Órdenes"
        ALLY_HOME --> AVAILABLE[📋 Órdenes Disponibles<br/>Auto-refresh 10s]
        ALLY_HOME --> ACTIVE[⚡ Mi Orden Activa]
        ALLY_HOME --> STATS[📊 Mis Estadísticas]
        
        AVAILABLE --> TAKE[🎯 Tomar Orden<br/>⏱️ 5:00 countdown]
        TAKE --> QR_VIEW[👀 Ver QR a Pagar]
        QR_VIEW --> BANK_APP[🏦 App Bancaria<br/>Externa]
        BANK_APP --> UPLOAD[📤 Subir Comprobante]
        UPLOAD --> SUCCESS_A[✅ USDT Recibido<br/>Auto-aprobado]
        
        ACTIVE --> QR_VIEW
        STATS --> EARNINGS[💰 Historial Ganancias]
    end
    
    subgraph "🔴 Flujo Admin - Monitoreo"
        ADMIN_HOME --> MONITOR[📊 Monitor Sistema<br/>Tiempo real]
        ADMIN_HOME --> USERS_MGMT[👥 Gestión Usuarios]
        ADMIN_HOME --> CONFIG[⚙️ Configuración]
        
        MONITOR --> LOGS[📝 Logs del Sistema]
        MONITOR --> ALERTS[🚨 Alertas Activas]
        
        CONFIG --> TIMEOUTS[⏰ Ajustar Timeouts]
        CONFIG --> LIMITS[💰 Límites de Órdenes]
        CONFIG --> RATES[💱 Gestión Cotizaciones]
        
        USERS_MGMT --> USER_DETAIL[👤 Detalle Usuario]
        USER_DETAIL --> PENALTIES[⚠️ Penalizaciones]
    end
    
    %% Casos de error y timeout
    WAITING -.->|Timeout 5min| TIMEOUT_AVAILABLE[⏰ Nadie tomó orden]
    TRACKING -.->|Timeout 5min| TIMEOUT_TAKEN[⏰ Aliado no completó]
    
    TIMEOUT_AVAILABLE --> REFUND[💰 Refund Automático]
    TIMEOUT_TAKEN --> REFUND
    REFUND --> USER_HOME
    
    %% Estilos
    classDef userFlow fill:#e3f2fd
    classDef allyFlow fill:#e8f5e8  
    classDef adminFlow fill:#ffebee
    classDef decision fill:#fff3e0
    classDef success fill:#c8e6c9
    classDef error fill:#ffcdd2
    classDef timeout fill:#ffecb3
    
    class USER_HOME,SCAN,QUOTE,PAY,WAITING,TRACKING,HISTORY_U,DETAIL_U userFlow
    class ALLY_HOME,AVAILABLE,TAKE,QR_VIEW,UPLOAD,ACTIVE,STATS,EARNINGS allyFlow
    class ADMIN_HOME,MONITOR,USERS_MGMT,CONFIG,LOGS,ALERTS,TIMEOUTS,LIMITS,RATES,USER_DETAIL,PENALTIES adminFlow
    class AUTH,ROLE decision
    class SUCCESS_U,SUCCESS_A success
    class ERROR_PAY error
    class TIMEOUT_AVAILABLE,TIMEOUT_TAKEN,REFUND timeout
```

## Estados de Navegación por Pantalla

### 📱 **App Estado Usuario**

```mermaid
stateDiagram-v2
    [*] --> Landing
    
    Landing --> UserDashboard : Wallet conectada
    Landing --> Landing : Wallet rechazada
    
    UserDashboard --> ScanQR : Escanear QR
    UserDashboard --> OrderHistory : Mis Órdenes
    UserDashboard --> Profile : Mi Perfil
    
    ScanQR --> QuoteScreen : QR válido
    ScanQR --> ScanError : QR inválido
    ScanError --> ScanQR : Reintentar
    
    QuoteScreen --> PaymentScreen : Confirmar
    QuoteScreen --> UserDashboard : Cancelar
    QuoteScreen --> QuoteExpired : Timeout 3min
    QuoteExpired --> ScanQR : Nueva cotización
    
    PaymentScreen --> WaitingScreen : Pago confirmado
    PaymentScreen --> PaymentError : Pago falló
    PaymentError --> QuoteScreen : Reintentar
    
    WaitingScreen --> TrackingScreen : Aliado asignado
    WaitingScreen --> TimeoutRefund : Timeout 5min
    
    TrackingScreen --> SuccessScreen : Orden completada
    TrackingScreen --> TimeoutRefund : Aliado timeout
    
    TimeoutRefund --> UserDashboard : USDT devuelto
    SuccessScreen --> UserDashboard : Nueva orden
    
    OrderHistory --> OrderDetail : Seleccionar orden
    OrderDetail --> OrderHistory : Volver
    OrderHistory --> UserDashboard : Volver
    
    Profile --> UserDashboard : Volver

    note right of QuoteScreen
        Countdown 3:00 visible
        Cotización fija durante timer
        Auto-refresh si expira
    end note

    note right of WaitingScreen
        Estado en tiempo real
        Buscando aliado...
        Notificación cuando asignen
    end note
```

### 🤝 **App Estado Aliado**

```mermaid
stateDiagram-v2
    [*] --> Landing
    
    Landing --> AllyDashboard : Wallet conectada
    
    AllyDashboard --> AvailableOrders : Ver órdenes
    AllyDashboard --> ActiveOrder : Mi orden activa
    AllyDashboard --> StatsScreen : Estadísticas
    AllyDashboard --> PenaltyScreen : Si penalizado
    
    AvailableOrders --> ProcessOrder : Tomar orden
    AvailableOrders --> AllyDashboard : Volver
    AvailableOrders --> OrderTaken : Orden tomada por otro
    OrderTaken --> AvailableOrders : Auto-refresh
    
    ProcessOrder --> UploadProof : Después de pagar
    ProcessOrder --> ProcessTimeout : Timeout 5min
    ProcessTimeout --> AllyDashboard : Penalizado 30min
    
    UploadProof --> SuccessScreen : Comprobante subido
    UploadProof --> ProcessTimeout : Timeout restante
    
    SuccessScreen --> AllyDashboard : USDT recibido
    
    ActiveOrder --> ProcessOrder : Continuar orden
    ActiveOrder --> AllyDashboard : Volver
    
    StatsScreen --> EarningsDetail : Ver ganancias
    StatsScreen --> AllyDashboard : Volver
    EarningsDetail --> StatsScreen : Volver
    
    PenaltyScreen --> AllyDashboard : Penalización expirada

    note right of ProcessOrder
        QR grande más datos bancarios
        Countdown 5:00 visible
        Botón Ya pagué
    end note

    note right of UploadProof
        Cámara o galería
        Preview imagen
        Validación tamaño menor 5MB
    end note
```

## Pantallas Detalladas por Tipo de Usuario

### 🌐 **Landing Page (Punto de Entrada)**

```mermaid
graph TB
    subgraph "🌐 Landing Page - Layout"
        HEADER[Kibo Logo\n🔗 Conectar Wallet]
        HERO[Paga cualquier QR bancario\nusando criptomonedas\n💡 Escanea QR → Paga USDT → Aliado procesa]
        FEATURES[✅ Pagos en segundos\n✅ Red de aliados confiables\n✅ Soporte BOB ↔ USDT\n✅ Tarifas competitivas]
        CTA_SECTION[🚀 Comenzar Ahora\n🤝 Ser Aliado]
        DEMO[📱 Ver Demo\n📊 Estadísticas en Vivo]
        FOOTER[📞 Contacto\n📄 Términos\n🔒 Seguridad]
    end
    
    subgraph "Interacciones"
        CONNECT_WALLET[Conectar Wallet via Privy]
        ROLE_DETECTION[Detectar rol del usuario]
        REDIRECT_DASHBOARD[Redirigir a dashboard correspondiente]
    end

    HEADER --> CONNECT_WALLET
    CTA_SECTION --> CONNECT_WALLET
    CONNECT_WALLET --> ROLE_DETECTION
    ROLE_DETECTION --> REDIRECT_DASHBOARD

    %% Estilos personalizados
    style HEADER fill:#e3f2fd
    style HERO fill:#e8f5e8
    style CTA_SECTION fill:#fff3e0
```

### 👤 **User Dashboard - Panel Principal**

```mermaid
graph TB
    subgraph "User Dashboard Mobile First"
        NAV[Inicio - Órdenes - Perfil - Logout]
        
        subgraph "Balance Card"
            BALANCE[Mi Balance 250.00 USDT - Red Polygon]
            REFRESH_BALANCE[Actualizar Balance]
        end
        
        subgraph "Nueva Orden Card Principal"
            QR_BTN[Escanear QR para Pagar]
            OR_TEXT[--- O ---]
            MANUAL_BTN[Ingresar datos manualmente - Próximamente]
        end
        
        subgraph "Órdenes Recientes Card"
            RECENT_HEADER[Mis Órdenes Recientes]
            ORDER1[150 BOB - 21.5 USDT - Completado]
            ORDER2[200 BOB - 28.7 USDT - En proceso]
            ORDER3[100 BOB - 14.3 USDT - Expirado]
            VIEW_ALL[Ver todas mis órdenes]
        end
        
        subgraph "Estadísticas Personales"
            STATS[Total procesado 1250 BOB - Ahorrado en fees 5 porciento]
        end
        
        subgraph "Estado del Sistema"
            SYSTEM_STATUS[Sistema operativo - 12 aliados activos]
        end
    end
    
    %% Interacciones
    QR_BTN --> SCAN_QR_FLOW[Flujo escanear QR]
    ORDER1 --> ORDER_DETAIL[Ver detalle orden]
    ORDER2 --> ORDER_TRACKING[Seguimiento en tiempo real]
    ORDER3 --> ORDER_DETAIL
    VIEW_ALL --> ORDER_HISTORY[Lista completa órdenes]
    
    style QR_BTN fill:#4caf50,color:#fff
    style ORDER2 fill:#fff3e0
    style SYSTEM_STATUS fill:#e8f5e8
```
