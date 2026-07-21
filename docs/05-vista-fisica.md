# Vista 4 — Física (Despliegue)

> **Modelo 4+1 · Vista Física.** Mapea los artefactos de software sobre nodos de infraestructura reales: servidores, zonas de disponibilidad, redes, bases de datos, servicios gestionados y clientes. Es la única vista que puede demostrar el cumplimiento de los requisitos de disponibilidad, escalabilidad y resiliencia. Su destinatario es el equipo de plataforma y operaciones.

**Cobertura:** despliegue multi-zona · segmentación de red · topología de datos · autoescalado · canalización de despliegue · recuperación ante fallos. **Total: 6 diagramas.**

---

## Convención de notación

Mermaid.js no implementa el diagrama UML de despliegue. Se emplea `flowchart` con esta convención:

| Elemento UML | Representación |
|---|---|
| Nodo de ejecución | `subgraph` con estereotipo |
| Artefacto desplegado | Rectángulo con extensión de archivo |
| Base de datos | Nodo cilíndrico `[( )]` |
| Servicio gestionado | Rectángulo con estereotipo *managed* |
| Canal de comunicación | Flecha etiquetada con el protocolo |
| Frontera de red | `subgraph` anidado con color según nivel de exposición |

---

## 1. Diagrama de despliegue general

```mermaid
flowchart TB
    subgraph CLI["Nodos cliente"]
        direction LR
        N1["Navegador de escritorio<br/><i>SPA · HTTPS/TLS 1.2+</i>"]:::cli
        N2["Navegador móvil<br/><i>web adaptativa</i>"]:::cli
        N3["Estación administrativa<br/><i>Tesorería · Seguridad</i>"]:::cli
    end

    INET["Internet pública"]:::net

    subgraph EDGE["Nodo de borde global · servicio gestionado"]
        direction LR
        E1["CDN<br/><i>recursos estáticos de la SPA</i>"]:::edge
        E2["WAF<br/><i>reglas OWASP Top 10</i>"]:::edge
        E3["Certificado TLS gestionado<br/><i>renovación automática</i>"]:::edge
        E4["Protección contra denegación<br/>de servicio"]:::edge
    end

    subgraph REG["Región cloud · Colombia / LATAM"]
        direction TB

        subgraph PUB["Subred pública"]
            direction LR
            LB["Balanceador de carga<br/><i>gestionado · multizona</i><br/>verificación de salud cada 10 s<br/>terminación TLS"]:::pub
            GWN["Nodo API Gateway<br/><i>PaaS · autoescalado 2-8</i><br/><b>gateway.jar</b>"]:::pub
        end

        subgraph AZA["Zona de Disponibilidad A"]
            direction TB
            subgraph APPA["Plan de servicio · subred privada A"]
                direction LR
                PA1["Pool Identidad<br/><b>identidad.jar</b><br/><i>2 a 6 instancias</i>"]:::app
                PA2["Pool Matrícula<br/><b>matricula.jar</b><br/><i>4 a 40 instancias</i>"]:::app
                PA3["Pool Académico<br/><b>academico.jar</b><br/><i>2 a 10 instancias</i>"]:::app
                PA4["Pool Evaluación<br/><b>evaluacion.jar</b><br/><i>2 a 12 instancias</i>"]:::app
                PA5["Pool Integración<br/><b>integracion.jar</b><br/><i>2 a 10 instancias</i>"]:::app
            end
            subgraph DBA["Subred aislada A"]
                direction LR
                DA1[("db_identidad<br/><i>primaria</i>")]:::dbp
                DA2[("db_matricula<br/><i>primaria</i>")]:::dbp
                DA3[("db_academico<br/><i>primaria</i>")]:::dbp
                DA4[("db_evaluacion<br/><i>primaria</i>")]:::dbp
                DA5[("db_integracion<br/><i>primaria</i>")]:::dbp
            end
        end

        subgraph AZB["Zona de Disponibilidad B"]
            direction TB
            subgraph APPB["Plan de servicio · subred privada B"]
                direction LR
                PB1["Pool Identidad<br/><b>identidad.jar</b><br/><i>2 a 6 instancias</i>"]:::app
                PB2["Pool Matrícula<br/><b>matricula.jar</b><br/><i>4 a 40 instancias</i>"]:::app
                PB3["Pool Académico<br/><b>academico.jar</b><br/><i>2 a 10 instancias</i>"]:::app
                PB4["Pool Evaluación<br/><b>evaluacion.jar</b><br/><i>2 a 12 instancias</i>"]:::app
                PB5["Pool Integración<br/><b>integracion.jar</b><br/><i>2 a 10 instancias</i>"]:::app
            end
            subgraph DBB["Subred aislada B"]
                direction LR
                DB1[("db_identidad<br/><i>réplica síncrona</i>")]:::dbr
                DB2[("db_matricula<br/><i>réplica síncrona</i>")]:::dbr
                DB3[("db_academico<br/><i>réplica síncrona</i>")]:::dbr
                DB4[("db_evaluacion<br/><i>réplica síncrona</i>")]:::dbr
                DB5[("db_integracion<br/><i>réplica síncrona</i>")]:::dbr
            end
        end

        subgraph SVC["Servicios gestionados regionales"]
            direction LR
            MQ["Cola de mensajes<br/><i>replicada entre zonas</i>"]:::svc
            OBS["Consola de observabilidad<br/><i>registros · métricas · alertas</i>"]:::svc
            VLT["Gestor de secretos<br/><i>credenciales · claves</i>"]:::svc
            BKP["Almacenamiento de respaldos<br/><i>cifrado · recuperación puntual</i>"]:::svc
        end
    end

    subgraph EXTP["Proveedores externos"]
        direction LR
        X1["Proveedor OIDC"]:::ext
        X2["Pasarela de Pagos"]:::ext
        X3["Biblioteca Digital"]:::ext
        X4["Gestión Documental"]:::ext
    end

    N1 --> INET
    N2 --> INET
    N3 --> INET
    INET -->|HTTPS 443| EDGE
    EDGE -->|HTTPS| LB
    LB --> GWN
    GWN -->|REST interno mTLS| APPA
    GWN -->|REST interno mTLS| APPB
    GWN -.valida token.-> X1

    PA1 --> DA1
    PA2 --> DA2
    PA3 --> DA3
    PA4 --> DA4
    PA5 --> DA5

    PB1 -.escritura a primaria.-> DA1
    PB2 -.escritura a primaria.-> DA2
    PB3 -.escritura a primaria.-> DA3
    PB4 -.escritura a primaria.-> DA4
    PB5 -.escritura a primaria.-> DA5

    DA1 <-->|replicación síncrona| DB1
    DA2 <-->|replicación síncrona| DB2
    DA3 <-->|replicación síncrona| DB3
    DA4 <-->|replicación síncrona| DB4
    DA5 <-->|replicación síncrona| DB5

    APPA --> MQ
    APPB --> MQ
    APPA -.telemetría.-> OBS
    APPB -.telemetría.-> OBS
    GWN -.telemetría.-> OBS
    APPA -.credenciales.-> VLT
    APPB -.credenciales.-> VLT
    DBA -.respaldo continuo.-> BKP
    DBB -.respaldo continuo.-> BKP

    PA5 -->|REST + CircuitBreaker| X2
    PA5 --> X3
    PA5 --> X4
    PB5 -->|REST + CircuitBreaker| X2
    PB5 --> X3
    PB5 --> X4

    classDef cli fill:#F5F5F5,stroke:#5F6368,color:#000
    classDef net fill:#FFFFFF,stroke:#5F6368,color:#000
    classDef edge fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef pub fill:#E8F0FE,stroke:#1967D2,stroke-width:2px,color:#000
    classDef app fill:#E6F4EA,stroke:#137333,color:#000
    classDef dbp fill:#F3E8FD,stroke:#8430CE,stroke-width:2px,color:#000
    classDef dbr fill:#F3E8FD,stroke:#8430CE,stroke-dasharray:4 3,color:#000
    classDef svc fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
```

### Justificación

**Dos zonas de disponibilidad en configuración activo-activo, no activo-pasivo.** Ambas zonas reciben tráfico permanentemente a través del balanceador. Un esquema activo-pasivo dejaría capacidad ociosa pagada durante todo el año y, más grave, la zona pasiva estaría fría y no probada el día que se necesite. Con activo-activo, la falla de una zona reduce la capacidad a la mitad pero **no produce indisponibilidad**, que es lo que exige el objetivo de 99,9 %.

**Escritura siempre contra la instancia primaria; la réplica existe para promoción.** Se descarta deliberadamente una configuración multi-maestro. Un sistema de cupos limitados con escritura en dos maestros produce conflictos de reserva irresolubles: dos estudiantes obtendrían el mismo último lugar. La primaria única preserva la consistencia que RF-02 exige, y la replicación síncrona ofrece un RPO cercano a cero sin sacrificarla. El costo aceptado es latencia de escritura entre zonas, del orden de milisegundos dentro de una misma región.

**El mismo artefacto se despliega en ambas zonas.** `matricula.jar` es binariamente idéntico en A y en B; solo cambia la configuración inyectada. Cualquier diferencia entre zonas sería un fallo latente que se manifestaría durante una conmutación — el peor momento posible para descubrirlo.

**Cola y observabilidad son servicios regionales, no por zona.** Si la cola viviera dentro de la zona A, su caída perdería los mensajes de reconciliación de pago y la garantía de resiliencia se rompería justo cuando más se necesita. Lo mismo aplica a la observabilidad: un sistema de monitoreo que muere con la zona que debía vigilar es inútil por definición.

---

## 2. Segmentación de red y superficie de exposición

```mermaid
flowchart TB
    INET["🌐 Internet pública"]:::inet

    subgraph N1["NIVEL 1 · Subred pública — expuesta a Internet"]
        direction LR
        A1["WAF + CDN"]:::pub
        A2["Balanceador de carga<br/>puerto 443 únicamente"]:::pub
        A3["API Gateway<br/>única entrada al sistema"]:::pub
    end

    subgraph N2["NIVEL 2 · Subred privada — sin dirección IP pública"]
        direction LR
        B1["Pool Identidad"]:::priv
        B2["Pool Matrícula"]:::priv
        B3["Pool Académico"]:::priv
        B4["Pool Evaluación"]:::priv
        B5["Pool Integración"]:::priv
    end

    subgraph N3["NIVEL 3 · Subred aislada — sin ruta hacia Internet"]
        direction LR
        C1[("db_identidad")]:::iso
        C2[("db_matricula")]:::iso
        C3[("db_academico")]:::iso
        C4[("db_evaluacion")]:::iso
        C5[("db_integracion")]:::iso
    end

    subgraph N4["Salida controlada"]
        D1["Pasarela NAT<br/>solo tráfico saliente"]:::nat
    end

    EXT["Proveedores externos<br/>Pagos · Biblioteca · Documental · OIDC"]:::ext

    INET -->|443| A1
    A1 --> A2 --> A3
    A3 -->|mTLS · puertos internos| B1
    A3 -->|mTLS| B2
    A3 -->|mTLS| B3
    A3 -->|mTLS| B4
    A3 -->|mTLS| B5

    B1 -->|5432 solo desde su pool| C1
    B2 -->|5432| C2
    B3 -->|5432| C3
    B4 -->|5432| C4
    B5 -->|5432| C5

    B5 --> D1 --> EXT
    A3 -.OIDC.-> D1

    INET -.->|❌ bloqueado| B2
    INET -.->|❌ bloqueado| C2
    B2 -.->|❌ bloqueado| C3

    classDef inet fill:#FCE8E6,stroke:#C5221F,stroke-width:3px,color:#000
    classDef pub fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef priv fill:#E6F4EA,stroke:#137333,color:#000
    classDef iso fill:#F3E8FD,stroke:#8430CE,stroke-width:3px,color:#000
    classDef nat fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
```

### Reglas de red aplicadas

| Origen | Destino | Puerto | Estado |
|---|---|---|---|
| Internet | Balanceador | 443 | ✅ Permitido |
| Internet | Pools de aplicación | cualquiera | ❌ Bloqueado |
| Internet | Bases de datos | cualquiera | ❌ Bloqueado |
| API Gateway | Pools de aplicación | interno mTLS | ✅ Permitido |
| Pool X | Base de datos X | 5432 | ✅ Permitido |
| Pool X | Base de datos Y | 5432 | ❌ Bloqueado |
| Pool Integración | NAT → Internet | 443 saliente | ✅ Permitido |
| Otros pools | NAT → Internet | cualquiera | ❌ Bloqueado |
| Bases de datos | Internet | cualquiera | ❌ Bloqueado |

### Justificación

**Los tres niveles de red convierten reglas de código en imposibilidades físicas.** La regla R7 de la [Vista de Desarrollo](04-vista-desarrollo.md) —"ningún módulo expone puerto público"— es una convención que un desarrollador podría violar por descuido. La segmentación de red hace que, aunque alguien publicara accidentalmente un endpoint sin autenticación, **no fuera alcanzable desde fuera**. La seguridad se sostiene por doble barrera: control de acceso en el Gateway y aislamiento de red por debajo.

**La regla R6 —una base de datos por módulo— también se aplica a nivel de red.** El pool de Matrícula no puede alcanzar `db_academico` ni siquiera si su código lo intentara. Esto elimina la vía más común de erosión arquitectónica: la consulta cruzada "temporal" que nunca se retira.

**Solo el pool de Integración tiene salida a Internet.** Los demás módulos no pueden contactar servicios externos aunque su código lo intentara. Esto refuerza a nivel de infraestructura la decisión de RF-05 de concentrar los adaptadores externos en un solo módulo, y reduce drásticamente la superficie de exfiltración en caso de compromiso.

---

## 3. Topología de datos, replicación y respaldo

```mermaid
flowchart TB
    subgraph ZA["Zona A — instancias primarias"]
        direction LR
        P1[("db_identidad<br/>primaria")]:::prim
        P2[("db_matricula<br/>primaria")]:::prim
        P3[("db_academico<br/>primaria")]:::prim
        P4[("db_evaluacion<br/>primaria")]:::prim
        P5[("db_integracion<br/>primaria")]:::prim
    end

    subgraph ZB["Zona B — réplicas síncronas"]
        direction LR
        R1[("db_identidad<br/>réplica")]:::rep
        R2[("db_matricula<br/>réplica")]:::rep
        R3[("db_academico<br/>réplica")]:::rep
        R4[("db_evaluacion<br/>réplica")]:::rep
        R5[("db_integracion<br/>réplica")]:::rep
    end

    subgraph RL["Réplicas de solo lectura"]
        direction LR
        L1[("réplica lectura<br/>db_matricula")]:::lec
        L2[("réplica lectura<br/>db_evaluacion")]:::lec
    end

    subgraph BK["Estrategia de respaldo"]
        direction TB
        B1["Instantánea completa<br/><i>diaria · retención 35 días</i>"]:::bkp
        B2["Registro de transacciones continuo<br/><i>recuperación a punto en el tiempo</i>"]:::bkp
        B3["Copia geográfica<br/><i>otra región · retención 12 meses</i>"]:::bkp
    end

    ESC["Escrituras<br/><i>desde ambas zonas</i>"]:::esc
    LEC["Lecturas analíticas<br/><i>paneles directivos</i>"]:::lect

    ESC --> P1
    ESC --> P2
    ESC --> P3
    ESC --> P4
    ESC --> P5

    P1 -->|síncrona| R1
    P2 -->|síncrona| R2
    P3 -->|síncrona| R3
    P4 -->|síncrona| R4
    P5 -->|síncrona| R5

    P2 -->|asíncrona| L1
    P4 -->|asíncrona| L2
    LEC --> L1
    LEC --> L2

    P1 --> B1
    P2 --> B1
    P3 --> B1
    P4 --> B1
    P5 --> B1
    B1 --> B2 --> B3

    classDef prim fill:#F3E8FD,stroke:#8430CE,stroke-width:3px,color:#000
    classDef rep fill:#F3E8FD,stroke:#8430CE,stroke-dasharray:4 3,color:#000
    classDef lec fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef bkp fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef esc fill:#E6F4EA,stroke:#137333,color:#000
    classDef lect fill:#E6F4EA,stroke:#137333,color:#000
```

### Objetivos de continuidad por módulo

| Base de datos | Criticidad | Replicación | RPO objetivo | RTO objetivo |
|---|---|---|---|---|
| `db_identidad` | Crítica — bloquea todo el sistema | Síncrona | ≈ 0 | < 15 min |
| `db_matricula` | Crítica — proceso de negocio central | Síncrona + lectura | ≈ 0 | < 15 min |
| `db_academico` | Alta — bloquea matrícula y evaluación | Síncrona | ≈ 0 | < 20 min |
| `db_evaluacion` | Media — tolera indisponibilidad breve | Síncrona + lectura | ≈ 0 | < 30 min |
| `db_integracion` | Alta — afecta pagos y certificados | Síncrona | ≈ 0 | < 20 min |

### Justificación

**Réplicas de lectura solo para Matrícula y Evaluación.** Son las dos bases con mayor volumen de consultas analíticas: expedientes para paneles directivos e históricos de calificaciones. Dirigir esas lecturas a una réplica asíncrona evita que la analítica compita por recursos con la operación transaccional durante el periodo de inscripciones. Las demás bases no justifican el costo adicional.

**Replicación síncrona para la réplica de conmutación, asíncrona para las de lectura.** La distinción es deliberada: la conmutación exige RPO cercano a cero y acepta el costo de latencia; la analítica tolera perfectamente unos segundos de retraso y no debe penalizar las escrituras.

**Copia geográfica en otra región con retención de 12 meses.** Protege contra la pérdida completa de una región y contra la corrupción lógica descubierta tardíamente —por ejemplo, una migración defectuosa detectada semanas después—. La replicación síncrona no protege contra este segundo caso: replica fielmente el error.

---

## 4. Configuración de autoescalado

```mermaid
flowchart LR
    subgraph MET["Métricas evaluadas"]
        direction TB
        M1["Uso de CPU"]:::met
        M2["Solicitudes en cola"]:::met
        M3["Latencia percentil 95"]:::met
        M4["Memoria en uso"]:::met
    end

    subgraph DEC["Motor de decisión"]
        direction TB
        D1{"¿CPU > 70 %<br/>durante 2 min?"}:::dec
        D2{"¿Cola > 100 solicitudes<br/>durante 1 min?"}:::dec
        D3{"¿CPU < 30 %<br/>durante 10 min?"}:::dec
        D4["Enfriamiento 5 min<br/>tras cada ajuste"]:::cool
    end

    subgraph ACC["Acciones"]
        direction TB
        A1["Escalar hacia afuera<br/>+50 % de instancias"]:::out
        A2["Escalar hacia adentro<br/>-25 % con drenado"]:::in
        A3["Mantener capacidad"]:::keep
    end

    M1 --> D1
    M2 --> D2
    M3 --> D1
    M4 --> D3
    D1 -->|Sí| A1
    D2 -->|Sí| A1
    D1 -->|No| D3
    D3 -->|Sí| A2
    D3 -->|No| A3
    A1 --> D4
    A2 --> D4
    D4 --> M1

    classDef met fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef dec fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef cool fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef out fill:#E6F4EA,stroke:#137333,color:#000
    classDef in fill:#E6F4EA,stroke:#137333,stroke-dasharray:4 3,color:#000
    classDef keep fill:#F5F5F5,stroke:#5F6368,color:#000
```

### Parámetros por módulo

| Módulo | Mínimo | Máximo | Umbral de crecimiento | Justificación del rango |
|---|---|---|---|---|
| **API Gateway** | 2 | 8 | CPU 60 % | Toda petición pasa por él; umbral más bajo para anticiparse |
| **Identidad** | 2 | 6 | CPU 70 % | Picos en inicio de sesión, resueltos con caché de sesión |
| **Matrícula** | 4 | 40 | CPU 70 % o cola > 100 | Módulo del pico estacional; rango más amplio del sistema |
| **Académico** | 2 | 10 | CPU 70 % | Consultado por Matrícula durante el pico, con carga proporcional |
| **Evaluación** | 2 | 12 | CPU 70 % | Su pico es el cierre de notas, desfasado del de matrícula |
| **Integración** | 2 | 10 | CPU 70 % | Limitado además por la capacidad de los proveedores externos |

### Justificación

**Los rangos son deliberadamente asimétricos, y esa asimetría es la refutación del problema heredado.** En el sistema monolítico, escalar significaba replicar la aplicación completa por culpa de un solo módulo saturado. Aquí, el primer día de inscripciones el pool de Matrícula crece hasta diez veces mientras Evaluación permanece en su base — y un docente consultando notas no percibe ninguna degradación.

**El mínimo de 2 instancias en todos los pools no es capacidad, es tolerancia a fallo.** Garantiza al menos una instancia viva por zona de disponibilidad: si una zona cae, el servicio continúa.

**Umbrales asimétricos entre crecer y reducir.** Escalar hacia arriba es agresivo (70 % durante 2 minutos) porque el costo de sub-aprovisionar es la caída del servicio en el día más visible del año. Escalar hacia abajo es conservador (30 % durante 10 minutos) porque el costo de mantener capacidad extra unos minutos es marginal frente al riesgo de un segundo pico.

**El drenado elegante al reducir es obligatorio.** Retirar una instancia con peticiones en vuelo dejaría matrículas a medio procesar en estados inconsistentes. El procedimiento —dejar de recibir tráfico nuevo, terminar lo en curso, luego apagar— es lo que hace seguro el escalado para un proceso transaccional.

**El módulo de Integración está limitado también por terceros.** Escalarlo a 40 instancias no aumentaría el rendimiento si la pasarela de pagos solo admite un número acotado de conexiones concurrentes; solo provocaría que el Circuit Breaker se abriera antes.

---

## 5. Canalización de despliegue

```mermaid
flowchart LR
    subgraph DEV["Desarrollo"]
        direction TB
        D1["Rama de característica"]:::dev
        D2["Solicitud de integración"]:::dev
    end

    subgraph CI["Integración continua"]
        direction TB
        C1["Compilar módulo<br/>afectado"]:::ci
        C2["Pruebas unitarias<br/>del dominio"]:::ci
        C3["Reglas arquitectónicas<br/>R1 a R8"]:::arch
        C4["Pruebas de integración"]:::ci
        C5["Pruebas de contrato"]:::ci
        C6["Análisis de seguridad"]:::ci
        C7["Publicar artefacto<br/>versionado"]:::ci
    end

    subgraph ENV["Ambientes"]
        direction TB
        E1["Desarrollo<br/><i>zona única</i>"]:::env
        E2["Preproducción<br/><i>espejo de producción</i>"]:::env
        E3["Producción<br/><i>multizona</i>"]:::prod
    end

    subgraph DEP["Estrategia de despliegue en producción"]
        direction TB
        P1["Despliegue azul-verde<br/>por módulo"]:::dep
        P2["Verificación de salud<br/>del nuevo conjunto"]:::dep
        P3{"¿Métricas<br/>correctas?"}:::dec
        P4["Conmutar tráfico<br/>gradualmente"]:::dep
        P5["Reversión automática<br/>al conjunto anterior"]:::err
    end

    D1 --> D2 --> C1 --> C2 --> C3
    C3 -->|Cumple| C4 --> C5 --> C6 --> C7
    C3 -->|Viola| FAIL["Build fallido"]:::err
    C7 --> E1 --> E2 --> E3
    E3 --> P1 --> P2 --> P3
    P3 -->|Sí| P4
    P3 -->|No| P5

    classDef dev fill:#F5F5F5,stroke:#5F6368,color:#000
    classDef ci fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef arch fill:#FEF7E0,stroke:#EA8600,stroke-width:3px,color:#000
    classDef env fill:#E6F4EA,stroke:#137333,color:#000
    classDef prod fill:#E6F4EA,stroke:#137333,stroke-width:3px,color:#000
    classDef dep fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef dec fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef err fill:#FCE8E6,stroke:#C5221F,color:#000
```

### Justificación

**Solo se compila y despliega el módulo afectado.** Es el beneficio operativo directo de la arquitectura modular: un cambio en Evaluación no requiere probar ni redesplegar Matrícula. Esto resuelve el problema heredado de *Acoplamiento Estructural*, donde cualquier cambio obligaba a validar la aplicación completa.

**Despliegue azul-verde por módulo con reversión automática.** Se levanta el conjunto nuevo en paralelo, se verifica su salud y solo entonces se conmuta el tráfico gradualmente. Si las métricas se degradan, la reversión es inmediata porque el conjunto anterior sigue activo. Durante el periodo de matrícula esta capacidad es indispensable: un despliegue defectuoso a las 8 de la mañana del primer día de inscripciones debe poder revertirse en segundos, no en el tiempo de un redespliegue.

**Las credenciales nunca viajan en el artefacto.** Se inyectan desde el gestor de secretos en tiempo de ejecución. Los artefactos son idénticos entre ambientes; solo cambia la configuración. Esto permite rotar credenciales sin redesplegar y cumple los requisitos de protección de datos de la normativa aplicable.

---

## 6. Procedimiento de conmutación ante falla de zona

```mermaid
sequenceDiagram
    autonumber
    participant HC as Verificación de salud
    participant LB as Balanceador de carga
    participant AZA as Zona A
    participant AZB as Zona B
    participant DBS as Servicio de base de datos
    participant AS as Autoescalado
    participant OBS as Observabilidad
    actor OPS as Equipo de Plataforma

    Note over AZA: Falla total de la Zona A

    HC->>AZA: Sondeo de salud
    AZA--xHC: Sin respuesta
    HC->>AZA: Reintento 2 y 3
    AZA--xHC: Sin respuesta
    Note over HC: Detección ≈ 30 s

    HC->>LB: Marcar Zona A como no disponible
    LB->>LB: Retirar de la rotación
    Note over LB: ≈ 10 s

    LB->>AZB: Dirigir el 100 % del tráfico
    activate AZB
    Note over AZB: Capacidad al 50 %.<br/>El servicio continúa disponible.

    HC->>OBS: Emitir alerta crítica
    OBS->>OPS: Notificación en menos de 5 min

    DBS->>DBS: Detectar pérdida de las primarias
    DBS->>AZB: Promover réplicas a primarias
    Note over DBS,AZB: ≈ 3 a 5 min.<br/>RPO ≈ 0 por replicación síncrona.
    DBS-->>AZB: Nuevas primarias operativas

    AZB->>AS: Métricas de CPU elevadas
    AS->>AZB: Escalar hacia afuera en la Zona B
    Note over AS,AZB: ≈ 5 a 10 min hasta<br/>recuperar capacidad plena

    AZB-->>LB: Capacidad restablecida
    deactivate AZB

    Note over HC,OPS: RTO total ≈ 15 min, por debajo del objetivo de 30 min

    Note over AZA: Restablecimiento de la Zona A

    AZA->>HC: Sondeo exitoso
    HC->>DBS: Reconstruir réplicas en la Zona A
    DBS->>AZA: Sincronizar desde las nuevas primarias
    AZA->>LB: Solicitar reincorporación
    LB->>AZA: Reincorporar gradualmente
    AS->>AZB: Reducir capacidad excedente
    Note over LB,AS: Retorno a operación normal<br/>sin interrupción del servicio
```

### Verificación de objetivos de continuidad

| Etapa | Tiempo estimado |
|---|---|
| Detección de la falla por verificación de salud | ≈ 30 s |
| Retiro de la zona de la rotación del balanceador | ≈ 10 s |
| Promoción de réplicas a primarias | 3 – 5 min |
| Recuperación de capacidad por autoescalado | 5 – 10 min |
| **RTO total** | **≈ 15 min** ✅ (objetivo < 30 min) |
| **RPO** por replicación síncrona | **≈ 0** ✅ (objetivo < 15 min) |

### Justificación

**El servicio nunca queda indisponible, solo degradado.** Entre la detección y la promoción de réplicas, la Zona B sigue atendiendo tráfico con capacidad reducida. Esta es la diferencia práctica entre una arquitectura activo-activo y una activo-pasivo: en la segunda, los mismos 15 minutos habrían sido de caída total.

**La reincorporación de la zona restablecida es gradual.** Devolver el 50 % del tráfico de golpe a instancias recién levantadas, con cachés vacías y conexiones frías, produciría una degradación visible. La reincorporación progresiva permite que las instancias se calienten.

**El procedimiento no requiere intervención humana para ejecutarse.** El equipo de plataforma recibe la alerta para supervisar y diagnosticar la causa raíz, pero la conmutación es automática. Un procedimiento que dependa de que alguien esté disponible a las 3 de la mañana no puede sustentar un objetivo de 99,9 %.

---

## Mapeo artefacto → nodo → requisito

| Artefacto | Nodo de despliegue | Instancias | Requisitos que satisface |
|---|---|---|---|
| `gateway.jar` | Subred pública, ambas zonas | 2 – 8 | RF-01, RNF-03 |
| `identidad.jar` | Subred privada A y B | 2 – 6 | RF-01 |
| `matricula.jar` | Subred privada A y B | 4 – 40 | RF-02, RNF-02 |
| `academico.jar` | Subred privada A y B | 2 – 10 | RF-03 |
| `evaluacion.jar` | Subred privada A y B | 2 – 12 | RF-04 |
| `integracion.jar` | Subred privada A y B | 2 – 10 | RF-05, RNF-04 |
| 5 bases de datos gestionadas | Subred aislada, primaria + réplica | 1 + 1 cada una | RNF-01, RNF-04 |
| Cola de mensajes | Servicio regional replicado | — | RF-04, RNF-04 |
| Consola de observabilidad | Servicio regional | — | RNF-05 |
| Gestor de secretos | Servicio regional | — | RNF-03 |
| WAF y CDN | Borde global | — | RNF-03 |

---

## Trazabilidad de la Vista Física

| Requisito | Elemento de infraestructura que lo satisface |
|---|---|
| **RNF-01** · Disponibilidad ≥ 99,9 % | Despliegue activo-activo en dos zonas, balanceador con verificación de salud, mínimo 2 instancias por pool |
| **RNF-02** · 15.000 concurrentes | Autoescalado de Matrícula hasta 40 instancias, limitación de tasa en el Gateway, réplicas de lectura |
| **RNF-03** · Seguridad y protección de datos | Segmentación en tres niveles de red, WAF, TLS gestionado, gestor de secretos, mTLS interno |
| **RNF-04** · RTO < 30 min, RPO < 15 min | Replicación síncrona, promoción automática, respaldo continuo, copia geográfica |
| **RNF-05** · Detección < 5 min | Consola de observabilidad regional con telemetría de los seis componentes |
| **Aislamiento entre módulos** | Pools independientes con autoescalado propio, bases de datos separadas, reglas de red por pool |
| **Costos elásticos** | Escalado hacia adentro fuera de los picos, servicios gestionados sin infraestructura propia |

---

## Cierre

Las cinco vistas del modelo 4+1 quedan documentadas con **48 diagramas** y trazabilidad completa hacia los 5 requisitos funcionales, los 5 requisitos no funcionales y los 11 actores identificados. Cada decisión arquitectónica registrada en el documento fuente tiene al menos una representación gráfica verificable en alguna de las vistas.

---

| ← Anterior | Índice | Siguiente → |
|---|---|---|
| [Vista de Desarrollo](04-vista-desarrollo.md) | [README](../README.md) | — |
