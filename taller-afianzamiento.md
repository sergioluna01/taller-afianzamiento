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
| Scope del Controller principal | | | |
| ¿Necesita transacciones? ¿Por qué? | | | |
| ¿Usar Facade? Justifique | | | |
| Principal desafío de concurrencia | | | |
| Patrón de persistencia más apropiado | | | |

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
2. ¿Qué le falta al TransferenciaService para garantizar atomicidad?
3. ¿Qué violaciones arquitectónicas tiene el JSP? Proponga una solución.

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

**1. Capa de Modelo (JavaBeans)** - Cree las clases necesarias:
   - `Cita` con todos sus atributos
   - `Medico` con especialidad
   - Incluya métodos de negocio donde corresponda

   

**2. Capa de Persistencia (DAO)** - Implemente:
   - `CitaDAO` con métodos: `guardar()`, `verificarDisponibilidad()`
   - `MedicoDAO` con método: `obtenerPorId()`
   - Use `@Resource` para el DataSource
   - Manejo apropiado de conexiones

**3. Capa de Servicio (Service)** - Implemente:
   - `CitaService` con método `reservarCita()`
   - Use `@Inject` para inyectar DAOs
   - Implemente manejo transaccional con `UserTransaction`
   - Método para calcular precio según especialidad

**4. Capa de Presentación (Managed Bean)** - Implemente:
   - `CitaController` con scope apropiado
   - `@Inject` del CitaService
   - Atributos del formulario con getters/setters
   - Método de acción `procesarReserva()`

**5. Patrón Facade (Opcional - Bonus)** - Implemente:
   - `ClinicaFacade` que coordine reservar cita Y enviar email de confirmación
   - Debe inyectar `CitaService` y `NotificacionService`

---

## PARTE III: TRANSICIÓN A FRAMEWORKS MODERNOS (40 minutos)

### Sección A: Conceptos Puente (20 minutos)

#### Actividad 3: Tabla Comparativa de Inyección de Dependencias

Complete la siguiente tabla comparativa entre Java EE CDI, Spring y PrimeFaces:

| Concepto | Java EE / CDI | Spring / Spring Boot | PrimeFaces + CDI |
|----------|---------------|----------------------|------------------|
| Anotación para inyección | `@Inject` | | |
| Anotación para bean de sesión | `@SessionScoped` | | |
| Anotación para bean de request | `@RequestScoped` | | |
| Anotación para singleton | `@ApplicationScoped` | | |
| Componente de servicio | `@Stateless` o CDI bean | | N/A |
| Transacciones declarativas | `@Transactional` (JTA) | | |
| Configuración | `beans.xml` | | |
| Managed Bean para vista | CDI Managed Bean | | |

**Investigue y complete:**
- Spring Boot usa `@Autowired` o `@Inject` para inyección
- Spring Boot scope equivalente: `@RequestScope`, `@SessionScope`, `@ApplicationScope`
- Spring Boot usa `@Service` para servicios
- PrimeFaces trabaja con los mismos managed beans CDI de JSF
- PrimeFaces scope especial: `@ViewScoped` (de JSF)

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
    private ProductoService productoService;
    
    // ¿Cómo manejar la inicialización?
    
    
    // Complete los métodos
}

// ¿Qué anotación usar para el servicio?
public class ProductoService {
    
    // ¿Cómo inyectar el repository?
    private ProductoRepository productoRepository;
    
    public List<Producto> listarTodos() {
        // En Spring Boot con JPA
        return productoRepository.findAll();
    }
}
```

**Preguntas guía:**
1. ¿Qué equivale `@Inject` en Spring?
2. ¿Qué equivale `@Stateless` en Spring?
3. ¿Cómo se declara un Repository en Spring Boot?
4. ¿Qué ventajas tiene Spring Boot sobre Java EE tradicional?

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