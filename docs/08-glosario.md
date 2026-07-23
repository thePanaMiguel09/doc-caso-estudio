# Glosario de Términos

> Consolida y amplía la *Tabla 1 · Terminología y Definiciones* del documento fuente. Reúne el vocabulario técnico empleado a lo largo de las cinco vistas 4+1, el registro de decisiones y el registro de riesgos, de modo que cualquier lector pueda interpretar la documentación sin recurrir a fuentes externas.

**Composición:** 28 términos originales del caso de estudio + 18 términos incorporados por esta documentación = **46 términos**.

Los términos se numeran de forma continua para facilitar su conteo y se organizan por categoría temática. Los marcados con 🆕 no figuraban en la Tabla 1 original y se añaden porque la documentación arquitectónica los utiliza de forma explícita.

---

## A · Arquitectura y estilos

| # | Término | Definición |
|---|---|---|
| 1 | **Arquitectura de Software** | Estructura de alto nivel que define los componentes de un sistema, sus responsabilidades y principios de diseño. |
| 2 | **Arquitectura en Capas (N-Tier)** | Organiza el sistema en capas con responsabilidades diferenciadas, donde cada capa solo se comunica con las adyacentes. |
| 3 | **Arquitectura Orientada a Servicios (SOA)** | Estilo en el que las funcionalidades se exponen como servicios independientes y reutilizables. |
| 4 | 🆕 **Modelo de Vistas 4+1** | Marco propuesto por Kruchten que describe la arquitectura mediante cinco vistas —Casos de Uso, Lógica, Procesos, Desarrollo y Física— para atender a distintos interesados. |
| 5 | 🆕 **Arquitectura Hexagonal (Puertos y Adaptadores)** | Estilo en el que la lógica de negocio (dominio) se aísla de la infraestructura mediante interfaces, de modo que todas las dependencias apunten hacia el núcleo. |
| 6 | 🆕 **Atributo de Calidad** | Propiedad medible del sistema —disponibilidad, escalabilidad, seguridad, resiliencia, mantenibilidad— clasificada según la norma ISO/IEC 25010:2011. |

---

## B · Modularidad y organización del código

| # | Término | Definición |
|---|---|---|
| 7 | **Módulo de Negocio** | Unidad de software desplegable de forma independiente, responsable de una capacidad de negocio específica y con su propia base de datos. |
| 8 | **Base de Datos por Módulo** | Cada módulo gestiona su propio esquema o instancia de base de datos, evitando el acoplamiento entre dominios. |
| 9 | 🆕 **Puerto (Port)** | Interfaz definida por el dominio que declara una capacidad que necesita (puerto de salida) o que ofrece (puerto de entrada), sin conocer su implementación. |
| 10 | 🆕 **Adaptador (Adapter)** | Implementación concreta de un puerto que conecta el dominio con una tecnología específica: base de datos, cliente REST, cola de mensajes o disparador por tiempo. |
| 11 | 🆕 **DTO (Data Transfer Object)** | Objeto sin lógica de negocio usado para transportar datos entre módulos o hacia el exterior, definido en la biblioteca de contratos compartida. |

---

## C · Diseño del dominio

| # | Término | Definición |
|---|---|---|
| 12 | 🆕 **Agregado y Raíz de Agregado** | Conjunto de objetos de dominio tratado como una unidad de consistencia; la raíz es el único punto de acceso al agregado desde el exterior. |
| 13 | 🆕 **Objeto de Valor (Value Object)** | Objeto de dominio inmutable, sin identidad propia, definido únicamente por sus atributos (por ejemplo, `EstadoFinanciero` o `FranjaHoraria`). |
| 14 | 🆕 **Evento de Dominio (Domain Event)** | Registro inmutable de un hecho relevante del negocio (por ejemplo, la publicación de una nota) que otros componentes pueden consumir de forma desacoplada. |
| 15 | 🆕 **Proyección de Lectura (Read Projection)** | Vista de datos precalculada y alimentada por eventos, usada para consultas analíticas sin acceder a las bases de datos de otros módulos. |

---

## D · Comunicación e integración

| # | Término | Definición |
|---|---|---|
| 16 | **API REST** | Estilo de interfaz basado en HTTP y JSON que expone recursos mediante operaciones estándar (GET, POST, PUT, DELETE). |
| 17 | **SOAP** | Protocolo de intercambio de mensajes basado en XML, con contratos formales mediante WSDL, usado en integraciones empresariales heredadas. |
| 18 | **API Gateway** | Componente que centraliza el enrutamiento, la autenticación y la limitación de tasa hacia los módulos internos. |
| 19 | **Cola de Mensajes Gestionada** | Servicio cloud para el envío y la recepción asíncrona de mensajes entre módulos. |
| 20 | **Comunicación Síncrona** | El emisor espera una respuesta inmediata del receptor antes de continuar. |
| 21 | **Comunicación Asíncrona** | El emisor no espera respuesta inmediata; el mensaje se procesa después, típicamente vía cola. |
| 22 | 🆕 **Cola de Mensajes Fallidos (Dead-Letter Queue)** | Cola donde se depositan los mensajes que no pudieron procesarse tras agotar los reintentos, para su análisis y reprocesamiento manual. |

---

## E · Infraestructura y despliegue

| # | Término | Definición |
|---|---|---|
| 23 | **PaaS (Platform as a Service)** | Modelo cloud donde el proveedor gestiona la infraestructura y el entorno de ejecución, sin administrar servidores directamente. |
| 24 | **Balanceador de Carga** | Distribuye las solicitudes entre varias instancias de un módulo para evitar su saturación. |
| 25 | **Autoescalado** | Mecanismo que agrega o retira automáticamente instancias según la demanda. |
| 26 | **Multi-AZ (Multi-Zona de Disponibilidad)** | Replica componentes en varias zonas físicas de un proveedor cloud para tolerar la falla de una zona. |
| 27 | **Base de Datos Gestionada** | Servicio de base de datos administrado por el proveedor cloud, que reduce la carga operativa. |
| 28 | 🆕 **Despliegue Azul-Verde (Blue-Green Deployment)** | Estrategia que levanta la nueva versión en paralelo a la vigente y conmuta el tráfico solo tras verificar su salud, permitiendo reversión inmediata. |
| 29 | 🆕 **Drenado Elegante (Graceful Draining)** | Procedimiento de apagado de una instancia que deja de recibir tráfico nuevo y termina las peticiones en curso antes de detenerse, evitando operaciones a medias. |

---

## F · Seguridad y control de acceso

| # | Término | Definición |
|---|---|---|
| 30 | **OAuth2 / OpenID Connect (OIDC)** | Protocolos de autorización y autenticación que permiten acceder a múltiples módulos con un mismo proveedor de identidad. |
| 31 | **RBAC (Role-Based Access Control)** | Asigna permisos a los usuarios según su rol en la organización. |
| 32 | **HTTPS / TLS** | Protocolo que cifra la comunicación cliente-servidor, garantizando la confidencialidad y la integridad de los datos. |
| 33 | **WAF (Web Application Firewall)** | Filtra y bloquea el tráfico malicioso dirigido a aplicaciones web. |
| 34 | 🆕 **JWT (JSON Web Token)** | Token firmado que transporta la identidad y los permisos del usuario, validado por el Gateway en cada petición. |
| 35 | 🆕 **mTLS (TLS Mutuo)** | Variante de TLS en la que tanto el cliente como el servidor presentan certificado, usada para autenticar la comunicación interna entre el Gateway y los módulos. |

---

## G · Resiliencia y disponibilidad

| # | Término | Definición |
|---|---|---|
| 36 | **Circuit Breaker** | Detecta fallas repetidas en un módulo o servicio externo y corta temporalmente las llamadas hacia él. |
| 37 | **Reintentos con Retroceso Exponencial** | Ante un fallo transitorio, reintenta la operación aumentando progresivamente el tiempo de espera. |
| 38 | **Idempotencia** | Propiedad de una operación que produce el mismo resultado sin importar cuántas veces se ejecute. |
| 39 | **Resiliencia** | Capacidad de continuar operando, o de recuperarse rápidamente, ante fallos. |
| 40 | **RPO / RTO** | Indicadores de continuidad: el RPO define la pérdida máxima tolerable de datos y el RTO el tiempo máximo de recuperación. |
| 41 | 🆕 **Bloqueo Optimista (Optimistic Locking)** | Control de concurrencia que permite lecturas en paralelo y detecta conflictos en la escritura mediante un número de versión, en lugar de bloquear el recurso. |
| 42 | 🆕 **Modo Degradado** | Estado en el que el sistema continúa operando con funcionalidad reducida ante el fallo de una dependencia, en lugar de rechazar por completo la operación. |

---

## H · Escalabilidad y observabilidad

| # | Término | Definición |
|---|---|---|
| 43 | **Escalabilidad** | Capacidad de soportar el incremento de usuarios o transacciones sin degradar el rendimiento. |
| 44 | **Observabilidad** | Capacidad de inferir el estado interno de un sistema a partir de métricas, registros (logs) y alertas. |

---

## I · Documentación arquitectónica

| # | Término | Definición |
|---|---|---|
| 45 | **Software Architecture Document (SAD)** | Documento técnico que describe la arquitectura, las decisiones, los requisitos y los atributos de calidad de un sistema. |
| 46 | 🆕 **Registro de Decisión Arquitectónica (ADR)** | Documento breve que registra una decisión arquitectónica significativa: su contexto, la decisión tomada, las alternativas descartadas y sus consecuencias. |

---

## Resumen por categoría

| Categoría | Términos | Cantidad |
|---|---|---|
| A · Arquitectura y estilos | 1 – 6 | 6 |
| B · Modularidad y organización del código | 7 – 11 | 5 |
| C · Diseño del dominio | 12 – 15 | 4 |
| D · Comunicación e integración | 16 – 22 | 7 |
| E · Infraestructura y despliegue | 23 – 29 | 7 |
| F · Seguridad y control de acceso | 30 – 35 | 6 |
| G · Resiliencia y disponibilidad | 36 – 42 | 7 |
| H · Escalabilidad y observabilidad | 43 – 44 | 2 |
| I · Documentación arquitectónica | 45 – 46 | 2 |
| | **Total** | **46** |

**Origen de los términos:** 28 provienen de la Tabla 1 del documento fuente y se conservan con su definición original; 18 (marcados 🆕) se incorporan por el vocabulario que introduce esta documentación arquitectónica.

---

| ← Anterior | Índice | Siguiente → |
|---|---|---|
| [Riesgos y Mitigación](07-riesgos-mitigacion.md) | [README](../README.md) | — |
