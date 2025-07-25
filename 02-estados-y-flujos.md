# 🔄 Kibo - Estados y Flujos de Proceso

## Estados de la Orden (State Machine)

```mermaid
stateDiagram-v2
    [*] --> PENDING_PAYMENT : Usuario envía QR + monto BOB
    
    PENDING_PAYMENT --> AVAILABLE : USDT recibido en escrow
    PENDING_PAYMENT --> EXPIRED : Timeout 3 minutos
    
    AVAILABLE --> TAKEN : Aliado toma la orden
    AVAILABLE --> EXPIRED : Timeout 5 minutos
    
    TAKEN --> COMPLETED : Aliado sube comprobante
    TAKEN --> EXPIRED : Timeout 5 minutos
    
    EXPIRED --> REFUNDED : Sistema devuelve USDT automáticamente
    COMPLETED --> [*] : USDT liberado al aliado
    REFUNDED --> [*] : Proceso terminado

    note right of PENDING_PAYMENT
        ⏱️ Cotización fija por 3 minutos
        💳 Usuario debe confirmar pago USDT
        🚫 Sin opción de cancelación manual
    end note

    note right of AVAILABLE
        🏦 USDT custodiado en escrow
        👁️ Visible para todos los aliados
        ⚡ Timeout para mantener velocidad UX
    end note

    note right of TAKEN
        🔒 Aliado asignado exclusivamente
        💰 Debe pagar QR y subir comprobante
        ⏰ Presión de tiempo para UX rápida
    end note

    note right of COMPLETED
        🚀 MVP: Comprobante = aprobación automática
        🚫 Sin verificación admin por ahora
        💸 Fondos liberados inmediatamente
    end note
```

## Descripción Detallada de Estados

### 🟡 **PENDING_PAYMENT** 
```
🎯 Qué significa: Orden creada, esperando pago USDT del usuario
⏱️ Duración: 3 minutos
🔄 Puede ir a: AVAILABLE, EXPIRED
✨ Trigger: Usuario envía QR + monto BOB

Características:
• Cotización USDT/BOB fija durante 3 minutos
• Usuario ve countdown en tiempo real
• No se puede cancelar manualmente
• Si no paga → orden se elimina (EXPIRED)
```

### 🔵 **AVAILABLE**
```
🎯 Qué significa: USDT recibido, orden disponible para aliados
⏱️ Duración: 5 minutos
🔄 Puede ir a: TAKEN, EXPIRED  
✨ Trigger: Confirmación de pago USDT en blockchain

Características:
• Fondos bloqueados en escrow
• Visible en dashboard de aliados
• Solo aliados del mismo país pueden verla
• Auto-refresh cada 10 segundos
• Si nadie toma → refund automático
```

### 🟣 **TAKEN**
```
🎯 Qué significa: Aliado asignado, debe procesar pago
⏱️ Duración: 5 minutos
🔄 Puede ir a: COMPLETED, EXPIRED
✨ Trigger: Aliado acepta orden disponible

Características:
• Orden bloqueada exclusivamente para un aliado
• QR mostrado solo al aliado asignado
• Debe pagar en app bancaria y subir comprobante
• Countdown visible para presión de tiempo
• Si no completa → penalización + refund
```

### 🟢 **COMPLETED**
```
🎯 Qué significa: Proceso exitoso, USDT liberado al aliado
⏱️ Duración: Estado final
🔄 Puede ir a: [Final]
✨ Trigger: Aliado sube comprobante (auto-aprobado en MVP)

Características:
• Comprobante subido = aprobación automática
• USDT transferido inmediatamente al aliado
• Notificaciones enviadas a usuario y aliado
• Estadísticas del aliado actualizadas
• Logs completos registrados
```

### 🔴 **EXPIRED**
```
🎯 Qué significa: Timeout en cualquier etapa del proceso
⏱️ Duración: Inmediato → REFUNDED
🔄 Puede ir a: REFUNDED
✨ Trigger: Cualquier timeout configurado

Características:
• Puede ocurrir en PENDING_PAYMENT, AVAILABLE, o TAKEN
• Si hay fondos en escrow → activa refund automático
• Aliado recibe penalización si estaba en TAKEN
• Usuario notificado del problema
```

### ⚪ **REFUNDED**
```
🎯 Qué significa: USDT devuelto al usuario original
⏱️ Duración: Estado final
🔄 Puede ir a: [Final]
✨ Trigger: Cualquier expiración con fondos en escrow

Características:
• USDT devuelto automáticamente al wallet original
• Hash de transacción registrado
• Usuario notificado del reembolso
• Orden marcada como cerrada
• No impacta reputación del usuario
```

## Flujo de Usuario (Crear y Pagar Orden)

```mermaid
sequenceDiagram
    participant U as 👤 Usuario
    participant F as 🖥️ Frontend
    participant API as ⚙️ Next.js API
    participant P as 💱 Pricing Service
    participant E as 🏦 Escrow Service
    participant DB as 🗄️ Supabase
    participant W as 🔗 Privy Wallet
    participant BC as ⛓️ Polygon

    Note over U,BC: FLUJO USUARIO: Crear y Pagar Orden

    U->>F: Escanea QR + ingresa 200 BOB
    F->>API: POST /api/quote
    Note right of API: {qrData, amountBOB: 200}
    
    API->>P: Calcular equivalente USDT
    P->>DB: SELECT rate FROM quotes WHERE active=true
    DB-->>P: 1 USDT = 6.97 BOB
    P-->>API: 28.69 USDT + 0.1 fee = 28.79 USDT
    
    API->>DB: INSERT INTO orders (status='PENDING_PAYMENT')
    Note right of DB: expires_at = NOW() + 3 minutes
    DB-->>API: Order ID: #025
    
    API-->>F: {orderId: "025", amountUSDT: 28.79, expires: "3:00"}
    F-->>U: "Paga 28.79 USDT en 3:00 minutos"
    
    Note over F: Countdown timer iniciado
    U->>F: Clic "Confirmar Pago"
    F->>W: requestTransaction(28.79 USDT)
    W->>U: "Aprobar transacción en wallet"
    U->>W: ✅ Aprobar
    W->>BC: Transfer 28.79 USDT → escrow_wallet
    
    BC->>E: Webhook: tx_hash confirmed
    E->>API: POST /api/orders/025/payment-confirmed
    API->>DB: UPDATE orders SET status='AVAILABLE'
    DB->>F: Realtime: order_status_changed
    F-->>U: "✅ Pago confirmado. Buscando aliado..."
    
    Note over U: Usuario puede cerrar app
    Note over U: Recibirá notificación cuando se complete
```

## Flujo de Aliado (Tomar y Procesar Orden)

```mermaid
sequenceDiagram
    participant AL as 🤝 Aliado
    participant F as 🖥️ Frontend
    participant API as ⚙️ Next.js API
    participant DB as 🗄️ Supabase
    participant BANK as 🏦 App Bancaria
    participant STORAGE as 📁 Supabase Storage

    Note over AL,STORAGE: FLUJO ALIADO: Procesar Orden

    AL->>F: Abre dashboard aliados
    F->>API: GET /api/orders/available
    API->>DB: SELECT * FROM available_orders_view
    DB-->>API: [Order #025: 200 BOB → 28.69 USDT, 4:15 restantes]
    API-->>F: Lista órdenes con timeouts
    F-->>AL: "💰 200 BOB → 28.69 USDT (⏰ 4:15)"
    
    Note over AL: Ve ganancia potencial y tiempo
    AL->>F: Clic "TOMAR ORDEN #025"
    F->>API: POST /api/orders/025/take
    
    API->>DB: UPDATE orders SET ally_id=AL, status='TAKEN'
    Note right of DB: expires_at = NOW() + 5 minutes
    API->>DB: INSERT INTO logs (action='ORDER_TAKEN')
    API-->>F: {qrData, expiresIn: "5:00", orderDetails}
    F-->>AL: "QR para pagar + countdown 5:00"
    
    Note over AL: Ve QR grande + datos bancarios
    AL->>BANK: Abre app bancaria (fuera de Kibo)
    AL->>BANK: Paga QR: 200 BOB a cuenta destino
    BANK-->>AL: "✅ Transferencia exitosa"
    
    AL->>F: Regresa a Kibo
    AL->>F: Clic "Subir Comprobante"
    F->>AL: Abre cámara/galería
    AL->>F: Selecciona foto comprobante
    
    F->>API: POST /api/orders/025/proof + FormData(image)
    API->>STORAGE: Upload imagen → /proofs/025.jpg
    STORAGE-->>API: URL: https://storage.../proofs/025.jpg
    API->>DB: UPDATE orders SET proof_url, status='COMPLETED'
    Note right of API: MVP: Auto-aprobación sin verificación
    
    API->>E: Trigger: liberar fondos al aliado
    E->>BC: Transfer 28.69 USDT → ally_wallet
    API->>DB: UPDATE escrow_accounts SET status='RELEASED'
    
    DB->>F: Realtime: order_completed
    F-->>AL: "✅ Completado! 28.69 USDT enviado"
    
    Note over AL: Puede tomar nueva orden inmediatamente
```

## Flujo de Sistema (Manejo de Timeouts Automático)

```mermaid
sequenceDiagram
    participant CRON as ⏰ Vercel Cron
    participant API as ⚙️ Next.js API
    participant DB as 🗄️ Supabase
    participant E as 🏦 Escrow Service
    participant N as 🔔 Notification
    participant U as 👤 Usuario
    participant AL as 🤝 Aliado

    Note over CRON,AL: FLUJO SISTEMA: Timeouts Automáticos

    loop Cada 30 segundos
        CRON->>API: POST /api/cron/check-timeouts
        API->>DB: SELECT * FROM orders WHERE expires_at < NOW()
        
        alt Orden #026 PENDING_PAYMENT expirada (3+ min)
            DB-->>API: Order #026, status='PENDING_PAYMENT'
            API->>DB: UPDATE orders SET status='EXPIRED'
            API->>N: Send notification
            N-->>U: "Tu orden #026 expiró. Intenta de nuevo."
            Note right of API: Sin refund - no hay fondos
        
        else Orden #027 AVAILABLE expirada (5+ min)
            DB-->>API: Order #027, status='AVAILABLE' 
            API->>DB: UPDATE orders SET status='EXPIRED'
            API->>E: processRefund(order #027)
            E->>BC: Transfer USDT back to user
            E->>DB: INSERT INTO refund_jobs (status='COMPLETED')
            API->>N: Send notification
            N-->>U: "Orden #027 expiró. USDT devuelto."
        
        else Orden #028 TAKEN expirada (5+ min)
            DB-->>API: Order #028, status='TAKEN', ally_id='AL123'
            API->>DB: UPDATE orders SET status='EXPIRED'
            API->>DB: INSERT INTO ally_penalties (ally_id='AL123', reason='TIMEOUT')
            API->>E: processRefund(order #028)
            E->>BC: Transfer USDT back to user
            API->>N: Send notifications
            N-->>U: "Orden #028 expiró. USDT devuelto."
            N-->>AL: "⚠️ Orden #028 expiró. Penalizado 30 min."
            
            Note right of API: Aliado bloqueado por 30 minutos
        end
    end

    Note over CRON: Sistema 100% automático
    Note over CRON: Corre en Vercel Cron Jobs
    Note over CRON: Sin intervención manual requerida
```

## Timeouts y Reglas de Negocio

### ⏰ **Timeouts por Estado**

```mermaid
graph TB
    subgraph "⏰ Timeouts Configurados"
        A[PENDING_PAYMENT<br/>3 minutos] -->|Timeout| B[EXPIRED<br/>Eliminar orden]
        C[AVAILABLE<br/>5 minutos] -->|Timeout| D[EXPIRED<br/>Refund automático]
        E[TAKEN<br/>5 minutos] -->|Timeout| F[EXPIRED<br/>Refund + penalizar]
    end
    
    subgraph "📊 Línea de Tiempo Exitosa"
        T0[00:00 Usuario escanea QR] --> T1[00:30 Ve cotización]
        T1 --> T2[02:00 Confirma pago]
        T2 --> T3[03:30 AVAILABLE]
        T3 --> T4[04:00 Aliado toma]
        T4 --> T5[07:00 Aliado paga banco]
        T5 --> T6[08:00 Sube comprobante]
        T6 --> T7[08:30 COMPLETED]
    end
    
    style B fill:#ffcdd2
    style D fill:#ffcdd2
    style F fill:#ffcdd2
    style T7 fill:#c8e6c9
```

### 🔒 **Reglas de Bloqueo**

| Regla | Descripción | Implementación |
|-------|-------------|----------------|
| **Un aliado, una orden** | Aliado solo puede tener 1 orden TAKEN a la vez | `WHERE ally_id IS NULL AND user NOT IN (SELECT ally_id FROM orders WHERE status='TAKEN')` |
| **Penalización por timeout** | Aliado que deja expirar se bloquea 30 min | `INSERT INTO ally_penalties (penalty_until = NOW() + INTERVAL '30 minutes')` |
| **No re-tomar orden expirada** | Mismo aliado no puede retomar orden que dejó expirar | `WHERE order_id NOT IN (SELECT order_id FROM ally_penalties WHERE ally_id = ?)` |
| **País específico** | Aliados solo ven órdenes de su país | `WHERE user.country = ally.country` |
| **Cotización única** | Una cotización por orden, fija hasta completar | `UPDATE quotes SET is_active=false WHERE order_id = ?` |

### 🚨 **Manejo de Errores Automático**

```mermaid
flowchart TD
    ERROR{Error Detectado} --> |Pago blockchain falla| REFUND_USER[Refund Inmediato]
    ERROR --> |Aliado no responde| PENALTY[Penalizar + Refund]
    ERROR --> |Sistema no disponible| QUEUE[Queue para retry]
    ERROR --> |Imagen muy grande| REJECT[Rechazar upload]
    
    REFUND_USER --> LOG[Registrar en logs]
    PENALTY --> LOG
    QUEUE --> LOG
    REJECT --> USER_NOTIFY[Notificar usuario]
    
    LOG --> ADMIN_ALERT[Alerta admin si crítico]
    USER_NOTIFY --> LOG
    
    style ERROR fill:#ffcdd2
    style REFUND_USER fill:#c8e6c9
    style PENALTY fill:#ffecb3
    style LOG fill:#e1f5fe
```

### 📊 **Métricas de Performance**

| Métrica | Target MVP | Medición |
|---------|------------|----------|
| **Tiempo promedio de orden** | < 10 minutos | `AVG(completed_at - created_at)` |
| **Tasa de éxito** | > 80% | `COMPLETED / TOTAL * 100` |
| **Tasa de timeout** | < 15% | `EXPIRED / TOTAL * 100` |
| **Aliados activos diarios** | > 5 aliados | `COUNT(DISTINCT ally_id) WHERE DATE(taken_at) = TODAY` |
| **Volumen diario** | > 1000 BOB | `SUM(amount_fiat) WHERE DATE(created_at) = TODAY` |

## Notificaciones en Tiempo Real

### 🔔 **Eventos de Notificación**

```mermaid
graph TB
    subgraph "👤 Usuario Recibe"
        U1[🔔 Aliado asignado a tu orden]
        U2[🔔 Pago completado exitosamente]
        U3[🔔 Orden expiró - USDT devuelto]
        U4[🔔 Error en el proceso]
    end
    
    subgraph "🤝 Aliado Recibe"
        A1[🔔 Nueva orden disponible en tu área]
        A2[🔔 Orden tomada exitosamente]
        A3[🔔 Comprobante procesado - USDT recibido]
        A4[🔔 Orden expiró - penalización aplicada]
    end
    
    subgraph "👨‍💼 Admin Recibe"
        AD1[🔔 Volumen alto de timeouts]
        AD2[🔔 API externa no responde]
        AD3[🔔 Escrow wallet saldo bajo]
        AD4[🔔 Aliado con comportamiento sospechoso]
    end

    subgraph "🎯 Triggers del Sistema"
        ORDER_TAKEN[Orden tomada]
        ORDER_COMPLETED[Orden completada]
        ORDER_EXPIRED[Orden expirada]
        SYSTEM_MONITOR[Monitor del sistema]
    end

    %% Conexiones de triggers a notificaciones
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

### 📱 **Implementación de Notificaciones**

```mermaid
sequenceDiagram
    participant DB as 🗄️ Database
    participant RT as ⚡ Supabase Realtime
    participant F as 🖥️ Frontend
    participant N as 📱 Push Notifications

    Note over DB,N: Sistema de Notificaciones en Tiempo Real

    DB->>RT: ORDER_STATUS_CHANGED event
    RT->>F: Realtime subscription update
    F->>F: Update UI state immediately
    
    alt Usuario online
        F->>F: Show in-app notification
        F->>N: Send push notification
    else Usuario offline
        F->>N: Queue push notification
        N->>Device: Deliver when online
    end
    
    Note over F: UI siempre sincronizada
    Note over N: Push como backup/reminder
```

---

**🎯 Importancia para el Equipo:**

- **Frontend**: Sabe exactamente qué estados mostrar y cuándo
- **Backend**: Lógica de transiciones automáticas clara
- **QA**: Casos de prueba definidos para cada transición
- **Product**: Timeouts optimizados para mejor UX
- **DevOps**: Sistema auto-gestionado que requiere mínima intervención

**🔑 Decisiones Clave:**
- **Timeouts agresivos**: 3-5-5 minutos para UX rápida
- **Auto-aprobación MVP**: Sin verificación manual para simplificar
- **Penalizaciones automáticas**: Mantienen calidad sin intervención
- **Refunds inmediatos**: Usuarios nunca pierden dinero por timeouts
