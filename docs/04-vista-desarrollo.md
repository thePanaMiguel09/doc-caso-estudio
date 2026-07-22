# Vista 3 — Desarrollo

> **Modelo 4+1 · Vista de Desarrollo.** Describe la organización estática del código: capas, paquetes, componentes, interfaces y dependencias de compilación. Su destinatario es el equipo de desarrollo y los ingenieros de plataforma.

**Cobertura:** los **4 diagramas exigidos por esta vista** (capas, paquetes, componentes y dependencias entre componentes), más 5 estructuras internas de módulo, 1 de bibliotecas compartidas y 1 de reglas de dependencia. **Total: 11 diagramas.**

---

## Los cuatro diagramas requeridos por esta vista

Esta vista debe entregar cuatro diagramas. Cada uno está en su propia sección, con un único bloque Mermaid listo para copiar:

| # | Diagrama requerido | Sección | Qué muestra |
|---|---|---|---|
| 1 | **Diagrama de capas** | [§1](#1-diagrama-de-capas) | Las cuatro capas de la arquitectura y la dirección de dependencia entre ellas |
| 2 | **Diagrama de paquetes** | [§2](#2-diagrama-de-paquetes) | Los paquetes de código y sus relaciones de importación |
| 3 | **Diagrama de componentes** | [§3](#3-diagrama-de-componentes) | Los componentes con sus interfaces provistas y requeridas |
| 4 | **Diagrama de dependencias entre componentes** | [§4](#4-diagrama-de-dependencias-entre-componentes) | El grafo dirigido de dependencias, clasificado por tipo |

Las secciones [§5](#5-estructura-interna--módulo-identidad-y-accesos-rf-01) a [§9](#9-estructura-interna--módulo-integración-externa-y-analítica-rf-05) detallan la estructura interna de cada módulo (arquitectura hexagonal) como complemento, y las secciones [§10](#10-bibliotecas-compartidas) a [§12](#12-organización-del-repositorio) documentan las bibliotecas compartidas, las reglas de dependencia y la organización del repositorio.

---

## Regla de oro de esta vista

Toda la organización del código está gobernada por una única regla verificable:

> **Ninguna dependencia de compilación cruza horizontalmente entre módulos de negocio, y ninguna dependencia apunta desde el dominio hacia la infraestructura.**

Si esa regla se rompe en un solo punto, el sistema deja de ser una arquitectura modular orientada a servicios y vuelve a ser el monolito acoplado que el caso de estudio describe como problema. Los diagramas de esta vista existen para hacer esa regla auditable, y la [sección 11](#11-reglas-de-dependencia-verificables-en-integración-continua) la convierte en pruebas automáticas del pipeline.

---

## Convención de notación

Mermaid.js no implementa nativamente los diagramas UML de paquetes y componentes. Se emplea `flowchart` con esta convención, uniforme en toda la vista:

| Elemento UML | Representación |
|---|---|
| Capa | `subgraph` con banda de color |
| Paquete | `subgraph` o rectángulo con nombre en notación de punto (`ups.matricula`) |
| Componente | Rectángulo con estereotipo en cursiva |
| Interfaz provista | Nodo hexagonal `{{ }}` con prefijo `I` |
| Dependencia de compilación | Flecha continua `-->` (o `==>` para resaltar capas) |
| Relación de importación entre paquetes | Flecha punteada `-.->` etiquetada `«import»` |
| Dependencia de ejecución síncrona (REST) | Flecha punteada `-.->` etiquetada `REST` |
| Dependencia de ejecución asíncrona (cola) | Flecha punteada `-.->` etiquetada `evento` |
| Base de datos | Nodo cilíndrico `[( )]` |

---

## 1. Diagrama de capas

Muestra las cuatro capas de la arquitectura y **la única dirección permitida de dependencia**: siempre hacia abajo. Ninguna capa conoce a la que está por encima de ella.

```mermaid
flowchart TB
    subgraph L1["CAPA 1 · Presentación y Borde"]
        direction LR
        L1A["ups.web.spa<br/><i>Portal web</i>"]:::c1
        L1B["ups.gateway<br/><i>Enrutamiento · Seguridad · Rate limit · WAF</i>"]:::c1
    end

    subgraph L2["CAPA 2 · Aplicación — Módulos de negocio"]
        direction LR
        L2A["ups.identidad"]:::c2
        L2B["ups.matricula"]:::c2
        L2C["ups.academico"]:::c2
        L2D["ups.evaluacion"]:::c2
        L2E["ups.integracion"]:::c2
    end

    subgraph L3["CAPA 3 · Comunicación"]
        direction LR
        L3A["ups.shared.rest"]:::c3
        L3B["ups.shared.messaging"]:::c3
        L3C["ups.shared.contracts"]:::c3
        L3D["ups.shared.observability"]:::c3
        L3E["ups.shared.security"]:::c3
    end

    subgraph L4["CAPA 4 · Datos"]
        direction LR
        L4A[("5 bases de datos<br/>una por módulo")]:::c4
        L4B["Cola de mensajes<br/><i>servicio gestionado</i>"]:::c4
    end

    L1 ==>|"invoca vía API"| L2
    L2 ==>|"usa"| L3
    L2 ==>|"persiste en"| L4
    L3 ==>|"transporta hacia"| L4

    NOTA["Regla de capas: las dependencias solo fluyen hacia abajo.<br/>Ninguna capa referencia a una capa superior."]:::nota

    classDef c1 fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef c2 fill:#E6F4EA,stroke:#137333,color:#000
    classDef c3 fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef c4 fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef nota fill:#FFFFFF,stroke:#5F6368,color:#000,stroke-dasharray:4 3
```

### Justificación

**Las cuatro capas corresponden exactamente a las definidas en la arquitectura**, sin fusionarlas ni inventar nuevas. La capa de Comunicación (capa 3) es la más fácil de omitir y la más importante: materializa la decisión de combinar REST síncrono con cola de mensajes. Sin ella, cada módulo implementaría su propio cliente HTTP y su propia política de reintentos, y el Circuit Breaker sería inconsistente entre módulos.

**La dirección de dependencia es estricta y unidireccional.** La capa de Presentación conoce a la de Aplicación, pero ningún módulo de negocio conoce al Gateway. Esto es lo que permite que un módulo se pruebe sin levantar el borde, y sostiene la decisión de que la seguridad resida únicamente en el Gateway.

---

## 2. Diagrama de paquetes

Muestra los paquetes de código y sus relaciones de importación. Cada módulo desplegable contiene los mismos tres subpaquetes internos (`adapter`, `application`, `domain`); todos importan de los paquetes compartidos `ups.shared.*`, y **ninguno importa a otro módulo de negocio**.

```mermaid
flowchart TB
    subgraph BORDE["ups.gateway"]
        direction LR
        PG1["gateway.routing"]:::gw
        PG2["gateway.security"]:::gw
        PG3["gateway.ratelimit"]:::gw
        PG4["gateway.waf"]:::gw
        PG5["gateway.audit"]:::gw
    end

    subgraph MODS["Módulos de negocio · misma estructura de paquetes"]
        direction LR
        subgraph MOD1["ups.matricula"]
            direction TB
            P1A["adapter"]:::adp
            P1B["application"]:::app
            P1C["domain"]:::dom
            P1A -.->|«access»| P1B
            P1B -.->|«access»| P1C
        end
        subgraph MOD2["ups.academico"]
            direction TB
            P2A["adapter"]:::adp
            P2B["application"]:::app
            P2C["domain"]:::dom
            P2A -.->|«access»| P2B
            P2B -.->|«access»| P2C
        end
        subgraph MOD3["ups.evaluacion · ups.identidad · ups.integracion"]
            direction TB
            P3A["adapter"]:::adp
            P3B["application"]:::app
            P3C["domain"]:::dom
            P3A -.->|«access»| P3B
            P3B -.->|«access»| P3C
        end
    end

    subgraph SHARED["Paquetes compartidos · ups.shared"]
        direction LR
        SH1["shared.rest"]:::sh
        SH2["shared.messaging"]:::sh
        SH3["shared.contracts"]:::sh
        SH4["shared.observability"]:::sh
        SH5["shared.security"]:::sh
    end

    BORDE -.->|«import»| SHARED
    MOD1 -.->|«import»| SHARED
    MOD2 -.->|«import»| SHARED
    MOD3 -.->|«import»| SHARED

    PROH["✗ Prohibido: ups.matricula «import» ups.academico<br/>Los módulos NO se importan entre sí (regla R2)"]:::proh
    MOD1 -.->|✗| MOD2

    classDef gw fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef adp fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef app fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef dom fill:#E6F4EA,stroke:#137333,stroke-width:2px,color:#000
    classDef sh fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef proh fill:#FCE8E6,stroke:#C5221F,color:#000,stroke-dasharray:4 3
```

### Justificación

**El diagrama de paquetes expresa la organización del código fuente**, no su comportamiento en ejecución. Su relación clave es la de importación (`«import»`), que es exactamente lo que un archivo de dependencias declara. La ausencia de una flecha de importación entre `ups.matricula` y `ups.academico` es lo que permite compilarlos y desplegarlos por separado.

**Dentro de cada módulo, las importaciones apuntan hacia el dominio.** `adapter` accede a `application`, y `application` accede a `domain`; nunca al revés. El paquete `domain` no importa ningún paquete externo, lo que permite probar las reglas de negocio sin infraestructura.

**Todos los módulos importan de `ups.shared`, pero `ups.shared` no importa de ningún módulo.** Es una dependencia estrictamente unidireccional: los paquetes compartidos son hojas del grafo de importación, lo que evita el ciclo que convertiría la biblioteca compartida en un monolito distribuido.

---

## 3. Diagrama de componentes

Muestra los componentes del sistema con las **interfaces que provee cada uno** (hexágonos `I...`) y las que **consume de otros**. A diferencia del diagrama de paquetes —que es organización de código—, este muestra las unidades lógicas en ejecución y sus contratos.

```mermaid
flowchart TB
    USR["Estudiante · Docente · Coordinador<br/>Tesorería · Directivos"]:::act
    SPA["Portal Web SPA<br/><i>«component»</i>"]:::c1
    GW["API Gateway<br/><i>«component»</i>"]:::c1

    subgraph IF1[" "]
        I1{{"IIdentidad"}}:::itf
        I2{{"IAuditoria"}}:::itf
    end
    MID["Módulo Identidad<br/><i>«component» · RF-01</i>"]:::c2

    subgraph IF2[" "]
        I3{{"IMatricula"}}:::itf
        I4{{"IExpediente"}}:::itf
    end
    MMAT["Módulo Matrícula<br/><i>«component» · RF-02</i>"]:::c2

    subgraph IF3[" "]
        I5{{"IOfertaAcademica"}}:::itf
        I6{{"IDisponibilidadCupo"}}:::itf
        I7{{"IReservaAula"}}:::itf
    end
    MACA["Módulo Gestión Académica<br/><i>«component» · RF-03</i>"]:::c2

    subgraph IF4[" "]
        I8{{"ICalificaciones"}}:::itf
        I9{{"IActas"}}:::itf
    end
    MEVA["Módulo Evaluación<br/><i>«component» · RF-04</i>"]:::c2

    subgraph IF5[" "]
        I10{{"IEstadoFinanciero"}}:::itf
        I11{{"IDocumentos"}}:::itf
        I12{{"IBiblioteca"}}:::itf
        I13{{"IAnalitica"}}:::itf
    end
    MINT["Módulo Integración<br/><i>«component» · RF-05</i>"]:::c2

    MQ["Cola de Mensajes<br/><i>«managed service»</i>"]:::c3
    OBS["Observabilidad<br/><i>«managed service»</i>"]:::c3

    E1["Proveedor OIDC"]:::ce
    E2["Pasarela de Pagos"]:::ce
    E3["Biblioteca Digital"]:::ce
    E4["Gestión Documental"]:::ce

    USR --> SPA
    SPA -->|HTTPS/TLS| GW

    MID --- I1
    MID --- I2
    MMAT --- I3
    MMAT --- I4
    MACA --- I5
    MACA --- I6
    MACA --- I7
    MEVA --- I8
    MEVA --- I9
    MINT --- I10
    MINT --- I11
    MINT --- I12
    MINT --- I13

    GW -.usa.-> I1
    GW -.usa.-> I3
    GW -.usa.-> I4
    GW -.usa.-> I5
    GW -.usa.-> I7
    GW -.usa.-> I8
    GW -.usa.-> I9
    GW -.usa.-> I11
    GW -.usa.-> I12
    GW -.usa.-> I13
    GW -.valida.-> E1

    MMAT -.usa.-> I6
    MMAT -.usa.-> I10
    MEVA -.usa.-> I5
    MINT -.usa.-> I4
    MINT -.usa.-> I7

    MMAT -.publica y consume.-> MQ
    MEVA -.publica.-> MQ
    MACA -.publica.-> MQ
    MINT -.publica y consume.-> MQ

    MID -.telemetría.-> OBS
    MMAT -.telemetría.-> OBS
    MACA -.telemetría.-> OBS
    MEVA -.telemetría.-> OBS
    MINT -.telemetría.-> OBS
    GW -.telemetría.-> OBS

    MINT --> E2
    MINT --> E3
    MINT --> E4

    classDef act fill:#F5F5F5,stroke:#5F6368,color:#000
    classDef c1 fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef c2 fill:#E6F4EA,stroke:#137333,color:#000
    classDef c3 fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef itf fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef ce fill:#FCE8E6,stroke:#C5221F,color:#000
```

### Catálogo de interfaces

| Interfaz | Provista por | Consumida por | Operaciones principales |
|---|---|---|---|
| `IIdentidad` | Identidad | Gateway | Autenticar, obtener perfil y scopes |
| `IAuditoria` | Identidad | Gateway, todos los módulos | Registrar evento de acceso |
| `IMatricula` | Matrícula | Gateway | Inscribir, modificar, cancelar |
| `IExpediente` | Matrícula | Gateway, Integración | Consultar historial, promedio, avance |
| `IOfertaAcademica` | Académico | Gateway, Evaluación | Consultar planes, grupos, docente titular |
| `IDisponibilidadCupo` | Académico | Matrícula | Verificar, reservar, liberar cupo |
| `IReservaAula` | Académico | Gateway, Integración | Reservar espacio, consultar ocupación |
| `ICalificaciones` | Evaluación | Gateway | Registrar, publicar, consultar notas |
| `IActas` | Evaluación | Gateway | Abrir, cerrar, reabrir acta |
| `IEstadoFinanciero` | Integración | Matrícula | Verificar paz y salvo, saldo |
| `IDocumentos` | Integración | Gateway | Emitir, archivar, verificar certificados |
| `IBiblioteca` | Integración | Gateway | Buscar recursos, otorgar acceso |
| `IAnalitica` | Integración | Gateway | Consultar indicadores directivos |

### Justificación

**Las interfaces se nombran por capacidad de negocio, no por entidad.** `IDisponibilidadCupo` en lugar de `IGrupoCRUD`. Una interfaz orientada a capacidad expone la operación mínima que el consumidor necesita y oculta el modelo interno del proveedor. `IGrupoCRUD` filtraría la estructura de datos de Gestión Académica hacia Matrícula y reconstruiría el acoplamiento que la arquitectura busca evitar.

**Un componente provee varias interfaces segregadas.** El módulo de Matrícula provee `IMatricula` (escritura, consumida por el estudiante) e `IExpediente` (lectura, consumida por los paneles). Segregarlas permite aplicar políticas RBAC distintas en el Gateway y escalar o cachear la lectura sin tocar la escritura.

**La cola aparece como componente, no como flecha.** Esto muestra que el módulo de Evaluación publica eventos sin conocer a ningún consumidor. Ese anonimato es la definición operativa del desacople de RF-04: agregar un consumidor de analítica que reaccione a las notas no requiere modificar ni redesplegar Evaluación.

---

## 4. Diagrama de dependencias entre componentes

Muestra el **grafo dirigido de dependencias** entre componentes, clasificado por tipo. Es la vista que responde a la pregunta *"¿quién depende de quién y de qué forma?"*, y la que permite verificar que no existan ciclos de dependencia síncrona.

```mermaid
flowchart LR
    SPA["Portal Web SPA"]:::cli
    GW["API Gateway"]:::edge
    MID["ups.identidad"]:::mod
    MMAT["ups.matricula"]:::mod
    MACA["ups.academico"]:::mod
    MEVA["ups.evaluacion"]:::mod
    MINT["ups.integracion"]:::mod
    SHR["ups.shared.*<br/><i>bibliotecas compartidas</i>"]:::shared
    MQ["Cola de mensajes"]:::infra
    EXT["Sistemas externos<br/>Pagos · Biblioteca · Documental · OIDC"]:::ext

    SPA -->|HTTP| GW
    GW -->|REST| MID
    GW -->|REST| MMAT
    GW -->|REST| MACA
    GW -->|REST| MEVA
    GW -->|REST| MINT

    MMAT -.REST.-> MACA
    MMAT -.REST.-> MINT
    MEVA -.REST.-> MACA
    MINT -.REST.-> MMAT

    MMAT -.evento.-> MQ
    MACA -.evento.-> MQ
    MEVA -.evento.-> MQ
    MINT -.evento.-> MQ
    MQ -.entrega.-> MMAT
    MQ -.entrega.-> MINT

    MID ==>|compilación| SHR
    MMAT ==>|compilación| SHR
    MACA ==>|compilación| SHR
    MEVA ==>|compilación| SHR
    MINT ==>|compilación| SHR
    GW ==>|compilación| SHR

    MINT -->|REST| EXT
    GW -->|OIDC| EXT

    classDef cli fill:#F5F5F5,stroke:#5F6368,color:#000
    classDef edge fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef mod fill:#E6F4EA,stroke:#137333,color:#000
    classDef shared fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef infra fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
```

### Tipos de dependencia representados

| Estilo de flecha | Tipo de dependencia | Momento | Ejemplo |
|---|---|---|---|
| `==>` continua gruesa | Compilación | Al construir | Todo módulo → `ups.shared.*` |
| `-->` continua | Invocación de borde o externa | Ejecución | Gateway → módulos; Integración → pasarela |
| `-.->` con `REST` | Ejecución síncrona entre módulos | Ejecución | Matrícula → Académico (cupo) |
| `-.->` con `evento` | Ejecución asíncrona vía cola | Ejecución diferida | Evaluación → Cola |

### Matriz de dependencias

| Componente | Depende de (compilación) | Depende de (REST síncrono) | Publica o consume (cola) |
|---|---|---|---|
| **Gateway** | `shared.*` | Los 5 módulos, OIDC | — |
| **Identidad** | `shared.*` | — | — |
| **Matrícula** | `shared.*` | Académico, Integración | Publica y consume |
| **Académico** | `shared.*` | — | Publica |
| **Evaluación** | `shared.*` | Académico | Publica |
| **Integración** | `shared.*` | Matrícula, sistemas externos | Publica y consume |
| **shared.\*** | — | — | — |

### Justificación

**No existe ningún ciclo de dependencia síncrona dentro de una misma cadena de llamadas.** El diagrama muestra una arista `Matrícula → Integración` y otra `Integración → Matrícula`, lo que a primera vista parece un ciclo. No lo es: son **flujos distintos sobre interfaces distintas** que nunca se anidan. Matrícula llama a Integración por `IEstadoFinanciero` durante la inscripción; Integración llama a Matrícula por `IExpediente` durante la emisión de un certificado. Ninguna de las dos operaciones ocurre dentro de la otra, de modo que no puede formarse un interbloqueo ni una cascada de tiempos de espera.

**Las dependencias de compilación forman un grafo acíclico dirigido** cuya única hoja es `ups.shared.*`. Esto es lo que hace posible compilar y desplegar cada módulo por separado, y es exactamente lo que la regla R2 del pipeline verifica automáticamente.

**Identidad es un componente sin dependencias salientes hacia otros módulos.** Solo depende de las bibliotecas compartidas. Esto lo convierte en el componente más estable del sistema, lo cual es deseable porque es del que dependen todos los demás a través del Gateway.

**Integración es el único componente con dependencias hacia sistemas externos.** Concentrar ahí toda la comunicación con terceros reduce la superficie de exposición a un solo componente auditable y permite cambiar un proveedor sin tocar el resto.

---

## 5. Estructura interna — Módulo Identidad y Accesos (RF-01)

> **Nota.** Las secciones 5 a 9 detallan la estructura interna de cada módulo siguiendo la **arquitectura hexagonal** (puertos y adaptadores). Son un complemento a los cuatro diagramas anteriores: muestran cómo se organiza *por dentro* cada componente de la [§3](#3-diagrama-de-componentes).

```mermaid
flowchart TB
    subgraph MOD["ups.identidad"]
        direction TB

        subgraph AIN["adapter.in · Adaptadores primarios"]
            direction LR
            A1["rest.AutenticacionController"]:::adp
            A2["rest.UsuarioController"]:::adp
            A3["rest.RolController"]:::adp
            A4["rest.AuditoriaController"]:::adp
        end

        subgraph APP["application · Casos de uso"]
            direction LR
            P1["port.in<br/><i>AutenticarUseCase</i><br/><i>GestionarUsuarioUseCase</i><br/><i>GestionarRolUseCase</i><br/><i>AuditarAccesoUseCase</i><br/><i>RevocarSesionUseCase</i>"]:::app
            S1["service<br/><i>AutenticacionService</i><br/><i>RbacService</i><br/><i>AuditoriaService</i>"]:::app
            P2["port.out<br/><i>UsuarioRepository</i><br/><i>RolRepository</i><br/><i>AuditoriaRepository</i><br/><i>ProveedorIdentidadPort</i>"]:::app
        end

        subgraph DOM["domain · Núcleo sin dependencias externas"]
            direction LR
            N1["model<br/><i>Usuario · Estudiante · Docente</i><br/><i>Coordinador · Tesorero</i><br/><i>Rol · Permiso · TokenOIDC</i><br/><i>SesionActiva · RegistroAuditoria</i>"]:::dom
            N2["service<br/><i>EvaluadorPermisos</i><br/><i>DetectorAnomalias</i>"]:::dom
            N3["exception<br/><i>CredencialInvalidaException</i><br/><i>CuentaBloqueadaException</i>"]:::dom
        end

        subgraph AOUT["adapter.out · Adaptadores secundarios"]
            direction LR
            B1["persistence<br/><i>JpaUsuarioRepository</i><br/><i>JpaRolRepository</i><br/><i>JpaAuditoriaRepository</i>"]:::adp
            B2["client<br/><i>OidcProviderClient</i>"]:::adp
            B3["cache<br/><i>SesionCacheAdapter</i>"]:::adp
        end
    end

    GW["API Gateway"]:::ext --> A1
    GW --> A2
    GW --> A3
    GW --> A4

    A1 --> P1
    A2 --> P1
    A3 --> P1
    A4 --> P1
    P1 --> S1
    S1 --> N1
    S1 --> N2
    S1 --> N3
    S1 --> P2

    B1 -.implementa.-> P2
    B2 -.implementa.-> P2
    B3 -.implementa.-> P2

    B1 --> DB[("db_identidad")]:::db
    B2 --> SH["ups.shared.rest"]:::sh
    SH --> OIDC["Proveedor OIDC"]:::ext

    classDef dom fill:#E6F4EA,stroke:#137333,stroke-width:3px,color:#000
    classDef app fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef adp fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef db fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef sh fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
```

**Decisión propia del módulo.** `SesionCacheAdapter` implementa un puerto de salida hacia caché distribuida. La validación de sesión ocurre en cada petición de toda la plataforma; resolverla contra la base de datos relacional convertiría al módulo de Identidad en el cuello de botella de todo el sistema. El puerto permite que el dominio ignore por completo que existe una caché.

`DetectorAnomalias` reside en el dominio, no en el adaptador: identificar un patrón de acceso sospechoso es una regla de negocio de seguridad, no un detalle técnico.

---

## 6. Estructura interna — Módulo Matrícula y Expediente (RF-02)

```mermaid
flowchart TB
    subgraph MOD["ups.matricula"]
        direction TB

        subgraph AIN["adapter.in · Adaptadores primarios"]
            direction LR
            A1["rest.MatriculaController"]:::adp
            A2["rest.ExpedienteController"]:::adp
            A3["messaging.ReconciliarPagoListener"]:::adp
            A4["messaging.ExpirarMatriculaListener"]:::adp
            A5["scheduler.ExpiracionScheduler"]:::adp
        end

        subgraph APP["application · Casos de uso"]
            direction LR
            P1["port.in<br/><i>InscribirAsignaturasUseCase</i><br/><i>ModificarMatriculaUseCase</i><br/><i>CancelarMatriculaUseCase</i><br/><i>ConsultarExpedienteUseCase</i><br/><i>ReconciliarMatriculaUseCase</i>"]:::app
            S1["service<br/><i>MatriculaService</i><br/><i>ExpedienteService</i><br/><i>ReconciliacionService</i>"]:::app
            P2["port.out<br/><i>MatriculaRepository</i><br/><i>ExpedienteRepository</i><br/><i>EstadoFinancieroPort</i><br/><i>DisponibilidadCupoPort</i><br/><i>EventPublisherPort</i>"]:::app
        end

        subgraph DOM["domain · Núcleo sin dependencias externas"]
            direction LR
            N1["model<br/><i>Matricula · DetalleMatricula</i><br/><i>ExpedienteEstudiantil</i><br/><i>EstadoFinanciero · PeriodoAcademico</i><br/><i>SolicitudCancelacion</i>"]:::dom
            N2["service<br/><i>ValidadorPrerrequisitos</i><br/><i>CalculadoraCreditos</i>"]:::dom
            N3["exception<br/><i>CupoAgotadoException</i><br/><i>PrerrequisitoException</i><br/><i>FueraDePlazoException</i>"]:::dom
        end

        subgraph AOUT["adapter.out · Adaptadores secundarios"]
            direction LR
            B1["persistence<br/><i>JpaMatriculaRepository</i><br/><i>JpaExpedienteRepository</i>"]:::adp
            B2["client<br/><i>IntegracionRestClient</i>"]:::adp
            B3["client<br/><i>AcademicoRestClient</i>"]:::adp
            B4["messaging<br/><i>QueueEventPublisher</i>"]:::adp
        end
    end

    GW["API Gateway"]:::ext --> A1
    GW --> A2
    Q1["Cola de mensajes"]:::ext --> A3
    Q1 --> A4

    A1 --> P1
    A2 --> P1
    A3 --> P1
    A4 --> P1
    A5 --> P1
    P1 --> S1
    S1 --> N1
    S1 --> N2
    S1 --> N3
    S1 --> P2

    B1 -.implementa.-> P2
    B2 -.implementa.-> P2
    B3 -.implementa.-> P2
    B4 -.implementa.-> P2

    B1 --> DB[("db_matricula")]:::db
    B2 --> SH1["ups.shared.rest<br/>CircuitBreaker"]:::sh
    B3 --> SH1
    B4 --> SH2["ups.shared.messaging"]:::sh
    SH1 -.REST.-> OTROS["Módulo Integración<br/>Módulo Académico"]:::ext
    SH2 --> Q2["Cola de mensajes"]:::ext

    classDef dom fill:#E6F4EA,stroke:#137333,stroke-width:3px,color:#000
    classDef app fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef adp fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef db fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef sh fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
```

**Decisiones propias del módulo.**

`ReconciliarPagoListener` es un adaptador de entrada, exactamente igual que un controlador REST. La reconciliación entra al módulo por la misma puerta (`port.in`) que una petición del usuario, reutilizando el mismo caso de uso, las mismas validaciones y las mismas transiciones de estado. Si la reconciliación tuviera su propio camino escribiendo directo a la base de datos, existirían dos rutas capaces de confirmar una matrícula con reglas divergentes.

`EstadoFinancieroPort` es una interfaz del **dominio**, y `IntegracionRestClient` su implementación en el adaptador. Esta inversión de dependencia se aplica precisamente donde el caso de estudio identifica *Rigidez para Integrar Terceros*: cambiar de proveedor de pagos significa escribir una clase nueva en `adapter.out.client` sin tocar una línea del núcleo.

`ExpiracionScheduler` es un tercer tipo de adaptador de entrada —disparado por tiempo— que ejecuta la expiración de matrículas vencidas. Los tres tipos de disparador (HTTP, cola, reloj) convergen en el mismo conjunto de casos de uso.

---

## 7. Estructura interna — Módulo Gestión Académica (RF-03)

```mermaid
flowchart TB
    subgraph MOD["ups.academico"]
        direction TB

        subgraph AIN["adapter.in · Adaptadores primarios"]
            direction LR
            A1["rest.PlanEstudioController"]:::adp
            A2["rest.GrupoController"]:::adp
            A3["rest.ReservaController"]:::adp
            A4["rest.HorarioController"]:::adp
            A5["rest.DisponibilidadController"]:::adp
            A6["messaging.LiberarCupoListener"]:::adp
        end

        subgraph APP["application · Casos de uso"]
            direction LR
            P1["port.in<br/><i>GestionarPlanEstudioUseCase</i><br/><i>PlanificarOfertaUseCase</i><br/><i>AsignarCargaDocenteUseCase</i><br/><i>ProgramarHorarioUseCase</i><br/><i>ReservarAulaUseCase</i><br/><i>ConsultarDisponibilidadUseCase</i>"]:::app
            S1["service<br/><i>OfertaAcademicaService</i><br/><i>ReservaService</i><br/><i>CupoService</i>"]:::app
            P2["port.out<br/><i>PlanEstudioRepository</i><br/><i>GrupoRepository</i><br/><i>AulaRepository</i><br/><i>ReservaRepository</i><br/><i>CargaRepository</i><br/><i>EventPublisherPort</i>"]:::app
        end

        subgraph DOM["domain · Núcleo sin dependencias externas"]
            direction LR
            N1["model<br/><i>PlanEstudio · Asignatura · Grupo</i><br/><i>Horario · FranjaHoraria</i><br/><i>Aula · ReservaAula</i><br/><i>CargaAcademica · Conflicto</i>"]:::dom
            N2["service<br/><i>MotorReservas</i><br/><i>DetectorConflictos</i><br/><i>ValidadorCargaDocente</i>"]:::dom
            N3["exception<br/><i>ConflictoHorarioException</i><br/><i>CapacidadInsuficienteException</i><br/><i>TopeCargaExcedidoException</i>"]:::dom
        end

        subgraph AOUT["adapter.out · Adaptadores secundarios"]
            direction LR
            B1["persistence<br/><i>JpaPlanEstudioRepository</i><br/><i>JpaGrupoRepository</i><br/><i>JpaAulaRepository</i><br/><i>JpaReservaRepository</i>"]:::adp
            B2["messaging<br/><i>QueueEventPublisher</i>"]:::adp
        end
    end

    GW["API Gateway"]:::ext --> A1
    GW --> A2
    GW --> A3
    GW --> A4
    OTR["Módulo Matrícula<br/>Módulo Evaluación"]:::ext -.REST.-> A5
    Q1["Cola de mensajes"]:::ext --> A6

    A1 --> P1
    A2 --> P1
    A3 --> P1
    A4 --> P1
    A5 --> P1
    A6 --> P1
    P1 --> S1
    S1 --> N1
    S1 --> N2
    S1 --> N3
    S1 --> P2

    B1 -.implementa.-> P2
    B2 -.implementa.-> P2

    B1 --> DB[("db_academico")]:::db
    B2 --> SH["ups.shared.messaging"]:::sh
    SH --> Q2["Cola de mensajes"]:::ext

    classDef dom fill:#E6F4EA,stroke:#137333,stroke-width:3px,color:#000
    classDef app fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef adp fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef db fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef sh fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
```

**Decisiones propias del módulo.**

Este es el único módulo **sin adaptadores de salida REST**: no llama a ningún otro módulo ni a ningún servicio externo. Solo persiste y publica eventos. Esta ausencia es deliberada y valiosa: convierte a Gestión Académica en un módulo hoja del grafo de dependencias, lo que significa que puede desplegarse y probarse en aislamiento total.

`DisponibilidadController` está separado de `GrupoController` porque expone una interfaz distinta con un consumidor distinto: los otros módulos consultan disponibilidad de cupo, mientras el coordinador administra grupos. La segregación permite aplicar políticas de acceso y de escalado diferenciadas.

`MotorReservas` y sus dos colaboradores viven en el **dominio**, no en la capa de aplicación. La detección de conflictos de horario es la regla de negocio central de RF-03; ubicarla en el dominio permite probarla exhaustivamente sin base de datos, lo que resulta indispensable dada la combinatoria de casos.

---

## 8. Estructura interna — Módulo Evaluación y Seguimiento (RF-04)

```mermaid
flowchart TB
    subgraph MOD["ups.evaluacion"]
        direction TB

        subgraph AIN["adapter.in · Adaptadores primarios"]
            direction LR
            A1["rest.ActaController"]:::adp
            A2["rest.ActividadController"]:::adp
            A3["rest.CalificacionController"]:::adp
            A4["rest.RevisionController"]:::adp
            A5["messaging.CierrePeriodoListener"]:::adp
        end

        subgraph APP["application · Casos de uso"]
            direction LR
            P1["port.in<br/><i>DefinirActividadUseCase</i><br/><i>RegistrarCalificacionUseCase</i><br/><i>PublicarCalificacionUseCase</i><br/><i>ConsultarCalificacionUseCase</i><br/><i>SolicitarRevisionUseCase</i><br/><i>CerrarActaUseCase</i>"]:::app
            S1["service<br/><i>ActaService</i><br/><i>CalificacionService</i><br/><i>RevisionService</i>"]:::app
            P2["port.out<br/><i>ActaRepository</i><br/><i>CalificacionRepository</i><br/><i>RevisionRepository</i><br/><i>OfertaAcademicaPort</i><br/><i>EventPublisherPort</i>"]:::app
        end

        subgraph DOM["domain · Núcleo sin dependencias externas"]
            direction LR
            N1["model<br/><i>ActaNotas · ActividadEvaluativa</i><br/><i>Calificacion · NotaDefinitiva</i><br/><i>SolicitudRevision · EventoNota</i><br/><i>HistorialCambioNota</i>"]:::dom
            N2["service<br/><i>CalculadoraDefinitivas</i><br/><i>ValidadorPorcentajes</i><br/><i>GeneradorHashActa</i>"]:::dom
            N3["exception<br/><i>ActaCerradaException</i><br/><i>NotaFueraDeRangoException</i><br/><i>RegistroIncompletoException</i>"]:::dom
        end

        subgraph AOUT["adapter.out · Adaptadores secundarios"]
            direction LR
            B1["persistence<br/><i>JpaActaRepository</i><br/><i>JpaCalificacionRepository</i>"]:::adp
            B2["client<br/><i>AcademicoRestClient</i>"]:::adp
            B3["messaging<br/><i>QueueEventPublisher</i>"]:::adp
        end
    end

    GW["API Gateway"]:::ext --> A1
    GW --> A2
    GW --> A3
    GW --> A4
    Q1["Cola de mensajes"]:::ext --> A5

    A1 --> P1
    A2 --> P1
    A3 --> P1
    A4 --> P1
    A5 --> P1
    P1 --> S1
    S1 --> N1
    S1 --> N2
    S1 --> N3
    S1 --> P2

    B1 -.implementa.-> P2
    B2 -.implementa.-> P2
    B3 -.implementa.-> P2

    B1 --> DB[("db_evaluacion")]:::db
    B2 --> SH1["ups.shared.rest"]:::sh
    B3 --> SH2["ups.shared.messaging"]:::sh
    SH1 -.REST.-> ACA["Módulo Académico"]:::ext
    SH2 --> Q2["Cola de mensajes"]:::ext

    classDef dom fill:#E6F4EA,stroke:#137333,stroke-width:3px,color:#000
    classDef app fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef adp fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef db fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef sh fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
```

**Decisiones propias del módulo.**

`GeneradorHashActa` reside en el dominio porque el sello de integridad del acta cerrada es una regla académica —garantizar que las notas consolidadas no fueron alteradas—, no un detalle criptográfico de infraestructura.

Este módulo **no consume ninguna interfaz síncrona salvo `OfertaAcademicaPort`**, usada únicamente para validar que el grupo existe y que el docente es su titular. Todo lo demás sale por cola. Es el módulo con menor acoplamiento síncrono del sistema, lo que explica por qué en la [Vista de Procesos](03-vista-procesos.md) permanece operativo mientras el módulo de Matrícula soporta el pico de inscripciones.

`ValidadorPorcentajes` implementa la invariante de que las actividades sumen 100 %. Reside en el dominio y se ejecuta como precondición de apertura del acta.

---

## 9. Estructura interna — Módulo Integración Externa y Analítica (RF-05)

```mermaid
flowchart TB
    subgraph MOD["ups.integracion"]
        direction TB

        subgraph AIN["adapter.in · Adaptadores primarios"]
            direction LR
            A1["rest.PagoController"]:::adp
            A2["rest.EstadoFinancieroController"]:::adp
            A3["rest.CertificadoController"]:::adp
            A4["rest.BibliotecaController"]:::adp
            A5["rest.PanelController"]:::adp
            A6["messaging.ReconciliarPagoListener"]:::adp
            A7["messaging.ProyeccionListener"]:::adp
        end

        subgraph APP["application · Casos de uso"]
            direction LR
            P1["port.in<br/><i>ProcesarPagoUseCase</i><br/><i>ConciliarPagoUseCase</i><br/><i>AplicarBeneficioUseCase</i><br/><i>EmitirCertificadoUseCase</i><br/><i>AccederBibliotecaUseCase</i><br/><i>ConsultarPanelUseCase</i>"]:::app
            S1["service<br/><i>PagoService</i><br/><i>ConciliacionService</i><br/><i>DocumentoService</i><br/><i>AnaliticaService</i>"]:::app
            P2["port.out<br/><i>TransaccionRepository</i><br/><i>DocumentoRepository</i><br/><i>ProyeccionRepository</i><br/><i>PasarelaPagoPort</i><br/><i>BibliotecaPort</i><br/><i>DocumentalPort</i><br/><i>ExpedientePort</i>"]:::app
        end

        subgraph DOM["domain · Núcleo sin dependencias externas"]
            direction LR
            N1["model<br/><i>TransaccionPago · BeneficioFinanciero</i><br/><i>Documento · PanelIndicadores</i><br/><i>Metrica · EstadoFinanciero</i>"]:::dom
            N2["service<br/><i>CalculadoraIndicadores</i><br/><i>EvaluadorRiesgoDesercion</i><br/><i>GeneradorCodigoVerificacion</i>"]:::dom
            N3["exception<br/><i>PagoDuplicadoException</i><br/><i>ServicioExternoException</i><br/><i>SaldoPendienteException</i>"]:::dom
        end

        subgraph AOUT["adapter.out · Adaptadores secundarios"]
            direction LR
            B1["persistence<br/><i>JpaTransaccionRepository</i><br/><i>JpaDocumentoRepository</i><br/><i>JpaProyeccionRepository</i>"]:::adp
            B2["client<br/><i>PasarelaPagoAdapter</i>"]:::adp
            B3["client<br/><i>BibliotecaAdapter</i>"]:::adp
            B4["client<br/><i>DocumentalAdapter</i>"]:::adp
            B5["client<br/><i>MatriculaRestClient</i>"]:::adp
            B6["messaging<br/><i>QueueEventPublisher</i>"]:::adp
        end
    end

    GW["API Gateway"]:::ext --> A1
    GW --> A3
    GW --> A4
    GW --> A5
    OTR["Módulo Matrícula"]:::ext -.REST.-> A2
    Q1["Cola de mensajes"]:::ext --> A6
    Q1 --> A7

    A1 --> P1
    A2 --> P1
    A3 --> P1
    A4 --> P1
    A5 --> P1
    A6 --> P1
    A7 --> P1
    P1 --> S1
    S1 --> N1
    S1 --> N2
    S1 --> N3
    S1 --> P2

    B1 -.implementa.-> P2
    B2 -.implementa.-> P2
    B3 -.implementa.-> P2
    B4 -.implementa.-> P2
    B5 -.implementa.-> P2
    B6 -.implementa.-> P2

    B1 --> DB[("db_integracion")]:::db
    B2 --> SH1["ups.shared.rest<br/>CircuitBreaker"]:::sh
    B3 --> SH1
    B4 --> SH1
    B5 --> SH1
    B6 --> SH2["ups.shared.messaging"]:::sh
    SH1 --> EXT1["Pasarela de Pagos"]:::ext
    SH1 --> EXT2["Biblioteca Digital"]:::ext
    SH1 --> EXT3["Gestión Documental"]:::ext
    SH2 --> Q2["Cola de mensajes"]:::ext

    classDef dom fill:#E6F4EA,stroke:#137333,stroke-width:3px,color:#000
    classDef app fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef adp fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef db fill:#F3E8FD,stroke:#8430CE,color:#000
    classDef sh fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef ext fill:#FCE8E6,stroke:#C5221F,color:#000
```

**Decisiones propias del módulo.**

Es el **único módulo con adaptadores hacia sistemas externos**, y esa concentración es deliberada. `ups.matricula` no llama a la pasarela de pagos: consume la interfaz `IEstadoFinanciero` que este módulo provee. Los beneficios son concretos: cambiar de proveedor toca un solo módulo; el Circuit Breaker y las credenciales viven en un solo lugar; y la superficie de exposición hacia terceros queda reducida a un componente auditable.

`ProyeccionListener` mantiene proyecciones de lectura alimentadas por eventos de los otros módulos. Esto permite que los paneles directivos consulten datos consolidados **sin ejecutar consultas en línea contra las bases de datos de los demás módulos**, evitando que la analítica compita por recursos con la operación transaccional durante el periodo de matrícula.

`EvaluadorRiesgoDesercion` reside en el dominio: el algoritmo que determina qué estudiantes están en riesgo es una regla de negocio institucional, no un reporte.

---

## 10. Bibliotecas compartidas

```mermaid
flowchart TB
    subgraph SH["shared · Bibliotecas transversales"]
        direction TB

        C1["<b>ups.shared.contracts</b><br/><i>DTOs de petición y respuesta</i><br/><i>Definiciones de eventos de dominio</i><br/><i>Tipos de identificador</i><br/><i>Enumeraciones compartidas</i><br/><br/>❌ Sin lógica de negocio<br/>❌ Sin entidades del dominio"]:::sh

        C2["<b>ups.shared.rest</b><br/><i>Cliente HTTP configurable</i><br/><i>CircuitBreaker</i><br/><i>PoliticaReintento</i><br/><i>Propagación de trazas</i><br/><i>Serialización estándar</i>"]:::sh

        C3["<b>ups.shared.messaging</b><br/><i>Publicador de eventos</i><br/><i>Consumidor con reintentos</i><br/><i>Cola de mensajes fallidos</i><br/><i>Garantía de entrega</i>"]:::sh

        C4["<b>ups.shared.observability</b><br/><i>Registro estructurado</i><br/><i>Métricas de negocio y técnicas</i><br/><i>Trazas distribuidas</i><br/><i>Correlación de peticiones</i>"]:::sh

        C5["<b>ups.shared.security</b><br/><i>Contexto de identidad</i><br/><i>Lectura de scopes del token</i><br/><i>Anotaciones de autorización</i><br/><br/>❌ No valida tokens<br/>ℹ️ Eso es del Gateway"]:::sh
    end

    M1["ups.identidad"]:::mod
    M2["ups.matricula"]:::mod
    M3["ups.academico"]:::mod
    M4["ups.evaluacion"]:::mod
    M5["ups.integracion"]:::mod
    GW["ups.gateway"]:::gw

    M1 --> C1
    M1 --> C2
    M1 --> C4
    M1 --> C5
    M2 --> C1
    M2 --> C2
    M2 --> C3
    M2 --> C4
    M2 --> C5
    M3 --> C1
    M3 --> C3
    M3 --> C4
    M3 --> C5
    M4 --> C1
    M4 --> C2
    M4 --> C3
    M4 --> C4
    M4 --> C5
    M5 --> C1
    M5 --> C2
    M5 --> C3
    M5 --> C4
    M5 --> C5
    GW --> C1
    GW --> C2
    GW --> C4
    GW --> C5

    classDef sh fill:#FEF7E0,stroke:#EA8600,color:#000
    classDef mod fill:#E6F4EA,stroke:#137333,color:#000
    classDef gw fill:#E8F0FE,stroke:#1967D2,color:#000
```

### Justificación

**`ups.shared.contracts` contiene únicamente tipos de datos serializables.** Es el riesgo clásico de las bibliotecas compartidas: crecen hasta convertirse en un monolito distribuido. Si llegara a contener la clase `Matricula` con sus métodos de negocio, los cinco módulos quedarían acoplados a los cambios del dominio de matrícula y se perdería el despliegue independiente. La restricción está expresada en el diagrama y verificada por la regla R5 del pipeline.

**El Circuit Breaker vive en `ups.shared.rest`, no en los adaptadores.** Los adaptadores declaran *qué* llaman; la biblioteca compartida decide *cómo* se protege esa llamada. Así, la política de resiliencia de RNF-04 se configura y audita en un solo lugar, y un desarrollador no puede olvidarse de aplicarla.

**`ups.shared.security` transporta el contexto de identidad pero no lo valida.** Es una distinción crítica: si validara tokens, cada módulo tendría su propia implementación de verificación y la política de seguridad dejaría de ser única. Su única responsabilidad es leer los scopes ya validados por el Gateway.

---

## 11. Reglas de dependencia verificables en integración continua

```mermaid
flowchart LR
    subgraph PIPE["Pipeline de integración continua"]
        direction TB
        P1["Compilación<br/>de los 5 módulos"]:::st
        P2["Pruebas unitarias<br/>del dominio"]:::st
        P3["<b>Verificación de reglas<br/>arquitectónicas</b>"]:::arch
        P4["Pruebas de integración<br/>por módulo"]:::st
        P5["Pruebas de contrato<br/>entre módulos"]:::st
        P6["Análisis de seguridad<br/>de dependencias"]:::st
        P7["Publicación de artefactos"]:::st
        P8["Despliegue"]:::st

        P1 --> P2 --> P3
        P3 -->|✅ Cumple| P4 --> P5 --> P6 --> P7 --> P8
        P3 -->|❌ Viola una regla| FAIL["Build fallido<br/>despliegue bloqueado"]:::err
    end

    subgraph RULES["Reglas verificadas"]
        direction TB
        R1["<b>R1</b> · domain sin dependencias externas"]:::r
        R2["<b>R2</b> · Ningún módulo depende de otro en compilación"]:::r
        R3["<b>R3</b> · Solo integracion referencia SDKs de terceros"]:::r
        R4["<b>R4</b> · Todo cliente HTTP pasa por shared.rest"]:::r
        R5["<b>R5</b> · shared.contracts sin lógica de negocio"]:::r
        R6["<b>R6</b> · Cada módulo accede solo a su BD"]:::r
        R7["<b>R7</b> · Ningún módulo expone puerto público"]:::r
        R8["<b>R8</b> · adapter no es referenciado por domain"]:::r
    end

    P3 -.evalúa.-> RULES

    classDef st fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef arch fill:#FEF7E0,stroke:#EA8600,stroke-width:3px,color:#000
    classDef r fill:#E6F4EA,stroke:#137333,color:#000
    classDef err fill:#FCE8E6,stroke:#C5221F,color:#000
```

### Tabla de reglas

| # | Regla | Requisito que protege | Herramienta |
|---|---|---|---|
| **R1** | `modules.*.domain` no depende de ningún paquete externo al propio dominio | Testabilidad, mantenibilidad | ArchUnit |
| **R2** | `modules.X` no declara dependencia de compilación hacia `modules.Y` | Despliegue independiente | ArchUnit / análisis de dependencias |
| **R3** | Solo `modules.integracion` referencia SDKs o clientes de terceros | RF-05, seguridad | ArchUnit |
| **R4** | Todo cliente HTTP saliente pasa por `shared.rest` | RNF-04 · Circuit Breaker consistente | ArchUnit |
| **R5** | `shared.contracts` no contiene clases con métodos de negocio | Prevención de monolito distribuido | ArchUnit |
| **R6** | Cada módulo accede exclusivamente a su propia base de datos | Base de datos por módulo | Revisión de configuración + pruebas |
| **R7** | Ningún módulo expone puerto público; solo el Gateway | RNF-03 | Infraestructura como código |
| **R8** | `domain` nunca referencia clases de `adapter` | Inversión de dependencia | ArchUnit |

### Justificación

**Estas reglas se ejecutan como pruebas automáticas, no como recomendaciones en un documento.** La erosión arquitectónica ocurre por atajos razonables bajo presión de entrega, y el periodo de matrícula es exactamente ese tipo de presión. Un test que falla es la única barrera que sobrevive a un viernes por la tarde.

**La verificación arquitectónica se ubica antes de las pruebas de integración**, no al final. Detectar una violación estructural después de ejecutar la suite completa desperdicia tiempo de pipeline y, sobre todo, permite que el desarrollador siga construyendo sobre una base incorrecta.

**Las pruebas de contrato entre módulos** verifican que un cambio en `IDisponibilidadCupo` no rompa a su consumidor sin que nadie lo note. Con cinco módulos desplegándose de forma independiente, es el mecanismo que sustituye a la garantía que daba la compilación única del monolito.

---

## 12. Organización del repositorio

```
ups-connect/
│
├── gateway/                             # Artefacto desplegable independiente
│   └── src/main/java/ups/gateway/
│       ├── routing/
│       ├── security/                    # Única validación de tokens
│       ├── ratelimit/
│       ├── waf/
│       └── audit/
│
├── modules/
│   ├── identidad/                       # RF-01
│   ├── matricula/                       # RF-02
│   ├── academico/                       # RF-03
│   ├── evaluacion/                      # RF-04
│   └── integracion/                     # RF-05
│       └── src/main/java/ups/<modulo>/
│           ├── adapter/
│           │   ├── in/
│           │   │   ├── rest/            # Controladores HTTP
│           │   │   ├── messaging/       # Consumidores de cola
│           │   │   └── scheduler/       # Disparadores por tiempo
│           │   └── out/
│           │       ├── persistence/     # Repositorios JPA
│           │       ├── client/          # Clientes REST
│           │       └── messaging/       # Publicadores
│           ├── application/
│           │   ├── port/in/             # Interfaces de casos de uso
│           │   ├── port/out/            # Interfaces requeridas
│           │   └── service/             # Orquestación
│           └── domain/                  # Núcleo sin dependencias
│               ├── model/
│               ├── service/
│               └── exception/
│
├── shared/
│   ├── contracts/                       # DTOs, eventos, identificadores
│   ├── rest/                            # Cliente HTTP, CircuitBreaker
│   ├── messaging/                       # Publicador y consumidor
│   ├── observability/                   # Registros, métricas, trazas
│   └── security/                        # Contexto de identidad
│
├── infra/                               # Infraestructura como código
│   ├── network/                         # Subredes, grupos de seguridad
│   ├── compute/                         # Planes de servicio, autoescalado
│   ├── data/                            # Bases de datos, réplicas, respaldos
│   └── observability/                   # Consola, alertas, paneles
│
├── .github/workflows/                   # Pipeline de integración continua
│   └── ci.yml
│
└── docs/architecture/                   # Este conjunto de vistas
```

### Justificación

**Los cinco módulos usan una plantilla interna idéntica.** Un desarrollador que se mueva de `evaluacion` a `matricula` encuentra la misma estructura de carpetas, los mismos nombres de paquete y la misma ubicación para cada tipo de clase. Con siete grupos de interesados y un equipo que rota, la uniformidad estructural reduce el costo de incorporación más que cualquier documento. Es una decisión de mantenibilidad (RNF-05), no de estética.

**`infra/` está versionada junto al código.** La infraestructura como código garantiza que la [Vista Física](05-vista-fisica.md) sea reproducible y que las reglas R6 y R7 —imposibles de verificar con ArchUnit— se auditen en la definición de red y de bases de datos.

---

## Trazabilidad de la Vista de Desarrollo

| Elemento arquitectónico | Materialización en el código |
|---|---|
| Cuatro capas | Diagrama de capas (§1) |
| Organización de paquetes sin acoplamiento cruzado | Diagrama de paquetes (§2), regla R2 |
| Cinco módulos con API REST propia | 13 interfaces provistas, diagrama de componentes (§3) |
| Ausencia de ciclos de dependencia síncrona | Diagrama de dependencias (§4) |
| Base de datos por módulo | Regla R6, ausencia de aristas cruzadas |
| Adaptadores desacoplados (RF-05) | Puertos de salida en los 5 módulos, regla R3 |
| Circuit Breaker (RNF-04) | `shared.rest`, regla R4 |
| Cola para casos puntuales | `shared.messaging`, diagrama de bibliotecas (§10) |
| Observabilidad centralizada (RNF-05) | `shared.observability`, 6 consumidores |
| Autenticación centralizada (RF-01) | `gateway.security` como único validador |
| Fin del acoplamiento heredado | Reglas R1, R2, R5, R8 verificadas en CI |

---

| ← Anterior | Índice | Siguiente → |
|---|---|---|
| [Vista de Procesos](03-vista-procesos.md) | [README](../README.md) | [Vista Física](05-vista-fisica.md) |
