# TALLER DE AFIANZAMIENTO: ARQUITECTURA EMPRESARIAL Y TRANSICIÓN A FRAMEWORKS MODERNOS


**Objetivo:** Consolidar conceptos de arquitectura empresarial Java EE y preparar la transición hacia PrimeFaces y Spring Boot

---

## PARTE I: REPASO Y CONSOLIDACIÓN DE CONCEPTOS (30 minutos)

### Actividad 1: Análisis Comparativo de Arquitecturas (15 minutos)

**Instrucciones:** Analice los siguientes tres escenarios y complete la tabla comparativa.

#### Escenario A: Sistema de Pedidos Online
- Múltiples usuarios hacen pedidos simultáneamente
- Cada pedido tiene múltiples ítems
- Se debe validar inventario y calcular total con impuestos
- Se registra auditoría de cada operación

#### Escenario B: Portal de Noticias
- Miles de usuarios leen noticias simultáneamente
- Pocos administradores publican contenido
- Las noticias rara vez cambian una vez publicadas
- Se necesita contar visitas por artículo

#### Escenario C: Sistema de Matrícula Universitaria
- Estudiantes se matriculan en un proceso de 5 pasos
- Pueden retroceder y cambiar selección de cursos
- Deben validar prerrequisitos y cupos disponibles
- Solo activo durante período de matrícula

**Complete la tabla:**

| Aspecto | Escenario A | Escenario B | Escenario C |
|---------|-------------|-------------|-------------|
| Scope del Controller principal |@RequestScoped - Cada pedido es una operación independiente que se completa en una sola petición | @ApplicationScoped - Las noticias se comparten entre todos los usuarios y no cambian frecuentemente|@SessionScoped o @ConversationScoped - Se necesita mantener el estado del proceso de matrícula mientras el estudiante navega entre los 5 pasos |
| ¿Necesita transacciones? ¿Por qué? |Sí, definitivamente. Hay que asegurar que la validación de inventario, creación del pedido, registro de ítems y auditoría se hagan como una sola unidad. Si algo falla, todo debe revertirse | Parcialmente. Solo al publicar noticias nuevas o actualizar contadores de visitas. La lectura no requiere transacciones|Sí, al finalizar. Aunque el usuario navegue entre pasos, la transacción se necesita al confirmar la matrícula para validar cupos y guardar todo junto, evitando inconsistencias |
| ¿Usar Facade? Justifique | Sí. El proceso involucra múltiples entidades (Pedido, Items, Inventario, Auditoría) y lógica de negocio. Un Facade coordinaría todo esto y simplificaría el controller| No necesariamente. La operación de lectura es simple y directa. Tal vez solo para la parte administrativa de publicación, pero no es necesario|Sí. El proceso de matrícula es complejo, involucra validación de prerrequisitos, verificación de cupos, y múltiples pasos. Un Facade ayudaría a encapsular toda esa lógica |
| Principal desafío de concurrencia | Control de inventario. Múltiples usuarios pueden intentar comprar el mismo producto simultáneamente. Se necesita locking optimista o pesimista para evitar sobreventa| Contador de visitas. Miles de usuarios leyendo al mismo tiempo pueden generar conflictos al incrementar el contador. Necesita manejo adecuado de concurrencia en las actualizaciones|Cupos disponibles. Varios estudiantes pueden intentar inscribirse en el mismo curso al mismo tiempo cuando quedan pocos cupos. Similar al inventario pero más crítico por ventanas de tiempo limitadas |
| Patrón de persistencia más apropiado |DAO con Repository. Se necesita acceso complejo a múltiples entidades con "queries" específicas para validaciones de inventario y reportes | DAO simple con caché. Las lecturas son frecuentes y los datos no cambian mucho, así que usar caché de segundo nivel de JPA mejoraría el rendimiento considerablemente| DAO con Repository. Necesita queries complejas para validar prerrequisitos, buscar cursos disponibles, y verificar conflictos de horarios entre los cursos seleccionados
|

### Actividad 2: Depuración de Código Problemático (15 minutos)

**Identifique y explique los problemas en el siguiente código:**

```java
// Código 1: Controller con problemas
@RequestScoped
public class CarritoComprasController {
    
    @Inject
    private ProductoService productoService;
    
    private List<Producto> productosEnCarrito; // Problema?
    private double totalCompra;
    
    public void agregarProducto(String codigo) {
        Producto p = productoService.buscar(codigo);
        productosEnCarrito.add(p);
        totalCompra += p.getPrecio();
    }
    
    public String finalizarCompra() {
        // Procesar compra
        return "confirmacion";
    }
}

// Código 2: Service sin manejo transaccional
public class TransferenciaService {
    
    @Inject
    private CuentaDAO cuentaDAO;
    
    public void transferir(String origen, String destino, double monto) 
            throws SQLException {
        Cuenta cuentaOrigen = cuentaDAO.buscar(origen);
        cuentaOrigen.setSaldo(cuentaOrigen.getSaldo() - monto);
        cuentaDAO.actualizar(cuentaOrigen);
        
        // ¿Qué pasa si falla aquí?
        Cuenta cuentaDestino = cuentaDAO.buscar(destino);
        cuentaDestino.setSaldo(cuentaDestino.getSaldo() + monto);
        cuentaDAO.actualizar(cuentaDestino);
    }
}

// Código 3: JSP con lógica de negocio
<%
    String cedula = request.getParameter("cedula");
    Connection conn = DriverManager.getConnection("jdbc:mysql://...");
    PreparedStatement stmt = conn.prepareStatement("SELECT * FROM cliente WHERE cedula=?");
    stmt.setString(1, cedula);
    ResultSet rs = stmt.executeQuery();
    
    if(rs.next()) {
        double saldo = rs.getDouble("saldo");
        if(saldo > 1000000) {
            out.println("Cliente VIP");
        }
    }
%>
```

**Preguntas:**
1. ¿Qué scope debería tener el CarritoComprasController? ¿Por qué?

    El CarritoComprasController debería tener @SessionScoped porque el carrito de compras debe persistir durante toda la sesión del usuario mientras navega por el sitio, agrega productos, revisa el carrito, etc. Solo se debe limpiar cuando finalice la compra o cierre sesión.

2. ¿Qué le falta al TransferenciaService para garantizar atomicidad?

    Le falta la anotación @Transactional para garantizar atomicidad.Con @Transactional, si algo falla entre las dos actualizaciones, todo se revierte automáticamente (rollback). Es como un "todo o nada".

3. ¿Qué violaciones arquitectónicas tiene el JSP? Proponga una solución.

- Lógica de acceso a datos en la vista: El JSP está creando conexiones JDBC y ejecutando queries directamente. Esto debería estar en un DAO.

- Lógica de negocio en la vista: La regla de "saldo mayor a 1 millón = VIP" es lógica de negocio y debería estar en un Service o en la entidad Cliente.

- No cierra recursos: lo que genera fugas de memoria y conexiones abiertas.

- Usa scriptlets: El código Java en el JSP es mala práctica. Debería usar JSTL.

### Solución:

Mover la conexión y consulta SQL a un ClienteDAO que use JPA.

Crear un ClienteService que llame al DAO y maneje la lógica de negocio.

Hacer un Controller que inyecte el service, reciba la cédula y guarde el cliente consultado.

El JSP solo debe mostrar los datos usando Expression Language, sin código Java ni lógica.

---

## PARTE II: EJERCICIO PRÁCTICO - REFACTORIZACIÓN (40 minutos)

### Ejercicio: Sistema de Reserva de Citas Médicas

**Contexto:** Tiene un sistema legacy con código mezclado. Debe refactorizarlo aplicando la arquitectura en capas correcta.

#### Código Legacy Proporcionado:

```java
// Todo en un solo servlet - MAL DISEÑO
@WebServlet("/reservarCita")
public class ReservaCitaServlet extends HttpServlet {
    
    protected void doPost(HttpServletRequest request, HttpServletResponse response) {
        try {
            // Obtener parámetros
            String cedulaPaciente = request.getParameter("cedula");
            String idMedico = request.getParameter("medico");
            String fecha = request.getParameter("fecha");
            String hora = request.getParameter("hora");
            
            // Conectar a BD directamente en el servlet
            Connection conn = DriverManager.getConnection(
                "jdbc:mysql://localhost/clinica", "root", "password");
            
            // Validar disponibilidad
            PreparedStatement ps1 = conn.prepareStatement(
                "SELECT COUNT(*) FROM citas WHERE id_medico=? AND fecha=? AND hora=?");
            ps1.setString(1, idMedico);
            ps1.setString(2, fecha);
            ps1.setString(3, hora);
            ResultSet rs = ps1.executeQuery();
            rs.next();
            
            if(rs.getInt(1) > 0) {
                response.getWriter().println("Horario no disponible");
                return;
            }
            
            // Insertar cita
            PreparedStatement ps2 = conn.prepareStatement(
                "INSERT INTO citas (cedula_paciente, id_medico, fecha, hora, estado) VALUES (?,?,?,?,?)");
            ps2.setString(1, cedulaPaciente);
            ps2.setString(2, idMedico);
            ps2.setString(3, fecha);
            ps2.setString(4, hora);
            ps2.setString(5, "RESERVADA");
            ps2.executeUpdate();
            
            // Calcular precio según especialidad
            PreparedStatement ps3 = conn.prepareStatement(
                "SELECT especialidad FROM medicos WHERE id=?");
            ps3.setString(1, idMedico);
            ResultSet rs2 = ps3.executeQuery();
            rs2.next();
            String especialidad = rs2.getString(1);
            
            double precio = 0;
            if(especialidad.equals("GENERAL")) precio = 50000;
            else if(especialidad.equals("ESPECIALISTA")) precio = 80000;
            
            response.getWriter().println("Cita reservada. Total: $" + precio);
            
        } catch(Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Tareas de Refactorización:

## Problema Original

El sistema original presentaba un diseño monolítico concentrado en un solo servlet de más de 80 líneas, lo que generaba una alta dependencia entre componentes y dificultaba el mantenimiento. Este servlet mezclaba diferentes responsabilidades como la lógica de presentación (manejo de `request` y `response`), la lógica de negocio (validaciones y cálculos), y el acceso directo a datos mediante consultas SQL. Además, no hacía uso de un pool de conexiones ni aplicaba un manejo transaccional adecuado, lo que incrementaba el riesgo de errores y afectaba la escalabilidad.

## Solución: Arquitectura en Capas

Para resolver estas deficiencias se implementó una refactorización basada en una arquitectura en capas, separando las responsabilidades en cuatro niveles principales: presentación, servicio, persistencia y modelo. Esta estructura mejora la mantenibilidad, facilita la reutilización del código, permite un control transaccional más seguro y reduce el acoplamiento entre componentes, logrando un sistema más modular y escalable.


**1. Capa de Modelo (JavaBeans)** - Cree las clases necesarias:
   - `Cita` con todos sus atributos
   - `Medico` con especialidad
   - Incluya métodos de negocio donde corresponda

### Diagrama de Clases
```
┌──────────────────────────────┐
│           <<Cita>>           │
├──────────────────────────────┤
│ - id: Long                   │
│ - cedulaPaciente: String     │
│ - medico: Medico             │
│ - fecha: LocalDate           │
│ - hora: LocalTime            │
│ - estado: EstadoCita         │
├──────────────────────────────┤
│ + getters/setters()          │
│ + calcularPrecio(): double   │
└──────────────────────────────┘
           │
           │ 1
           ▼
┌─────────────────────────────────┐
│           <<Medico>>            │
├─────────────────────────────────┤
│ - id: Long                      │
│ - nombre: String                │
│ - especialidad: Especialidad    │
│ - precioConsulta: double        │
├─────────────────────────────────┤
│ + getters/setters()             │
│ + getPrecioConsulta(): double   │
└─────────────────────────────────┘
```


### Enumeraciones
```
┌──────────────────────────────┐       ┌────────────────────────────────┐
│       <<enumeration>>        │       │        <<enumeration>>         │
│          EstadoCita          │       │         Especialidad           │
├──────────────────────────────┤       ├────────────────────────────────┤
│ • RESERVADA                  │       │ • GENERAL (50000)              │
│ • CONFIRMADA                 │       │ • ESPECIALISTA (80000)         │
│ • CANCELADA                  │       │ • CIRUJANO (120000)            │
│ • COMPLETADA                 │       │ • PEDIATRA (60000)             │
└──────────────────────────────┘       └────────────────────────────────┘

```


### Características Clave

- **Cita:** Contiene los métodos de negocio para calcular el precio de la consulta según la especialidad del médico asociado.  
- **Medico:** Encapsula los atributos del médico y su especialidad, permitiendo obtener el precio correspondiente a su tipo de consulta.  
- **Enumeraciones:** Definen de forma controlada los estados válidos de una cita y las especialidades médicas con sus precios asociados.  
- **Serializable:** Todas las clases implementan la interfaz `Serializable` para facilitar el manejo de sesión y la persistencia de objetos.




**2. Capa de Persistencia (DAO)** - Implemente:
   - `CitaDAO` con métodos: `guardar()`, `verificarDisponibilidad()`
   - `MedicoDAO` con método: `obtenerPorId()`
   - Use `@Resource` para el DataSource
   - Manejo apropiado de conexiones

### Diagrama de Componentes

```
┌─────────────────────────────────────────┐
│        <<interface>> CitaDAO            │
├─────────────────────────────────────────┤
│ + guardar(cita: Cita): void             │
│ + verificarDisponibilidad(              │
│     medico: Long,                       │
│     fecha: LocalDate,                   │
│     hora: LocalTime): boolean           │
│ + obtenerPorId(id: Long): Cita          │
│ + listarPorPaciente(String): List<Cita> │
└─────────────────────────────────────────┘
                    △
                    │ implements
                    │
┌─────────────────────────────────────────┐
│       CitaDAOImpl                       │
│       @Stateless                        │
├─────────────────────────────────────────┤
│ - ds: DataSource                        │
│   @Resource(lookup="java:/jdbc/...")    │
├─────────────────────────────────────────┤
│ + guardar(cita: Cita): void             │
│ + verificarDisponibilidad(...): boolean │
│ + obtenerPorId(id: Long): Cita          │
│ + listarPorPaciente(String): List       │
│ - getConnection(): Connection           │
│ - closeResources(...): void             │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│       <<interface>> MedicoDAO           │
├─────────────────────────────────────────┤
│ + obtenerPorId(id: Long): Medico        │
│ + listarPorEspecialidad(String): List   │
│ + listarTodos(): List<Medico>           │
└─────────────────────────────────────────┘
                    △
                    │ implements
                    │
┌─────────────────────────────────────────┐
│       MedicoDAOImpl                     │
│       @Stateless                        │
├─────────────────────────────────────────┤
│ - ds: DataSource                        │
│   @Resource(lookup="java:/jdbc/...")    │
├─────────────────────────────────────────┤
│ + obtenerPorId(id: Long): Medico        │
│ + listarPorEspecialidad(String): List   │
│ + listarTodos(): List<Medico>           │
└─────────────────────────────────────────┘
```

### Características Clave

**@Stateless:** EJBs sin estado para alta concurrencia.  
**@Resource:** Inyección automática del `DataSource` desde el servidor.  
**Connection Pool:** Uso eficiente de conexiones mediante `DataSource`.  
**Try-with-resources:** Cierre automático de recursos (`Connection`, `Statement`, `ResultSet`).  
**PreparedStatement:** Prevención de ataques **SQL Injection**.




**3. Capa de Servicio (Service)** - Implemente:
   - `CitaService` con método `reservarCita()`
   - Use `@Inject` para inyectar DAOs
   - Implemente manejo transaccional con `UserTransaction`
   - Método para calcular precio según especialidad

### Diagrama de Componentes

```
┌─────────────────────────────────────────────────┐
│            CitaService                          │
│            @Stateless                           │
├─────────────────────────────────────────────────┤
│ - citaDAO: CitaDAO           @Inject            │
│ - medicoDAO: MedicoDAO       @Inject            │
│ - userTransaction: UserTransaction @Resource    │
├─────────────────────────────────────────────────┤
│ + reservarCita(                                 │
│     cedulaPaciente: String,                     │
│     idMedico: Long,                             │
│     fecha: LocalDate,                           │
│     hora: LocalTime): Cita                      │
│ + calcularPrecio(medico: Medico): double        │
│ + confirmarCita(idCita: Long): void             │
└─────────────────────────────────────────────────┘
              │                    │
              │ @Inject            │ @Inject
              ▼                    ▼
    ┌──────────────┐      ┌──────────────┐
    │   CitaDAO    │      │  MedicoDAO   │
    └──────────────┘      └──────────────┘
```

### Flujo del Método reservarCita()

```
1. ┌─────────────────────────────────────┐
   │ userTransaction.begin()             │
   │ Iniciar transacción                 │
   └─────────────────────────────────────┘
                  ↓
2. ┌─────────────────────────────────────┐
   │ medicoDAO.obtenerPorId(idMedico)    │
   │ Obtener datos del médico            │
   └─────────────────────────────────────┘
                  ↓
3. ┌─────────────────────────────────────┐
   │ citaDAO.verificarDisponibilidad()   │
   │ Verificar horario disponible        │
   └─────────────────────────────────────┘
                  ↓
4. ┌─────────────────────────────────────┐
   │ ¿Disponible?                        │
   │   NO → rollback() + Exception       │
   │   SÍ → Continuar                    │
   └─────────────────────────────────────┘
                  ↓
5. ┌─────────────────────────────────────┐
   │ Crear objeto Cita                   │
   │ Calcular precio                     │
   └─────────────────────────────────────┘
                  ↓
6. ┌─────────────────────────────────────┐
   │ citaDAO.guardar(cita)               │
   │ Persistir en BD                     │
   └─────────────────────────────────────┘
                  ↓
7. ┌─────────────────────────────────────┐
   │ userTransaction.commit()            │
   │ Confirmar transacción               │
   └─────────────────────────────────────┘
                  ↓
8. ┌─────────────────────────────────────┐
   │ return cita                         │
   │ Retornar objeto persistido          │
   └─────────────────────────────────────┘
```



**4. Capa de Presentación (Managed Bean)** - Implemente:
   - `CitaController` con scope apropiado
   - `@Inject` del CitaService
   - Atributos del formulario con getters/setters
   - Método de acción `procesarReserva()`


### Diagrama de Componentes

```
┌─────────────────────┐
│  reservarCita.xhtml │
│  (Vista JSF)        │
├─────────────────────┤
│ • h:inputText       │
│ • h:commandButton   │
│ • h:message         │
└─────────────────────┘
           │
           │ binding
           ▼
┌─────────────────────────────────────────┐
│       CitaController                    │
│       @Named("citaController")          │
│       @RequestScoped                    │
├─────────────────────────────────────────┤
│ - citaService: CitaService  @Inject     │
│                                         │
│ // Atributos del formulario             │
│ - cedulaPaciente: String                │
│ - idMedico: Long                        │
│ - fecha: LocalDate                      │
│ - hora: LocalTime                       │
│ - mensaje: String                       │
│ - citaReservada: Cita                   │
├─────────────────────────────────────────┤
│ + procesarReserva(): String             │
│ + getters/setters()                     │
│ + init() @PostConstruct                 │
└─────────────────────────────────────────┘
           │
           │ @Inject
           ▼
┌─────────────────────┐
│   CitaService       │
│   @Stateless        │
└─────────────────────┘


### Flujo del Método procesarReserva()

┌──────────────────────────────────────────────┐
│ 1. Validar datos del formulario              │
│    - Campos requeridos                       │
│    - Formato de fecha/hora                   │
│    - Cédula válida                           │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ 2. Llamar citaService.reservarCita()         │
│    con todos los parámetros                  │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ 3. Capturar objeto Cita retornado            │
│    citaReservada = resultado                 │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ 4. Establecer mensaje de éxito               │
│    "Cita reservada. Total: $" + precio       │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ 5. Retornar navegación                       │
│    - null: quedarse en página                │
│    - "confirmacion": ir a confirmación       │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ 6. Manejo de excepciones                     │
│    catch → mensaje de error + return null    │
└──────────────────────────────────────────────┘
```



**5. Patrón Facade (Opcional - Bonus)** - Implemente:
   - `ClinicaFacade` que coordine reservar cita Y enviar email de confirmación
   - Debe inyectar `CitaService` y `NotificacionService`


### Diagrama de Arquitectura

```
┌─────────────────────────────────────────────────┐
│         CitaController                          │
│         @Named / @RequestScoped                 │
└─────────────────────────────────────────────────┘
                      │
                      │ usa
                      ▼
┌─────────────────────────────────────────────────┐
│         <<Facade>>                              │
│         ClinicaFacade                           │
│         @Stateless                              │
├─────────────────────────────────────────────────┤
│ - citaService: CitaService          @Inject     │
│ - notificacionService: NotificationService      │
│                                     @Inject     │
├─────────────────────────────────────────────────┤
│ + reservarCitaConNotificacion(...)              │
│     → Orquesta múltiples servicios              │
└─────────────────────────────────────────────────┘
              │                    │
              │ @Inject            │ @Inject
              ▼                    ▼
    ┌──────────────────┐  ┌──────────────────────┐
    │  CitaService     │  │ NotificacionService  │
    │  @Stateless      │  │ @Stateless           │
    ├──────────────────┤  ├──────────────────────┤
    │ - citaDAO        │  │ - mailSession        │
    │ - medicoDAO      │  │   @Resource          │
    │ - userTx         │  ├──────────────────────┤
    ├──────────────────┤  │ + enviarConfirmacion │
    │ + reservarCita() │  │ + enviarRecordatorio │
    │ + confirmarCita()│  └──────────────────────┘
    └──────────────────┘
```


### Flujo del Método reservarCitaConNotificacion()

```
┌─────────────────────────────────────────────┐
│ 1. Llamar citaService.reservarCita()        │
│    → Crear y persistir la cita              │
└─────────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 2. ¿Reserva exitosa?                        │
│    NO → Lanzar excepción                    │
│    SÍ → Continuar                           │
└─────────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 3. Obtener email del paciente               │
│    (de BD o servicio externo)               │
└─────────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 4. Llamar notificacionService               │
│    .enviarConfirmacionCita(cita, email)     │
└─────────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 5. Retornar cita reservada                  │
└─────────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 6. Manejo de excepciones                    │
│    - Rollback si es necesario               │
│    - Log de errores                         │
└─────────────────────────────────────────────┘
```


### Estructura de Capas Implementada

```
┌───────────────────────────────────────────────────┐
│  PRESENTACIÓN (CitaController)                    │
│  • JSF Managed Bean                               │
│  • Validación de entrada                          │
│  • Navegación                                     │
└───────────────────────────────────────────────────┘
                      ↓ @Inject
┌───────────────────────────────────────────────────┐
│  FACADE (ClinicaFacade) - OPCIONAL                │
│  • Coordina múltiples servicios                   │
│  • Operaciones complejas de negocio               │
└───────────────────────────────────────────────────┘
                      ↓ @Inject
┌───────────────────────────────────────────────────┐
│  SERVICIO (CitaService)                           │
│  • Lógica de negocio                              │
│  • Manejo transaccional (UserTransaction)         │
│  • Validaciones de negocio                        │
└───────────────────────────────────────────────────┘
                      ↓ @Inject
┌───────────────────────────────────────────────────┐
│  PERSISTENCIA (CitaDAO, MedicoDAO)                │
│  • CRUD operations                                │
│  • Consultas SQL                                  │
│  • Gestión de conexiones                          │
└───────────────────────────────────────────────────┘
                      ↓ usa
┌───────────────────────────────────────────────────┐
│  MODELO (Cita, Medico, Enums)                     │
│  • Entidades de dominio                           │
│  • Lógica de dominio                              │
│  • Validaciones básicas                           │
└───────────────────────────────────────────────────┘
```


---

## PARTE III: TRANSICIÓN A FRAMEWORKS MODERNOS (40 minutos)

### Sección A: Conceptos Puente (20 minutos)

#### Actividad 3: Tabla Comparativa de Inyección de Dependencias

Complete la siguiente tabla comparativa entre Java EE CDI, Spring y PrimeFaces:

| Concepto | Java EE / CDI | Spring / Spring Boot | PrimeFaces + CDI |
|----------|---------------|----------------------|------------------|
| Anotación para inyección |`@Inject`|`@Autowired` o `@Inject`|`@Inject `|
| Anotación para bean de sesión | `@SessionScoped` | `@SessionScope`  |`@SessionScoped`|
| Anotación para bean de request | `@RequestScoped` | `@RequestScope` | `@RequestScoped` |
| Anotación para singleton | `@ApplicationScoped` |`@Singleton` o `@Component` |`@ApplicationScoped`|
| Componente de servicio | `@Stateless` o CDI bean |`@Service`| N/A |
| Transacciones declarativas | `@Transactional` (JTA) | `@Transactional` |N/A se delega al servicio CDI|
| Configuración | `beans.xml` |`@Configuration` + `application.properties/YAML` |`beans.xml` |
| Managed Bean para vista | CDI Managed Bean `@Named` | `@Controller`|`@Named + @ViewScoped` |

**Investigue y complete:**
- Spring Boot usa `@Autowired` o `@Inject` 
- Spring Boot scope equivalente: `@RequestScope`, `@SessionScope`, `@ApplicationScope` 
- Spring Boot usa `@Service` para servicios 
- PrimeFaces trabaja con los mismos managed beans CDI de JSF. Rta: Si, PrimeFaces no define sus propios beans, usa los mismos managed beans de CDI/JSF.
- PrimeFaces scope especial: `@ViewScoped` (de JSF). Rta: Permite mantener el estado mientras el usuario permanece en la misma vista.


#### Actividad 4: De Java EE a Spring Boot

**Dado este código Java EE CDI:**

```java
@Named
@SessionScoped
public class ProductoController implements Serializable {
    
    @Inject
    private ProductoService productoService;
    
    private List<Producto> productos;
    
    @PostConstruct
    public void init() {
        productos = productoService.listarTodos();
    }
    
    // getters y setters
}

@Stateless
public class ProductoService {
    
    @Inject
    private ProductoDAO productoDAO;
    
    public List<Producto> listarTodos() {
        return productoDAO.findAll();
    }
}
```

**Conviértalo a Spring Boot:**

```java
// Complete el código Spring Boot equivalente
@Controller
@SessionAttributes("productos")
public class ProductoController {
    
    // ¿Qué anotación usar para inyección?
    @Autowired o @Injection // Inyección de dependencias de Spring 
    private ProductoService productoService;
    
    // ¿Cómo manejar la inicialización?
    @PostConstruct // Método que se ejecuta después de la construcción del bean y la inyección de dependencias
    public void init() {
        productos = productoService.listarTodos();
    }
    
    // Complete los métodos

    @GetMapping("/productos") // Mapea peticiones GET a /productos
    @ResponseBody // Indica que el retorno se serializa directamente en el cuerpo de la respuesta
    public List<Producto> listarProductos(Model model) {
        List<Producto> productos = productoService.listarTodos();
        model.addAttribute("productos", productos); // Agrega la lista al modelo para la vista
        return productos;
    
}



// ¿Qué anotación usar para el servicio?
@Service // Marca la clase como un componente de servicio de Spring (capa de lógica de negocio)
public class ProductoService {
    
    // ¿Cómo inyectar el repository?
    @Autowired o @Injection // Inyección de dependencias de Sprin
    private ProductoRepository productoRepository;
    
    public List<Producto> listarTodos() {
        // En Spring Boot con JPA
        return productoRepository.findAll(); // Spring Data JPA proporciona este método automáticamente
    }
}
```

**Preguntas guía:**
1. ¿Qué equivale `@Inject` en Spring? Rta: En Spring, el equivalente a @Inject de Java EE CDI es @Autowired, aunque ambas anotaciones (@Inject y @Autowired) funcionan en Spring Boot. 
2. ¿Qué equivale `@Stateless` en Spring? Rta: En Spring, el equivalente a @Stateless de Java EE es @Service.
3. ¿Cómo se declara un Repository en Spring Boot? Rta: En Spring Boot se declara como una interfaz que extiende de JpaRepository u otros tipos de Repositorios. 
Ejemplo:
```
@Repository  // 
public interface ProductoRepository extends JpaRepository<Producto, Long> {
    // Spring Data JPA genera automáticamente la implementación.
    // Algunos metodos disponibles findAll(), save(), deleteById(), etc.
}
```
4. ¿Qué ventajas tiene Spring Boot sobre Java EE tradicional? 
Rta: 
-Spring Boot tiene una configuración más simplificada.
-Tiene servidor embebido, Tomcat o Jetty ya incluido.
-Desarrollo más ágil: 
-Auto-configuración: 
    Spring Boot configura automáticamente beans según las dependencias
    Spring Boot DevTools: Recarga automática durante desarrollo
    Starters: Dependencias preconfiguradas

| Característica | Java EE Tradicional | Spring Boot |
|----------------|---------------------|-------------|
| Configuración | XML mas extenso | application.properties mas simple|
| Servidor | Externo (pesado y más complejo de configurar) | Embebido  |
| Tiempo de inicio | Puede resultar un poco más lento | Es más rápido |
| Microservicios | Complejo | Nativo |

### Sección B: Introducción a PrimeFaces (20 minutos)

#### Actividad 5: Comparación JSP vs PrimeFaces

**Analice estos dos códigos equivalentes:**

**JSP Tradicional:**
```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>

<form action="guardarProducto" method="post">
    <label>Nombre:</label>
    <input type="text" name="nombre" value="${producto.nombre}"/>
    
    <label>Precio:</label>
    <input type="number" name="precio" value="${producto.precio}"/>
    
    <input type="submit" value="Guardar"/>
</form>

<table>
    <tr>
        <th>Código</th>
        <th>Nombre</th>
        <th>Precio</th>
    </tr>
    <c:forEach items="${productos}" var="p">
    <tr>
        <td>${p.codigo}</td>
        <td>${p.nombre}</td>
        <td>${p.precio}</td>
    </tr>
    </c:forEach>
</table>
```

**PrimeFaces (XHTML con JSF):**
```xhtml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="http://xmlns.jcp.org/jsf/html"
      xmlns:p="http://primefaces.org/ui">
<h:head>
    <title>Productos</title>
</h:head>
<h:body>
    <h:form>
        <p:panelGrid columns="2">
            <p:outputLabel value="Nombre:"/>
            <p:inputText value="#{productoController.producto.nombre}"/>
            
            <p:outputLabel value="Precio:"/>
            <p:inputNumber value="#{productoController.producto.precio}"/>
        </p:panelGrid>
        
        <p:commandButton value="Guardar" 
                         action="#{productoController.guardar}"
                         update="tablaProductos"/>
    </h:form>
    
    <p:dataTable id="tablaProductos" 
                 value="#{productoController.productos}" 
                 var="p">
        <p:column headerText="Código">
            <h:outputText value="#{p.codigo}"/>
        </p:column>
        <p:column headerText="Nombre">
            <h:outputText value="#{p.nombre}"/>
        </p:column>
        <p:column headerText="Precio">
            <h:outputText value="#{p.precio}"/>
        </p:column>
    </p:dataTable>
</h:body>
</html>
```

**Responda:**
1. ¿Qué ventajas observa en PrimeFaces sobre JSP tradicional?
2. ¿Cómo se vinculan los componentes PrimeFaces con el Managed Bean?
3. ¿Qué es `update="tablaProductos"` y qué tecnología usa (AJAX)?
4. ¿El Managed Bean cambia al usar PrimeFaces? ¿Por qué?

#### Actividad 6: Componentes Avanzados PrimeFaces

**Investigue y describa casos de uso para estos componentes PrimeFaces:**

1. **`<p:dataTable>` con LazyDataModel**
   - ¿Cuándo usar paginación lazy?
   - ¿Qué ventaja tiene sobre cargar toda la lista?

2. **`<p:dialog>`**
   - ¿Cómo mejora la UX comparado con páginas separadas?
   - ¿Se requiere JavaScript manual?

3. **`<p:autoComplete>`**
   - ¿Cómo se implementa el método de búsqueda en el bean?
   - ¿Es necesario AJAX?

4. **`<p:fileUpload>`**
   - ¿Cómo se procesa el archivo en el Managed Bean?
   - ¿Dónde se guardaría en la arquitectura en capas?

---

## PARTE IV: CASO INTEGRADOR FINAL (10 minutos)

### Diseño de Arquitectura Completa

**Escenario:** Sistema de E-commerce con carrito de compras, gestión de inventario y procesamiento de pagos.

**Tarea:** Diseñe la arquitectura completa especificando:

1. **Capas y componentes:**
   - Entidades (JavaBeans)
   - DAOs
   - Services
   - Facades (si aplica)
   - Controllers
   - Vistas (JSP/PrimeFaces)

2. **Scopes de cada Managed Bean:**
   - `LoginController`: ¿?
   - `ProductoCatalogoController`: ¿?
   - `CarritoComprasController`: ¿?
   - `CheckoutMultipasoController`: ¿?

3. **Puntos donde se requieren transacciones:**
   - Liste las operaciones que deben ser transaccionales

4. **Transición a tecnologías modernas:**
   - ¿Qué cambiaría al migrar a Spring Boot?
   - ¿Qué componentes PrimeFaces usaría en el catálogo?
   - ¿Cómo implementaría el carrito con AJAX de PrimeFaces?

---

## RECURSOS COMPLEMENTARIOS

### Para profundizar en PrimeFaces:
- PrimeFaces Showcase: https://www.primefaces.org/showcase/
- Tutoriales de DataTable con LazyDataModel
- Componentes AJAX y actualización parcial

### Para profundizar en Spring Boot:
- Spring Initializr para crear proyectos
- Comparación: `@Inject` vs `@Autowired`
- Spring Data JPA vs DAOs manuales
- `@RestController` para APIs REST

### Conceptos clave para próximas clases:
- **PrimeFaces**: LazyDataModel, AJAX, diálogos modales, FileUpload
- **Spring Boot**: Anotaciones (`@Service`, `@Repository`, `@Autowired`), Spring Data JPA, configuración con `application.properties`
- **Arquitectura moderna**: Microservicios, APIs REST, separación frontend/backend

---

## ENTREGABLE

**Documento PDF o MD con:**
1. Tablas completadas de la Parte I
2. Análisis de problemas y soluciones propuestas
3. Código refactorizado del ejercicio práctico (Parte II)
4. Tablas comparativas completadas (Parte III)
5. Diseño de arquitectura del caso integrador (Parte IV)

**Fecha de entrega:** 31 octubre 2025

---

**FIN DEL TALLER**
