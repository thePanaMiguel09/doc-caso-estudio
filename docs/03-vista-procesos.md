# Vista 2 — Procesos

> **Modelo 4+1 · Vista de Procesos.** Describe el comportamiento dinámico en tiempo de ejecución: flujos de control, puntos de decisión, validaciones, concurrencia, sincronización y contención por recursos compartidos. Su destinatario es el equipo de desarrollo y el equipo de plataforma.

**Cobertura:** 6 diagramas de actividades (uno por módulo, más el proceso asíncrono de reconciliación) · 5 máquinas de estado · 2 procesos de infraestructura. **Total: 13 diagramas.**

---

## Convención de notación

Mermaid.js no implementa carriles (*swimlanes*) en diagramas de actividad. Se emplea `flowchart` con **codificación por color** según el proceso responsable, que preserva la información esencial: qué unidad de ejecución realiza cada acción y dónde el control cruza un límite de red.

| Color | Proceso responsable |
|---|---|
| 🔵 Azul | API Gateway — capa de borde |
| 🟢 Verde | Módulo de negocio propietario del flujo |
| 🟡 Amarillo | Otro módulo invocado por REST |
| 🔴 Rojo | Sistema externo o ruta de error |
| 🟣 Morado | Proceso asíncrono dirigido por cola |

| Símbolo | Significado UML |
|---|---|
| `([ ])` | Nodo inicial / final |
| `[ ]` | Acción |
| `{ }` | Nodo de decisión o fusión |
| Barra `═══` | Nodo `fork` / `join` de concurrencia |

---

## 1. Proceso de autenticación y autorización — RF-01

```mermaid
flowchart TD
    INI([Inicio: usuario solicita acceso]) --> A1

    A1["Portal envía credenciales<br/>por HTTPS/TLS 1.2+"]:::gw
    A1 --> A2["WAF filtra tráfico<br/>según OWASP Top 10"]:::gw
    A2 --> D1{"¿Tráfico<br/>malicioso?"}:::gw
    D1 -->|Sí| E1["Bloquear y registrar<br/>en auditoría"]:::err
    E1 --> FIN1([Fin: acceso bloqueado])
    D1 -->|No| A3["Redirigir a proveedor OIDC<br/>flujo Authorization Code"]:::gw

    A3 --> A4["Proveedor valida<br/>credenciales"]:::ext
    A4 --> D2{"¿Credenciales<br/>válidas?"}:::ext
    D2 -->|No| E2["Incrementar contador<br/>de intentos fallidos"]:::err
    E2 --> D3{"¿Supera<br/>umbral?"}:::err
    D3 -->|Sí| E3["Bloquear cuenta<br/>temporalmente"]:::err
    E3 --> A13["Registrar anomalía<br/>en auditoría"]:::mod
    A13 --> FIN2([Fin: cuenta bloqueada])
    D3 -->|No| FIN3([Fin: credenciales inválidas])

    D2 -->|Sí| A5["Intercambiar código<br/>por tokens"]:::gw
    A5 --> A6["Consultar perfil<br/>en Módulo Identidad"]:::mod

    A6 --> FK1[" "]:::fork
    FK1 --> A7["Cargar roles<br/>del usuario"]:::mod
    FK1 --> A8["Cargar permisos<br/>asociados"]:::mod
    FK1 --> A9["Verificar estado<br/>de la cuenta"]:::mod
    A7 --> JN1[" "]:::fork
    A8 --> JN1
    A9 --> JN1

    JN1 --> D4{"¿Cuenta<br/>activa?"}:::mod
    D4 -->|No| E4["Denegar acceso<br/>cuenta inactiva"]:::err
    E4 --> A13
    D4 -->|Sí| A10["Construir scopes<br/>desde matriz RBAC"]:::mod
    A10 --> A11["Crear SesionActiva<br/>y emitir credencial"]:::mod
    A11 --> A12["Registrar LOGIN_EXITOSO<br/>en auditoría"]:::mod
    A12 --> FIN4([Fin: sesión establecida])

    FIN4 -.-> B1

    B1["Solicitud a recurso protegido<br/>con Bearer token"]:::gw
    B1 --> D5{"¿Token válido<br/>y no expirado?"}:::gw
    D5 -->|Expirado| B2["Renovar con<br/>refresh_token"]:::gw
    B2 --> D6{"¿Renovación<br/>exitosa?"}:::gw
    D6 -->|Sí| D7
    D6 -->|No| E5["401 Unauthorized<br/>reautenticar"]:::err
    E5 --> FIN5([Fin: sesión expirada])
    D5 -->|Inválido| E5
    D5 -->|Válido| D7{"¿Scope cubre<br/>el recurso?"}:::gw

    D7 -->|No| E6["Registrar ACCESO_DENEGADO<br/>y alertar"]:::err
    E6 --> E7["403 Forbidden<br/>en menos de 2 s"]:::err
    E7 --> FIN6([Fin: acceso denegado])
    D7 -->|Sí| B3["Enrutar al módulo destino<br/>con identidad validada"]:::mod
    B3 --> B4["Módulo procesa<br/>sin revalidar identidad"]:::mod
    B4 --> FIN7([Fin: recurso entregado])

    classDef gw fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef mod fill:#E6F4EA,stroke:#137333,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
    classDef err fill:#FCE8E6,stroke:#C5221F,color:#000
    classDef fork fill:#5F6368,stroke:#202124,color:#fff,height:6px
```

### Justificación — RF-01

**El módulo destino no revalida la identidad.** La validación ocurre exclusivamente en el Gateway; los módulos reciben la identidad ya verificada y confían en el borde. Esto evita cinco implementaciones divergentes de RBAC y concentra la política de seguridad en un punto auditable. La contrapartida obligatoria —que ningún módulo sea alcanzable directamente desde Internet— se garantiza mediante segmentación de red en la [Vista Física](05-vista-fisica.md).

**La carga de roles, permisos y estado de cuenta se ejecuta en paralelo.** Son consultas independientes; serializarlas triplicaría la latencia de una operación que ocurre en cada inicio de sesión de toda la comunidad universitaria.

**El bloqueo por intentos fallidos protege contra fuerza bruta**, y toda anomalía se registra antes de responder, garantizando que no exista ninguna ventana donde un rechazo ocurra sin dejar rastro auditable.

---

## 2. Proceso de matrícula — RF-02

Proceso crítico del negocio. Integra validación académica concurrente, verificación financiera obligatoria, manejo de fallo del servicio externo y reconciliación asíncrona.

```mermaid
flowchart TD
    INI([Inicio: estudiante confirma selección]) --> A1

    A1["Generar clave de idempotencia<br/>y enviar solicitud"]:::gw
    A1 --> D0{"¿Supera límite<br/>de tasa?"}:::gw
    D0 -->|Sí| E0["429 Too Many Requests<br/>con Retry-After"]:::err
    E0 --> FIN0([Fin: reintentar más tarde])
    D0 -->|No| A2["Validar token y rol<br/>en Gateway"]:::gw
    A2 --> A3["Enrutar a Módulo Matrícula"]:::mod

    A3 --> D1{"¿Clave de idempotencia<br/>ya procesada?"}:::mod
    D1 -->|Sí| A4["Recuperar respuesta<br/>original"]:::mod
    A4 --> FIN1([Fin: respuesta idéntica sin duplicar])
    D1 -->|No| D2{"¿Periodo de matrícula<br/>abierto?"}:::mod
    D2 -->|No| E1["409 Fuera del periodo<br/>de matrícula"]:::err
    E1 --> FIN2([Fin: rechazada])
    D2 -->|Sí| A5["Crear Matricula<br/>estado EN_VALIDACION"]:::mod

    A5 --> FK1[" "]:::fork
    FK1 --> A6["Verificar cupo del grupo<br/>con versión optimista"]:::otro
    FK1 --> A7["Validar prerrequisitos<br/>contra el expediente"]:::otro
    FK1 --> A8["Verificar tope<br/>de créditos"]:::mod
    FK1 --> A9["Detectar solapamiento<br/>de horarios"]:::mod
    A6 --> JN1[" "]:::fork
    A7 --> JN1
    A8 --> JN1
    A9 --> JN1

    JN1 --> A10["Consolidar todos<br/>los resultados"]:::mod
    A10 --> D3{"¿Todas las validaciones<br/>son correctas?"}:::mod
    D3 -->|No| E2["Marcar RECHAZADA<br/>con el detalle de cada regla"]:::err
    E2 --> FIN3([Fin: 409 con motivos])

    D3 -->|Sí| A11["Solicitar verificación<br/>financiera"]:::otro
    A11 --> D4{"¿Circuit Breaker<br/>permite el paso?"}:::otro
    D4 -->|No, circuito abierto| A12["Fallo rápido sin<br/>intentar la llamada"]:::err
    D4 -->|Sí| A13["Invocar pasarela<br/>tiempo de espera 3 s"]:::ext
    A13 --> D5{"¿Respuesta<br/>exitosa?"}:::ext
    D5 -->|No| A14["Registrar fallo<br/>en el contador"]:::err
    A14 --> D6{"¿Supera umbral<br/>de fallos?"}:::err
    D6 -->|Sí| A15["Abrir circuito<br/>y emitir alerta"]:::err
    A15 --> A12
    D6 -->|No| A12
    D5 -->|Sí| A16["Obtener EstadoFinanciero<br/>confiable"]:::otro

    A12 --> A17["EstadoFinanciero<br/>marcado degradado"]:::err
    A17 --> A18["Reservar cupo PROVISIONAL<br/>con vencimiento 72 h"]:::otro
    A18 --> A19["Marcar PENDIENTE_PAGO"]:::mod
    A19 --> A20["Publicar ReconciliarPago<br/>en la cola"]:::mod
    A20 --> FIN4([Fin: 202 pago en verificación])

    A16 --> D7{"¿Estudiante a<br/>paz y salvo?"}:::mod
    D7 -->|No| A21["Generar orden de pago"]:::mod
    A21 --> A18
    D7 -->|Sí| A22["Reservar cupo<br/>en firme"]:::otro
    A22 --> D8{"¿Conflicto de<br/>versión optimista?"}:::otro
    D8 -->|Sí| A23["Releer estado<br/>actual del grupo"]:::otro
    A23 --> D9{"¿Aún hay<br/>cupo?"}:::otro
    D9 -->|No| E3["Marcar RECHAZADA<br/>cupo agotado"]:::err
    E3 --> FIN5([Fin: 409 cupo agotado])
    D9 -->|Sí| A22
    D8 -->|No| A24["Marcar CONFIRMADA"]:::mod
    A24 --> A25["Actualizar expediente<br/>y publicar evento"]:::mod
    A25 --> FIN6([Fin: 201 matrícula confirmada])

    classDef gw fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef mod fill:#E6F4EA,stroke:#137333,color:#000
    classDef otro fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
    classDef err fill:#FCE8E6,stroke:#C5221F,color:#000
    classDef fork fill:#5F6368,stroke:#202124,color:#fff,height:6px
```

### Proceso asíncrono de reconciliación

```mermaid
flowchart TD
    INI([Inicio: mensaje ReconciliarPago en cola]):::async --> R1

    R1["Consumir mensaje<br/>de la cola"]:::async
    R1 --> R2["Esperar según retroceso<br/>exponencial 1s, 2s, 4s..."]:::async
    R2 --> R3["Reintentar consulta con<br/>la misma clave idempotente"]:::async
    R3 --> D1{"¿Pasarela<br/>restablecida?"}:::async

    D1 -->|No| D2{"¿Dentro de la<br/>ventana de 72 h?"}:::async
    D2 -->|Sí| R4["Reencolar con espera<br/>incrementada"]:::async
    R4 --> R2
    D2 -->|No| R5["Liberar cupo<br/>provisional"]:::otro
    R5 --> R6["Marcar matrícula<br/>EXPIRADA"]:::mod
    R6 --> R7["Notificar expiración<br/>al estudiante"]:::async
    R7 --> FIN1([Fin: matrícula expirada])

    D1 -->|Sí| R8["Cerrar circuito"]:::async
    R8 --> D3{"¿Estudiante a<br/>paz y salvo?"}:::async
    D3 -->|No| R9["Mantener PENDIENTE_PAGO<br/>y notificar orden de pago"]:::mod
    R9 --> FIN2([Fin: pendiente de pago del estudiante])
    D3 -->|Sí| R10["Convertir cupo provisional<br/>en cupo firme"]:::otro
    R10 --> R11["Marcar CONFIRMADA"]:::mod
    R11 --> R12["Notificar confirmación<br/>al estudiante"]:::async
    R12 --> FIN3([Fin: reconciliación exitosa, RTO cumplido])

    classDef mod fill:#E6F4EA,stroke:#137333,color:#000
    classDef otro fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef async fill:#F3E8FD,stroke:#8430CE,color:#000
```

### Justificación — RF-02

| Decisión | Razón |
|---|---|
| **Verificación de idempotencia antes de crear cualquier registro** | Durante el pico, el doble clic y el reintento del navegador son masivos. Si la verificación estuviera después de la creación, existiría una ventana de carrera que produciría matrículas duplicadas precisamente cuando el sistema está más cargado — el fallo aparecería solo bajo la condición en que más daño hace. |
| **Fallo rápido cuando el circuito está abierto** | No se intenta la llamada. Sin esto, miles de peticiones esperarían tiempos de espera de 3 s, agotando el pool de conexiones del módulo y afectando a estudiantes que solo querían consultar notas. El Circuit Breaker no protege a la pasarela: protege a UPS-Connect de la pasarela. |
| **Cuatro validaciones concurrentes con `join` obligatorio** | Ninguna validación se omite aunque otra ya haya fallado: el estudiante debe recibir todos los motivos de rechazo de una vez, no uno por intento. |
| **Reconciliador con nodo inicial propio** | Es un proceso autónomo dirigido por cola. Puede ejecutarse minutos u horas después, en otra instancia, incluso tras un redespliegue del módulo. Dibujarlo como continuación del flujo síncrono sugeriría que el usuario espera. |
| **Manejo explícito del conflicto de versión optimista** | Entre la lectura del cupo y la reserva en firme transcurre tiempo real, y otro estudiante pudo tomar el lugar. Omitir esta rama produciría un diagrama que solo funciona con un usuario a la vez. |

---

## 3. Proceso de programación académica — RF-03

```mermaid
flowchart TD
    INI([Inicio: coordinador planifica el semestre]) --> A1

    A1["Seleccionar plan de estudio<br/>vigente del programa"]:::mod
    A1 --> D1{"¿Plan vigente<br/>y validado?"}:::mod
    D1 -->|No| E1["Bloquear: el plan debe<br/>estar aprobado"]:::err
    E1 --> FIN1([Fin: planificación bloqueada])
    D1 -->|Sí| A2["Crear grupos por asignatura<br/>con cupo previsto"]:::mod

    A2 --> A3["Asignar docente<br/>a cada grupo"]:::mod
    A3 --> A4["Consultar carga académica<br/>actual del docente"]:::mod
    A4 --> D2{"¿Excede el tope<br/>contractual?"}:::mod
    D2 -->|Sí| E2["Rechazar asignación<br/>y sugerir docentes alternos"]:::err
    E2 --> A3
    D2 -->|No| A5["Proponer franjas horarias<br/>para el grupo"]:::mod

    A5 --> A6["Seleccionar aula<br/>según tipo y capacidad"]:::mod
    A6 --> D3{"¿El aula admite<br/>el tamaño del grupo?"}:::mod
    D3 -->|No| E3["Rechazar: capacidad<br/>insuficiente"]:::err
    E3 --> A6
    D3 -->|Sí| A7["Invocar MotorReservas"]:::mod

    A7 --> FK1[" "]:::fork
    FK1 --> A8["Detectar conflicto<br/>de aula en la franja"]:::mod
    FK1 --> A9["Detectar conflicto<br/>de docente"]:::mod
    FK1 --> A10["Detectar solapamiento<br/>del propio grupo"]:::mod
    A8 --> JN1[" "]:::fork
    A9 --> JN1
    A10 --> JN1

    JN1 --> A11["Consolidar conflictos<br/>clasificados por severidad"]:::mod
    A11 --> D4{"¿Hay conflictos<br/>bloqueantes?"}:::mod

    D4 -->|Sí| A12["Calcular franjas<br/>alternativas viables"]:::mod
    A12 --> A13["Presentar conflicto<br/>con alternativas ordenadas"]:::err
    A13 --> D5{"¿Coordinador acepta<br/>una alternativa?"}:::mod
    D5 -->|Sí| A5
    D5 -->|No| FIN2([Fin: programación no resuelta])

    D4 -->|No| A14["Iniciar transacción"]:::mod
    A14 --> A15["Persistir ReservaAula<br/>estado CONFIRMADA"]:::mod
    A15 --> A16["Actualizar CargaAcademica<br/>del docente"]:::mod
    A16 --> A17["Actualizar Horario<br/>del grupo"]:::mod
    A17 --> A18["Confirmar transacción"]:::mod
    A18 --> D6{"¿Hubo advertencias<br/>no bloqueantes?"}:::mod
    D6 -->|Sí| A19["Registrar advertencias<br/>en el resultado"]:::mod
    A19 --> A20
    D6 -->|No| A20["Publicar ReservaConfirmada<br/>en la cola"]:::async
    A20 --> A21["Notificar al docente<br/>de forma asíncrona"]:::async
    A21 --> D7{"¿Quedan grupos<br/>por programar?"}:::mod
    D7 -->|Sí| A2
    D7 -->|No| A22["Publicar oferta<br/>académica del periodo"]:::mod
    A22 --> FIN3([Fin: oferta publicada])

    classDef mod fill:#E6F4EA,stroke:#137333,color:#000
    classDef err fill:#FCE8E6,stroke:#C5221F,color:#000
    classDef async fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef fork fill:#5F6368,stroke:#202124,color:#fff,height:6px
```

### Justificación — RF-03

**La detección de conflictos ocurre antes de la transacción de escritura, y los tres detectores se ejecutan concurrentemente.** El motor de reservas actúa como precondición, no como validación posterior. Ningún camino del diagrama permite persistir una asignación conflictiva, lo que hace cumplible el requisito de RF-03 de evitar conflictos de programación.

**Las cuatro escrituras van en una sola transacción.** Reserva, carga docente y horario deben confirmarse atómicamente: una reserva persistida sin actualizar la carga del docente permitiría sobreasignarlo en la siguiente iteración.

**Los conflictos se clasifican por severidad.** Un conflicto bloqueante detiene el flujo; una advertencia lo permite pero queda registrada. Un rechazo binario obligaría al coordinador a resolver manualmente situaciones que el sistema puede tolerar de forma informada.

**El bucle de programación permite procesar todos los grupos del semestre** sin abandonar el proceso ante cada conflicto individual.

---

## 4. Proceso de evaluación académica — RF-04

```mermaid
flowchart TD
    INI([Inicio: docente inicia el periodo evaluativo]) --> A1

    A1["Definir actividades evaluativas<br/>del grupo"]:::mod
    A1 --> D1{"¿Los porcentajes<br/>suman 100 %?"}:::mod
    D1 -->|No| E1["Rechazar: la distribución<br/>debe sumar 100 %"]:::err
    E1 --> A1
    D1 -->|Sí| A2["Abrir ActaNotas<br/>del grupo"]:::mod

    A2 --> A3["Registrar calificaciones<br/>de una actividad"]:::mod
    A3 --> D2{"¿El acta está<br/>abierta?"}:::mod
    D2 -->|No| E2["409 El acta está cerrada"]:::err
    E2 --> D8
    D2 -->|Sí| D3{"¿La nota está<br/>en rango válido?"}:::mod
    D3 -->|No| E3["400 Nota fuera de rango"]:::err
    E3 --> A3
    D3 -->|Sí| A4["Iniciar transacción"]:::mod
    A4 --> A5["Persistir Calificacion<br/>estado REGISTRADA"]:::mod
    A5 --> A6["Registrar en<br/>HistorialCambioNota"]:::mod
    A6 --> A7["Confirmar transacción"]:::mod

    A7 --> D4{"¿Docente decide<br/>publicar ahora?"}:::mod
    D4 -->|No| A3
    D4 -->|Sí| D5{"¿Todos los estudiantes<br/>tienen nota?"}:::mod
    D5 -->|No| E4["409 Registro incompleto<br/>con lista de faltantes"]:::err
    E4 --> A3

    D5 -->|Sí| A8["Marcar actividad<br/>como PUBLICADA"]:::mod
    A8 --> A9["Generar un EventoNota<br/>por estudiante"]:::mod
    A9 --> A10["Publicar lote de eventos<br/>en la cola"]:::async
    A10 --> A11["Responder al docente<br/>transacción finalizada"]:::mod

    A10 -.-> N1["Consumidor de notificaciones<br/>procesa el lote"]:::async
    N1 --> N2["Resolver canal según<br/>preferencias del estudiante"]:::async
    N2 --> N3["Entregar notificación"]:::async
    N3 --> D6{"¿Entrega<br/>exitosa?"}:::async
    D6 -->|No| N4["Reencolar con<br/>retroceso exponencial"]:::async
    N4 --> N2
    D6 -->|Sí| FINN([Fin: estudiante notificado])

    A11 --> D7{"¿Quedan actividades<br/>por evaluar?"}:::mod
    D7 -->|Sí| A3
    D7 -->|No| A12["Calcular notas<br/>definitivas del grupo"]:::mod
    A12 --> A13["Cerrar ActaNotas<br/>y generar hash de integridad"]:::mod
    A13 --> A14["Consolidar resultados<br/>en el expediente"]:::otro
    A14 --> FIN1([Fin: acta cerrada])

    D8{"¿Existe autorización<br/>de reapertura?"}:::mod
    D8 -->|No| FIN2([Fin: modificación no permitida])
    D8 -->|Sí| A15["Reabrir acta<br/>registrando autorización"]:::mod
    A15 --> A16["Modificar nota<br/>con motivo obligatorio"]:::mod
    A16 --> A17["Registrar cambio<br/>en el historial"]:::mod
    A17 --> A18["Publicar evento<br/>de MODIFICACION"]:::async
    A18 --> A13

    classDef mod fill:#E6F4EA,stroke:#137333,color:#000
    classDef otro fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef err fill:#FCE8E6,stroke:#C5221F,color:#000
    classDef async fill:#F3E8FD,stroke:#8430CE,color:#000
```

### Justificación — RF-04

**El registro y la publicación son operaciones separadas.** Un docente registra notas progresivamente sin que los estudiantes vean resultados parciales; publicar es un acto deliberado. Fusionarlas obligaría a completar todo el grupo en una sola sesión.

**La transacción del docente termina al publicar el evento en la cola.** El flujo de notificación se muestra como rama desprendida (línea punteada) para indicar que corre en un proceso independiente. Si el servicio de notificaciones estuviera caído, las notas quedan igualmente publicadas y consultables: **el desacople protege la operación académica de los fallos del canal de comunicación**.

**La validación de suma 100 % es una precondición de apertura del acta**, no un chequeo al cierre. Detectarlo al final obligaría a rehacer toda la evaluación.

**La modificación posterior al cierre exige autorización y motivo**, y queda registrada en el historial inmutable. Es un requisito de trazabilidad académica y legal.

---

## 5. Proceso de integración externa y analítica — RF-05

```mermaid
flowchart TD
    INI([Inicio: solicitud que requiere servicio externo]) --> A1

    A1["Identificar adaptador<br/>correspondiente"]:::mod
    A1 --> A2["Consultar estado del<br/>CircuitBreaker del servicio"]:::mod
    A2 --> D1{"¿Estado del<br/>circuito?"}:::mod

    D1 -->|ABIERTO| D2{"¿Venció la ventana<br/>de apertura?"}:::mod
    D2 -->|No| E1["Fallo rápido sin<br/>consumir recursos"]:::err
    D2 -->|Sí| A3["Pasar a SEMIABIERTO<br/>y permitir una prueba"]:::mod
    D1 -->|SEMIABIERTO| A3
    D1 -->|CERRADO| A4["Invocar servicio externo<br/>con tiempo de espera"]:::ext
    A3 --> A4

    A4 --> D3{"¿Respuesta<br/>exitosa?"}:::ext
    D3 -->|Sí| A5["Registrar éxito<br/>y cerrar circuito"]:::mod
    A5 --> A6["Retornar resultado<br/>confiable"]:::mod
    A6 --> D4{"¿Operación<br/>tolera degradación?"}:::mod

    D3 -->|No| A7["Registrar fallo<br/>e incrementar contador"]:::err
    A7 --> D5{"¿Supera umbral<br/>de fallos?"}:::err
    D5 -->|Sí| A8["Abrir circuito<br/>y emitir alerta"]:::err
    A8 --> A9["Notificar a Observabilidad<br/>en menos de 5 min"]:::mod
    A9 --> E1
    D5 -->|No| E1

    E1 --> D4
    D4 -->|Sí, matrícula| A10["Retornar resultado<br/>marcado como degradado"]:::mod
    A10 --> A11["Encolar tarea<br/>de reconciliación"]:::async
    A11 --> FIN1([Fin: operación en modo degradado])

    D4 -->|No, certificado| A12["Rechazar la operación<br/>503 Service Unavailable"]:::err
    A12 --> FIN2([Fin: operación no ejecutada])

    D4 -->|Resultado confiable| A13["Completar la operación<br/>solicitada"]:::mod
    A13 --> A14["Persistir resultado<br/>y publicar evento"]:::mod
    A14 --> FIN3([Fin: operación completada])

    INI2([Inicio: actualización de panel directivo]):::mod --> P1
    P1["Consumir eventos de dominio<br/>de todos los módulos"]:::async
    P1 --> P2["Actualizar proyecciones<br/>de lectura"]:::mod
    P2 --> FK1[" "]:::fork
    FK1 --> P3["Calcular tasa de<br/>ocupación de aulas"]:::mod
    FK1 --> P4["Calcular índice de<br/>riesgo de deserción"]:::mod
    FK1 --> P5["Calcular tasa de<br/>matrícula efectiva"]:::mod
    FK1 --> P6["Calcular recaudo<br/>acumulado"]:::mod
    P3 --> JN1[" "]:::fork
    P4 --> JN1
    P5 --> JN1
    P6 --> JN1
    JN1 --> P7["Comparar con el<br/>periodo anterior"]:::mod
    P7 --> D6{"¿Algún indicador<br/>supera su umbral?"}:::mod
    D6 -->|Sí| P8["Emitir alerta temprana<br/>al coordinador"]:::async
    P8 --> P9
    D6 -->|No| P9["Publicar panel<br/>actualizado"]:::mod
    P9 --> FIN4([Fin: panel disponible])

    classDef mod fill:#E6F4EA,stroke:#137333,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
    classDef err fill:#FCE8E6,stroke:#C5221F,color:#000
    classDef async fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef fork fill:#5F6368,stroke:#202124,color:#fff,height:6px
```

### Justificación — RF-05

**La decisión "¿la operación tolera degradación?" es el corazón de este proceso.** No todas las operaciones deben comportarse igual ante un fallo externo:

| Operación | Comportamiento ante fallo | Razón |
|---|---|---|
| Verificación financiera para matrícula | Continúa en modo degradado | Una matrícula provisional se reconcilia después sin daño |
| Emisión de certificado | Se rechaza con `503` | Un documento oficial emitido sobre información no confirmada ya circuló y no puede retractarse |

Esta asimetría deliberada es una decisión arquitectónica, no una inconsistencia: **el modo degradado se concede en función del costo del error, no de la conveniencia técnica**.

**El estado `SEMIABIERTO` prueba la recuperación con una sola petición** antes de reabrir el tráfico completo, evitando saturar un servicio que apenas se está restableciendo y provocar una reapertura inmediata del circuito.

**El panel directivo consume eventos y mantiene proyecciones de lectura**, en lugar de consultar en línea las bases de datos de los otros módulos. Esto evita que la analítica compita por recursos con la operación transaccional durante el periodo de matrícula.

---

## 6. Máquina de estados — Sesión de usuario (RF-01)

```mermaid
stateDiagram-v2
    direction LR

    [*] --> NoAutenticado

    NoAutenticado --> Autenticando : solicita acceso
    Autenticando --> NoAutenticado : credenciales inválidas
    Autenticando --> Bloqueado : supera umbral de intentos
    Autenticando --> Activa : autenticación exitosa

    Bloqueado --> NoAutenticado : vence el bloqueo temporal
    Bloqueado --> [*] : desactivación administrativa

    Activa --> Activa : renovación de token
    Activa --> Expirada : inactividad prolongada
    Activa --> Revocada : revocación administrativa
    Activa --> Cerrada : cierre voluntario

    Expirada --> Activa : refresh_token válido
    Expirada --> NoAutenticado : refresh_token vencido

    Revocada --> [*]
    Cerrada --> [*]

    note right of Revocada
        Estado terminal inmediato.
        Permite al administrador cortar
        el acceso sin esperar la
        expiración natural del token.
    end note
```

**Justificación.** La separación entre `Expirada` y `Revocada` es lo que hace posible el caso de uso UC-06.1: `Expirada` es recuperable mediante `refresh_token`; `Revocada` es terminal e irreversible. Sin esta distinción, un administrador que detecte una cuenta comprometida no tendría forma de cortar el acceso antes del vencimiento natural del token.

---

## 7. Máquina de estados — Matrícula (RF-02)

```mermaid
stateDiagram-v2
    direction LR

    [*] --> EN_VALIDACION : solicitud con clave idempotente

    EN_VALIDACION --> RECHAZADA : validación académica fallida
    EN_VALIDACION --> VERIFICANDO_PAGO : validaciones correctas

    state VERIFICANDO_PAGO {
        direction LR
        [*] --> ConsultaSincrona
        ConsultaSincrona --> Degradado : circuito abierto o timeout
        ConsultaSincrona --> Resuelto : la pasarela responde
        Degradado --> [*]
        Resuelto --> [*]
    }

    VERIFICANDO_PAGO --> CONFIRMADA : paz y salvo, cupo en firme
    VERIFICANDO_PAGO --> PENDIENTE_PAGO : saldo pendiente o dato degradado

    PENDIENTE_PAGO --> CONFIRMADA : reconciliación exitosa
    PENDIENTE_PAGO --> EXPIRADA : vencen las 72 h
    PENDIENTE_PAGO --> CANCELADA : el estudiante desiste

    CONFIRMADA --> MODIFICADA : agregar o quitar asignaturas
    MODIFICADA --> VERIFICANDO_PAGO : recalcular y reverificar
    CONFIRMADA --> CANCELADA : cancelación dentro de plazo
    CONFIRMADA --> CERRADA : cierre del periodo académico

    RECHAZADA --> [*]
    EXPIRADA --> [*]
    CANCELADA --> [*]
    CERRADA --> [*]

    note right of PENDIENTE_PAGO
        Tres salidas obligatorias.
        Un estado de espera con una sola
        salida feliz sería una fuga
        garantizada de cupos.
    end note

    note right of CERRADA
        Estado terminal e inmutable.
        Habilita el registro de
        calificaciones del periodo.
    end note
```

**Justificación.** `MODIFICADA` retorna a `VERIFICANDO_PAGO` y nunca directo a `CONFIRMADA`. Si un estudiante agrega asignaturas, el valor cambia y **el paz y salvo previo deja de ser válido**. Permitir el salto directo reabriría por la puerta de atrás la desconexión académico-financiera que el proyecto busca eliminar. Es una regla que solo se hace visible en la máquina de estados: ni el diagrama de clases ni el de secuencia la revelan.

La separación entre `CONFIRMADA` y `CERRADA` establece la frontera de inmutabilidad. Sin ella, un docente podría estar calificando a un grupo cuya composición cambia simultáneamente.

---

## 8. Máquina de estados — Reserva de aula (RF-03)

```mermaid
stateDiagram-v2
    direction LR

    [*] --> SOLICITADA : el coordinador propone

    SOLICITADA --> EN_VALIDACION : invocar MotorReservas
    EN_VALIDACION --> RECHAZADA : conflicto bloqueante
    EN_VALIDACION --> CON_ADVERTENCIA : conflicto no bloqueante
    EN_VALIDACION --> CONFIRMADA : sin conflictos

    CON_ADVERTENCIA --> CONFIRMADA : el coordinador acepta
    CON_ADVERTENCIA --> RECHAZADA : el coordinador desiste

    CONFIRMADA --> REPROGRAMADA : cambio de franja u aula
    REPROGRAMADA --> EN_VALIDACION : revalidar la nueva propuesta

    CONFIRMADA --> LIBERADA : cancelación del grupo
    CONFIRMADA --> FINALIZADA : concluye el periodo

    RECHAZADA --> [*]
    LIBERADA --> [*]
    FINALIZADA --> [*]

    note right of REPROGRAMADA
        Toda reprogramación vuelve
        a validarse. No existe camino
        que persista una asignación
        sin pasar por el motor.
    end note
```

**Justificación.** El estado `REPROGRAMADA` obliga a revalidar. Es el punto donde suelen introducirse conflictos en los sistemas académicos: un cambio de aula aparentemente inocuo genera un choque de horario para el docente. Al forzar el retorno a `EN_VALIDACION`, ningún camino evade el motor de reservas.

`CON_ADVERTENCIA` es un estado real, no un mensaje: permite que el coordinador tome una decisión informada sobre conflictos tolerables sin que el sistema imponga un rechazo binario.

---

## 9. Máquina de estados — Acta de notas (RF-04)

```mermaid
stateDiagram-v2
    direction LR

    [*] --> BORRADOR : docente define actividades

    BORRADOR --> BORRADOR : ajustar porcentajes
    BORRADOR --> ABIERTA : porcentajes suman 100 %

    ABIERTA --> ABIERTA : registrar calificaciones
    ABIERTA --> PARCIALMENTE_PUBLICADA : publicar una actividad
    PARCIALMENTE_PUBLICADA --> PARCIALMENTE_PUBLICADA : publicar otra actividad
    PARCIALMENTE_PUBLICADA --> COMPLETA : todas las actividades publicadas

    COMPLETA --> CERRADA : el docente cierra el acta
    CERRADA --> EN_REVISION : solicitud de revisión aceptada
    EN_REVISION --> CERRADA : revisión resuelta
    CERRADA --> REABIERTA : autorización del coordinador
    REABIERTA --> COMPLETA : correcciones aplicadas

    CERRADA --> CONSOLIDADA : cierre del periodo académico
    CONSOLIDADA --> [*]

    note right of CERRADA
        Genera hash de integridad.
        Toda modificación posterior
        exige autorización explícita
        y queda en el historial.
    end note

    note right of CONSOLIDADA
        Estado terminal absoluto.
        Las notas pasan al expediente
        y ya no admiten cambios.
    end note
```

**Justificación.** `PARCIALMENTE_PUBLICADA` refleja la realidad operativa: un docente publica el primer parcial mucho antes que el examen final. Modelar solo `ABIERTA` y `CERRADA` obligaría a mantener todo oculto hasta el final del semestre, algo que ningún docente aceptaría y que contradice la expectativa del estudiante de "consulta oportuna de calificaciones".

La cadena `CERRADA → REABIERTA → COMPLETA → CERRADA` permite corregir errores legítimos sin destruir la trazabilidad, mientras `CONSOLIDADA` establece un punto de no retorno definitivo al cierre del periodo.

---

## 10. Máquina de estados — Transacción de pago (RF-05)

```mermaid
stateDiagram-v2
    direction LR

    [*] --> CREADA : se genera la orden de pago

    CREADA --> ENVIADA : se remite a la pasarela
    CREADA --> ANULADA : el estudiante desiste antes de pagar

    ENVIADA --> APROBADA : la pasarela confirma
    ENVIADA --> RECHAZADA : fondos insuficientes o error del medio
    ENVIADA --> INDETERMINADA : sin respuesta de la pasarela

    INDETERMINADA --> APROBADA : la conciliación confirma el pago
    INDETERMINADA --> RECHAZADA : la conciliación confirma el fallo
    INDETERMINADA --> INDETERMINADA : reintento con retroceso exponencial
    INDETERMINADA --> VENCIDA : se agotan los reintentos

    RECHAZADA --> ENVIADA : el estudiante reintenta con otro medio
    APROBADA --> CONCILIADA : coincide con el extracto de tesorería
    APROBADA --> EN_DISPUTA : discrepancia detectada
    EN_DISPUTA --> CONCILIADA : resuelta a favor
    EN_DISPUTA --> REVERSADA : resuelta en contra

    CONCILIADA --> [*]
    REVERSADA --> [*]
    ANULADA --> [*]
    VENCIDA --> [*]

    note right of INDETERMINADA
        Estado crítico. La clave de
        idempotencia garantiza que los
        reintentos no generen doble cobro.
    end note
```

**Justificación.** `INDETERMINADA` es el estado que la mayoría de los diseños omite y el que más incidentes produce en producción: la pasarela no respondió, pero el cargo pudo haberse aplicado. Modelarlo explícitamente obliga a resolverlo mediante conciliación, y la clave de idempotencia garantiza que los reintentos consulten el estado real en lugar de generar un segundo cobro.

`EN_DISPUTA` y `REVERSADA` dan soporte al requisito de trazabilidad de cada transacción para efectos de auditoría que expresa el personal de tesorería.

---

## 11. Proceso de escalado bajo demanda (RNF-02)

```mermaid
flowchart TD
    subgraph ENT["Tráfico entrante"]
        A["Más de 15.000 estudiantes<br/>primer día de inscripciones"]:::cli
    end

    subgraph BORDE["Capa de borde"]
        B["WAF filtra tráfico malicioso"]:::gw
        C{"¿Excede la cuota<br/>por usuario?"}:::gw
        D["429 Too Many Requests<br/>con Retry-After"]:::err
        E["Balanceador distribuye<br/>con verificación de salud"]:::gw
    end

    subgraph APP["Capa de aplicación"]
        F["Pool del Módulo Matrícula<br/>escala de 4 a 40 instancias"]:::mod
        G["Pool del Módulo Evaluación<br/>permanece en su base"]:::aislado
        H["Pool del Módulo Académico<br/>escala moderadamente"]:::mod
        I["Pool del Módulo Identidad<br/>escala con los inicios de sesión"]:::mod
    end

    subgraph AUTO["Control de autoescalado"]
        J["Recolectar métricas de CPU,<br/>cola y percentil 95"]:::auto
        K{"¿CPU mayor a 70 %<br/>durante 2 min?"}:::auto
        L["Escalar hacia afuera<br/>añadir instancias"]:::auto
        M{"¿CPU menor a 30 %<br/>durante 10 min?"}:::auto
        N["Escalar hacia adentro<br/>con drenado elegante"]:::auto
        O["Periodo de enfriamiento<br/>evita la oscilación"]:::auto
    end

    subgraph OBS["Observabilidad"]
        P["Métricas, registros y trazas<br/>centralizados"]:::obs
        Q{"¿Degradación<br/>mayor al 20 %?"}:::obs
        R["Alertar al Equipo de Plataforma<br/>en menos de 5 min"]:::err
    end

    A --> B --> C
    C -->|Sí| D --> A
    C -->|No| E
    E --> F
    E --> H
    E --> I
    F --> J
    H --> J
    I --> J
    J --> K
    K -->|Sí| L --> O --> J
    K -->|No| M
    M -->|Sí| N --> O
    M -->|No| J

    F -.telemetría.-> P
    G -.telemetría.-> P
    H -.telemetría.-> P
    I -.telemetría.-> P
    P --> Q
    Q -->|Sí| R
    Q -->|No| P

    classDef cli fill:#F5F5F5,stroke:#5F6368,color:#000
    classDef gw fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef mod fill:#E6F4EA,stroke:#137333,color:#000
    classDef aislado fill:#E8F0FE,stroke:#1967D2,stroke-width:3px,color:#000
    classDef auto fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef obs fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef err fill:#FCE8E6,stroke:#C5221F,color:#000
```

### Justificación — RNF-02

| Decisión | Razón |
|---|---|
| **Limitación de tasa antes del autoescalado, no en su lugar** | Son mecanismos complementarios con tiempos de reacción distintos: la limitación actúa en milisegundos; el autoescalado tarda decenas de segundos en aprovisionar. Sin el primero, el pico inicial tumba el módulo antes de que el segundo reaccione. |
| **Umbrales asimétricos: 70 % / 2 min para crecer, 30 % / 10 min para reducir** | Escalar hacia arriba debe ser agresivo (el costo de sub-aprovisionar es la caída en el día más visible del año); escalar hacia abajo debe ser conservador (el costo de sobre-aprovisionar unos minutos es marginal frente al riesgo de un segundo pico). |
| **Periodo de enfriamiento obligatorio** | Impide la oscilación, donde el sistema crea y destruye instancias sin cesar y termina peor que sin autoescalado. |
| **Drenado elegante al reducir** | Retirar una instancia con peticiones en vuelo dejaría matrículas a medio procesar en estados inconsistentes. |
| **El Módulo Evaluación aparece sin escalar** | Está en el diagrama precisamente para mostrar que **no cambia**. Es la refutación gráfica del problema heredado de *Fragilidad Ante Picos de Demanda*: en el monolito, saturar la matrícula degradaba la consulta de notas. |

---

## 12. Proceso de contención por cupo bajo alta concurrencia

Situación crítica: N estudiantes compiten simultáneamente por el último cupo de un grupo.

```mermaid
sequenceDiagram
    autonumber
    participant E1 as Estudiante A
    participant E2 as Estudiante B
    participant E3 as Estudiante C
    participant M1 as Instancia Matrícula 1
    participant M2 as Instancia Matrícula 2
    participant ACA as Módulo Académico
    participant DB as BD Académico

    Note over E1,DB: Grupo con cupoMaximo 30, cupoOcupado 29, versión 41

    par Tres solicitudes simultáneas
        E1->>M1: POST /matriculas grupo X
    and
        E2->>M2: POST /matriculas grupo X
    and
        E3->>M1: POST /matriculas grupo X
    end

    M1->>ACA: GET disponibilidad grupo X
    M2->>ACA: GET disponibilidad grupo X
    ACA->>DB: SELECT grupo
    DB-->>ACA: cupoOcupado 29, versión 41
    ACA-->>M1: Hay cupo, versión 41
    ACA-->>M2: Hay cupo, versión 41

    Note over M1,M2: Las tres solicitudes creen tener cupo.<br/>Aquí es donde un diseño ingenuo falla.

    M1->>ACA: POST reservar-cupo grupo X, versión 41
    activate ACA
    ACA->>DB: UPDATE grupo SET cupoOcupado 30, version 42 WHERE version 41
    DB-->>ACA: 1 fila afectada
    ACA-->>M1: 200 Cupo reservado
    deactivate ACA
    M1-->>E1: 201 Matrícula confirmada

    M2->>ACA: POST reservar-cupo grupo X, versión 41
    activate ACA
    ACA->>DB: UPDATE grupo SET cupoOcupado 30, version 42 WHERE version 41
    DB-->>ACA: 0 filas afectadas, conflicto de versión
    ACA->>DB: SELECT grupo
    DB-->>ACA: cupoOcupado 30, versión 42, sin cupo
    ACA-->>M2: 409 Cupo agotado
    deactivate ACA
    M2-->>E2: 409 El grupo se llenó

    M1->>ACA: POST reservar-cupo grupo X, versión 41
    activate ACA
    ACA->>DB: UPDATE con versión obsoleta
    DB-->>ACA: 0 filas afectadas
    ACA-->>M1: 409 Cupo agotado
    deactivate ACA
    M1-->>E3: 409 El grupo se llenó

    Note over E1,DB: Resultado: exactamente un cupo asignado.<br/>Sin bloqueo pesimista ni serialización global.
```

### Justificación — control de concurrencia

**Se usa bloqueo optimista, no pesimista.** Un bloqueo pesimista sobre el grupo serializaría a todos los estudiantes que intentan matricular esa asignatura: con 15.000 usuarios concurrentes, la fila de espera haría inviable el requisito de degradación ≤ 20 %.

El bloqueo optimista permite que todas las lecturas ocurran en paralelo y solo serializa la escritura final, que es una operación de milisegundos. El conflicto se detecta por el contador de versión: `UPDATE ... WHERE version = 41` afecta cero filas si otra transacción ya incrementó la versión.

**El conflicto no es un error del sistema, es información válida de negocio.** El estudiante recibe un `409` claro —"el grupo se llenó"— en lugar de un tiempo de espera agotado o un error genérico. La rama de reintento del [proceso de matrícula](#2-proceso-de-matrícula--rf-02) relee el estado antes de rendirse, cubriendo el caso en que otro estudiante haya cancelado en el intervalo.

**Este mecanismo es la razón por la cual `Grupo` concentra el cupo** en la [Vista Lógica](02-vista-logica.md): si el contador viviera en `Asignatura`, toda la asignatura sería un punto único de serialización global en lugar de un grupo individual.

---

## Trazabilidad de la Vista de Procesos

| Requisito / Atributo | Elemento que lo materializa |
|---|---|
| RF-01 · Autenticación centralizada | Proceso 1 · Máquina de estados de sesión |
| RF-02 · Verificación financiera obligatoria | Proceso 2 · Máquina de estados de matrícula |
| RF-03 · Sin conflictos de programación | Proceso 3 · Máquina de estados de reserva |
| RF-04 · Notificación desacoplada | Proceso 4 · Máquina de estados de acta |
| RF-05 · Integración desacoplada | Proceso 5 · Máquina de estados de pago |
| RNF-01 · Disponibilidad | Fallo rápido y modo degradado en procesos 2 y 5 |
| RNF-02 · 15.000 concurrentes | Proceso 11 · Proceso 12 |
| RNF-03 · Seguridad y auditoría | Proceso 1, ramas de rechazo y registro |
| RNF-04 · Circuit Breaker y reintentos | Proceso 2 asíncrono · Proceso 5 |
| RNF-05 · Detección en menos de 5 min | Bucle de observabilidad del proceso 11 |

---

| ← Anterior | Índice | Siguiente → |
|---|---|---|
| [Vista Lógica](02-vista-logica.md) | [README](../README.md) | [Vista de Desarrollo](04-vista-desarrollo.md) |
