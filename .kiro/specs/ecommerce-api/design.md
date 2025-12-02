# Documento de Diseño - Sistema de Gestión de Pedidos E-commerce

## Overview

El sistema de gestión de pedidos será desarrollado como una API REST usando Spring Boot WebFlux con arquitectura reactiva. La aplicación implementará una arquitectura híbrida con bases de datos relacionales y no relacionales, siguiendo patrones de diseño sólidos para mantener la separación de responsabilidades y facilitar el testing y mantenimiento.

### Technology Stack
- **Framework:** Spring Boot 3.x con WebFlux (reactive)
- **Base de Datos Relacional:** PostgreSQL con R2DBC para clientes y productos
- **Base de Datos No Relacional:** MongoDB con Spring Data MongoDB Reactive para pedidos
- **Validación:** Bean Validation (JSR-303)
- **Testing:** JUnit 5, Mockito, TestContainers
- **Containerización:** Docker y Docker Compose

## Architecture

### Arquitectura Híbrida con Bases de Datos Duales

```
┌─────────────────────────────────────────────────────────────┐
│                    Web Layer                                │
│              (Controllers, DTOs)                            │
├─────────────────────────────────────────────────────────────┤
│                  Service Layer                              │
│           (Business Logic, Transacciones)                   │
├─────────────────────────────────────────────────────────────┤
│                Repository Layer                             │
│  ┌─────────────────────┐    ┌─────────────────────────────┐ │
│  │   PostgreSQL Repos  │    │     MongoDB Repos           │ │
│  │ (Clientes/Productos)│    │      (Pedidos)              │ │
│  └─────────────────────┘    └─────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│                Database Layer                               │
│  ┌─────────────────────┐    ┌─────────────────────────────┐ │
│  │    PostgreSQL       │    │       MongoDB               │ │
│  │   (Relacional)      │    │   (No Relacional)           │ │
│  └─────────────────────┘    └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Package Structure
```
src/main/java/com/ecommerce/orders/
├── config/              # Configuraciones de BD y aplicación
│   ├── PostgreSQLConfig.java
│   ├── MongoConfig.java
│   └── WebFluxConfig.java
├── controller/          # REST controllers
├── dto/                # Data Transfer Objects
├── entity/             # Entidades de base de datos
│   ├── postgresql/     # Entidades PostgreSQL (Cliente, Producto)
│   └── mongodb/        # Documentos MongoDB (Pedido)
├── exception/          # Excepciones personalizadas
├── repository/         # Repositorios de datos
│   ├── postgresql/     # Repositorios R2DBC
│   └── mongodb/        # Repositorios MongoDB Reactive
├── service/            # Lógica de negocio
└── util/               # Clases utilitarias
```

## Components and Interfaces

### Entidades PostgreSQL (Datos Maestros)

#### Cliente Entity
```java
@Table("clientes")
public class Cliente {
    @Id
    private Long id;
    private String nombre;
    private String apellido;
    private String email;
    private String telefono;
    private String direccion;
    private LocalDateTime fechaCreacion;
    private LocalDateTime fechaActualizacion;
}
```

#### Producto Entity
```java
@Table("productos")
public class Producto {
    @Id
    private Long id;
    private String nombre;
    private String descripcion;
    private BigDecimal precio;
    private Integer stock;
    private String categoria;
    private Boolean activo;
    private LocalDateTime fechaCreacion;
    private LocalDateTime fechaActualizacion;
}
```

### Documentos MongoDB (Pedidos)

#### Pedido Document
```java
@Document(collection = "pedidos")
public class Pedido {
    @Id
    private String id;
    private Long clienteId;
    private List<ItemPedido> productos;
    private BigDecimal total;
    private EstadoPedido estado;
    private LocalDateTime fechaCreacion;
    private LocalDateTime fechaActualizacion;
}
```

#### ItemPedido (Embedded Document)
```java
public class ItemPedido {
    private Long productoId;
    private String nombreProducto;
    private Integer cantidad;
    private BigDecimal precioUnitario;
    private BigDecimal subtotal;
}
```

### Enums

#### EstadoPedido
```java
public enum EstadoPedido {
    PENDIENTE,
    CONFIRMADO,
    PROCESANDO,
    ENVIADO,
    ENTREGADO,
    CANCELADO
}
```

### Service Interfaces

#### ClienteService
```java
public interface ClienteService {
    Flux<ClienteDto> findAll();
    Mono<ClienteDto> findById(Long id);
    Mono<ClienteDto> create(CreateClienteDto dto);
    Mono<ClienteDto> update(Long id, UpdateClienteDto dto);
    Mono<Void> delete(Long id);
    Mono<Boolean> exists(Long id);
}
```

#### ProductoService
```java
public interface ProductoService {
    Flux<ProductoDto> findAll();
    Flux<ProductoDto> findByCategoria(String categoria);
    Mono<ProductoDto> findById(Long id);
    Mono<ProductoDto> create(CreateProductoDto dto);
    Mono<ProductoDto> update(Long id, UpdateProductoDto dto);
    Mono<Void> delete(Long id);
    Mono<Boolean> hasStock(Long productoId, Integer cantidad);
    Mono<Boolean> exists(Long id);
}
```

#### PedidoService
```java
public interface PedidoService {
    Mono<PedidoDto> create(CreatePedidoDto dto);
    Flux<PedidoDto> findAll();
    Mono<PedidoDto> findById(String id);
    Flux<PedidoDto> findByClienteId(Long clienteId);
    Mono<PedidoDto> update(String id, UpdatePedidoDto dto);
    Mono<Void> delete(String id);
    Mono<PedidoDto> updateEstado(String id, EstadoPedido estado);
}
```

#### TransactionService
```java
public interface TransactionService {
    Mono<PedidoDto> createPedidoTransactional(CreatePedidoDto dto);
    Mono<PedidoDto> updatePedidoTransactional(String pedidoId, UpdatePedidoDto dto);
}
```

### REST Endpoints

#### Cliente Controller
- `GET /api/clientes` - Listar todos los clientes
- `GET /api/clientes/{id}` - Obtener cliente por ID
- `POST /api/clientes` - Crear nuevo cliente
- `PUT /api/clientes/{id}` - Actualizar cliente
- `DELETE /api/clientes/{id}` - Eliminar cliente

#### Producto Controller
- `GET /api/productos` - Listar todos los productos
- `GET /api/productos/{id}` - Obtener producto por ID
- `GET /api/productos/categoria/{categoria}` - Obtener productos por categoría
- `POST /api/productos` - Crear nuevo producto
- `PUT /api/productos/{id}` - Actualizar producto
- `DELETE /api/productos/{id}` - Eliminar producto

#### Pedido Controller
- `GET /api/pedidos` - Listar todos los pedidos (con paginación)
- `GET /api/pedidos/{id}` - Obtener detalles de pedido específico
- `GET /api/pedidos/cliente/{clienteId}` - Obtener pedidos por cliente
- `POST /api/pedidos` - Crear nuevo pedido
- `PUT /api/pedidos/{id}` - Actualizar pedido
- `PUT /api/pedidos/{id}/estado` - Actualizar estado del pedido
- `DELETE /api/pedidos/{id}` - Eliminar pedido

## Data Models

### PostgreSQL Schema (Datos Maestros)

```sql
-- Tabla de clientes
CREATE TABLE clientes (
    id BIGSERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    apellido VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    telefono VARCHAR(20),
    direccion TEXT,
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabla de productos
CREATE TABLE productos (
    id BIGSERIAL PRIMARY KEY,
    nombre VARCHAR(200) NOT NULL,
    descripcion TEXT,
    precio DECIMAL(10,2) NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0,
    categoria VARCHAR(100),
    activo BOOLEAN DEFAULT true,
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índices para optimización
CREATE INDEX idx_productos_categoria ON productos(categoria);
CREATE INDEX idx_productos_activo ON productos(activo);
CREATE INDEX idx_clientes_email ON clientes(email);
```

### MongoDB Schema (Pedidos)

```javascript
// Colección: pedidos
{
  "_id": ObjectId("..."),
  "clienteId": 123,
  "productos": [
    {
      "productoId": 456,
      "nombreProducto": "Producto A",
      "cantidad": 2,
      "precioUnitario": 25.50,
      "subtotal": 51.00
    },
    {
      "productoId": 789,
      "nombreProducto": "Producto B", 
      "cantidad": 1,
      "precioUnitario": 15.75,
      "subtotal": 15.75
    }
  ],
  "total": 66.75,
  "estado": "PENDIENTE",
  "fechaCreacion": ISODate("2024-01-15T10:30:00Z"),
  "fechaActualizacion": ISODate("2024-01-15T10:30:00Z")
}

// Índices MongoDB
db.pedidos.createIndex({ "clienteId": 1 })
db.pedidos.createIndex({ "estado": 1 })
db.pedidos.createIndex({ "fechaCreacion": -1 })
```

### DTOs Structure

#### Request DTOs
- `CreateClienteDto`: nombre, apellido, email, telefono, direccion
- `UpdateClienteDto`: nombre, apellido, email, telefono, direccion (actualizaciones parciales)
- `CreateProductoDto`: nombre, descripcion, precio, stock, categoria
- `UpdateProductoDto`: nombre, descripcion, precio, stock, categoria (actualizaciones parciales)
- `CreatePedidoDto`: clienteId, productos (List<ItemPedidoDto>)
- `UpdatePedidoDto`: productos (List<ItemPedidoDto>)
- `ItemPedidoDto`: productoId, cantidad

#### Response DTOs
- `ClienteDto`: id, nombre, apellido, email, telefono, direccion, fechaCreacion
- `ProductoDto`: id, nombre, descripcion, precio, stock, categoria, activo, fechaCreacion
- `PedidoDto`: id, clienteId, cliente (ClienteDto), productos (List<ItemPedidoResponseDto>), total, estado, fechaCreacion
- `ItemPedidoResponseDto`: productoId, nombreProducto, cantidad, precioUnitario, subtotal
- `ErrorResponseDto`: mensaje, codigo, timestamp, detalles

## Error Handling

### Global Exception Handler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ValidationException.class)
    public Mono<ResponseEntity<ErrorResponseDto>> handleValidation(ValidationException ex);
    
    @ExceptionHandler(ClienteNotFoundException.class)
    public Mono<ResponseEntity<ErrorResponseDto>> handleClienteNotFound(ClienteNotFoundException ex);
    
    @ExceptionHandler(ProductoNotFoundException.class)
    public Mono<ResponseEntity<ErrorResponseDto>> handleProductoNotFound(ProductoNotFoundException ex);
    
    @ExceptionHandler(PedidoNotFoundException.class)
    public Mono<ResponseEntity<ErrorResponseDto>> handlePedidoNotFound(PedidoNotFoundException ex);
    
    @ExceptionHandler(InsufficientStockException.class)
    public Mono<ResponseEntity<ErrorResponseDto>> handleInsufficientStock(InsufficientStockException ex);
    
    @ExceptionHandler(TransactionException.class)
    public Mono<ResponseEntity<ErrorResponseDto>> handleTransaction(TransactionException ex);
    
    @ExceptionHandler(DatabaseException.class)
    public Mono<ResponseEntity<ErrorResponseDto>> handleDatabase(DatabaseException ex);
}
```

### Custom Exceptions
- `ClienteNotFoundException`: Cliente no encontrado (404)
- `ProductoNotFoundException`: Producto no encontrado (404)
- `PedidoNotFoundException`: Pedido no encontrado (404)
- `ValidationException`: Errores de validación de datos (400)
- `InsufficientStockException`: Stock insuficiente (400)
- `TransactionException`: Errores transaccionales (500)
- `DatabaseException`: Errores de base de datos (500)

## Testing Strategy

### Unit Tests
- **Service Layer**: Mockear repositories y validar lógica de negocio transaccional
- **Repository Layer**: TestContainers con PostgreSQL y MongoDB reales
- **Controller Layer**: WebTestClient para testing de endpoints reactivos

### Integration Tests
- **Database Integration**: TestContainers para pruebas con ambas BD
- **Transaction Integration**: Validar consistencia entre PostgreSQL y MongoDB
- **End-to-End**: Flujos completos de gestión de pedidos

### Test Structure
```
src/test/java/com/ecommerce/orders/
├── integration/         # Integration tests
│   ├── database/       # Tests de integración BD
│   └── transaction/    # Tests transaccionales
├── service/            # Service unit tests
├── repository/         # Repository tests
│   ├── postgresql/     # Tests R2DBC
│   └── mongodb/        # Tests MongoDB Reactive
├── controller/         # Controller tests
└── util/              # Test utilities
```

### Testing Tools
- **JUnit 5**: Framework de testing principal
- **Mockito**: Para mocking de dependencias
- **TestContainers**: Para testing con PostgreSQL y MongoDB reales
- **WebTestClient**: Para testing de endpoints reactivos
- **StepVerifier**: Para testing de Flux/Mono
- **Embedded MongoDB**: Para tests unitarios rápidos

## Gestión Transaccional

### Patrón Saga para Consistencia Distribuida

Dado que usamos dos bases de datos diferentes, implementaremos el patrón Saga para mantener consistencia:

```java
@Service
public class TransactionServiceImpl implements TransactionService {
    
    @Override
    public Mono<PedidoDto> createPedidoTransactional(CreatePedidoDto dto) {
        return validateCliente(dto.getClienteId())
            .then(validateProductos(dto.getProductos()))
            .then(reservarStock(dto.getProductos()))
            .then(crearPedido(dto))
            .onErrorResume(this::compensate);
    }
    
    private Mono<Void> compensate(Throwable error) {
        // Lógica de compensación para revertir cambios
        return liberarStock()
            .then(Mono.error(new TransactionException("Error en transacción", error)));
    }
}
```

### Validaciones Transaccionales

1. **Validar Cliente**: Verificar existencia en PostgreSQL
2. **Validar Productos**: Verificar existencia y stock en PostgreSQL
3. **Reservar Stock**: Actualizar stock en PostgreSQL
4. **Crear Pedido**: Insertar en MongoDB
5. **Compensación**: En caso de error, revertir cambios de stock

### Configuración Docker

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d

  mongodb:
    image: mongo:7
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
      MONGO_INITDB_DATABASE: ecommerce
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db

  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - mongodb
    environment:
      SPRING_PROFILES_ACTIVE: docker
      POSTGRES_URL: jdbc:postgresql://postgres:5432/ecommerce
      MONGODB_URI: mongodb://admin:password@mongodb:27017/ecommerce

volumes:
  postgres_data:
  mongodb_data:
```