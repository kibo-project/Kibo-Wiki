# 📋 Kibo - Use Cases and User Stories

## Main MVP Epics

```mermaid
mindmap
  root((🚀 Kibo MVP))
    🔐 Authentication
      Connect Wallet
      Detect User Role
      Maintain Session
      Disconnect Wallet
    
    👤 User Payments
      Scan Bank QR
      View Real-time Quote
      Pay USDT via Privy
      Track Order Status
      View Order History
      
    🤝 Ally Processes
      View Available Orders
      Take Exclusive Order
      Pay QR in Banking App
      Upload Payment Receipt
      Receive USDT Automatically
      View Earnings Statistics
      
    ⚙️ Automatic System
      Calculate Quotes
      Handle Timeouts
      Process Refunds
      Release Funds
      Penalize Allies
      Audit Logs
      
    👨‍💼 Admin Monitor
      Real-time Dashboard
      Manage Configuration
      Review System Logs
      Administer Users
      Automatic Alerts
```

## 🔐 EPIC 1: Authentication and Session Management

### **US001: Connect Wallet as User**
```gherkin
As a new user
I want to connect my crypto wallet
So I can make payments with my cryptocurrencies

GIVEN I am on the Kibo landing page
WHEN I click "Connect Wallet"
THEN Privy opens to select my wallet
AND I can connect MetaMask, WalletConnect, Coinbase Wallet
AND the system automatically detects my wallet address
AND my profile is created with role="user" in the database
AND I am redirected to the User Dashboard
AND I see my updated USDT balance

Error Flows:
❌ IF I reject the connection → Return to landing with message
❌ IF my wallet has no USDT → Warning but I can continue
❌ IF there's a network error → Message "Try again" with retry

Technical Criteria:
✅ Use Privy SDK for authentication
✅ Save wallet_address in users table
✅ Generate JWT token for session
✅ Verify USDT balance on Mantle
✅ Handle states: loading, success, error

Estimation: 2 days
Priority: Critical
Dependencies: Privy + Supabase configuration
```

### **US002: Connect Wallet as Ally**
```gherkin
As a person interested in being an ally
I want to register as a Kibo ally
So I can earn USDT by processing payments from other users

GIVEN I am on the landing page
WHEN I click "Become an Ally"
THEN I connect my wallet the same as a normal user
AND I fill out an additional form:
  - Country of operation (Bolivia for now)
  - Banks I handle (Banco Unión, BNB, etc.)
  - Accept specific ally terms
AND my role is marked as role="ally" in the database
AND I am redirected to the Ally Dashboard
AND I see available orders immediately

Specific Validations:
✅ Wallet cannot already be registered with another role
✅ Country must be in supported countries list
✅ Must have minimum 1 USDT to verify functional wallet
✅ Form must validate banking data

Estimation: 3 days
Priority: Critical
Dependencies: US001 completed
```

### **US003: Maintain Active Session**
```gherkin
As an authenticated user (user or ally)
I want my session to remain active
So I don't have to reconnect constantly

GIVEN I have already connected my wallet
WHEN I close and reopen the application
OR when I navigate between pages
THEN my session remains active automatically
AND I don't need to reconnect wallet
AND my data loads automatically
AND my role is detected correctly

Special Cases:
✅ Session expires after 7 days of inactivity
✅ If wallet address changes → force re-authentication
✅ If JWT token error → automatic logout

Estimation: 1 day
Priority: High
```

## 👤 EPIC 2: User - Make Crypto-to-Fiat Payments

### **US004: Scan Bank QR to Pay (Simplified MVP)**
```gherkin
As a user with USDT
I want to scan a bank payment QR
So I can pay it using my cryptocurrencies

GIVEN I am on the User Dashboard
WHEN I click "Scan QR"
THEN the device's native camera opens
AND I can scan any QR (MVP: no automatic validation)
AND the system saves the QR image to show to the ally
AND it allows me to manually enter:
  - Amount in BOB
  - Basic banking information (optional for MVP)
AND validates that the amount is between 10-10,000 BOB
AND automatically proceeds to calculate quote

Edge Cases Handled:
❌ Camera not available → Option to upload image from gallery
❌ Amount out of range → "Amount must be between 10-10,000 BOB"
❌ Image too large → Automatic compression

MVP Technical Criteria:
✅ Use browser's native camera API
✅ Save QR image without parsing content
✅ Client-side and server-side amount validation
✅ Fallback to manual upload if camera fails
❌ Automatic QR parsing (post-MVP)
❌ Specific validation of Bolivian banks (post-MVP)

Estimation: 2 days
Priority: Critical
Dependencies: None (simplified MVP)
```

### **US005: View Quote and Confirm Payment**
```gherkin
As a user who scanned a valid QR
I want to see the exact quote before paying
So I can decide whether to proceed with the transaction

GIVEN I scanned a valid QR and entered BOB amount
WHEN the system calculates the quote
THEN I see clearly on screen:
  - Amount in BOB that I want to pay
  - Current USDT/BOB quote and source
  - Exact amount in USDT that I will pay
  - Estimated network fee in USDT
  - Exact final total
  - 3:00 countdown timer to decide
AND I can confirm or cancel the transaction
AND if I don't decide in 3 minutes, the quote updates automatically

Real-time Validations:
✅ Verify that I have sufficient USDT in wallet
✅ Confirm that quote is not older than 30 seconds
✅ Validate that Mantle network is operational
✅ Show warning if balance insufficient

Possible Flows:
✅ Confirm → Proceed to payment flow
✅ Cancel → Return to dashboard without creating order
✅ Timeout → Recalculate quote automatically
✅ Insufficient funds → Show error + suggest getting more USDT

Estimation: 3 days
Priority: Critical
Dependencies: US004 + quote API integration
```

### **US006: Pay USDT to Centralized Escrow**
```gherkin
As a user who confirmed the quote
I want to transfer my USDT securely
So that an ally can process my fiat payment

GIVEN I confirmed the displayed quote
WHEN I click "Confirm and Pay"
THEN Privy opens my connected wallet
AND I see the exact transaction details:
  - Recipient: Kibo's escrow address
  - Amount: exact calculated USDT amount
  - Network: Mantle
  - Estimated gas fee
AND I confirm the transaction in my wallet
AND the system detects the payment on blockchain
AND my order is marked as status="AVAILABLE"
AND I see "Looking for available ally..." with spinner

Clear Loading States:
🔄 "Confirming transaction..." (until confirmed on blockchain)
🔄 "Transaction pending..." (waiting for confirmations)
🔄 "Looking for ally..." (until someone takes the order)

Handled Errors:
❌ Transaction rejected by user → Return to quote
❌ Insufficient gas fee → "You need more ETH for gas"
❌ Network congestion → "Network congested, retrying..."
❌ Transaction failed → "Transaction error, try again"

Technical Criteria:
✅ Integrate with Privy for transaction handling
✅ Listen to blockchain events for confirmation
✅ Create record in orders table with status="PENDING_PAYMENT"
✅ Update to "AVAILABLE" when payment is confirmed
✅ Configure webhook or polling to detect transactions

Estimation: 5 days
Priority: Critical
Dependencies: US005 + escrow wallet configuration
```

### **US007: Track My Order Status in Real Time**
```gherkin
As a user who has already paid their order
I want to see the progress of my payment in real time
So I know when it will be completed

DADO que ya transferí USDT al escrow
CUANDO estoy en la pantalla de seguimiento de orden
ENTONCES veo actualizaciones automáticas instantáneas via Supabase Realtime:
  - ⏳ "Buscando aliado disponible..." (status=AVAILABLE)
  - 👤 "Aliado Juan (0x56...78) asignado" (status=TAKEN) - actualización < 1 segundo  
  - 💳 "Aliado procesando pago bancario..." (status=TAKEN)
  - ✅ "¡Pago completado exitosamente!" (status=COMPLETED) - actualización instantánea
Y veo countdown de timeouts calculados en tiempo real
Y recibo notificaciones push automáticas en momentos clave
Y puedo ver detalles expandidos de la orden sin recargar

Información Mostrada por Estado:
📋 AVAILABLE: "Buscando aliado... (⏱️ expira en 4:30)"
🤝 TAKEN: "Aliado procesando... (⏱️ debe completar en 3:15)"
✅ COMPLETED: "Pago completado - aliado recibió USDT"
❌ EXPIRED: "Orden expiró - USDT devuelto a tu wallet"

Notificaciones Push:
🔔 Aliado asignado a tu orden
🔔 Pago completado exitosamente  
🔔 Orden expirada - USDT reembolsado

Criterios Técnicos MVP:
✅ Implementar Supabase Realtime subscription para orders
✅ Initial load con GET /api/orders/:id una sola vez
✅ Push automático de cambios de estado sin polling
✅ Cálculo de countdown en frontend con expires_at
✅ Notificaciones browser nativas para eventos críticos

Estimación: 2 días (reducida gracias a Supabase Realtime)
Prioridad: Alta
Dependencias: US006 + Supabase configurado
```

### **US008: Ver Historial de Mis Órdenes**
```gherkin
Como usuario que ha hecho varios pagos
Quiero ver el historial completo de mis órdenes
Para revisar transacciones pasadas y gestionar activas

DADO que estoy logueado en User Dashboard
CUANDO navego a "Mis Órdenes"
ENTONCES veo lista paginada de todas mis órdenes:
  - Estado actual con íconos claros
  - Monto en BOB y USDT pagado
  - Fecha y hora de creación
  - Duración total del proceso
  - Aliado que la procesó (si completada)
Y puedo filtrar por estado: Todas, Activas, Completadas, Expiradas
Y puedo ver detalles completos de cada orden
Y puedo exportar mi historial en CSV

Detalles por Orden:
📋 ID orden, timestamps, montos, quote usado
🤝 Info del aliado (si asignado)
⛓️ Hashes de transacciones blockchain
📄 Comprobante de pago (si completada)

Estimación: 2 días
Prioridad: Media
Dependencias: US007 completado
```

## 🤝 EPIC 3: Ally - Process Orders and Earn USDT

### **US009: View Available Orders to Process**
```gherkin
As a registered ally
I want to see the orders I can process
So I can choose which ones to take based on my capacity

GIVEN I am logged in as an ally
WHEN I access my dashboard
THEN I see "Available Orders" section that updates every 10s
AND I see for each available order:
  - Amount in BOB to pay
  - My exact profit in USDT
  - Time remaining before expiration
  - Destination bank
  - Prominent "TAKE" button
AND I see only orders I can process:
  - From my country (Bolivia)
  - That are not taken by another ally
  - Only if I don't have an active order
  - Only if I'm not penalized
AND orders are sorted by urgency (closest to expiring first)

Automatic Filters:
✅ Only orders with status="AVAILABLE"
✅ Only from my country="BO"
✅ Solo si mi ally_id no está en otra orden TAKEN
✅ Solo si no tengo penalizaciones activas
✅ Excluir órdenes que dejé expirar antes

Estados Especiales:
📭 Si no hay órdenes: "No hay órdenes disponibles - te notificaremos"
🚫 Si estoy penalizado: "Penalizado hasta 15:30 - no puedes tomar órdenes"
⚡ Si tengo orden activa: "Completa tu orden actual antes de tomar otra"

Estimación: 3 días
Prioridad: Crítica
Dependencias: US001, US002 + tabla ally_penalties
```

### **US010: Tomar Orden Disponible Exclusivamente**
```gherkin
Como aliado viendo órdenes disponibles
Quiero tomar una orden específica
Para ganar USDT procesándola exclusivamente

DADO que veo una orden disponible
CUANDO hago clic en "TOMAR ORDEN"
ENTONCES la orden se asigna exclusivamente a mí (ally_id)
Y el status cambia de "AVAILABLE" a "TAKEN"
Y otros aliados ya no pueden verla ni tomarla
Y soy redirigido a pantalla de procesamiento
Y veo todos los detalles para procesar el pago:
  - QR code grande y claro para escanear
  - Datos bancarios para copiar manualmente
  - Monto exacto a pagar en BOB
  - Referencia única para la transferencia
  - Countdown timer 5:00 para completar
Y la orden expires_at se actualiza a NOW() + 5 minutos

Validaciones al Tomar:
❌ Si alguien más la tomó 1 segundo antes → "Orden ya tomada por otro aliado"
❌ Si estoy penalizado → "No puedes tomar órdenes mientras estés penalizado"
❌ Si ya tengo orden activa → "Completa tu orden actual primero"
❌ Si la orden expiró → "Esta orden ya expiró"

Información Mostrada:
📱 QR code optimizado para scan móvil
🏦 Banco: Unión | Cuenta: 123456789 | Referencia: KBO025
💰 Monto exacto: 300.00 BOB
⏰ Tiempo límite: 5:00 countdown
🎯 Mi ganancia: 43.06 USDT

Estimación: 4 días
Prioridad: Crítica
Dependencias: US009 + manejo concurrencia base de datos
```

### **US011: Pagar QR en App Bancaria Externa**
```gherkin
Como aliado que tomó una orden
Quiero instrucciones claras para pagar el QR
Para completar el proceso eficientemente

DADO que tomé una orden y estoy en pantalla de procesamiento
CUANDO veo los detalles del pago a realizar
ENTONCES tengo opciones claras para procesar:
  - QR code grande para escanear con mi app bancaria
  - Datos bancarios para copiar/pegar manualmente
  - Botón "Copiar datos" que copia al clipboard
  - Instrucciones paso a paso claras
  - Botón "Abrir app bancaria" (si es posible)
Y puedo alternar entre vista QR y vista datos textuales
Y veo countdown prominente del tiempo restante
Y hay botón "Ya pagué - Subir comprobante" para siguiente paso

Flujo Esperado del Aliado:
1. Ver QR/datos en Kibo
2. Abrir app bancaria (externa a Kibo)  
3. Escanear QR o ingresar datos manualmente
4. Realizar transferencia bancaria
5. Recibir confirmación del banco
6. Regresar a Kibo
7. Subir foto del comprobante

Ayudas UX:
📱 QR optimizado para cámaras móviles
📋 "Copiar datos" funciona en todos los browsers
🔄 Botón refresh por si QR no carga
⏰ Warning cuando quedan < 1 minuto
📞 Enlace a soporte si hay problemas

Estimación: 2 días
Prioridad: Alta
Dependencias: US010 + investigación deep linking apps bancarias
```

### **US012: Subir Comprobante de Pago**
```gherkin
Como aliado que ya pagó el QR bancario
Quiero subir el comprobante de mi transferencia
Para recibir mis USDT automáticamente

DADO que ya realicé el pago bancario
CUANDO regreso a Kibo y hago clic en "Subir Comprobante"
ENTONCES puedo:
  - Tomar foto directamente con cámara
  - Seleccionar imagen existente de galería
  - Ver preview de la imagen antes de enviar
  - Confirmar que se lee claramente:
    * Monto transferido (debe coincidir)
    * Fecha y hora del pago
    * Banco y cuenta destino
    * Referencia de la transferencia
Y el sistema valida la imagen:
  - Tamaño < 5MB
  - Formato JPG o PNG
  - No está corrupta
Y al enviar, el comprobante se sube a Supabase Storage
Y automáticamente la orden se marca como status="COMPLETED"
Y recibo confirmación: "USDT enviado a tu wallet"

Validaciones de Imagen:
✅ Tamaño máximo 5MB
✅ Formatos permitidos: JPG, PNG, HEIC
✅ Imagen no puede estar completamente negra/blanca
✅ Debe tener dimensiones mínimas (ej: 200x200px)

MVP: Auto-aprobación
🤖 Comprobante subido = Aprobación automática inmediata
🚫 Sin verificación manual por ahora
📝 Logs completos para auditoría futura

Post-upload:
✅ USDT transferido inmediatamente al wallet del aliado
✅ Estadísticas del aliado actualizadas (+1 successful_order)
✅ Notificaciones enviadas a usuario y aliado
✅ Orden marcada como completada con timestamp

Estimación: 3 días
Prioridad: Crítica
Dependencias: US011 + Supabase Storage + auto-aprobación logic
```

### **US013: Ver Mis Estadísticas y Ganancias**
```gherkin
Como aliado que ha procesado órdenes
Quiero ver mis estadísticas de rendimiento
Para monitorear mis ganancias y mejorar mi servicio

DADO que soy aliado con órdenes procesadas
CUANDO navego a mi sección "Estadísticas"
ENTONCES veo dashboard con:

Métricas del Día:
- Órdenes completadas hoy
- USDT ganado hoy  
- Tiempo promedio de procesamiento
- Órdenes disponibles vs tomadas

Métricas Históricas:
- Total órdenes completadas
- Total USDT ganado
- Tasa de éxito (completadas vs expiradas)
- Tiempo promedio histórico
- Tendencia de ganancias (gráfico semanal)

Estado Actual:
- ¿Tengo penalizaciones activas?
- ¿Tengo orden activa pendiente?
- Mi reputación (futuro)
- Ranking vs otros aliados (futuro)

Detalles Útiles:
📊 Gráfico de ganancias por día/semana
⏰ Análisis de mejores horarios para estar activo
🏆 Badges por hitos (ej: "100 órdenes completadas")
💡 Tips para mejorar velocidad de procesamiento

Estimación: 2 días
Prioridad: Media
Dependencias: US012 + ally_dashboard_view en BD
```

## ⚙️ ÉPICA 4: Sistema Automático - Backbone del MVP

### **US014: Calcular y Actualizar Cotizaciones Automáticamente**
```gherkin
Como sistema de Kibo
Quiero mantener cotizaciones USDT/BOB actualizadas
Para ofrecer precios justos y competitivos a usuarios

DADO que el sistema está operando
CUANDO pasan 30 segundos desde la última actualización
ENTONCES ejecuto job automático que:
  - Consulta binance_api/coingecko API para precio USDT/USD actual
  - Consulta fuente confiable para tasa USD/BOB
  - Calcula tasa USDT/BOB resultante
  - Guarda nuevo quote en tabla quotes
  - Marca quote anterior como is_active=false
Y las nuevas órdenes usan automáticamente el quote más reciente
Y las órdenes en progreso mantienen su quote fijo original

Manejo de Errores:
❌ API externa no responde → Usar último quote válido + log warning
❌ Cambio > 5% vs precio anterior → Log alerta para admin review
❌ No hay quote en últimos 10 min → Bloquear creación nuevas órdenes

Fuentes de Datos:
🔧 Primaria: binance_api/coingecko API (gratis, confiable)
🔧 Fallback: CoinMarketCap API
🔧 Tasa BOB: Banco Central Bolivia o servicio financiero local

Implementación Técnica:
✅ Vercel Cron Job cada 30 segundos
✅ Next.js API route: /api/cron/update-quotes
✅ Retry automático 3 veces si falla
✅ Logs en tabla quotes para auditoría

Estimación: 3 días
Prioridad: Crítica
Dependencias: Investigación APIs quotes + configuración cron jobs
```

### **US015: Manejar Timeouts y Expiraciones Automáticamente**
```gherkin
Como sistema de Kibo
Quiero procesar órdenes expiradas automáticamente
Para mantener el flujo sin intervención manual

DADO que ejecuto job de limpieza cada 30 segundos
CUANDO encuentro órdenes con expires_at < NOW()
ENTONCES según el estado de cada orden:

PENDING_PAYMENT expirada (3+ minutos):
- Cambiar status a "EXPIRED"
- Eliminar orden (no hay fondos que devolver)
- Log: "Order {id} expired in PENDING_PAYMENT"
- Notificar usuario: "Tu orden expiró sin pagar"

AVAILABLE expirada (5+ minutos):
- Cambiar status a "EXPIRED"  
- Crear refund_job para devolver USDT al usuario
- Ejecutar refund automático via wallet escrow
- Log: "Order {id} expired in AVAILABLE, refund processed"
- Notificar usuario: "Orden expiró, USDT devuelto"

TAKEN expirada (5+ minutos):
- Cambiar status a "EXPIRED"
- Crear ally_penalty para el aliado (30 min block)
- Crear refund_job para devolver USDT al usuario
- Log: "Order {id} expired in TAKEN, ally {id} penalized"
- Notificar usuario: "Orden expiró, USDT devuelto"  
- Notificar aliado: "Orden expiró, penalizado 30 minutos"

Todos los timeouts:
✅ Se registran en tabla logs con metadata completa
✅ Métricas agregadas para dashboard admin
✅ Alertas automáticas si > 30% órdenes expiran

Implementación:
✅ Vercel Cron Job: /api/cron/check-timeouts
✅ Función SQL optimizada para encontrar expiradas
✅ Transacciones atómicas para evitar race conditions
✅ Idempotencia para múltiples ejecuciones

Estimación: 4 días
Prioridad: Crítica
Dependencias: US014 + refund system + penalty system
```

### **US016: Procesar Refunds Automáticamente**
```gherkin
Como sistema de Kibo
Quiero devolver USDT automáticamente cuando algo falla
Para que usuarios nunca pierdan dinero por problemas del sistema

DADO que una orden expira con fondos en escrow (AVAILABLE o TAKEN)
CUANDO el sistema detecta la expiración
ENTONCES ejecuto proceso de refund automático:
  - Crear registro en tabla refund_jobs con status="PENDING"
  - Transferir USDT desde wallet escrow al wallet original del usuario
  - Esperar confirmación de transacción en blockchain
  - Marcar refund_job como status="COMPLETED"
  - Actualizar escrow_account como status="REFUNDED"
  - Registrar tx_hash de la transacción refund
  - Enviar notificación push al usuario
Y todo el proceso es completamente automático
Y se registra hash de transacción para verificación

Validaciones de Seguridad:
✅ Solo refund si hay fondos confirmados en escrow
✅ Solo una vez por orden (idempotencia)
✅ Verificar saldo suficiente en wallet escrow antes de transferir
✅ Timeout del refund job si no se procesa en 10 minutos

Estados del Refund Job:
📋 PENDING: Creado, esperando procesamiento
🔄 PROCESSING: Transacción enviada, esperando confirmación
✅ COMPLETED: USDT devuelto exitosamente  
❌ FAILED: Error en el proceso, requiere intervención manual

Manejo de Errores:
❌ Wallet escrow sin fondos → Alerta crítica admin
❌ Network down → Reintentar cada 5 minutos
❌ Transaction failed → Marcar como FAILED + alerta admin

Estimación: 3 días
Prioridad: Alta
Dependencias: US015 + configuración wallet escrow para envíos
```

### **US017: Liberar Fondos al Aliado Automáticamente**
```gherkin
Como sistema de Kibo
Quiero transferir USDT al aliado cuando complete correctamente
Para automatizar el pago sin intervención manual

DADO que un aliado sube comprobante válido
CUANDO la orden se marca como status="COMPLETED"
ENTONCES ejecuto liberación automática de fondos:
  - Transferir USDT desde wallet escrow al wallet del aliado
  - Marcar escrow_account como status="RELEASED"
  - Registrar tx_hash de la transacción de liberación
  - Incrementar successful_orders del aliado en +1
  - Actualizar last_active del aliado a NOW()
  - Enviar notificaciones push a aliado y usuario
  - Registrar en logs: "Funds released to ally"
Y el proceso es inmediato tras subir comprobante (MVP auto-approval)

Validaciones Antes de Liberar:
✅ Orden debe estar en status="TAKEN" → "COMPLETED"
✅ ally_id debe coincidir con quien tomó la orden
✅ Debe haber fondos suficientes en escrow
✅ proof_url debe existir y ser válida
✅ No debe haberse liberado antes (idempotencia)

MVP: Auto-liberación Inmediata
🤖 Comprobante subido = Liberación automática sin verificación
🚫 Sin review manual en esta etapa
📊 Métricas para detectar patrones sospechosos después

Post-liberación:
✅ Usuario notificado: "Pago completado exitosamente"
✅ Aliado notificado: "43.06 USDT enviado a tu wallet"
✅ Estadísticas actualizadas en tiempo real
✅ Aliado queda disponible para tomar nueva orden

Estimación: 3 días
Prioridad: Crítica
Dependencias: US012 + wallet escrow configurado para envíos
```

## 👨‍💼 ÉPICA 5: Admin - Monitoreo y Configuración

### **US018: Dashboard de Monitoreo en Tiempo Real**
```gherkin
Como administrador del sistema
Quiero ver el estado operativo de Kibo en tiempo real
Para detectar problemas y supervisar el rendimiento

DADO que accedo al admin panel con rol="admin"
CUANDO veo el dashboard principal
ENTONCES veo métricas actualizadas cada 30 segundos:

Métricas del Día Actual:
- Total órdenes creadas hoy
- Órdenes completadas vs expiradas  
- Volumen en BOB y USDT procesado
- Número de aliados únicos activos
- Tiempo promedio de procesamiento
- Tasa de éxito general

Estado del Sistema:
- Uptime de la aplicación
- Estado de APIs externas (binance_api/coingecko, etc.)
- Saldo del wallet escrow
- Órdenes en cada estado (gráfico en tiempo real)
- Alertas activas del sistema

Últimas Órdenes (Live Feed):
- 10 órdenes más recientes con estado actual
- Posibilidad de drill-down en cualquier orden
- Filtros por estado, usuario, aliado, fecha

Alertas Automáticas Configuradas:
🚨 > 30% órdenes expirando en la última hora
🚨 Aliado con 3+ timeouts seguidos  
🚨 API de cotizaciones sin responder > 5 minutos
🚨 Wallet escrow con saldo < 100 USDT
🚨 Sistema con > 10% error rate

Estimación: 4 días
Prioridad: Media
Dependencias: admin_metrics_view + alerting system
```

### **US019: Gestionar Configuración del Sistema Dinámicamente**
```gherkin
Como administrador
Quiero ajustar parámetros del sistema sin redeploy
Para optimizar la operación según datos reales

DADO que estoy en sección "Configuración" del admin panel
CUANDO edito parámetros operativos
ENTONCES puedo modificar en tiempo real:

Timeouts (en minutos):
- PENDING_PAYMENT_TIMEOUT: 3 → editable
- AVAILABLE_TIMEOUT: 5 → editable  
- TAKEN_TIMEOUT: 5 → editable
- ALLY_PENALTY_DURATION: 30 → editable

Límites de Órdenes:
- MIN_ORDER_AMOUNT_BOB: 10 → editable
- MAX_ORDER_AMOUNT_BOB: 10000 → editable
- MAX_DAILY_ORDERS_PER_USER: sin límite → configurable

Parámetros de Quotes:
- QUOTE_UPDATE_INTERVAL_SECONDS: 30 → editable
- PRICE_CHANGE_ALERT_THRESHOLD: 5% → editable
- QUOTE_SOURCE: binance_api/coingecko → seleccionable

Y todos los cambios aplican inmediatamente sin reiniciar
Y se registra quién cambió qué parámetro y cuándo
Y hay validaciones para evitar configuraciones problemáticas

Validaciones de Configuración:
✅ Timeouts mínimo 1 minuto, máximo 60 minutos
✅ Límites coherentes (min < max)
✅ Solo valores numéricos positivos donde corresponde
✅ Cambios críticos requieren confirmación adicional

Historial de Cambios:
📝 Log completo de quién cambió qué configuración
📅 Posibilidad de revertir a configuración anterior
⚠️ Alertas si configuración causa problemas operativos

Estimación: 2 días
Prioridad: Baja
Dependencias: system_config table + validation functions
```

### **US020: Administrar Usuarios y Penalizaciones**
```gherkin
Como administrador
Quiero gestionar usuarios problemáticos
Para mantener la calidad del servicio

DADO que estoy en sección "Gestión de Usuarios"
CUANDO reviso la lista de usuarios/aliados
ENTONCES puedo:

Ver Información Detallada:
- Lista de todos los usuarios con filtros por rol
- Estadísticas de cada aliado (éxito, timeouts, ganancias)
- Historial completo de órdenes por usuario
- Penalizaciones activas y históricas
- Actividad reciente y patrones de uso

Acciones sobre Aliados Problemáticos:
- Ver detalle de timeouts y razones
- Extender penalización manual (ej: de 30min a 2 horas)
- Bloquear aliado temporalmente (ej: 24 horas)
- Marcar aliado como "bajo revisión"
- Ver todas las órdenes que procesó con detalles

Casos de Uso Típicos:
🔍 Aliado con patrón de muchos timeouts
🔍 Usuario con comportamiento sospechoso (muchas órdenes canceladas)
🔍 Verificar legitimidad de comprobantes específicos
🔍 Investigar quejas o problemas reportados

Reportes Generables:
📊 Aliados más efectivos del mes
📊 Usuarios con mayor volumen
📊 Análisis de timeouts por horario/día
📊 Tendencias de crecimiento de usuarios

Limitaciones MVP:
🚫 No editar datos de usuarios directamente
🚫 No eliminar órdenes completadas
✅ Solo gestión de penalizaciones y monitoreo
✅ Logs completos de todas las acciones admin

Estimación: 3 días
Prioridad: Baja
Dependencias: US018 + ally_penalties management
```

## 🚨 ÉPICA 6: Manejo de Errores y Casos Edge

### **US021: Manejar Fallos de Red Blockchain Graciosamente**
```gherkin
Como sistema resiliente
Quiero manejar problemas de conectividad blockchain
Para que la experiencia de usuario sea robusta

DADO que hay problemas en mantle network
CUANDO un usuario intenta realizar acciones que requieren blockchain
ENTONCES el sistema responde apropiadamente:

Durante Creación de Orden:
- Detectar si mantle RPC está respondiendo
- Si red lenta (> 30s para confirmación): mostrar "Red congestionada, puede tomar más tiempo"
- Si red no disponible: mostrar "Red temporalmente no disponible, intenta en unos minutos"
- Pausar temporalmente creación de nuevas órdenes
- Mostrar banner de estado en toda la app

Durante Procesamiento de Refunds:
- Encolar refunds pendientes para reintentar cada 5 minutos
- Mantener estado "PROCESSING" hasta que red se recupere
- No marcar como "FAILED" inmediatamente
- Notificar usuarios que refund está "en proceso"

Durante Liberación de Fondos a Aliados:
- Similar a refunds: encolar y reintentar
- Mantener órdenes en estado especial "RELEASING_FUNDS"
- Notificar aliado que "transferencia en proceso"

Estados de Red Monitoreados:
🟢 Operativo: < 30s confirmación promedio
🟡 Lento: 30s - 2min confirmación promedio  
🔴 No disponible: > 5min sin respuesta o error rate > 50%

Recovery Automático:
✅ Intentar reconectar cada 60 segundos
✅ Procesar cola de transacciones pendientes cuando red se recupere
✅ Notificar usuarios cuando servicio se restaure completamente

Estimación: 3 días
Prioridad: Alta
Dependencias: Blockchain monitoring + queue system
```

### **US022: Recuperar Órdenes en Estados Inconsistentes**
```gherkin
Como sistema de mantenimiento
Quiero recuperar órdenes en estados problemáticos
Para evitar pérdida de fondos o datos corruptos

DADO que ejecuto job de limpieza diario a las 3:00 AM
CUANDO encuentro órdenes en estados inconsistentes
ENTONCES ejecuto recovery automático:

Órdenes Huérfanas (edge cases):
- TAKEN > 24 horas sin movimiento → forzar EXPIRED + refund
- AVAILABLE > 2 horas sin movimiento → investigar y posible refund
- Fondos en escrow sin orden válida → alert crítico admin
- Orders con status="COMPLETED" pero sin tx_hash_release → investigar

Inconsistencias de Datos:
- Órdenes COMPLETED sin comprobante → marcar para review
- Escrow con fondos pero orden EXPIRED → forzar refund
- Ally_penalties con penalty_until en el pasado → limpiar automáticamente
- Quotes sin is_active=true en últimos 10 minutos → alert

Procesos de Recovery:
✅ Forzar refund automático para casos seguros
✅ Generar alertas admin para casos que requieren investigación  
✅ Registrar todos los recovery actions en logs especiales
✅ Generar reporte semanal de órdenes recuperadas

Prevención:
✅ Timeouts agresivos previenen mayoría de casos  
✅ Transacciones atómicas en operaciones críticas
✅ Validaciones en múltiples capas
✅ Monitoring continuo de estados inconsistentes

Estimación: 2 días
Prioridad: Media
Dependencias: US015, US016 + comprehensive logging
```

## 📊 Development Plan by Sprint

### **🚀 Sprint 1 (2 weeks) - Authentication and Core User**
**Objective**: User can scan QR, view quote and pay USDT

**Included User Stories:**
- [ ] US001: Connect User Wallet (2d)
- [ ] US004: Scan Bank QR (4d) 
- [ ] US005: View Quote (3d)
- [ ] US006: Pay USDT (5d)
- [ ] US014: Automatic Quotes (3d)

**Tareas Técnicas Críticas:**
- [ ] Setup Privy + Supabase + Vercel
- [ ] Investigar formatos QR bancarios Bolivia
- [ ] Configurar wallet escrow para recibir USDT
- [ ] Implementar API binance_api/coingecko para quotes
- [ ] Setup Vercel Cron Jobs

**Criterios de Éxito Sprint 1:**
✅ Usuario puede conectar wallet via Privy
✅ Usuario puede escanear QR bancario boliviano  
✅ Usuario ve quote USDT/BOB en tiempo real
✅ Usuario puede transferir USDT al escrow
✅ Sistema actualiza cotizaciones cada 30s automáticamente

**Entregable**: Demo de usuario pagando QR con crypto

---

### **🤝 Sprint 2 (2 semanas) - Core Aliado y Flujo Completo**
**Objetivo**: Aliados pueden procesar órdenes y completar flujo end-to-end

**User Stories Incluidas:**
- [ ] US002: Conectar Wallet Aliado (3d)
- [ ] US009: Ver Órdenes Disponibles (3d)
- [ ] US010: Tomar Orden (4d)
- [ ] US011: Pagar QR (2d)
- [ ] US012: Subir Comprobante (3d)
- [ ] US017: Liberar Fondos Automático (3d)

**Tareas Técnicas Críticas:**
- [ ] Implementar concurrencia segura para tomar órdenes
- [ ] Configurar Supabase Storage para comprobantes
- [ ] Configurar wallet escrow para enviar USDT
- [ ] Implementar auto-aprobación de comprobantes
- [ ] Sistema de penalizaciones básico

**Criterios de Éxito Sprint 2:**
✅ Aliado puede registrarse y ver órdenes disponibles
✅ Aliado puede tomar orden exclusivamente 
✅ Aliado puede subir comprobante de pago
✅ Sistema libera USDT automáticamente al aliado
✅ Flujo completo usuario → aliado → completado funcional

**Entregable**: MVP funcional con transacciones reales

---

### **⚙️ Sprint 3 (2 semanas) - Sistema Automático y Robustez**
**Objetivo**: Sistema auto-gestionado con timeouts y manejo de errores

**User Stories Incluidas:**
- [ ] US015: Timeouts Automáticos (4d)
- [ ] US016: Refunds Automáticos (3d)
- [ ] US007: Rastrear Orden Usuario (3d)
- [ ] US021: Fallos de Red (3d)
- [ ] US022: Recovery Órdenes (2d)

**Tareas Técnicas Críticas:**
- [ ] Implementar job timeouts con Vercel Cron
- [ ] Sistema de refunds automáticos
- [ ] Supabase Realtime para tracking órdenes
- [ ] Manejo de errores blockchain
- [ ] Job de recovery y limpieza

**Criterios de Éxito Sprint 3:**
✅ Timeouts automáticos funcionan para todos los estados
✅ Refunds automáticos devuelven USDT cuando corresponde
✅ Sistema maneja graciosamente fallos de red blockchain
✅ Usuario puede rastrear estado de orden en tiempo real
✅ Recovery automático de órdenes problemáticas

**Entregable**: Sistema robusto que opera sin intervención manual

---

### **📊 Sprint 4 (1 semana) - Admin Panel y Pulimiento**
**Objetivo**: Herramientas administrativas y UX final optimizada

**User Stories Incluidas:**
- [ ] US018: Dashboard Admin (4d)
- [ ] US019: Configuración Sistema (2d)
- [ ] US003: Mantener Sesión (1d)
- [ ] US008: Historial Órdenes Usuario (2d)
- [ ] US013: Estadísticas Aliado (2d)
- [ ] Pulimiento UX y testing (3d)

**Tareas Técnicas Críticas:**
- [ ] Dashboard admin con métricas tiempo real
- [ ] Sistema configuración dinámico
- [ ] Views optimizadas para estadísticas
- [ ] Testing completo end-to-end
- [ ] Performance optimization

**Criterios de Éxito Sprint 4:**
✅ Admin puede monitorear sistema en tiempo real
✅ Admin puede ajustar timeouts y configuraciones dinámicamente
✅ UX pulida y optimizada para móvil
✅ Testing completo y bugs críticos resueltos
✅ Sistema production-ready

**Entregable**: MVP completo listo para usuarios reales

## 🎯 Criterios de Aceptación del MVP Completo

### **✅ Funcionalidades Core Validadas**
- [x] **Usuario puede pagar QR bancario boliviano con USDT** 
  - Escanear QR → Ver quote → Pagar → Completado
- [x] **Aliado puede procesar pagos y ganar USDT**
  - Ver órdenes → Tomar → Pagar en banco → Subir comprobante → Recibir USDT
- [x] **Sistema maneja timeouts automáticamente**
  - 3-5-5 minutos por estado, refunds automáticos
- [x] **Refunds automáticos funcionan correctamente**
  - Ningún usuario pierde dinero por timeouts
- [x] **Admin puede monitorear y configurar el sistema**
  - Dashboard tiempo real + configuración dinámica

### **✅ Métricas de Éxito MVP**
- **Tiempo promedio procesamiento**: < 10 minutos end-to-end
- **Tasa de éxito**: > 80% órdenes completadas exitosamente
- **Tasa de timeout**: < 15% órdenes expiradas
- **Uptime del sistema**: > 99% disponibilidad
- **UX mobile-first**: Funciona perfecto en dispositivos móviles
- **Cero pérdida de fondos**: Todos los fallos resultan en refund automático

### **✅ Seguridad y Confiabilidad MVP**
- **Fondos siempre seguros**: Escrow centralizado funcional con refunds automáticos
- **No pérdida de dinero**: Sistema designed to never lose user funds
- **Logs completos**: Auditoría detallada de todas las acciones
- **Timeouts agresivos**: Previenen órdenes colgadas indefinidamente
- **Penalizaciones automáticas**: Mantienen calidad de aliados sin intervención

### **✅ Experiencia de Usuario MVP**
- **Onboarding simple**: Conectar wallet y empezar a usar inmediatamente
- **Feedback claro**: Usuario siempre sabe qué está pasando y qué hacer
- **Timers visibles**: Countdown en tiempo real para presión temporal apropiada
- **Notificaciones útiles**: Push notifications en momentos clave
- **Mobile optimizado**: UI/UX diseñada mobile-first

## 🚨 Riesgos y Mitigaciones

### **🔴 Riesgos Técnicos Altos**
| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| **API quotes no confiable** | Media | Alto | Múltiples fuentes + fallbacks |
| **mantle network issues** | Media | Alto | Queue system + retry logic |
| **Escrow wallet comprometida** | Baja | Crítico | Multi-sig + monitoring + limits |
| **Concurrencia en tomar órdenes** | Alta | Medio | Database locks + validación doble |

### **🟡 Riesgos de Negocio Medios**
| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| **Aliados maliciosos** | Media | Alto | Logs + penalizaciones + review posterior |
| **Usuarios abandono por UX** | Media | Alto | Testing extensivo + feedback loops |
| **Regulaciones cripto Bolivia** | Baja | Alto | Monitoring legal + compliance research |
| **Competencia directa** | Alta | Medio | Focus en UX superior + network effects |

### **🟢 Riesgos Técnicos Bajos**
- **Supabase downtime**: Rare, good SLA, status page monitoring
- **Vercel deployment issues**: Rare, good track record, easy rollback
- **QR parsing failures**: Fallback to manual input, multiple libraries

## 📈 Roadmap Post-MVP (Futuras Versiones)

### **🔄 Versión 1.1 - Mejoras de Confianza (Mes 2)**
- [ ] Verificación manual opcional de comprobantes
- [ ] Sistema de reputación para aliados
- [ ] OCR automático de comprobantes
- [ ] Múltiples wallets escrow para distribución de riesgo

### **🌍 Versión 1.2 - Expansión Geográfica (Mes 3-4)**
- [ ] Soporte para México (MXN)
- [ ] Soporte para Argentina (ARS)
- [ ] Múltiples tokens (USDC, DAI)
- [ ] Múltiples redes (Base, Arbitrum)

### **⚡ Versión 1.3 - Optimización Avanzada (Mes 5-6)**
- [ ] Smart contracts descentralizados
- [ ] Sistema de dispute resolution
- [ ] API pública para terceros
- [ ] App móvil nativa

---

**🎯 Entregables Finales para el Equipo:**

- **📋 Product Owner**: 22 User Stories priorizadas con criterios de aceptación detallados
- **👨‍💻 Developers**: Tareas específicas con estimaciones y dependencias claras
- **🧪 QA**: Criterios de aceptación testeable y casos edge documentados
- **📊 DevOps**: Plan de 4 sprints con dependencies y milestones claros
- **💼 Stakeholders**: Roadmap con entregables concretos y métricas de éxito

**🔑 Próximos Pasos Inmediatos:**
1. **Setup técnico**: Configurar Privy + Supabase + Vercel
2. **Research QR formats**: Investigar formatos QR bancarios bolivianos
3. **Wallet escrow**: Configurar wallet para recibir/enviar USDT
4. **Sprint 1 kickoff**: Comenzar con US001 - Conectar Wallet Usuario

**🚀 ¡MVP Kibo listo para desarrollo!**
