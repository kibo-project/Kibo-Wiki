# 🎨 Kibo - Navigation and User Experience

## General Navigation Map

```mermaid
graph TD
    START[🌐 Landing Page<br/>kibo.app] --> AUTH{👤 Connect Wallet?}
    
    AUTH -->|Wallet connected| ROLE{🎭 What role does user have?}
    AUTH -->|Rejects connection| START
    
    ROLE -->|role: 'user'| USER_HOME[👤 User Dashboard]
    ROLE -->|role: 'ally'| ALLY_HOME[🤝 Ally Dashboard] 
    ROLE -->|role: 'admin'| ADMIN_HOME[👨‍💼 Admin Panel]
    
    subgraph "🔵 User Flow - Make Payments"
        USER_HOME --> SCAN[📱 Scan QR]
        USER_HOME --> HISTORY_U[📋 My Orders]
        
        SCAN --> QUOTE[💱 View Quote<br/>⏱️ 3:00 countdown]
        QUOTE -->|Confirm| PAY[💳 Pay USDT<br/>Privy Wallet]
        QUOTE -->|Cancel| USER_HOME
        
        PAY -->|Success| WAITING[⏳ Waiting for Ally<br/>Status: AVAILABLE]
        PAY -->|Error| ERROR_PAY[❌ Payment Error]
        
        WAITING --> TRACKING[👀 Track Order<br/>Status: TAKEN]
        TRACKING --> SUCCESS_U[✅ Payment Completed<br/>Status: COMPLETED]
        
        HISTORY_U --> DETAIL_U[🔍 Order Details]
        ERROR_PAY --> USER_HOME
    end
    
    subgraph "🟢 Ally Flow - Process Orders"
        ALLY_HOME --> AVAILABLE[📋 Available Orders<br/>Auto-refresh 10s]
        ALLY_HOME --> ACTIVE[⚡ My Active Order]
        ALLY_HOME --> STATS[📊 My Statistics]
        
        AVAILABLE --> TAKE[🎯 Take Order<br/>⏱️ 5:00 countdown]
        TAKE --> QR_VIEW[👀 View QR to Pay]
        QR_VIEW --> BANK_APP[🏦 Banking App<br/>External]
        BANK_APP --> UPLOAD[📤 Upload Receipt]
        UPLOAD --> SUCCESS_A[✅ USDT Received<br/>Auto-approved]
        
        ACTIVE --> QR_VIEW
        STATS --> EARNINGS[💰 Earnings History]
    end
    
    subgraph "🔴 Admin Flow - Monitoring"
        ADMIN_HOME --> MONITOR[📊 System Monitor<br/>Real time]
        ADMIN_HOME --> USERS_MGMT[👥 User Management]
        ADMIN_HOME --> CONFIG[⚙️ Configuration]
        
        MONITOR --> LOGS[📝 System Logs]
        MONITOR --> ALERTS[🚨 Active Alerts]
        
        CONFIG --> TIMEOUTS[⏰ Adjust Timeouts]
        CONFIG --> LIMITS[💰 Order Limits]
        CONFIG --> RATES[💱 Quote Management]
        
        USERS_MGMT --> USER_DETAIL[👤 User Details]
        USER_DETAIL --> PENALTIES[⚠️ Penalties]
    end
    
    %% Error and timeout cases
    WAITING -.->|Timeout 5min| TIMEOUT_AVAILABLE[⏰ Nobody took order]
    TRACKING -.->|Timeout 5min| TIMEOUT_TAKEN[⏰ Ally didn't complete]
    
    TIMEOUT_AVAILABLE --> REFUND[💰 Automatic Refund]
    TIMEOUT_TAKEN --> REFUND
    REFUND --> USER_HOME
    
    %% Styles
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

## Navigation States by Screen

### 📱 **User App State**

```mermaid
stateDiagram-v2
    [*] --> Landing
    
    Landing --> UserDashboard : Wallet connected
    Landing --> Landing : Wallet rejected
    
    UserDashboard --> ScanQR : Scan QR
    UserDashboard --> OrderHistory : My Orders
    UserDashboard --> Profile : My Profile
    
    ScanQR --> QuoteScreen : Valid QR
    ScanQR --> ScanError : Invalid QR
    ScanError --> ScanQR : Retry
    
    QuoteScreen --> PaymentScreen : Confirm
    QuoteScreen --> UserDashboard : Cancel
    QuoteScreen --> QuoteExpired : Timeout 3min
    QuoteExpired --> ScanQR : New quote
    
    PaymentScreen --> WaitingScreen : Payment confirmed
    PaymentScreen --> PaymentError : Payment failed
    PaymentError --> QuoteScreen : Retry
    
    WaitingScreen --> TrackingScreen : Ally assigned
    WaitingScreen --> TimeoutRefund : Timeout 5min
    
    TrackingScreen --> SuccessScreen : Order completed
    TrackingScreen --> TimeoutRefund : Ally timeout
    
    TimeoutRefund --> UserDashboard : USDT returned
    SuccessScreen --> UserDashboard : New order
    
    OrderHistory --> OrderDetail : Select order
    OrderDetail --> OrderHistory : Go back
    OrderHistory --> UserDashboard : Go back
    
    Profile --> UserDashboard : Go back

    note right of QuoteScreen
        Countdown 3:00 visible
        Fixed quote during timer
        Auto-refresh if expires
    end note

    note right of WaitingScreen
        Real-time status
        Searching for ally...
        Notification when assigned
    end note
```

### 🤝 **Ally App State**

```mermaid
stateDiagram-v2
    [*] --> Landing
    
    Landing --> AllyDashboard : Wallet connected
    
    AllyDashboard --> AvailableOrders : View orders
    AllyDashboard --> ActiveOrder : My active order
    AllyDashboard --> StatsScreen : Statistics
    AllyDashboard --> PenaltyScreen : If penalized
    
    AvailableOrders --> ProcessOrder : Take order
    AvailableOrders --> AllyDashboard : Go back
    AvailableOrders --> OrderTaken : Order taken by another
    OrderTaken --> AvailableOrders : Auto-refresh
    
    ProcessOrder --> UploadProof : After paying
    ProcessOrder --> ProcessTimeout : Timeout 5min
    ProcessTimeout --> AllyDashboard : Penalized 30min
    
    UploadProof --> SuccessScreen : Receipt uploaded
    UploadProof --> ProcessTimeout : Remaining timeout
    
    SuccessScreen --> AllyDashboard : USDT received
    
    ActiveOrder --> ProcessOrder : Continue order
    ActiveOrder --> AllyDashboard : Go back
    
    StatsScreen --> EarningsDetail : View earnings
    StatsScreen --> AllyDashboard : Go back
    EarningsDetail --> StatsScreen : Go back
    
    PenaltyScreen --> AllyDashboard : Penalty expired

    note right of ProcessOrder
        Large QR plus banking data
        Countdown 5:00 visible
        "Already paid" button
    end note

    note right of UploadProof
        Camera or gallery
        Image preview
        Size validation under 5MB
    end note
```

## Detailed Screens by User Type

### 🌐 **Landing Page (Entry Point)**

```mermaid
graph TB
    subgraph "🌐 Landing Page - Layout"
        HEADER[Kibo Logo\n🔗 Connect Wallet]
        HERO[Pay any bank QR code\nusing cryptocurrencies\n💡 Scan QR → Pay USDT → Ally processes]
        FEATURES[✅ Payments in seconds\n✅ Reliable ally network\n✅ BOB ↔ USDT support\n✅ Competitive fees]
        CTA_SECTION[🚀 Get Started\n🤝 Become an Ally]
        DEMO[📱 View Demo\n📊 Live Statistics]
        FOOTER[📞 Contact\n📄 Terms\n🔒 Security]
    end
    
    subgraph "Interactions"
        CONNECT_WALLET[Connect Wallet via Privy]
        ROLE_DETECTION[Detect user role]
        REDIRECT_DASHBOARD[Redirect to corresponding dashboard]
    end

    HEADER --> CONNECT_WALLET
    CTA_SECTION --> CONNECT_WALLET
    CONNECT_WALLET --> ROLE_DETECTION
    ROLE_DETECTION --> REDIRECT_DASHBOARD

    %% Custom styles
    style HEADER fill:#e3f2fd
    style HERO fill:#e8f5e8
    style CTA_SECTION fill:#fff3e0
```

### 👤 **User Dashboard - Main Panel**

```mermaid
graph TB
    subgraph "User Dashboard Mobile First"
        NAV[Home - Orders - Profile - Logout]
        
        subgraph "Balance Card"
            BALANCE[My Balance 250.00 USDT - Mantle network]
            REFRESH_BALANCE[Refresh Balance]
        end
        
        subgraph "New Order Main Card"
            QR_BTN[Scan QR to Pay]
            OR_TEXT[--- OR ---]
            MANUAL_BTN[Enter data manually - Coming Soon]
        end
        
        subgraph "Recent Orders Card"
            RECENT_HEADER[My Recent Orders]
            ORDER1[150 BOB - 21.5 USDT - Completed]
            ORDER2[200 BOB - 28.7 USDT - In Progress]
            ORDER3[100 BOB - 14.3 USDT - Expired]
            VIEW_ALL[View all my orders]
        end
        
        subgraph "Personal Statistics"
            STATS[Total processed 1250 BOB - Saved in fees 5 percent]
        end
        
        subgraph "System Status"
            SYSTEM_STATUS[System operational - 12 active allies]
        end
    end
    
    %% Interactions
    QR_BTN --> SCAN_QR_FLOW[Scan QR flow]
    ORDER1 --> ORDER_DETAIL[View order details]
    ORDER2 --> ORDER_TRACKING[Real-time tracking]
    ORDER3 --> ORDER_DETAIL
    VIEW_ALL --> ORDER_HISTORY[Complete order list]
    
    style QR_BTN fill:#4caf50,color:#fff
    style ORDER2 fill:#fff3e0
    style SYSTEM_STATUS fill:#e8f5e8
```
