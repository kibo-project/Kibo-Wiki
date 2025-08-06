# 🔄 Kibo - States and Process Flows

## Order States (State Machine)

```mermaid
stateDiagram-v2
    [*] --> PENDING_PAYMENT : User sends QR + BOB amount
    
    PENDING_PAYMENT --> AVAILABLE : USDT received in escrow
    PENDING_PAYMENT --> EXPIRED : 3 minute timeout
    
    AVAILABLE --> TAKEN : Aliado takes the order
    AVAILABLE --> EXPIRED : 5 minute timeout
    
    TAKEN --> COMPLETED : Aliado uploads receipt
    TAKEN --> EXPIRED : 5 minute timeout
    
    EXPIRED --> REFUNDED : System returns USDT automatically
    COMPLETED --> [*] : USDT released to aliado
    REFUNDED --> [*] : Process terminated

    note right of PENDING_PAYMENT
        ⏱️ Fixed quote for 3 minutes
        💳 User must confirm USDT payment
        🚫 No manual cancellation option
    end note

    note right of AVAILABLE
        🏦 USDT held in escrow
        👁️ Visible to all aliados
        ⚡ Timeout to maintain UX speed
    end note

    note right of TAKEN
        🔒 Aliado assigned exclusively
        💰 Must pay QR and upload receipt
        ⏰ Time pressure for fast UX
    end note

    note right of COMPLETED
        🚀 MVP: Receipt = automatic approval
        🚫 No admin verification for now
        💸 Funds released immediately
    end note
```

## Detailed State Descriptions

### 🟡 **PENDING_PAYMENT** 
```
🎯 What it means: Order created, waiting for user's USDT payment
⏱️ Duration: 3 minutes
🔄 Can go to: AVAILABLE, EXPIRED
✨ Trigger: User sends QR + BOB amount

Characteristics:
• USDT/BOB quote fixed for 3 minutes
• User sees real-time countdown
• Cannot be cancelled manually
• If not paid → order deleted (EXPIRED)
```

### 🔵 **AVAILABLE**
```
🎯 What it means: USDT received, order available for aliados
⏱️ Duration: 5 minutes
🔄 Can go to: TAKEN, EXPIRED  
✨ Trigger: USDT payment confirmation on blockchain

Characteristics:
• Funds locked in escrow
• Visible on aliados dashboard
• Only aliados from same country can see it
• Auto-refresh every 10 seconds
• If no one takes it → automatic refund
```

### 🟣 **TAKEN**
```
🎯 What it means: Aliado assigned, must process payment
⏱️ Duration: 5 minutes
🔄 Can go to: COMPLETED, EXPIRED
✨ Trigger: Aliado accepts available order

Characteristics:
• Order locked exclusively for one aliado
• QR shown only to assigned aliado
• Must pay in banking app and upload receipt
• Visible countdown for time pressure
• If not completed → penalty + refund
```

### 🟢 **COMPLETED**
```
🎯 What it means: Successful process, USDT released to aliado
⏱️ Duration: Final state
🔄 Can go to: [Final]
✨ Trigger: Aliado uploads receipt (auto-approved in MVP)

Characteristics:
• Receipt uploaded = automatic approval
• USDT transferred immediately to aliado
• Notifications sent to user and aliado
• Aliado statistics updated
• Complete logs recorded
```

### 🔴 **EXPIRED**
```
🎯 What it means: Timeout at any stage of the process
⏱️ Duration: Immediate → REFUNDED
🔄 Can go to: REFUNDED
✨ Trigger: Any configured timeout

Characteristics:
• Can occur in PENDING_PAYMENT, AVAILABLE, or TAKEN
• If funds in escrow → triggers automatic refund
• Aliado receives penalty if was in TAKEN
• User notified of the issue
```

### ⚪ **REFUNDED**
```
🎯 What it means: USDT returned to original user
⏱️ Duration: Final state
🔄 Can go to: [Final]
✨ Trigger: Any expiration with funds in escrow

Characteristics:
• USDT automatically returned to original wallet
• Transaction hash recorded
• User notified of refund
• Order marked as closed
• Does not impact user reputation
```

## User Flow (Create and Pay Order)

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant F as 🖥️ Frontend
    participant API as ⚙️ Next.js API
    participant Q as 💱 Quote Service
    participant E as 🏦 Escrow Service
    participant DB as 🗄️ Supabase
    participant W as 🔗 Privy Wallet
    participant BC as ⛓️ mantle

    Note over U,BC: USER FLOW: Create and Pay Order

    U->>F: Scans QR + enters 200 BOB
    F->>API: POST /api/quote
    Note right of API: {qrData, amountBOB: 200}
    
    API->>Q: Calculate USDT equivalent
    Q->>DB: SELECT rate FROM quotes WHERE active=true
    DB-->>Q: 1 USDT = 6.97 BOB
    Q-->>API: 28.69 USDT + 0.1 fee = 28.79 USDT
    
    API->>DB: INSERT INTO orders (status='PENDING_PAYMENT')
    Note right of DB: expires_at = NOW() + 3 minutes
    DB-->>API: Order ID: #025
    
    API-->>F: {orderId: "025", amountUSDT: 28.79, expires: "3:00"}
    F-->>U: "Pay 28.79 USDT in 3:00 minutes"
    
    Note over F: Countdown timer started
    U->>F: Click "Confirm Payment"
    F->>W: requestTransaction(28.79 USDT)
    W->>U: "Approve transaction in wallet"
    U->>W: ✅ Approve
    W->>BC: Transfer 28.79 USDT → escrow_wallet
    
    BC->>E: Webhook: tx_hash confirmed
    E->>API: POST /api/orders/025/payment-confirmed
    API->>DB: UPDATE orders SET status='AVAILABLE'
    DB->>F: Realtime: order_status_changed
    F-->>U: "✅ Payment confirmed. Finding aliado..."
    
    Note over U: User can close app
    Note over U: Will receive notification when completed
```

## Aliado Flow (Take and Process Order)

```mermaid
sequenceDiagram
    participant AL as 🤝 Aliado
    participant F as 🖥️ Frontend
    participant API as ⚙️ Next.js API
    participant DB as 🗄️ Supabase
    participant BANK as 🏦 Banking App
    participant STORAGE as 📁 Supabase Storage

    Note over AL,STORAGE: ALIADO FLOW: Process Order

    AL->>F: Opens aliados dashboard
    F->>API: GET /api/orders/available
    API->>DB: SELECT * FROM available_orders_view
    DB-->>API: [Order #025: 200 BOB → 28.69 USDT, 4:15 remaining]
    API-->>F: List orders with timeouts
    F-->>AL: "💰 200 BOB → 28.69 USDT (⏰ 4:15)"
    
    Note over AL: Sees potential profit and time
    AL->>F: Click "TAKE ORDER #025"
    F->>API: POST /api/orders/025/take
    
    API->>DB: UPDATE orders SET ally_id=AL, status='TAKEN'
    Note right of DB: expires_at = NOW() + 5 minutes
    API->>DB: INSERT INTO logs (action='ORDER_TAKEN')
    API-->>F: {qrData, expiresIn: "5:00", orderDetails}
    F-->>AL: "QR to pay + countdown 5:00"
    
    Note over AL: Sees large QR + banking data
    AL->>BANK: Opens banking app (outside Kibo)
    AL->>BANK: Pays QR: 200 BOB to destination account
    BANK-->>AL: "✅ Transfer successful"
    
    AL->>F: Returns to Kibo
    AL->>F: Click "Upload Receipt"
    F->>AL: Opens camera/gallery
    AL->>F: Selects receipt photo
    
    F->>API: POST /api/orders/025/proof + FormData(image)
    API->>STORAGE: Upload image → /proofs/025.jpg
    STORAGE-->>API: URL: https://storage.../proofs/025.jpg
    API->>DB: UPDATE orders SET proof_url, status='COMPLETED'
    Note right of API: MVP: Auto-approval without verification
    
    API->>E: Trigger: release funds to aliado
    E->>BC: Transfer 28.69 USDT → ally_wallet
    API->>DB: UPDATE escrow_accounts SET status='RELEASED'
    
    DB->>F: Realtime: order_completed
    F-->>AL: "✅ Completed! 28.69 USDT sent"
    
    Note over AL: Can take new order immediately
```

## System Flow (Automatic Timeout Management)

```mermaid
sequenceDiagram
    participant CRON as ⏰ Vercel Cron
    participant API as ⚙️ Next.js API
    participant DB as 🗄️ Supabase
    participant E as 🏦 Escrow Service
    participant N as 🔔 Notification
    participant U as 👤 User
    participant AL as 🤝 Aliado

    Note over CRON,AL: SYSTEM FLOW: Automatic Timeouts

    loop Every 30 seconds
        CRON->>API: POST /api/cron/check-timeouts
        API->>DB: SELECT * FROM orders WHERE expires_at < NOW()
        
        alt Order #026 PENDING_PAYMENT expired (3+ min)
            DB-->>API: Order #026, status='PENDING_PAYMENT'
            API->>DB: UPDATE orders SET status='EXPIRED'
            API->>N: Send notification
            N-->>U: "Your order #026 expired. Try again."
            Note right of API: No refund - no funds
        
        else Order #027 AVAILABLE expired (5+ min)
            DB-->>API: Order #027, status='AVAILABLE' 
            API->>DB: UPDATE orders SET status='EXPIRED'
            API->>E: processRefund(order #027)
            E->>BC: Transfer USDT back to user
            E->>DB: INSERT INTO refund_jobs (status='COMPLETED')
            API->>N: Send notification
            N-->>U: "Order #027 expired. USDT returned."
        
        else Order #028 TAKEN expired (5+ min)
            DB-->>API: Order #028, status='TAKEN', ally_id='AL123'
            API->>DB: UPDATE orders SET status='EXPIRED'
            API->>DB: INSERT INTO ally_penalties (ally_id='AL123', reason='TIMEOUT')
            API->>E: processRefund(order #028)
            E->>BC: Transfer USDT back to user
            API->>N: Send notifications
            N-->>U: "Order #028 expired. USDT returned."
            N-->>AL: "⚠️ Order #028 expired. Penalized 30 min."
            
            Note right of API: Aliado blocked for 30 minutes
        end
    end

    Note over CRON: 100% automatic system
    Note over CRON: Runs on Vercel Cron Jobs
    Note over CRON: No manual intervention required
```

## Timeouts and Business Rules

### ⏰ **Timeouts by State**

```mermaid
graph TB
    subgraph "⏰ Configured Timeouts"
        A[PENDING_PAYMENT<br/>3 minutes] -->|Timeout| B[EXPIRED<br/>Delete order]
        C[AVAILABLE<br/>5 minutes] -->|Timeout| D[EXPIRED<br/>Automatic refund]
        E[TAKEN<br/>5 minutes] -->|Timeout| F[EXPIRED<br/>Refund + penalize]
    end
    
    subgraph "📊 Successful Timeline"
        T0[00:00 User scans QR] --> T1[00:30 Sees quote]
        T1 --> T2[02:00 Confirms payment]
        T2 --> T3[03:30 AVAILABLE]
        T3 --> T4[04:00 Aliado takes]
        T4 --> T5[07:00 Aliado pays bank]
        T5 --> T6[08:00 Uploads receipt]
        T6 --> T7[08:30 COMPLETED]
    end
    
    style B fill:#ffcdd2
    style D fill:#ffcdd2
    style F fill:#ffcdd2
    style T7 fill:#c8e6c9
```

### 🔒 **Locking Rules**

| Rule | Description | Implementation |
|------|-------------|----------------|
| **One aliado, one order** | Aliado can only have 1 TAKEN order at a time | `WHERE ally_id IS NULL AND user NOT IN (SELECT ally_id FROM orders WHERE status='TAKEN')` |
| **Timeout penalty** | Aliado who lets order expire gets blocked 30 min | `INSERT INTO ally_penalties (penalty_until = NOW() + INTERVAL '30 minutes')` |
| **No retaking expired order** | Same aliado cannot retake order they let expire | `WHERE order_id NOT IN (SELECT order_id FROM ally_penalties WHERE ally_id = ?)` |
| **Country specific** | Aliados only see orders from their country | `WHERE user.country = ally.country` |
| **Unique quote** | One quote per order, fixed until completion | `UPDATE quotes SET is_active=false WHERE order_id = ?` |

### 🚨 **Automatic Error Handling**

```mermaid
flowchart TD
    ERROR{Error Detected} --> |Blockchain payment fails| REFUND_USER[Immediate Refund]
    ERROR --> |Aliado doesn't respond| PENALTY[Penalize + Refund]
    ERROR --> |System unavailable| QUEUE[Queue for retry]
    ERROR --> |Image too large| REJECT[Reject upload]
    
    REFUND_USER --> LOG[Record in logs]
    PENALTY --> LOG
    QUEUE --> LOG
    REJECT --> USER_NOTIFY[Notify user]
    
    LOG --> ADMIN_ALERT[Admin alert if critical]
    USER_NOTIFY --> LOG
    
    style ERROR fill:#ffcdd2
    style REFUND_USER fill:#c8e6c9
    style PENALTY fill:#ffecb3
    style LOG fill:#e1f5fe
```

### 📊 **Performance Metrics**

| Metric | MVP Target | Measurement |
|--------|------------|-------------|
| **Average order time** | < 10 minutes | `AVG(completed_at - created_at)` |
| **Success rate** | > 80% | `COMPLETED / TOTAL * 100` |
| **Timeout rate** | < 15% | `EXPIRED / TOTAL * 100` |
| **Daily active aliados** | > 5 aliados | `COUNT(DISTINCT ally_id) WHERE DATE(taken_at) = TODAY` |
| **Daily volume** | > 1000 BOB | `SUM(amount_fiat) WHERE DATE(created_at) = TODAY` |

## Real-Time Notifications

### 🔔 **Notification Events**

```mermaid
graph TB
    subgraph "👤 User Receives"
        U1[🔔 Aliado assigned to your order]
        U2[🔔 Payment completed successfully]
        U3[🔔 Order expired - USDT returned]
        U4[🔔 Error in process]
    end
    
    subgraph "🤝 Aliado Receives"
        A1[🔔 New order available in your area]
        A2[🔔 Order taken successfully]
        A3[🔔 Receipt processed - USDT received]
        A4[🔔 Order expired - penalty applied]
    end
    
    subgraph "👨‍💼 Admin Receives"
        AD1[🔔 High volume of timeouts]
        AD2[🔔 External API not responding]
        AD3[🔔 Escrow wallet low balance]
        AD4[🔔 Aliado with suspicious behavior]
    end

    subgraph "🎯 System Triggers"
        ORDER_TAKEN[Order taken]
        ORDER_COMPLETED[Order completed]
        ORDER_EXPIRED[Order expired]
        SYSTEM_MONITOR[System monitor]
    end

    %% Connections from triggers to notifications
    ORDER_TAKEN --> U1
    ORDER_COMPLETED --> U2
    ORDER_COMPLETED --> A3
    ORDER_EXPIRED --> U3
    ORDER_EXPIRED --> A4
    
    SYSTEM_MONITOR --> AD1
    SYSTEM_MONITOR --> AD2
    SYSTEM_MONITOR --> AD3
    SYSTEM_MONITOR --> AD4
    
    classDef user fill:#e3f2fd
    classDef ally fill:#e8f5e8
    classDef admin fill:#ffebee
    classDef trigger fill:#fff3e0
    
    class U1,U2,U3,U4 user
    class A1,A2,A3,A4 ally
    class AD1,AD2,AD3,AD4 admin
    class ORDER_TAKEN,ORDER_COMPLETED,ORDER_EXPIRED,SYSTEM_MONITOR trigger
```

### 📱 **Notification Implementation**

```mermaid
sequenceDiagram
    participant DB as 🗄️ Database
    participant RT as ⚡ Supabase Realtime
    participant F as 🖥️ Frontend
    participant N as 📱 Push Notifications

    Note over DB,N: Real-Time Notification System

    DB->>RT: ORDER_STATUS_CHANGED event
    RT->>F: Realtime subscription update
    F->>F: Update UI state immediately
    
    alt User online
        F->>F: Show in-app notification
        F->>N: Send push notification
    else User offline
        F->>N: Queue push notification
        N->>Device: Deliver when online
    end
    
    Note over F: UI always synchronized
    Note over N: Push as backup/reminder
```

---

**🎯 Importance for the Team:**

- **Frontend**: Knows exactly which states to show and when
- **Backend**: Clear automatic transition logic
- **QA**: Defined test cases for each transition
- **Product**: Optimized timeouts for better UX
- **DevOps**: Self-managed system requiring minimal intervention

**🔑 Key Decisions:**
- **Aggressive timeouts**: 3-5-5 minutes for fast UX
- **MVP auto-approval**: No manual verification to simplify
- **Automatic penalties**: Maintain quality without intervention
- **Immediate refunds**: Users never lose money due to timeouts
