**Caso de Estudio: Sistema Integral de Gestión Académica para una Universidad** 

**Miguel Ángel Chavez Barrera** 

**Frank Danilo Marin Cedeño** 

# **Ingeniería De Software 3** 

# **Ingeniería De Sistemas** 

# **Universidad De La Amazonía** 

**Florencia, Caquetá** 

**Julio De 2026** 

# **Tabla de contenido** 

|Introducción.........................................................................................................................4|
|---|
|Tipo de Arquitectura Seleccionada..................................................................................4|
|Descripción General del Sistema.....................................................................................5|
|Problema que Busca Resolver.........................................................................................6|
|Acoplamiento Estructural del Sistema Heredado........................................................6|
|Fragilidad Ante Picos De Demanda.............................................................................6|
|Superficie de Seguridad Concentrada..........................................................................6|
|Desconexión Académico-Financiera...........................................................................6|
|Rigidez para Integrar Terceros.....................................................................................7|
|Costos Fijos de Infraestructura....................................................................................7|
|Propósito del Documento................................................................................................7|
|Alcance del Proyecto.......................................................................................................7|
|Identidad y Accesos (RF-01).......................................................................................8|
|Matrícula y Expediente Estudiantil (RF-02)................................................................8|
|Gestión Académica de Docentes, Horarios y Aulas (RF-03)......................................8|
|Evaluación y Seguimiento Académico (RF-04)..........................................................8|
|Integración Externa y Analítica Directiva (RF-05).....................................................8|
|Público Objetivo.............................................................................................................. 9|
|Terminología y Definiciones.............................................................................................10|



|Objetivos del Proyecto.......................................................................................................14|
|---|
|Objetivo 1: Cobertura Funcional e Interoperabilidad....................................................14|
|Objetivo 2: Disponibilidad y Resiliencia.......................................................................14|
|Objetivo 3: Escalabilidad bajo Demanda......................................................................14|
|Objetivo 4: Sostenibilidad Tecnológica y Alineación con Políticas Públicas...............14|
|Stakeholders.......................................................................................................................15|
|Requisitos Arquitectónicos................................................................................................17|
|Requisitos Funcionales..................................................................................................17|
|Requisitos no Funcionales.............................................................................................19|
|Escenarios de Atributos de Calidad...................................................................................20|
|Escenario 1, Disponibilidad / Resiliencia......................................................................20|
|Escenario 2, Escalabilidad.............................................................................................21|
|Escenario 3, Seguridad..................................................................................................22|
|Relación entre los Requisitos Arquitectónicos y la Arquitectura Propuesta..................23|
|Referencias........................................................................................................................25|



# **Lista de Tablas** 

|Tabla 1 Terminología y Definiciones.................................................................................10|
|---|
|Tabla 2 Stakeholders..........................................................................................................15|
|Tabla 3 Requisitos Funcionales.........................................................................................17|
|Tabla 4 Requisitos no Funcionales....................................................................................19|
|Tabla 5 Escenario de Disponibilidad/ Resiliencia.............................................................20|
|Tabla 6 Escenario de Escalabilidad...................................................................................21|
|Tabla 7 Escenario de Seguridad.........................................................................................22|



# **Introducción** 

# **Tipo de Arquitectura Seleccionada** 

La arquitectura seleccionada para el desarrollo de UPS-Connect que es el nombre que se propone para el sistema, entendida como una variante de la arquitectura orientada a servicios (Erl, 2016), esta se compone de cinco módulos de negocio independientes que son Identidad, Matrícula, Gestión Académica, Evaluación e Integración Externa, cada uno de ellos expone su propia API REST y se despliega sobre una plataforma cloud bajo el modelo PaaS (Microsoft, 2023), esta organización se estructura en cuatro capas (Fowler, 2002) que son un API Gateway que centraliza la autenticación y el enrutamiento de las solicitudes, una capa de aplicación donde residen los módulos de negocio, una capa de comunicación que combina el estilo REST síncrono con una cola de mensajes para casos puntuales como los pagos, y una capa de datos donde cada módulo administra su propia base de datos (Kleppmann, 2017), esto permite que cada componente evolucione sin comprometer el funcionamiento integral de la plataforma. 

# **Descripción General del Sistema** 

UPS-Connect es la plataforma de gestión académica de la Universidad Pública del Sur, esta se encuentra formada por módulos de negocio independientes que se despliegan sobre una infraestructura cloud multizona mediante un modelo PaaS, esta  plataforma centraliza los procesos de matrícula, gestión docente, horarios y aulas, evaluación académica, integración con pagos, bibliotecas digitales y gestión documental, además de disponer los paneles de control que los directivos necesitan para la toma de decisiones, convirtiendo la plataforma en el eje sobre el cual se realiza todo lo relacionado con la operación académica de la institución. 

# **Problema que Busca Resolver** 

El sistema académico con el que actualmente cuenta la UPS es monolítico, obsoleto y difícil de escalar, esta situación genera una serie de problemas estructurales que afectan al funcionamiento integral de la institución y que UPS-Connect busca resolver, a continuación, se describen cada uno de estos problemas: 

# **_Acoplamiento Estructural del Sistema Heredado_** 

Un cambio realizado sobre cualquier módulo obliga a probar y desplegar la aplicación completa, esto ralentiza la evolución del sistema y aumenta el riesgo en cada actualización que se realiza. 

# **_Fragilidad Ante Picos De Demanda_** 

Al no existir un aislamiento entre los componentes, la saturación de un proceso como puede ser el de matrícula termina afectando funciones que no se relacionan con él, como la consulta de las notas o el acceso de los docentes. 

# **_Superficie de Seguridad Concentrada_** 

La existencia de un único circulo de seguridad para todos los datos académicos, financieros y personales incrementa el impacto que tendría cualquier vulnerabilidad explotada dentro del sistema. Esta concentración de riesgo debe abordarse en línea con el bien jurídico de la protección de la información y de los datos establecido en la Ley 1273 de 2009 (Congreso de la República de Colombia, 2009). 

# **_Desconexión Académico-Financiera_** 

Actualmente hay validaciones manuales para confirmar si un estudiante se encuentra a paz y salvo antes de permitirle acceder a clases o solicitar certificados, esto se traduce en demoras y depende de la disposición del personal presente en cada momento. 

# **_Rigidez para Integrar Terceros_** 

Conectar nuevas pasarelas de pago, bibliotecas digitales o gestores de documentos exige modificaciones profundas sobre el núcleo del sistema en lugar de resolverse mediante simples adaptadores desacoplados. 

# **_Costos Fijos de Infraestructura_** 

El sistema actual se ejecuta sobre servidores propios que fueron dimensionados para el pico máximo de demanda, esto implica sobrecostos permanentes y dificulta escalar de forma elástica únicamente cuando el contexto lo requiere. 

# **Propósito del Documento** 

Más allá de la descripción de la arquitectura elegida, el propósito central de este documento es dejar constancia y trazabilidad de las decisiones arquitectónicas que se tomarán durante el diseño de UPS-Connect, esto incluye qué se decidió, por qué razón se decidió y qué impacto tiene cada una de esas decisiones sobre los requisitos y los atributos de calidad del sistema, de esta manera el documento funciona como una memoria técnica y como referencia para las futuras iteraciones, auditorías o módulos que se incorporen, esto para garantizar que las decisiones que se tomen después se mantengan guiadas con el equilibrio entre simplicidad, escalabilidad, resiliencia y seguridad que se estableció desde el inicio del proyecto. 

# **Alcance del Proyecto** 

El proyecto abarca el diseño, el desarrollo y la implementación de los módulos que se definen dentro de los Requisitos Funcionales RF-01 a RF-05, a continuación, se describe cada uno de ellos: 

# **_Identidad y Accesos (RF-01)_** 

Corresponde a la autenticación centralizada mediante OAuth2/OpenID Connect, el control 

de acceso basado en roles o RBAC y la emisión de credenciales digitales, esto permite que la 

identidad se gestione desde un único punto dentro de la plataforma. 

# **_Matrícula y Expediente Estudiantil (RF-02)_** 

Comprende la inscripción, la modificación y la cancelación de asignaturas, incorporando 

la verificación del estado financiero del estudiante antes de que la matrícula sea confirmada. 

# **_Gestión Académica de Docentes, Horarios y Aulas (RF-03)_** 

Abarca la administración de la carga académica, los planes de estudio y la asignación de 

los espacios físicos, esto sin que se generen conflictos de programación entre las actividades. 

# **_Evaluación y Seguimiento Académico (RF-04)_** 

Consiste en el registro y la publicación de las calificaciones, incorporando las notificaciones hacia los interesados cuando se presenten cambios relevantes. 

# **_Integración Externa y Analítica Directiva (RF-05)_** 

Corresponde a los adaptadores desacoplados hacia las pasarelas de pago, la biblioteca digital y la gestión documental, sumado a los paneles de control que los directivos utilizan para la toma de decisiones. 

Por su parte quedan fuera del alcance de esta primera fase la migración histórica completa de la información anterior a cinco años y el desarrollo de aplicaciones móviles nativas, esto porque ambas podrán abordarse durante las fases posteriores mediante nuevos módulos que consuman las mismas APIs que ya fueron definidas dentro de la arquitectura. 

# **Público Objetivo** 

# **_Equipo de Desarrollo y Plataforma_** 

Corresponde a los desarrolladores de cada módulo y a los ingenieros de infraestructura 

cloud, quienes utilizarán este documento como guía durante el diseño, el despliegue y la operación de la plataforma. 

# **_Líderes de Proyecto y Product Owners_** 

Quienes lo utilizarán para validar que la solución construida efectivamente satisface las necesidades del negocio universitario. 

# **_Directivos Universitarios y Coordinadores_** 

Quienes lo utilizarán para comprender el impacto operativo de la plataforma y las nuevas capacidades de analítica que se encuentran disponibles. 

# **_Equipo de Seguridad de la Información_** 

Quienes lo utilizarán para validar el modelo de autenticación, la gestión de identidades y el cumplimiento normativo en materia de protección de datos. 

# **_Estudiantes y docentes_** 

Quienes harán uso frecuente de la plataforma para sus actividades cotidianas. 

# **Terminología y Definiciones** 

_Tabla 1 Terminología y Definiciones_ 

|**Término**|**Concepto**|
|---|---|
|**Arquitectura de**<br>**Software**|Estructura de alto nivel que define los componentes de un<br>sistema, sus responsabilidades y principios de diseño.|
|**Arquitectura en Capas**<br>**(N-Tier)**|Organiza el sistema en capas con responsabilidades diferenciadas,<br>donde cada capa solo se comunica con las adyacentes.|
|**Arquitectura**<br>**Orientada a Servicios**<br>**(SOA)**|Estilo en el que las funcionalidades se exponen como servicios<br>independientes y reutilizables.|
|**Módulo de Negocio**|Unidad de software desplegable de forma independiente,<br>responsable de una capacidad de negocio específica y con su propia base<br>de datos.|
|**API REST**|Estilo de interfaz basado en HTTP y JSON que expone recursos<br>mediante operaciones estándar (GET, POST, PUT, DELETE).|
||Protocolo de intercambio de mensajes basado en XML, con|
|**SOAP**|contratos formales mediante WSDL, usado en integraciones<br>empresariales heredadas.|
||Componente que centraliza el enrutamiento, la autenticación y la|
|**API Gateway**|limitación de tasa hacia los módulos internos.|
|**PaaS (Platform as a**<br>**Service)**|Modelo cloud donde el proveedor gestiona la infraestructura y el|



|**Término**|**Concepto**|
|---|---|
||entorno de ejecución, sin administrar servidores directamente.|
|**Balanceador de Carga**|Distribuye las solicitudes entre varias instancias de un módulo<br>para evitar su saturación.|
|**Autoescalado**|Mecanismo que agrega o retira automáticamente instancias según<br>la demanda.|
|**Multi-AZ (Multi-Zona**<br>**de Disponibilidad)**|Replica componentes en varias zonas físicas de un proveedor<br>cloud para tolerar la falla de una zona.|
|**Base de Datos**<br>**Gestionada**|Servicio de base de datos administrado por el proveedor cloud,<br>que reduce la carga operativa.|
|**Base de Datos por**<br>**Módulo**|Cada módulo gestiona su propio esquema o instancia de base de<br>datos, evitando acoplamiento entre dominios.|
|**Cola de Mensajes**<br>**Gestionada**|Servicio cloud para envío y recepción asíncrona de mensajes<br>entre módulos.|
|**Comunicación**<br>**Síncrona**|El emisor espera una respuesta inmediata del receptor antes de<br>continuar.|
|**Comunicación**<br>**Asíncrona**|El emisor no espera respuesta inmediata; el mensaje se procesa<br>después, típicamente vía cola.|
|**OAuth2 / OpenID**<br>**Connect (OIDC)**|Protocolos de autorización y autenticación que permiten acceder|



|**Término**|**Concepto**|
|---|---|
||a múltiples módulos con un mismo proveedor de identidad.|
|**RBAC (Role-Based**<br>**Access Control)**|Asigna permisos a los usuarios según su rol en la organización.|
||Protocolo que cifra la comunicación cliente-servidor,|
|**HTTPS / TLS**|garantizando confidencialidad e integridad de los datos.|
|**WAF (Web**<br>**Application Firewall)**|Filtra y bloquea tráfico malicioso dirigido a aplicaciones web.|
||Detecta fallas repetidas en un módulo o servicio externo y corta|
|**Circuit Breaker**|temporalmente las llamadas hacia él.|
|**Reintentos con**<br>**Retroceso Exponencial**|Ante un fallo transitorio, reintenta la operación aumentando<br>progresivamente el tiempo de espera.|
|**Idempotencia**|Propiedad de una operación que produce el mismo resultado sin<br>importar cuántas veces se ejecute.|
|**Observabilidad**|Capacidad de inferir el estado interno de un sistema a partir de<br>métricas, logs y alertas.|
|**RPO / RTO**|Indicadores de continuidad; el RPO define la pérdida máxima<br>tolerable de datos y el RTO el tiempo máximo de recuperación.|
|**Escalabilidad**|Capacidad de soportar el incremento de usuarios o transacciones<br>sin degradar el rendimiento.|
|**Resiliencia**|Capacidad de continuar operando, o recuperarse rápidamente,|



|**Término**|**Concepto**|
|---|---|
||ante fallos.|
||Documento técnico que describe la arquitectura, las decisiones,|
|**Software Architecture**||
|**Document (SAD)**|los requisitos y los atributos de calidad de un sistema.|



# **Objetivos del Proyecto** 

# **Objetivo 1: Cobertura Funcional e Interoperabilidad** 

Este objetivo busca automatizar el 100% de los procesos de matrícula, gestión docente, horarios y evaluación académica, esto con el propósito de eliminar los procesos manuales que actualmente sostiene el sistema monolítico (RF-02, RF-03, RF-04). 

# **Objetivo 2: Disponibilidad y Resiliencia** 

Este objetivo busca garantizar una disponibilidad igual o superior al 99,9% de los módulos críticos durante los periodos de inscripción, esto mediante el despliegue multizona y las bases de datos con réplica automática, estableciendo un RTO menor a 30 minutos (RNF-01, RNF-04). 

# **Objetivo 3: Escalabilidad bajo Demanda** 

Este objetivo busca soportar al menos 15.000 usuarios concurrentes durante el proceso de matrícula manteniendo una degradación de rendimiento no mayor al 20%, esto mediante el autoescalado nativo que ofrece la plataforma PaaS (RNF-02). 

# **Objetivo 4: Sostenibilidad Tecnológica y Alineación con Políticas Públicas.** 

Este objetivo busca reducir en al menos un 70% el uso de papel y los desplazamientos físicos, esto mediante la infraestructura cloud y la digitalización de los trámites, manteniéndose en línea con el Decreto 1008 de 2018 (Ministerio de Tecnologías de la Información y las Comunicaciones de Colombia, 2018) y con los ODS 4 y 9 de la Agenda 2030. 

# **Stakeholders** 

_Tabla 2 Stakeholders_ 

|**Stakeholder / Rol**|**Descripción**|**Necesidades o Expectativas**|
|---|---|---|
|**1. Estudiante**<br>**(Rol: Usuario final)**|Corresponde al usuario<br>principal del sistema, este<br>consume<br>los<br>servicios<br>académicos y realiza los trámites<br>financieros a través de la<br>aplicación web o móvil.|Inscripción de asignaturas sin<br>interrupciones durante la matrícula<br>(RF-02).<br>Transparencia<br>y<br>disponibilidad de canales de pago<br>(RF-05). Consulta oportuna de<br>calificaciones y notificaciones<br>(RF-04).|
|**2. Docente (Rol:**|Corresponde<br>al<br>responsable de impartir las<br>|Registro ágil y confiable de<br>calificaciones y asistencia (RF-04).<br>Reserva de aulas o laboratorios sin|
|**Usuario académico)**|clases, registrar las evaluaciones<br>y utilizar los recursos físicos con<br>los que cuenta la universidad.|choques de horario (RF-03).<br>Notificaciones<br>ante<br>cambios<br>relevantes.|
|**3. Coordinador**<br>**Académico**<br>**(Rol:**<br>**Administrador**<br>**de**<br>**dominio académico)**|Corresponde a quien<br>planifica la oferta académica del<br>semestre, asigna los docentes y<br>supervisa<br>el<br>rendimiento<br>estudiantil durante el periodo.|Herramientas centralizadas<br>para construir la oferta académica<br>(RF-03). Paneles de control sobre<br>ocupación y rendimiento (RF-05).<br>Alertas tempranas ante riesgos de<br>deserción.|



|**Stakeholder / Rol**|**Descripción**|**Necesidades o Expectativas**|
|---|---|---|
|**4. Personal de**<br>**Tesorería**<br>**(Rol:**<br>**Administrador**|Corresponde a quien<br>gestiona el recaudo, la<br>conciliación de los pagos y la<br>aplicación de los beneficios|Verificación automática del<br>estado de pagos antes de confirmar la<br>matrícula<br>(RF-02,<br>RF-05).<br>Trazabilidad de cada transacción para|
|**financiero)**|financieros dentro de la<br>institución.|efectos de auditoría.|
|**5.**<br>**Administrador**<br>**de**<br>**Identidad y Seguridad**|Corresponde<br>al<br>responsable de las políticas de<br>autenticación, autorización y<br>|Visibilidad centralizada de<br>accesos y anomalías. Cumplimiento<br>verificable de la Ley 1581 de 2012 y|
|**(Rol:**<br>**Oficial**<br>**de**<br>**seguridad / IAM)**|protección de datos entre los<br>módulos que conforman la<br>plataforma.|buenas prácticas de seguridad como<br>OWASP Top 10 (RF-01, RNF-03).|
|**6. Equipo de**<br>**Plataforma**<br>**(Rol:**<br>**Operaciones**<br>**/**|Corresponde<br>al<br>encargado de operar la<br>infraestructura<br>PaaS,<br>los<br>ili d dli  l|Aislamiento real entre<br>módulos que evite el efecto dominó<br>ante fallos. Métricas, logs y alertas<br>centralizadas para detectar incidentes|
|**Infraestructura**<br>**Cloud)**|ppenes e espegue y e<br>monitoreo continuo de los<br>módulos.|rápidamente,<br>sin<br>requerir<br>infraestructura de orquestación<br>propia.|
|**7.**<br>**Alta**|Corresponde a quien|Retorno<br>de<br>inversión|
|**Dirección (Rectoría /**|patrocina el proyecto y toma las|mediante la reducción de la deserción|



|**Stakeh**|**older / Rol**<br>**Descripción**|**Necesidades o Expectativas**|
|---|---|---|
|**Vicerrect**<br>**equisitos**<br>_Tab_<br>|**orías)**<br>decisiones estratégicas a partir de<br>los indicadores que la plataforma<br>genera.<br> <br><br> <br> <br>**Requisitos Arquitectónicos**<br>**Funcionales**<br>_la 3 Requisitos Funcionales_<br>|estudiantil y la optimización del uso<br>de<br>infraestructura<br>física.<br>Cumplimiento de políticas públicas<br>de gobierno digital y sostenibilidad.<br>|
|**Código**|**Requisito Funcional**|**Justificación Arquitectónica**|
||El sistema debe permitir la autenticación y<br>autorización centralizada de estudiantes, docentes,|<br> <br>Este requisito centraliza la<br>gestión de la identidad y permite|
|RF-01|coordinadores, personal de tesorería y<br>administradores, mediante OAuth2/OpenID<br>Connect y control de acceso basado en roles<br>(RBAC).|<br> <br> <br>que el API Gateway aplique una<br>política de acceso consistente<br>sobre todos los módulos de la<br>plataforma.|
||El sistema debe gestionar el proceso<br>integral de matrícula y el expediente estudiantil|<br> <br>Este requisito corresponde<br>al proceso crítico del negocio|
|RF-02|(inscripción, modificación, cancelación, historial),<br>verificando el estado financiero del estudiante<br>antes de confirmar la matrícula.|<br> <br>universitario y es el que exige<br>mayor disponibilidad y mayor<br>consistencia frente al módulo|



# **Requisitos Funcionales** 

|**Código**|**Requisito Funcional**|**Justificación Arquitectónica**|
|---|---|---|
|||financiero.|
||El sistema debe administrar docentes,|Este requisito aísla la<br>lógica de programación académica|
|RF-03|asignaturas, carga académica, horarios y aulas,<br>evitando conflictos de programación mediante un<br>motor de reservas dedicado.|dentro<br>de<br>un<br>módulo<br>independiente, esto permite que se<br>escale por separado del resto de la<br>plataforma.|
|RF-04|El sistema debe permitir el registro,<br>consulta y publicación de calificaciones y<br>actividades evaluativas, notificando a los<br>interesados ante cambios relevantes.|Este requisito aprovecha la<br>cola de mensajes gestionada para<br>desacoplar el registro de las notas<br>del envío de las notificaciones<br>hacia los interesados.|
|||Este requisito favorece la|
||El sistema debe integrarse de forma|interoperabilidad<br>institucional|
|RF-05|desacoplada con servicios externos (pasarelas de<br>pago, biblioteca digital, gestión documental) y|mediante adaptadores REST, esto<br>sin que el núcleo académico quede|
||generar paneles de control para directivos.|acoplado a los proveedores<br>externos.|



# **Requisitos no Funcionales** 

Los requisitos no funcionales que se describen a continuación se clasifican de acuerdo con 

los atributos de calidad definidos en la norma ISO/IEC 25010:2011 (International Organization for Standardization, 2011). 

_Tabla 4 Requisitos no Funcionales_ 

|**Código**|**Requisito No Funcional**|**Atributo de Calidad**|
|---|---|---|
|**RNF-01**|El sistema deberá garantizar una disponibilidad<br>igual o superior al 99,9 % de los módulos críticos durante<br>los periodos académicos activos, mediante despliegue<br>multi-zona gestionado por la plataforma PaaS.|Disponibilidad|
|**RNF-02**|El sistema deberá soportar al menos 15.000 usuarios<br>concurrentes durante el proceso de matrícula, mediante<br>autoescalado horizontal gestionado nativamente por la<br>plataforma cloud.|Escalabilidad|
||Toda la comunicación, tanto externa como entre<br>módulos, deberá cifrarse mediante HTTPS/TLS 1.2 o||
|**RNF-03**|superior, con autenticación OAuth2/OIDC y control de<br>acceso RBAC, cumpliendo con la Ley 1581 de 2012<br>(Congreso de la República de Colombia, 2012).|Seguridad|
|**RNF-04**|Ante la falla de un módulo individual o de un<br>servicio externo, el sistema deberá aislar el impacto<br>mediante Circuit Breaker y reintentos con retroceso<br>exponencial, con un tiempo objetivo de recuperación (RTO)|Resiliencia|



|**Código**|**Requisito No Funcional**|**Atributo de Calidad**|
|---|---|---|
||menor a 30 minutos y una pérdida máxima de datos (RPO)<br>menor a 15 minutos.||
||El sistema deberá centralizar los registros (logs),||
|**RNF-05**|métricas y alertas de todos los módulos en una única consola<br>de observabilidad, permitiendo detectar incidentes en|Observabilidad /<br>Mantenibilidad|
||menos de 5 minutos desde su ocurrencia.||



# **Escenarios de Atributos de Calidad** 

A continuación, se presentan los escenarios de atributos de calidad siguiendo la estructura 

de Fuente, Estímulo, Ambiente, Artefacto, Respuesta y Medida de respuesta propuesta por Bass, Clements y Kazman (2021). 

# **Escenario 1, Disponibilidad / Resiliencia** 

_Tabla 5 Escenario de Disponibilidad/ Resiliencia_ 

|**Atributo**|**_Disponibilidad / Resiliencia_**|
|---|---|
|**Fuente**|Corresponde a los miles de estudiantes que utilizan la plataforma y al<br>propio sistema de pagos externo.|
|**Estímulo**|Se presenta cuando el servicio de pagos externo deja de responder<br>durante el periodo oficial de matrícula.|
|**Ambiente**|Corresponde al periodo oficial de matrículas, caracterizado por una|



||alta concurrencia de usuarios.|
|---|---|
|**Artefacto**|Corresponde al Módulo de Matrícula junto al adaptador REST de<br>integración con la pasarela de pagos.|
||El Circuit Breaker del cliente HTTP corta las llamadas hacia el<br>servicio externo tras un número definido de fallos, por su parte la matrícula|
|**Respuesta**|queda registrada en estado 'pendiente de pago' y un proceso de reintento con<br>retroceso exponencial reconcilia el estado cuando el servicio se restablece,<br>esto sin que el resto de estudiantes quede bloqueado.|
||El resto de la plataforma mantiene una disponibilidad igual o superior|
|**Medida de**|al 99,9 % aún cuando el servicio externo se encuentre caído, además la|
|**respuesta**|conciliación se completa en menos de 30 minutos una vez ocurrido el<br>restablecimiento.|



# **Escenario 2, Escalabilidad** 

# _Tabla 6 Escenario de Escalabilidad_ 

|**Atributo**|**_Escalabilidad_**|
|---|---|
|**Fuente**|Corresponde al incremento masivo y simultáneo de usuarios sobre la<br>plataforma.|
|**Estímulo**|Se presenta cuando más de 15.000 estudiantes acceden de manera<br>simultánea al módulo de matrícula durante el primer día de inscripciones.|



|**Ambiente**|Corresponde a un periodo de alta demanda estacional dentro del<br>calendario académico.|
|---|---|
|**Artefacto**|Corresponde al Módulo de Matrícula desplegado sobre la plataforma<br>PaaS, ubicado detrás del API Gateway y del balanceador de carga.|
||El autoescalado nativo de la plataforma cloud añade de manera<br>automática nuevas instancias del módulo de Matrícula en función del uso de|
|**Respuesta**|CPU y del número de solicitudes en cola, por su parte el API Gateway aplica<br>la limitación de tasa con el objetivo de proteger al módulo durante el pico de<br>demanda.|
|**Medida de**<br>**respuesta**|La degradación del tiempo de respuesta no supera el 20 % respecto a<br>la operación normal, además no se registran caídas del servicio durante el<br>evento.|



# **Escenario 3, Seguridad** 

_Tabla 7 Escenario de Seguridad_ 

|**Atributo**|**_Seguridad_**|
|---|---|
|**Fuente**|Corresponde a un usuario autenticado dentro de la plataforma con rol<br>de estudiante.|
|**Estímulo**|Se presenta cuando un estudiante intenta acceder mediante la|
||manipulación de una solicitud a un endpoint del módulo financiero que se|



||encuentra reservado para el personal de tesorería.|
|---|---|
|**Ambiente**|Corresponde a la operación normal de la plataforma, con tráfico<br>autenticado a través del API Gateway.|
|**Artefacto**|Corresponde al API Gateway, al proveedor de identidad<br>OAuth2/OIDC y a la política RBAC del módulo financiero.|
||El API Gateway valida el token y el rol del usuario, al no contar con|
|**Respuesta**|los permisos requeridos la solicitud es rechazada antes de llegar al módulo<br>financiero, además el intento queda registrado dentro del sistema de auditoría.|
|**Medida de**<br>**respuesta**|El acceso se bloquea en menos de 2 segundos sin que se expongan<br>datos financieros, además el evento queda disponible para auditoría en menos<br>de 5 minutos.|



# **Relación entre los Requisitos Arquitectónicos y la Arquitectura Propuesta** 

La elección de una arquitectura modular en capas orientada a servicios REST y desplegada sobre una plataforma cloud PaaS permite satisfacer de manera directa los requisitos que fueron identificados anteriormente, esto porque el aislamiento por módulo y el patrón Circuit Breaker son los que sostienen la disponibilidad y la resiliencia del sistema, por su parte el autoescalado y el balanceo de carga que el proveedor cloud gestiona de forma nativa son los que sostienen la escalabilidad ante los picos de demanda que se presentan durante los periodos de inscripción, y finalmente la combinación de OAuth2/OIDC, RBAC y el cifrado HTTPS/TLS es la que sostiene la seguridad que exige la normativa colombiana de protección de datos. A diferencia de una 

arquitectura de microservicios completa con orquestación de contenedores y mensajería distribuida (Richardson, 2018), estos mecanismos se obtienen en gran medida como servicios gestionados del proveedor cloud, esto se traduce en una reducción importante de la complejidad operativa sin que se sacrifiquen los atributos de calidad que resultan prioritarios para el proyecto, garantizando así que las decisiones arquitectónicas tomadas respondan al contexto real de la Universidad Pública del Sur y no a un ideal técnico desconectado de sus capacidades. 

# **Referencias** 

Bass, L., Clements, P., & Kazman, R. (2021). Software architecture in practice (4th ed.). Addison-Wesley. 

Congreso de la República de Colombia. (2012). Ley 1581 de 2012, por la cual se dictan disposiciones generales para la protección de datos personales. Diario Oficial. 

Congreso de la República de Colombia. (2009). Ley 1273 de 2009, por medio de la cual se modifica el Código Penal y se crea el bien jurídico de la protección de la información y de los datos. Diario Oficial. 

Erl, T. (2016). Service-oriented architecture: Analysis and design for services and microservices (2nd ed.). Prentice Hall. 

Fowler, M. (2002). Patterns of enterprise application architecture. Addison-Wesley. 

International Organization for Standardization. (2011). ISO/IEC 25010:2011 — Systems and software engineering — Systems and software Quality Requirements and Evaluation (SQuaRE). ISO. 

Kleppmann, M. (2017). Designing data-intensive applications. O'Reilly Media. 

Microsoft. (2023). Azure well-architected framework. https://learn.microsoft.com/azure/well-architected/ 

Ministerio de Tecnologías de la Información y las Comunicaciones de Colombia. (2018). Decreto 1008 de 2018, por el cual se establecen los lineamientos generales de la Política de Gobierno Digital. 

Naciones Unidas. (2015). Transformar nuestro mundo: la Agenda 2030 para el Desarrollo Sostenible (A/RES/70/1). 

OWASP Foundation. (2023). OWASP top 10: The ten most critical web application security risks. https://owasp.org/ 

Richardson, C. (2018). Microservices patterns: With examples in Java. Manning 

Publications. 

