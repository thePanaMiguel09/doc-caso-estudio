# Vista 1 — Lógica

> **Modelo 4+1 · Vista Lógica.** Describe la estructura estática del dominio —clases, atributos, operaciones, asociaciones y cardinalidades— y la colaboración dinámica entre objetos mediante diagramas de secuencia. Su destinatario es el equipo de desarrollo y los analistas funcionales.

**Cobertura:** 6 diagramas de clases (mapa de dominio + 5 módulos) · 5 diagramas de secuencia · 1 mapa de referencias inter-módulo. **Total: 12 diagramas.**

---

## Principio rector de esta vista

El documento arquitectónico establece **una base de datos por módulo**. Esa decisión impone una restricción que gobierna todo el modelo de clases: el sistema **no puede** representarse como un único modelo entidad-relación con claves foráneas cruzadas. Hacerlo contradiría la arquitectura en el mismo documento que pretende describirla.

En consecuencia se aplican dos reglas verificables:

| Ámbito | Relación permitida | Notación |
|---|---|---|
| **Dentro de un módulo** | Asociación navegable, composición, agregación, herencia | `-->` `*--` `o--` `<|--` |
| **Entre módulos** | Únicamente dependencia hacia referencias por identificador | `..>` con etiqueta del identificador |

Esto se conoce como **frontera de agregado**: es la traducción de la decisión "base de datos por módulo" al nivel de clases. Ninguna asociación navegable cruza un límite modular en toda esta vista.

---

## 1. Mapa de dominio — agregados y fronteras

Vista de conjunto de los agregados raíz de cada módulo y sus referencias cruzadas.

```mermaid
flowchart TB
    subgraph A["Módulo Identidad · RF-01"]
        A1["<b>Usuario</b><br/><i>agregado raíz</i>"]
        A2["Rol"]
        A3["RegistroAuditoria<br/><i>agregado raíz</i>"]
    end

    subgraph B["Módulo Matrícula · RF-02"]
        B1["<b>Matricula</b><br/><i>agregado raíz</i>"]
        B2["ExpedienteEstudiantil<br/><i>agregado raíz</i>"]
    end

    subgraph C["Módulo Gestión Académica · RF-03"]
        C1["<b>PlanEstudio</b><br/><i>agregado raíz</i>"]
        C2["<b>Grupo</b><br/><i>agregado raíz</i>"]
        C3["<b>Aula</b><br/><i>agregado raíz</i>"]
        C4["CargaAcademica<br/><i>agregado raíz</i>"]
    end

    subgraph D["Módulo Evaluación · RF-04"]
        D1["<b>ActaNotas</b><br/><i>agregado raíz</i>"]
        D2["SolicitudRevision<br/><i>agregado raíz</i>"]
    end

    subgraph E["Módulo Integración · RF-05"]
        E1["<b>TransaccionPago</b><br/><i>agregado raíz</i>"]
        E2["Documento<br/><i>agregado raíz</i>"]
        E3["PanelIndicadores"]
    end

    B1 -.->|EstudianteId| A1
    B1 -.->|GrupoId| C2
    C2 -.->|DocenteId| A1
    C4 -.->|DocenteId| A1
    D1 -.->|GrupoId| C2
    D1 -.->|EstudianteId| A1
    E1 -.->|MatriculaId| B1
    E2 -.->|EstudianteId| A1
    E3 -.->|lectura| B2
    E3 -.->|lectura| C3
    E3 -.->|lectura| D1
    B1 -.->|IEstadoFinanciero| E1

    classDef ag fill:#E6F4EA,stroke:#137333,color:#000
    class A1,A2,A3,B1,B2,C1,C2,C3,C4,D1,D2,E1,E2,E3 ag
```

**Justificación.** Todas las flechas inter-módulo son punteadas y llevan como etiqueta un **identificador**, no una referencia a objeto. `Matricula` almacena un `GrupoId` de tipo `UUID`, no un puntero a la instancia de `Grupo`. Esta es la propiedad que permite que las cinco bases de datos sean físicamente independientes y que un módulo se despliegue sin requerir la presencia del otro.

---

## 2. Diagrama de clases — Módulo Identidad y Accesos (RF-01)

```mermaid
classDiagram
    direction TB

    class Usuario {
        <<abstract>>
        -UUID id
        -String documento
        -String nombreCompleto
        -String correoInstitucional
        -EstadoCuenta estado
        -DateTime fechaCreacion
        -DateTime ultimoAcceso
        +autenticar(credenciales) TokenOIDC
        +tieneRol(nombreRol) bool
        +tienePermiso(recurso, accion) bool
        +activar() void
        +desactivar(motivo) void
        +registrarAcceso() void
    }

    class Estudiante {
        -String codigoEstudiantil
        -UUID programaId
        -int semestreActual
        -decimal promedioAcumulado
        -EstadoAcademico estadoAcademico
        +estaHabilitadoParaMatricula() bool
    }

    class Docente {
        -String codigoDocente
        -TipoVinculacion vinculacion
        -int horasContratadas
        -String titulacionMaxima
        +horasDisponibles(periodo) int
        +puedeAsumirCarga(horas) bool
    }

    class Coordinador {
        -UUID programaACargo
        -DateTime fechaDesignacion
        +administraPrograma(programaId) bool
    }

    class Tesorero {
        -String centroCosto
        -decimal topeAprobacion
        +puedeAprobar(monto) bool
    }

    class AdministradorSeguridad {
        -String nivelAcceso
        +puedeRevocarSesiones() bool
    }

    class Rol {
        -UUID id
        -String nombre
        -String descripcion
        -bool esSistema
        +agregarPermiso(permiso) void
        +revocarPermiso(permisoId) void
        +permisos() List~Permiso~
    }

    class Permiso {
        -UUID id
        -String recurso
        -Accion accion
        -String descripcion
        +permite(recurso, accion) bool
    }

    class TokenOIDC {
        -String accessToken
        -String refreshToken
        -String subject
        -List~String~ scopes
        -DateTime emitidoEn
        -DateTime expiraEn
        +esValido() bool
        +contieneScope(scope) bool
        +tiempoRestante() Duration
        +revocar() void
    }

    class CredencialDigital {
        -UUID id
        -String numeroSerie
        -String huellaFirma
        -DateTime vigenteDesde
        -DateTime vigenteHasta
        +estaVigente() bool
        +verificarFirma() bool
    }

    class RegistroAuditoria {
        -UUID id
        -UUID usuarioId
        -String recurso
        -Accion accion
        -ResultadoAcceso resultado
        -String direccionIp
        -String agenteUsuario
        -DateTime ocurridoEn
        +esAnomalia() bool
        +serializar() String
    }

    class SesionActiva {
        -UUID id
        -UUID usuarioId
        -String identificadorDispositivo
        -DateTime iniciadaEn
        -DateTime ultimaActividad
        +estaExpirada() bool
        +renovar() void
        +cerrar() void
    }

    Usuario <|-- Estudiante
    Usuario <|-- Docente
    Usuario <|-- Coordinador
    Usuario <|-- Tesorero
    Usuario <|-- AdministradorSeguridad

    Usuario "1" o-- "1..*" Rol : posee
    Rol "1" *-- "1..*" Permiso : agrupa
    Usuario "1" --> "0..*" TokenOIDC : emite
    Usuario "1" --> "0..1" CredencialDigital : porta
    Usuario "1" --> "0..*" RegistroAuditoria : genera
    Usuario "1" --> "0..*" SesionActiva : mantiene
    TokenOIDC "1" --> "1" SesionActiva : vincula
```

### Decisiones de diseño — RF-01

| Decisión | Justificación |
|---|---|
| Herencia en `Usuario` con cinco subclases | Los cinco perfiles comparten identidad, credenciales y ciclo de vida, y difieren solo en atributos propios. Es el único punto del sistema donde la generalización es apropiada. |
| `Rol` en agregación (`o--`) y `Permiso` en composición (`*--`) | Un rol existe independientemente del usuario que lo porta; un permiso no tiene sentido fuera del rol que lo agrupa y muere con él. |
| `TokenOIDC` como entidad, no como cadena | Permite modelar expiración, revocación y scopes de forma explícita, y hace verificable el corte de acceso del escenario de seguridad. |
| `RegistroAuditoria` como agregado raíz independiente | La auditoría debe sobrevivir a la eliminación de un usuario. Si fuera parte del agregado `Usuario`, desactivar una cuenta borraría su rastro — inaceptable bajo la Ley 1581 de 2012. |
| `SesionActiva` separada de `TokenOIDC` | Permite revocación centralizada (UC-06.1) sin esperar la expiración natural del token. |

---

## 3. Diagrama de clases — Módulo Matrícula y Expediente (RF-02)

```mermaid
classDiagram
    direction TB

    class Matricula {
        -UUID id
        -UUID estudianteId
        -PeriodoAcademico periodo
        -EstadoMatricula estado
        -DateTime fechaCreacion
        -DateTime fechaConfirmacion
        -decimal valorTotal
        -String claveIdempotencia
        +agregarAsignatura(grupoId, creditos) void
        +cancelarAsignatura(detalleId) void
        +confirmar() Resultado
        +marcarPendientePago() void
        +expirar() void
        +totalCreditos() int
        +excedeTopeCreditos() bool
        +puedeModificarse() bool
    }

    class DetalleMatricula {
        -UUID id
        -UUID grupoId
        -UUID asignaturaId
        -int creditos
        -EstadoDetalle estado
        -TipoInscripcion tipo
        -DateTime inscritoEn
        +anular(motivo) void
        +esRepeticion() bool
    }

    class ExpedienteEstudiantil {
        -UUID id
        -UUID estudianteId
        -decimal creditosAprobados
        -decimal creditosCursados
        -decimal promedioPonderado
        -int periodosCursados
        +historialMatriculas() List~Matricula~
        +recalcularPromedio() decimal
        +haAprobado(asignaturaId) bool
        +porcentajeAvance() decimal
    }

    class EstadoFinanciero {
        <<value object>>
        -bool pazYSalvo
        -decimal saldoPendiente
        -DateTime consultadoEn
        -bool datoDegradado
        -String referenciaConsulta
        +permiteMatricula() bool
        +esConfiable() bool
        +haExpirado() bool
    }

    class PeriodoAcademico {
        <<value object>>
        -int anio
        -int numero
        -DateTime inicioMatricula
        -DateTime finMatricula
        -DateTime finCancelacion
        +estaAbierto() bool
        +permiteCancelacion() bool
        +etiqueta() String
    }

    class ValidadorPrerrequisitos {
        <<service>>
        +validar(expediente, asignaturaId) Resultado
        +validarConjunto(expediente, asignaturas) List~Resultado~
        +verificarTopeCreditos(matricula) bool
        +verificarSolapamientoHorario(detalles) bool
    }

    class ReconciliacionService {
        <<service>>
        +reconciliar(matriculaId) Resultado
        +expirarVencidas(limite) int
        +calcularProximoIntento(intentos) Duration
    }

    class SolicitudCancelacion {
        -UUID id
        -UUID matriculaId
        -String motivo
        -EstadoSolicitud estado
        -DateTime solicitadaEn
        +aprobar() void
        +rechazar(razon) void
        +generaNotaCredito() bool
    }

    Matricula "1" *-- "1..*" DetalleMatricula : contiene
    Matricula "1" --> "1" PeriodoAcademico : pertenece_a
    Matricula "1" --> "0..1" EstadoFinanciero : evalua
    Matricula "1" --> "0..1" SolicitudCancelacion : origina
    ExpedienteEstudiantil "1" o-- "0..*" Matricula : historiza
    ValidadorPrerrequisitos ..> Matricula : valida
    ValidadorPrerrequisitos ..> ExpedienteEstudiantil : consulta
    ReconciliacionService ..> Matricula : reconcilia
```

### Decisiones de diseño — RF-02

| Decisión | Justificación |
|---|---|
| `EstadoFinanciero` como *value object* con `datoDegradado` | El módulo de Matrícula **no posee** el dato financiero: lo consulta y lo trata como fotografía inmutable con marca temporal. El indicador `datoDegradado` permite distinguir "verificado y aprobado" de "no se pudo verificar". Sin él, una caída de la pasarela sería indistinguible de un paz y salvo confirmado — un fallo simultáneo de seguridad y de negocio. |
| `claveIdempotencia` en `Matricula` | Durante el pico de inscripciones el doble clic y el reintento del navegador son masivos. Esta clave garantiza que N envíos idénticos produzcan una sola matrícula. Es la contrapartida obligatoria de la política de reintentos de RNF-04. |
| `DetalleMatricula` en composición | Un detalle no existe fuera de su matrícula: su ciclo de vida está totalmente contenido. La composición lo expresa formalmente. |
| `PeriodoAcademico` como *value object* con reglas de fecha | Concentra las reglas temporales —¿está abierta la matrícula?, ¿se permite cancelar?— en un objeto inmutable reutilizable, en lugar de dispersarlas en condicionales por todo el servicio. |
| `ValidadorPrerrequisitos` y `ReconciliacionService` como servicios sin estado | Son reglas que involucran varias entidades y no pertenecen naturalmente a ninguna. Ubicarlas dentro de `Matricula` produciría el antipatrón de entidad-Dios. |

---

## 4. Diagrama de clases — Módulo Gestión Académica (RF-03)

```mermaid
classDiagram
    direction TB

    class PlanEstudio {
        -UUID id
        -UUID programaId
        -String version
        -int creditosTotales
        -int semestresPrevistos
        -EstadoPlan estado
        -DateTime vigenteDesde
        +asignaturasDeSemestre(n) List~Asignatura~
        +estaVigente() bool
        +validarIntegridad() Resultado
    }

    class Asignatura {
        -UUID id
        -String codigo
        -String nombre
        -int creditos
        -int horasSemanales
        -TipoAsignatura tipo
        -List~UUID~ prerrequisitos
        +requierePrerrequisito(id) bool
        +requiereLaboratorio() bool
        +esElectiva() bool
    }

    class Grupo {
        -UUID id
        -UUID asignaturaId
        -UUID docenteId
        -String identificador
        -int cupoMaximo
        -int cupoOcupado
        -int cupoProvisional
        -int version
        -EstadoGrupo estado
        +hayCupo() bool
        +cupoDisponible() int
        +reservarCupo() bool
        +reservarCupoProvisional(ttl) bool
        +confirmarProvisional() void
        +liberarCupo() void
        +estaLleno() bool
    }

    class FranjaHoraria {
        <<value object>>
        -DiaSemana dia
        -Time horaInicio
        -Time horaFin
        +duracionMinutos() int
        +seSolapaCon(otra) bool
        +etiqueta() String
    }

    class Horario {
        -UUID id
        -UUID grupoId
        -EstadoHorario estado
        +franjas() List~FranjaHoraria~
        +horasSemanales() int
        +chocaCon(otro) bool
    }

    class Aula {
        -UUID id
        -String codigo
        -String edificio
        -int piso
        -TipoEspacio tipo
        -int capacidad
        -bool tieneProyector
        -EstadoAula estado
        +admite(cantidadEstudiantes) bool
        +estaDisponible(franja) bool
        +esLaboratorio() bool
    }

    class ReservaAula {
        -UUID id
        -UUID aulaId
        -UUID grupoId
        -UUID solicitanteId
        -EstadoReserva estado
        -TipoReserva tipo
        -DateTime creadaEn
        +confirmar() void
        +liberar() void
        +esRecurrente() bool
    }

    class CargaAcademica {
        -UUID id
        -UUID docenteId
        -PeriodoAcademico periodo
        -int horasAsignadas
        -int horasTope
        -EstadoCarga estado
        +agregarGrupo(grupoId, horas) Resultado
        +excedeTope() bool
        +horasDisponibles() int
        +porcentajeOcupacion() decimal
    }

    class MotorReservas {
        <<service>>
        +detectarConflictoAula(reserva) List~Conflicto~
        +detectarConflictoDocente(docenteId, franja) List~Conflicto~
        +detectarConflictoGrupo(grupoId, franja) List~Conflicto~
        +asignarEspacio(grupo, requisitos) ReservaAula
        +proponerAlternativas(conflicto) List~FranjaHoraria~
    }

    class Conflicto {
        <<value object>>
        -TipoConflicto tipo
        -UUID recursoAfectado
        -String descripcion
        -Severidad severidad
        +esBloqueante() bool
    }

    PlanEstudio "1" *-- "1..*" Asignatura : define
    Asignatura "1" --> "0..*" Grupo : se_dicta_en
    Grupo "1" *-- "1" Horario : tiene
    Horario "1" *-- "1..*" FranjaHoraria : compone
    Grupo "1" --> "1..*" ReservaAula : requiere
    Aula "1" o-- "0..*" ReservaAula : es_reservada
    CargaAcademica "1" o-- "0..*" Grupo : agrupa
    MotorReservas ..> ReservaAula : valida
    MotorReservas ..> CargaAcademica : verifica
    MotorReservas ..> Conflicto : produce
    MotorReservas ..> Horario : analiza
```

### Decisiones de diseño — RF-03

| Decisión | Justificación |
|---|---|
| `Grupo` concentra el control de cupo con `version` | Es el recurso escaso disputado por miles de usuarios simultáneos. Aislar allí la exclusión mutua permite bloqueo optimista sobre un objeto pequeño en vez de sobre el agregado completo de matrícula. Si el cupo viviera en `Asignatura`, toda la asignatura sería un punto de serialización global. |
| `cupoProvisional` separado de `cupoOcupado` | Permite retener un lugar mientras se resuelve el pago, sin contarlo como confirmado. Es el soporte estructural de la resiliencia de RNF-04. |
| `Aula` con atributo `TipoEspacio` en lugar de subclases | Una universidad agrega tipos de espacio con frecuencia. El tipo como dato es evolutivo; como subclase exigiría redespliegue. |
| `MotorReservas` como servicio con tres detectores de conflicto | RF-03 nombra explícitamente el "motor de reservas dedicado". Los tres tipos de conflicto —aula ocupada, docente con choque, grupo con solapamiento— son independientes y deben poder evaluarse en paralelo. |
| `Conflicto` como *value object* con severidad | Distingue conflictos bloqueantes de advertencias, permitiendo que el coordinador tome decisiones informadas en lugar de recibir un rechazo binario. |

---

## 5. Diagrama de clases — Módulo Evaluación y Seguimiento (RF-04)

```mermaid
classDiagram
    direction TB

    class ActaNotas {
        -UUID id
        -UUID grupoId
        -UUID docenteId
        -PeriodoAcademico periodo
        -EstadoActa estado
        -DateTime fechaApertura
        -DateTime fechaCierre
        -String hashIntegridad
        +abrir() void
        +cerrar() Resultado
        +reabrir(autorizacion) void
        +estaCompleta() bool
        +porcentajeRegistrado() decimal
        +calcularDefinitivas() List~NotaDefinitiva~
    }

    class ActividadEvaluativa {
        -UUID id
        -UUID actaId
        -String nombre
        -TipoActividad tipo
        -decimal porcentaje
        -DateTime fechaAplicacion
        -DateTime fechaLimiteRegistro
        -EstadoActividad estado
        +estaVigente() bool
        +permiteRegistro() bool
        +cerrarRegistro() void
    }

    class Calificacion {
        -UUID id
        -UUID actividadId
        -UUID estudianteId
        -decimal nota
        -EstadoNota estado
        -String observacion
        -DateTime registradaEn
        -DateTime publicadaEn
        +registrar(valor) void
        +modificar(valor, motivo) EventoNota
        +publicar() EventoNota
        +estaEnRango() bool
        +esAprobatoria() bool
    }

    class NotaDefinitiva {
        <<value object>>
        -UUID estudianteId
        -decimal valor
        -bool aprobada
        -DateTime calculadaEn
        +equivalenteLiteral() String
    }

    class SolicitudRevision {
        -UUID id
        -UUID calificacionId
        -UUID estudianteId
        -String argumento
        -EstadoSolicitud estado
        -DateTime solicitadaEn
        -DateTime resueltaEn
        +responder(decision, nuevaNota) void
        +estaEnPlazo() bool
        +escalar() void
    }

    class EventoNota {
        <<domain event>>
        -UUID id
        -TipoEventoNota tipo
        -UUID destinatarioId
        -UUID calificacionId
        -String resumen
        -DateTime ocurridoEn
        +aMensaje() Mensaje
        +requiereNotificacion() bool
    }

    class HistorialCambioNota {
        -UUID id
        -UUID calificacionId
        -decimal valorAnterior
        -decimal valorNuevo
        -UUID modificadaPor
        -String motivo
        -DateTime ocurridoEn
        +esModificacionPostCierre() bool
    }

    ActaNotas "1" *-- "1..*" ActividadEvaluativa : consolida
    ActividadEvaluativa "1" *-- "0..*" Calificacion : evalua
    ActaNotas "1" --> "0..*" NotaDefinitiva : produce
    Calificacion "1" --> "0..*" EventoNota : emite
    Calificacion "1" --> "0..*" HistorialCambioNota : registra
    Calificacion "1" --> "0..*" SolicitudRevision : recibe
    SolicitudRevision "1" --> "0..1" EventoNota : dispara
```

### Decisiones de diseño — RF-04

| Decisión | Justificación |
|---|---|
| `EventoNota` como *domain event* en lugar de llamada directa a notificaciones | RF-04 exige desacoplar el registro de notas del envío de notificaciones mediante cola gestionada. El evento es la representación lógica de esa cola: `Calificacion` lo produce y termina su transacción; quién lo consuma es irrelevante para el módulo. |
| `HistorialCambioNota` como entidad separada | Una calificación modificada tras la publicación tiene implicaciones académicas y legales. El historial inmutable garantiza trazabilidad y hace auditables las modificaciones extemporáneas. |
| `ActaNotas` con `hashIntegridad` y estados de cierre/reapertura | El cierre convierte el acta en documento inmutable. La reapertura requiere autorización explícita y queda registrada. Sin esta frontera, un docente podría calificar a un grupo cuya composición cambia simultáneamente. |
| `NotaDefinitiva` como *value object* calculado, no almacenado como entidad | Se deriva de las calificaciones y sus porcentajes. Persistirla como entidad independiente crearía dos fuentes de verdad susceptibles de divergir. |
| Suma de `porcentaje` de actividades validada en `ActaNotas` | La regla "los porcentajes deben sumar 100 %" es una invariante del agregado, no de la actividad individual. Reside en el agregado raíz. |

---

## 6. Diagrama de clases — Módulo Integración Externa y Analítica (RF-05)

```mermaid
classDiagram
    direction TB

    class AdaptadorPagos {
        <<adapter>>
        -String urlBase
        -String identificadorComercio
        -Duration tiempoEspera
        +consultarEstadoFinanciero(estudianteId) EstadoFinanciero
        +iniciarPago(orden) TransaccionPago
        +confirmarPago(referencia) bool
        +anularPago(referencia, motivo) bool
    }

    class AdaptadorBiblioteca {
        <<adapter>>
        -String urlBase
        +buscarRecurso(consulta) List~Recurso~
        +otorgarAcceso(usuarioId) TokenAcceso
        +revocarAcceso(usuarioId) void
    }

    class AdaptadorDocumental {
        <<adapter>>
        -String urlBase
        -String contenedor
        +generarCertificado(tipo, datos) Documento
        +almacenar(documento) String
        +recuperar(identificador) Documento
        +verificarAutenticidad(codigo) bool
    }

    class CircuitBreaker {
        <<policy>>
        -String nombreServicio
        -EstadoCircuito estado
        -int umbralFallos
        -int fallosConsecutivos
        -Duration ventanaApertura
        -DateTime abiertoDesde
        +ejecutar(operacion) Resultado
        +permitePaso() bool
        +registrarExito() void
        +registrarFallo() void
        +abrir() void
        +cerrar() void
        +intentarSemiabierto() bool
    }

    class PoliticaReintento {
        <<policy>>
        -int intentosMaximos
        -Duration esperaInicial
        -decimal factorMultiplicador
        -Duration esperaMaxima
        +proximaEspera(numeroIntento) Duration
        +debeReintentar(intento, error) bool
    }

    class TransaccionPago {
        -UUID id
        -UUID matriculaId
        -UUID estudianteId
        -String referenciaExterna
        -decimal monto
        -MedioPago medio
        -EstadoPago estado
        -String claveIdempotencia
        -DateTime creadaEn
        -DateTime conciliadaEn
        +conciliar() bool
        +estaVencida() bool
        +puedeReintentarse() bool
    }

    class BeneficioFinanciero {
        -UUID id
        -UUID estudianteId
        -TipoBeneficio tipo
        -decimal porcentajeDescuento
        -PeriodoAcademico periodo
        -EstadoBeneficio estado
        +aplicarA(monto) decimal
        +estaVigente() bool
    }

    class Documento {
        -UUID id
        -TipoDocumento tipo
        -UUID estudianteId
        -String codigoVerificacion
        -String rutaAlmacenamiento
        -String hashContenido
        -DateTime emitidoEn
        -DateTime vigenteHasta
        +estaVigente() bool
        +urlVerificacion() String
    }

    class PanelIndicadores {
        -UUID id
        -TipoPanel tipo
        -PeriodoAcademico periodo
        -DateTime actualizadoEn
        +tasaOcupacionAulas() Metrica
        +indiceRiesgoDesercion() Metrica
        +tasaMatriculaEfectiva() Metrica
        +rendimientoPorPrograma() List~Metrica~
        +recaudoAcumulado() Metrica
    }

    class Metrica {
        <<value object>>
        -String nombre
        -decimal valor
        -String unidad
        -decimal variacionPeriodoAnterior
        -DateTime calculadaEn
        +tendencia() Tendencia
        +superaUmbral() bool
    }

    AdaptadorPagos "1" --> "0..*" TransaccionPago : gestiona
    AdaptadorPagos "1" --> "1" CircuitBreaker : protegido_por
    AdaptadorBiblioteca "1" --> "1" CircuitBreaker : protegido_por
    AdaptadorDocumental "1" --> "1" CircuitBreaker : protegido_por
    CircuitBreaker "1" --> "1" PoliticaReintento : coordina_con
    AdaptadorDocumental "1" --> "0..*" Documento : produce
    TransaccionPago "1" --> "0..1" BeneficioFinanciero : aplica
    PanelIndicadores "1" *-- "1..*" Metrica : agrega
```

### Decisiones de diseño — RF-05

| Decisión | Justificación |
|---|---|
| `CircuitBreaker` como clase de primer nivel compartida por los tres adaptadores | Elevarlo a clase del modelo lógico —en lugar de dejarlo como detalle invisible de infraestructura— hace explícito que los tres adaptadores externos están protegidos por la misma política, y da un lugar concreto donde parametrizar umbrales por proveedor. |
| `claveIdempotencia` en `TransaccionPago` | Sin ella, la política de reintentos con retroceso exponencial de RNF-04 produciría **doble cobro** al estudiante. Es un caso donde un requisito de resiliencia genera, implementado ingenuamente, un defecto financiero grave. |
| Estado `SEMIABIERTO` en `CircuitBreaker` | Permite probar la recuperación del servicio externo con una sola petición en lugar de reabrir el tráfico completo de golpe, evitando reabrir el circuito y saturar un servicio que apenas se está restableciendo. |
| `PoliticaReintento` separada de `CircuitBreaker` | Son mecanismos distintos con responsabilidades distintas: el primero decide *cuándo volver a intentar*, el segundo decide *si vale la pena intentar*. Acoplarlos impediría configurarlos independientemente. |
| `Documento` con `codigoVerificacion` y `hashContenido` | Permite verificación pública de autenticidad de certificados sin exponer la base de datos, requisito habitual en trámites académicos. |
| `Metrica` como *value object* con variación | Un indicador sin comparación temporal no soporta la toma de decisiones directiva que exige RF-05. |

---

## 7. Diagrama de secuencia — RF-01 · Autenticación y autorización

**Flujo:** UC-01 Autenticarse + UC-02 Autorizar acceso por rol, incluyendo el intento de acceso no autorizado.

```mermaid
sequenceDiagram
    autonumber
    actor USR as Usuario
    participant SPA as Portal Web
    participant GW as API Gateway
    participant IDP as Proveedor OIDC
    participant MID as Módulo Identidad
    participant DB as BD Identidad
    participant OBS as Observabilidad

    USR->>SPA: Ingresa credenciales
    SPA->>GW: POST /auth/login
    activate GW
    GW->>IDP: Redirección OAuth2 Authorization Code
    activate IDP
    IDP->>IDP: Valida credenciales
    IDP-->>GW: Código de autorización
    GW->>IDP: Intercambia código por tokens
    IDP-->>GW: access_token + id_token + refresh_token
    deactivate IDP

    GW->>MID: GET /usuarios/por-subject
    activate MID
    MID->>DB: SELECT usuario, roles, permisos
    DB-->>MID: Perfil completo
    MID->>MID: Construye scopes desde RBAC
    MID->>DB: INSERT SesionActiva
    MID->>DB: INSERT RegistroAuditoria LOGIN_EXITOSO
    MID-->>GW: Perfil + scopes
    deactivate MID

    GW->>OBS: Métrica de autenticación exitosa
    GW-->>SPA: 200 + token con scopes
    deactivate GW
    SPA-->>USR: Sesión iniciada

    Note over USR,OBS: Acceso posterior a un recurso protegido

    USR->>SPA: Solicita recurso financiero
    SPA->>GW: GET /finanzas/reportes + Bearer token
    activate GW
    GW->>GW: Valida firma y expiración del token

    alt Token válido y scope suficiente
        GW->>MID: Enruta la solicitud
        MID-->>GW: 200 Datos del recurso
        GW-->>SPA: 200 OK
    else Scope insuficiente
        GW->>MID: POST /auditoria ACCESO_DENEGADO
        activate MID
        MID->>DB: INSERT RegistroAuditoria
        MID-->>GW: Registrado
        deactivate MID
        GW->>OBS: Alerta de intento no autorizado
        GW-->>SPA: 403 Forbidden
        SPA-->>USR: Acceso denegado
        Note over GW,OBS: Bloqueo en menos de 2 s.<br/>La solicitud nunca alcanza<br/>el módulo financiero.
    else Token expirado
        GW-->>SPA: 401 Unauthorized
        SPA->>GW: POST /auth/refresh
        GW->>IDP: Renueva con refresh_token
        IDP-->>GW: Nuevo access_token
        GW-->>SPA: 200 Token renovado
    end
    deactivate GW
```

**Justificación.** El corte de autorización ocurre **en el Gateway, antes del primer participante de negocio**. La barra de activación del módulo financiero ni siquiera se abre en la rama de rechazo. Esta posición del `alt` en el diagrama comunica visualmente lo que exige el requisito de seguridad: bloqueo rápido sin exponer datos. Adicionalmente, el registro de auditoría se emite **antes** de responder al cliente, garantizando que no exista una ventana donde el rechazo ocurra sin dejar rastro.

---

## 8. Diagrama de secuencia — RF-02 · Inscripción con fallo de la pasarela

**Flujo:** UC-07 Inscribir asignaturas cuando el servicio de pagos externo no responde. Es el flujo más complejo del sistema: atraviesa cuatro módulos y ejercita Circuit Breaker, cola de mensajes, idempotencia y reconciliación.

```mermaid
sequenceDiagram
    autonumber
    actor EST as Estudiante
    participant GW as API Gateway
    participant MAT as Módulo Matrícula
    participant ACA as Módulo Académico
    participant INT as Módulo Integración
    participant CB as CircuitBreaker
    participant PAY as Pasarela de Pagos
    participant MQ as Cola de Mensajes
    participant DB as BD Matrícula
    participant OBS as Observabilidad

    EST->>GW: POST /matriculas + Idempotency-Key
    GW->>GW: Rate limiting y validación de token
    GW->>MAT: Enruta solicitud
    activate MAT

    MAT->>DB: SELECT por clave de idempotencia
    DB-->>MAT: No existe, continuar
    MAT->>DB: INSERT Matricula estado EN_VALIDACION

    Note over MAT,ACA: Validaciones académicas concurrentes

    par Verificación de cupo
        MAT->>ACA: GET /grupos/{id}/disponibilidad
        activate ACA
        ACA->>ACA: Grupo.hayCupo con versión optimista
        ACA-->>MAT: 200 Cupo disponible, versión 42
        deactivate ACA
    and Validación de prerrequisitos
        MAT->>ACA: POST /prerrequisitos/validar
        activate ACA
        ACA-->>MAT: 200 Prerrequisitos cumplidos
        deactivate ACA
    and Validación de tope de créditos
        MAT->>MAT: ValidadorPrerrequisitos.verificarTopeCreditos
    end

    alt Alguna validación falla
        MAT->>DB: UPDATE estado RECHAZADA
        MAT-->>EST: 409 Conflict con el detalle de la regla
    end

    Note over MAT,PAY: Verificación financiera obligatoria

    MAT->>INT: GET /estado-financiero/{estudianteId}
    activate INT
    INT->>CB: ejecutar consulta
    activate CB
    CB->>CB: permitePaso

    alt Circuito CERRADO y pasarela responde
        CB->>PAY: GET /paz-y-salvo
        PAY-->>CB: 200 pazYSalvo true
        CB->>CB: registrarExito
        CB-->>INT: EstadoFinanciero confiable
        INT-->>MAT: 200 degradado false
        MAT->>ACA: POST /grupos/{id}/reservar-cupo
        ACA-->>MAT: 200 Cupo confirmado
        MAT->>DB: UPDATE estado CONFIRMADA
        MAT->>MQ: Publica MatriculaConfirmada
        MAT-->>EST: 201 Created

    else Circuito ABIERTO o tiempo de espera agotado
        CB->>PAY: GET /paz-y-salvo
        PAY--xCB: Tiempo de espera agotado
        CB->>CB: registrarFallo, supera umbral
        CB->>CB: abrir circuito, ventana 60 s
        CB->>OBS: Alerta CIRCUITO_ABIERTO
        CB-->>INT: Fallo del servicio externo
        deactivate CB
        INT-->>MAT: 200 EstadoFinanciero degradado true
        deactivate INT

        MAT->>ACA: POST /grupos/{id}/reservar-provisional
        activate ACA
        ACA->>ACA: cupoProvisional incrementado, TTL 72 h
        ACA-->>MAT: 200 Cupo retenido
        deactivate ACA
        MAT->>DB: UPDATE estado PENDIENTE_PAGO
        MAT->>MQ: Publica ReconciliarPago
        MAT-->>EST: 202 Accepted, pago en verificación
        deactivate MAT

        Note over MQ,PAY: Reconciliación asíncrona,<br/>fuera del camino del usuario

        loop Retroceso exponencial 1s, 2s, 4s hasta 30 min
            MQ->>INT: Consume ReconciliarPago
            activate INT
            INT->>CB: ejecutar con Idempotency-Key
            activate CB
            alt Pasarela restablecida
                CB->>PAY: GET /paz-y-salvo
                PAY-->>CB: 200 pazYSalvo true
                CB->>CB: cerrar circuito
                CB-->>INT: EstadoFinanciero confiable
                INT->>MAT: PATCH /matriculas/{id}/confirmar
                activate MAT
                MAT->>ACA: POST /grupos/{id}/confirmar-provisional
                MAT->>DB: UPDATE estado CONFIRMADA
                MAT->>MQ: Publica NotificarEstudiante
                deactivate MAT
                INT->>OBS: Circuito cerrado, RTO cumplido
            else Servicio aún caído
                CB-->>INT: Fallo persistente
                INT-->>MQ: Reencola con espera incrementada
            end
            deactivate CB
            deactivate INT
        end

        alt Ventana de 72 h agotada
            MQ->>MAT: Publica ExpirarMatricula
            activate MAT
            MAT->>ACA: POST /grupos/{id}/liberar-provisional
            MAT->>DB: UPDATE estado EXPIRADA
            MAT->>MQ: Publica NotificarExpiracion
            deactivate MAT
        end
    end
```

**Justificación de las decisiones críticas:**

1. **La matrícula se persiste antes de consultar a la pasarela.** Si se consultara primero y el servicio estuviera caído, la solicitud del estudiante se perdería por completo. Al persistir un estado intermedio, el sistema siempre tiene un registro sobre el cual reconciliar.

2. **Respuesta `202 Accepted`, no `500`.** El fallo de un tercero no se propaga como fallo al usuario. Devolver `500` acoplaría la disponibilidad percibida de UPS-Connect a la de la pasarela de pagos, anulando el objetivo de 99,9 %.

3. **Cupo provisional con vencimiento de 72 horas.** Si el cupo se confirmara sin verificar el pago, un estudiante moroso bloquearía un lugar indefinidamente. Si no se retuviera nada, el estudiante perdería el cupo por un fallo ajeno. El cupo provisional con expiración protege a ambas partes, y la rama de expiración cierra el ciclo de vida evitando fuga de cupos.

4. **La clave de idempotencia atraviesa las tres fases.** Se recibe en la petición inicial, se verifica contra la base de datos antes de crear nada, y se reenvía en cada reintento de reconciliación. Un estudiante que pulse el botón cinco veces durante el pico genera una sola matrícula y un solo cobro.

---

## 9. Diagrama de secuencia — RF-03 · Reserva de aula con detección de conflictos

**Flujo:** UC-18 Reservar aula + UC-19 Detectar conflictos de programación.

```mermaid
sequenceDiagram
    autonumber
    actor COO as Coordinador
    participant GW as API Gateway
    participant ACA as Módulo Académico
    participant MR as MotorReservas
    participant DB as BD Académico
    participant MQ as Cola de Mensajes

    COO->>GW: POST /reservas grupo, aula, franjas
    GW->>ACA: Enruta solicitud
    activate ACA

    ACA->>DB: SELECT Aula, Grupo, CargaAcademica
    DB-->>ACA: Entidades cargadas
    ACA->>ACA: Aula.admite cantidad de estudiantes

    alt Capacidad insuficiente
        ACA-->>COO: 409 El aula no admite el tamaño del grupo
    end

    ACA->>MR: detectarConflictos reserva propuesta
    activate MR

    par Detección concurrente de conflictos
        MR->>DB: SELECT reservas del aula en la franja
        DB-->>MR: Reservas existentes
        MR->>MR: detectarConflictoAula
    and
        MR->>DB: SELECT horarios del docente en la franja
        DB-->>MR: Franjas del docente
        MR->>MR: detectarConflictoDocente
    and
        MR->>DB: SELECT horario actual del grupo
        DB-->>MR: Franjas del grupo
        MR->>MR: detectarConflictoGrupo
    end

    MR->>MR: Consolida lista de conflictos
    MR-->>ACA: Lista de conflictos con severidad
    deactivate MR

    alt Existen conflictos bloqueantes
        ACA->>MR: proponerAlternativas
        activate MR
        MR->>DB: SELECT franjas libres compatibles
        DB-->>MR: Franjas candidatas
        MR-->>ACA: Alternativas ordenadas por conveniencia
        deactivate MR
        ACA-->>COO: 409 Conflicto detectado con alternativas
        Note over ACA,COO: Ninguna asignación conflictiva<br/>llega a persistirse

    else Solo advertencias no bloqueantes
        ACA->>DB: BEGIN TRANSACTION
        ACA->>DB: INSERT ReservaAula estado CONFIRMADA
        ACA->>DB: UPDATE CargaAcademica horas asignadas
        ACA->>DB: UPDATE Horario del grupo
        ACA->>DB: COMMIT
        ACA->>MQ: Publica ReservaConfirmada
        ACA-->>COO: 201 Created con advertencias
        MQ->>MQ: Notifica al docente afectado

    else Sin conflictos
        ACA->>DB: BEGIN TRANSACTION
        ACA->>DB: INSERT ReservaAula estado CONFIRMADA
        ACA->>DB: UPDATE CargaAcademica
        ACA->>DB: UPDATE Horario
        ACA->>DB: COMMIT
        ACA->>MQ: Publica ReservaConfirmada
        ACA-->>COO: 201 Created
    end
    deactivate ACA
```

**Justificación.** La detección de conflictos ocurre **dentro de la misma transacción** que la escritura, y los tres detectores se ejecutan concurrentemente porque son independientes entre sí. La rama de conflicto bloqueante retorna sin persistir nada: el motor de reservas actúa como precondición de escritura, no como validación posterior. Esto es lo que hace cumplible el requisito de RF-03 de evitar conflictos de programación.

La propuesta de alternativas convierte un rechazo en una acción constructiva, reduciendo los ciclos de ensayo y error del coordinador durante la planificación semestral.

---

## 10. Diagrama de secuencia — RF-04 · Registro, publicación y notificación de notas

**Flujo:** UC-23 Registrar + UC-24 Publicar + UC-28 Notificar, incluyendo modificación posterior a la publicación.

```mermaid
sequenceDiagram
    autonumber
    actor DOC as Docente
    participant GW as API Gateway
    participant EVA as Módulo Evaluación
    participant DB as BD Evaluación
    participant MQ as Cola de Mensajes
    participant NOT as Servicio de Notificaciones
    actor EST as Estudiante

    DOC->>GW: POST /actividades/{id}/calificaciones
    GW->>EVA: Enruta solicitud
    activate EVA

    EVA->>DB: SELECT ActaNotas del grupo
    DB-->>EVA: Acta en estado ABIERTA

    alt Acta cerrada
        EVA-->>DOC: 409 El acta está cerrada
    end

    EVA->>EVA: ActividadEvaluativa.permiteRegistro
    EVA->>EVA: Calificacion.estaEnRango

    alt Nota fuera de rango permitido
        EVA-->>DOC: 400 Nota inválida
    end

    EVA->>DB: BEGIN TRANSACTION
    EVA->>DB: INSERT Calificacion estado REGISTRADA
    EVA->>DB: INSERT HistorialCambioNota
    EVA->>DB: COMMIT
    EVA-->>DOC: 201 Calificaciones registradas
    deactivate EVA

    Note over DOC,EST: Publicación, acción explícita del docente

    DOC->>GW: POST /actividades/{id}/publicar
    GW->>EVA: Enruta solicitud
    activate EVA
    EVA->>DB: SELECT calificaciones de la actividad
    DB-->>EVA: Conjunto completo

    alt Registro incompleto
        EVA-->>DOC: 409 Faltan estudiantes por calificar
    end

    EVA->>DB: UPDATE estado PUBLICADA
    EVA->>EVA: Genera EventoNota por estudiante
    EVA->>MQ: Publica lote de EventoNota
    Note over EVA,MQ: La transacción del docente<br/>termina aquí. El envío es<br/>responsabilidad de otro proceso.
    EVA-->>DOC: 200 Calificaciones publicadas
    deactivate EVA

    MQ->>NOT: Consume EventoNota
    activate NOT
    NOT->>NOT: Resuelve canal según preferencias
    NOT-->>EST: Notificación de nota publicada
    deactivate NOT

    EST->>GW: GET /calificaciones
    GW->>EVA: Enruta solicitud
    EVA-->>EST: 200 Calificaciones publicadas

    Note over DOC,EST: Modificación posterior a la publicación

    DOC->>GW: PATCH /calificaciones/{id} con motivo
    GW->>EVA: Enruta solicitud
    activate EVA
    EVA->>DB: SELECT Calificacion y Acta
    alt Acta cerrada sin autorización
        EVA-->>DOC: 403 Requiere reapertura autorizada
    end
    EVA->>DB: BEGIN TRANSACTION
    EVA->>DB: UPDATE Calificacion
    EVA->>DB: INSERT HistorialCambioNota con motivo
    EVA->>DB: COMMIT
    EVA->>MQ: Publica EventoNota tipo MODIFICACION
    EVA-->>DOC: 200 Nota modificada
    deactivate EVA

    MQ->>NOT: Consume evento de modificación
    NOT-->>EST: Notificación de cambio de nota
```

**Justificación.** El registro y la publicación son operaciones **separadas y explícitas**. Un docente puede registrar notas progresivamente sin que los estudiantes vean resultados parciales; la publicación es un acto deliberado que dispara las notificaciones. Fusionar ambas obligaría al docente a completar todo el grupo en una sola sesión.

La transacción del docente termina al publicar el evento en la cola. Si el servicio de notificaciones estuviera caído, las notas quedan igualmente publicadas y consultables: **el desacople protege la operación académica de los fallos del canal de comunicación**, que es exactamente lo que RF-04 persigue.

---

## 11. Diagrama de secuencia — RF-05 · Emisión de certificado académico

**Flujo:** UC-33 Emitir certificado + UC-34 Archivar documento, con verificación de paz y salvo.

```mermaid
sequenceDiagram
    autonumber
    actor EST as Estudiante
    participant GW as API Gateway
    participant INT as Módulo Integración
    participant CB as CircuitBreaker
    participant PAY as Pasarela de Pagos
    participant MAT as Módulo Matrícula
    participant GDO as Gestión Documental
    participant DB as BD Integración
    participant MQ as Cola de Mensajes

    EST->>GW: POST /certificados tipo NOTAS
    GW->>INT: Enruta solicitud
    activate INT

    Note over INT,PAY: Verificación de paz y salvo obligatoria

    INT->>CB: ejecutar consulta de estado financiero
    activate CB
    alt Circuito cerrado
        CB->>PAY: GET /paz-y-salvo
        PAY-->>CB: 200 pazYSalvo false, saldo pendiente
        CB-->>INT: EstadoFinanciero confiable
    else Circuito abierto
        CB-->>INT: Servicio no disponible
        INT-->>EST: 503 Verificación no disponible, reintente
        Note over INT,EST: No se emite el documento<br/>sin verificación confiable
    end
    deactivate CB

    alt Estudiante con saldo pendiente
        INT-->>EST: 403 Debe estar a paz y salvo
    end

    INT->>MAT: GET /expedientes/{estudianteId}
    activate MAT
    MAT-->>INT: 200 Historial académico consolidado
    deactivate MAT

    INT->>INT: Compone contenido del certificado
    INT->>INT: Genera código de verificación único

    INT->>GDO: POST /documentos generar y firmar
    activate GDO
    GDO->>GDO: Aplica plantilla y firma digital
    GDO-->>INT: 201 Documento firmado con ruta
    deactivate GDO

    INT->>DB: BEGIN TRANSACTION
    INT->>DB: INSERT Documento con hash y código
    INT->>DB: COMMIT

    INT->>MQ: Publica DocumentoEmitido
    INT-->>EST: 201 Created con enlace de descarga
    deactivate INT

    MQ->>MQ: Notifica emisión al estudiante

    Note over EST,GDO: Verificación pública posterior

    EST->>GW: GET /certificados/verificar/{codigo}
    GW->>INT: Enruta solicitud
    activate INT
    INT->>DB: SELECT Documento por código
    DB-->>INT: Documento con hash
    INT->>INT: Verifica vigencia e integridad
    INT-->>EST: 200 Documento auténtico y vigente
    deactivate INT
```

**Justificación.** La verificación de paz y salvo es **bloqueante** para la emisión de certificados: a diferencia de la matrícula, aquí no existe modo degradado. Si el estado financiero no puede verificarse con confianza, el sistema responde `503` en lugar de emitir un documento oficial sobre información no confirmada. Esta asimetría deliberada respecto al flujo de matrícula refleja que **el costo del error es distinto**: una matrícula provisional se reconcilia; un certificado emitido indebidamente ya circuló.

El código de verificación permite a terceros validar la autenticidad del documento sin acceso al sistema, requisito habitual en trámites académicos e institucionales.

---

## 12. Mapa de referencias inter-módulo

Resumen de todas las dependencias entre módulos y su mecanismo de comunicación.

```mermaid
flowchart LR
    subgraph SINC["Comunicación síncrona · REST"]
        direction TB
        S1["Matrícula → Académico<br/><i>IDisponibilidadCupo</i><br/>Consulta y reserva de cupo"]
        S2["Matrícula → Integración<br/><i>IEstadoFinanciero</i><br/>Verificación de paz y salvo"]
        S3["Integración → Matrícula<br/><i>IExpediente</i><br/>Datos para certificados y paneles"]
        S4["Evaluación → Académico<br/><i>IOfertaAcademica</i><br/>Validación de grupo y docente"]
        S5["Integración → Académico<br/><i>IReservaAula</i><br/>Ocupación para paneles"]
        S6["Gateway → Identidad<br/><i>IIdentidad</i><br/>Perfil y scopes RBAC"]
    end

    subgraph ASINC["Comunicación asíncrona · Cola de mensajes"]
        direction TB
        A1["Evaluación → Notificaciones<br/><i>EventoNota</i>"]
        A2["Matrícula → Integración<br/><i>ReconciliarPago</i>"]
        A3["Matrícula → Notificaciones<br/><i>MatriculaConfirmada</i><br/><i>NotificarExpiracion</i>"]
        A4["Académico → Notificaciones<br/><i>ReservaConfirmada</i>"]
        A5["Integración → Notificaciones<br/><i>DocumentoEmitido</i>"]
    end

    classDef s fill:#E8F0FE,stroke:#1967D2,color:#000
    classDef a fill:#FEF7E0,stroke:#EA8600,color:#000
    class S1,S2,S3,S4,S5,S6 s
    class A1,A2,A3,A4,A5 a
```

### Criterio de elección entre síncrono y asíncrono

| Situación | Mecanismo | Razón |
|---|---|---|
| El emisor **necesita la respuesta** para continuar | REST síncrono | Verificar cupo o paz y salvo determina si la operación puede proceder |
| El emisor **no depende** del resultado | Cola de mensajes | Notificar al estudiante no condiciona el registro de la nota |
| La operación **debe sobrevivir** a la caída del receptor | Cola de mensajes | La reconciliación de pago debe ejecutarse aunque el módulo estuviera caído al fallar la pasarela |
| Se requiere **consistencia inmediata** | REST síncrono | La reserva de cupo no admite ventanas de inconsistencia bajo alta concurrencia |

**Regla verificable:** el diagrama contiene **cero** dependencias circulares síncronas. Matrícula llama a Integración y Integración llama a Matrícula, pero sobre interfaces distintas y en flujos distintos, nunca dentro de una misma cadena de llamadas — lo que evita interbloqueos y cascadas de tiempo de espera.

---

| ← Anterior | Índice | Siguiente → |
|---|---|---|
| [Casos de Uso](01-vista-casos-uso.md) | [README](../README.md) | [Vista de Procesos](03-vista-procesos.md) |
