# Explicación de POO aplicada en Sys.Ventas (para exposición / nota 20)

Este documento resume **dónde se aplican los conceptos de Programación Orientada a Objetos** en el proyecto y **cómo se conectan las clases/capas**.

## 1) Conceptos de POO y dónde se aplican

### Clase
Una **clase** es una plantilla que define atributos y métodos.
- `src/main/java/pe/edu/upeu/model/Producto.java` → clase de dominio `Producto`.
- `src/main/java/pe/edu/upeu/model/Cliente.java` → clase de dominio `Cliente`.
- `src/main/java/pe/edu/upeu/controller/ProductoController.java` → clase controladora de UI.
- `src/main/java/pe/edu/upeu/controller/VentaController.java` → clase controladora de ventas/POS.
- `src/main/java/pe/edu/upeu/controller/LoginController.java` → clase controladora de login.
- `src/main/java/pe/edu/upeu/service/impl/ProductoServiceImp.java` → clase de servicio.
- `src/main/java/pe/edu/upeu/repository/ProductoRepository.java` → clase repositorio.
- `src/main/java/pe/edu/upeu/repository/AbstractJpaRepository.java` → clase abstracta base.
- `src/main/java/pe/edu/upeu/config/AppContext.java`, `Conexion.java`, `DatabaseConfig.java` → clases de configuración/integración.

### Objeto
Un **objeto** es una instancia concreta de una clase en tiempo de ejecución.
- En `ProductoController.validarFormulario(...)` se crea un objeto: `Producto producto = Producto.builder()...build();`
- En `VentaController` se crean/usan objetos `Venta`, `VentaDetalle`, `Cliente`, `Comprobante`, `CarritoItem`.
- `AppContext` crea objetos con `new ProductoRepository()`, `new ProductoServiceImp(...)`, `new ProductoController(...)`, etc.
- JavaFX crea objetos de vista/control (`Stage`, `Scene`, `FXMLLoader`, `TableView`, etc.).

### Métodos
Los **métodos** son comportamientos/acciones de una clase u objeto.
- Modelo/servicios/repos: `guardar`, `actualizar`, `eliminar`, `listarTodo`, `buscarPorId`, `mapRow`, `insert`, etc.
- Controladores: `initialize()`, `listarProductos()`, `validarFormulario()`, `buscarClienteCdni()`, `registrarVenta()`, `generarComprobante()`, `login()`.
- Infraestructura: `DatabaseConfig.getConnection()`, `init()`, `Conexion.obtenerConexion()`.
- Cada método encapsula una tarea específica (listar, validar, guardar, conectar, etc.).

### Abstracción
La **abstracción** muestra lo esencial y oculta detalles de implementación.
- Interfaces (contratos):
  - `service/ICrudGenericoService.java`
  - `repository/ICrudGenericoRepository.java`
  - `service/IProductoService.java`, `service/IClienteService.java`, etc.
- Clase abstracta:
  - `repository/AbstractJpaRepository.java` define estructura común y deja métodos abstractos para las subclases.
- Fachadas/servicios de infraestructura:
  - `config/Conexion.java` y `config/DatabaseConfig.java` ocultan detalles de conexión/pool/HikariCP.
- Beneficio: los controladores/servicios trabajan con contratos sin conocer detalles SQL o de conexión.

### Encapsulamiento
El **encapsulamiento** protege el estado interno y controla el acceso mediante modificadores y métodos.
- Atributos `private` en modelos (`Producto`, `Cliente`, etc.) y acceso por getters/setters (Lombok `@Data`).
- Dependencias privadas en servicios/controladores (`private final IProductoService productoService;`, `private final ProductoRepository productoRepository;`).
- Estado interno oculto: `AppContext` tiene `contenedor` privado y constructor privado; `DatabaseConfig` maneja internamente el pool.
- Detalles de JDBC/SQL encapsulados en repositorios y no expuestos a la UI.

### Herencia
La **herencia** permite reutilizar/extender comportamiento de una clase base.
- Repositorios concretos heredan de clase abstracta base:
  - `ProductoRepository extends AbstractJpaRepository<Producto, Long>`
  - `ClienteRepository extends AbstractJpaRepository<Cliente, Long>`
  - `UsuarioRepository extends AbstractJpaRepository<Usuario, Long>`
  - `VentaRepository`, `VentaDetalleRepository`, etc. (misma estructura)
- Herencia de interfaces:
  - `IProductoService extends ICrudGenericoService<Producto, Long>`
  - `IClienteService extends ICrudGenericoService<Cliente, Long>`

### Polimorfismo
El **polimorfismo** permite usar distintas implementaciones a través de un mismo tipo (interfaz/clase base).
- **Por implementación de interfaz:**
  - `ProductoServiceImp implements IProductoService`
  - `ClienteServiceImp implements IClienteService`
  - Controladores reciben `IProductoService`, `IClienteService` (tipo abstracto), no la clase concreta.
- **Por sobrescritura (@Override):**
  - `ProductoRepository` sobrescribe `insert`, `updateRow`, `mapRow`, `getTableName`, etc. definidos en `AbstractJpaRepository`.
  - `ProductoServiceImp` sobrescribe métodos del contrato `IProductoService/ICrudGenericoService`.
- **Por uso polimórfico en tiempo de ejecución:**
  - `AppContext` registra/retorna beans por tipo/interfaz y compone dependencias.
  - `FXMLLoader.setControllerFactory(context::getBean)` obtiene controladores según el tipo solicitado.

## 2) Dónde se conectan las clases y capas (flujo del sistema)

### Arquitectura por capas (conexión principal)
1. **Vista (FXML / UI JavaFX)**
   - `src/main/resources/view/*.fxml` (login, maingui, main_producto, main_venta, main_comprobante)
   - Define la interfaz y dispara eventos.
2. **Controladores (presentación/lógica de UI)**
   - `controller/LoginController.java`, `MainGuiController.java`, `ProductoController.java`, `VentaController.java`, `MainComprobanteController.java`
   - Reciben eventos, validan datos de pantalla, crean/usan objetos del modelo y llaman a servicios.
3. **Servicios (lógica de negocio / orquestación)**
   - Interfaces: `service/I*Service.java`, `ICrudGenericoService.java`
   - Implementaciones: `service/impl/*ServiceImp.java`
   - Aplican reglas y coordinan operaciones usando repositorios.
4. **Repositorios (acceso a datos / persistencia)**
   - Interfaz base: `repository/ICrudGenericoRepository.java`
   - Clase abstracta: `repository/AbstractJpaRepository.java`
   - Concretos: `repository/*Repository.java`
   - Ejecutan SQL, mapean `ResultSet` ↔ objetos y encapsulan JDBC.
5. **Infraestructura / BD**
   - `config/Conexion.java` (fachada)
   - `config/DatabaseConfig.java` (HikariCP, propiedades, ejecución DDL)
   - `application.properties` + `schema_neon.sql`
   - Base de datos: **Neon PostgreSQL**

### Flujo ejemplo: CRUD de Productos
- `main_producto.fxml` (vista) → evento → `ProductoController`
- `ProductoController` crea/usa objeto `Producto` y llama a `IProductoService`
- `ProductoServiceImp` (implementa interfaz, polimorfismo) delega a `ProductoRepository`
- `ProductoRepository` (hereda de `AbstractJpaRepository`, sobrescribe métodos) ejecuta SQL
- `AbstractJpaRepository`/`Conexion`/`DatabaseConfig` obtienen conexión y acceden a Neon
- Resultado vuelve en sentido inverso hasta la tabla/UI

### Flujo ejemplo: Login
- `login.fxml` → `LoginController.login()`
- `LoginController` usa `IUsuarioService` (abstracción/polimorfismo)
- `UsuarioServiceImp` consulta `UsuarioRepository`
- `UsuarioRepository` valida credenciales en la BD (Neon)
- Si es correcto, se crea/actualiza estado de sesión (`SessionManager`) y se carga `maingui.fxml`

### Flujo ejemplo: Venta + Comprobante
- `main_venta.fxml` → `VentaController`
- Búsqueda cliente: primero en BD (`IClienteService`/repos), si no existe consulta `ConsultaDNI` (API apisperu) y autocompleta
- Registro de venta: `VentaController` crea objetos `Venta`/`VentaDetalle`, llama servicios/repos, actualiza stock de `Producto`
- Generación de comprobante: crea objeto `Comprobante` y abre `main_comprobante.fxml` con `MainComprobanteController`
- `MainComprobanteController` lista ventas (BD), muestra vista previa y exporta/imprime PDF

### Integración/Conexión de dependencias
- `AppContext` actúa como integrador:
  - crea objetos de repositorio, servicio y controlador
  - conecta dependencias (ej. `new ProductoServiceImp(getBean(ProductoRepository.class))`)
  - entrega instancias a JavaFX (`controllerFactory`)

## 3) Archivos con comentarios POO agregados en el código
Se agregaron comentarios explicativos en (entre otros):
- `model/Producto.java`, `model/Cliente.java`
- `service/ICrudGenericoService.java`, `service/IProductoService.java`
- `repository/ICrudGenericoRepository.java`, `repository/AbstractJpaRepository.java`, `repository/ProductoRepository.java`
- `service/impl/ProductoServiceImp.java`
- `controller/ProductoController.java`, `controller/VentaController.java`, `controller/LoginController.java`
- `config/AppContext.java`, `config/Conexion.java`, `config/DatabaseConfig.java`

Estos comentario
