# NexusPOS

## Sistema integral de ventas, inventario y administración

NexusPOS es un proyecto de práctica Full Stack para construir un sistema de punto de venta e inventario con una arquitectura **monolítica**. El objetivo es aprender implementando funcionalidades reales: desde la autenticación hasta la confirmación transaccional de ventas, con una interfaz Angular y una API REST en Spring Boot.

El monolito tendrá módulos separados y responsabilidades claras. No se dividirá en microservicios; los conceptos de microservicios se estudiarán como ejercicios de diseño y comparación, sin cambiar la arquitectura principal.

> Este repositorio funciona también como guía de aprendizaje. Avanza por etapas, registra decisiones y explica por qué elegiste cada solución.

## Objetivos

- Administrar usuarios, roles, clientes, categorías, productos e inventario.
- Registrar, confirmar y cancelar ventas con historial auditable.
- Consultar reportes y alertas de existencias bajas.
- Aplicar Java, POO, Spring Boot, REST, seguridad, persistencia y SQL.
- Construir una interfaz Angular que consuma la API.
- Practicar Maven, Git, Docker, pruebas y trabajo con Scrum.

## Alcance funcional

### Módulos

- Autenticación y autorización.
- Usuarios, roles y perfil de usuario.
- Clientes.
- Categorías y productos.
- Inventario y movimientos.
- Ventas y detalles de venta.
- Dashboard, reportes y auditoría básica.
- Importación de productos desde CSV.

### Roles iniciales

| Rol | Capacidades principales |
| --- | --- |
| `ADMIN` | Administra usuarios, catálogo, inventario, ventas y reportes; puede cancelar ventas. |
| `VENDEDOR` | Consulta catálogo y clientes, registra clientes, crea ventas y consulta sus ventas. |
| `CONSULTA` | Consulta la información que tenga autorizada. |

Un usuario puede tener varios roles y un rol puede pertenecer a varios usuarios. El perfil del usuario se modelará por separado para practicar relaciones JPA.

## Requisitos del dominio

### Usuarios y autenticación

- Datos mínimos: identificador, username, contraseña cifrada, nombre, apellidos, correo, estado y fechas de registro/actualización.
- Username y correo únicos entre usuarios activos.
- Estados: `ACTIVO`, `ELIMINADO` y `BLOQUEADO`. Un usuario eliminado o bloqueado no puede iniciar sesión.
- La contraseña nunca se guarda en texto plano; se utiliza un algoritmo de hashing integrado con Spring Security.
- Perfil con datos adicionales como teléfono, dirección, fecha de nacimiento y avatar.
- Inicio de sesión mediante `POST /api/auth/login`; al autenticarse se entrega un JWT con usuario, roles y expiración.

### Clientes

- Datos mínimos: nombre, apellidos, correo, teléfono, dirección, estado y fechas de registro/actualización.
- Correo y teléfono únicos entre clientes activos.
- Estados `ACTIVO` y `ELIMINADO`. Se usa borrado lógico; las consultas normales excluyen registros eliminados.

### Categorías

- Nombre, descripción y estado.
- Nombre único entre categorías activas.
- No se puede eliminar una categoría con productos activos.
- Una categoría tiene muchos productos; cada producto pertenece a una categoría.

### Productos

- SKU, nombre, descripción, precio de compra, precio de venta, stock, stock mínimo, categoría, estado y fechas.
- SKU único. Precios mayores que cero, stock nunca negativo y stock mínimo igual o mayor a cero.
- Cada producto pertenece a una categoría.
- Hay stock bajo cuando `stock <= stockMinimo`. La API ofrece esta consulta y el Dashboard la muestra.
- Los cambios de existencias se registran como movimientos; no se permite cambiar stock sin dejar historial.

### Movimientos de inventario

Cada movimiento registra producto, tipo, cantidad, stock anterior y nuevo, usuario responsable, fecha y motivo. Tipos mínimos: `ENTRADA`, `VENTA`, `CANCELACION_VENTA`, `AJUSTE_POSITIVO` y `AJUSTE_NEGATIVO`. Los movimientos son históricos y no se eliminan.

### Ventas y detalles

- Una venta contiene folio, cliente, vendedor, fecha, subtotal, descuento, impuestos, total y estado.
- Estados: `BORRADOR`, `CONFIRMADA` y `CANCELADA`.
- Una venta contiene uno o más detalles; cada detalle registra producto, cantidad, precio unitario, descuento y subtotal.
- Las cantidades deben ser positivas y los productos deben estar activos.
- El backend obtiene los precios y calcula subtotal, descuentos, impuestos y total. Nunca confía en cálculos enviados por el navegador.

### Confirmación y cancelación de ventas

Al confirmar una venta, el backend valida existencias, descuenta el stock, crea los movimientos y cambia el estado, todo en una sola transacción. Si falla cualquier paso, revierte la operación completa.

Solo se cancelan ventas confirmadas. La cancelación devuelve los productos al inventario, genera movimientos `CANCELACION_VENTA`, conserva el historial e impide una segunda cancelación. También es transaccional.

Debe controlarse la concurrencia: si queda una unidad disponible y dos usuarios intentan venderla, no pueden confirmarse ambas ventas. Como ejercicio avanzado, implementar bloqueo optimista con versión u otro mecanismo apropiado.

### Descuentos

Soportar al menos ausencia de descuento, descuento porcentual y descuento por cantidad. Diseñar las estrategias mediante interfaces e implementaciones polimórficas. Incluir una clase abstracta solo cuando exista una razón de dominio clara y documentar por qué conviene frente a una interfaz.

## Requisitos técnicos

### Stack

**Backend:** Java 17 o superior, Spring Boot, Spring Web, Spring Data JPA, Hibernate, Spring Security, JWT, Bean Validation, Maven y OpenAPI/Swagger. Lombok es opcional.

**Base de datos:** PostgreSQL. Mantener las reglas de negocio desacopladas del motor de base de datos para facilitar una eventual migración a otro sistema relacional.

**Frontend:** Angular, TypeScript, HTML, CSS, Angular Router, HttpClient, RxJS y Reactive Forms.

**Pruebas:** JUnit 5, Mockito, Spring Boot Test y MockMvc o equivalente en backend; utilizar el framework de pruebas configurado para Angular en frontend.

**Herramientas de trabajo:** Git, GitHub, Docker, Docker Compose y Postman. IntelliJ IDEA y VS Code son opciones de entorno de desarrollo.

### Arquitectura monolítica

Mantener separación interna de responsabilidades, sin acceso al repositorio desde los controladores:

```text
Controller → Service → Repository → Database
                   ↕
             DTO / Mapper / Entity
```

Organizar como mínimo paquetes equivalentes a `controller`, `dto`, `entity`, `repository`, `service`, `mapper`, `exception`, `security`, `config`, `utils` y `enums`. Las reglas de negocio pertenecen principalmente a `Service`; los controladores coordinan HTTP y los DTO evitan exponer entidades directamente.

Usar inyección por constructor. Documentar qué objetos son Spring Beans (`@Component`, `@Service`, `@Repository`, `@RestController`, `@Configuration`) y comparar inyección por constructor, setter y campo.

### API REST

- Usar correctamente `GET`, `POST`, `PUT`, `PATCH` y `DELETE`.
- `PUT` representa una actualización completa y `PATCH` un cambio parcial o una transición de estado (por ejemplo confirmar/cancelar venta o bloquear usuario).
- Ejemplos: `GET /api/productos`, `GET /api/productos/{id}`, `POST /api/productos`, `PUT /api/productos/{id}`, `PATCH /api/ventas/{id}/confirmar`.
- Listados principales con paginación, tamaño y orden: `page`, `size`, `sort`.
- Productos filtrables por nombre, SKU, categoría, estado y stock bajo.
- Ventas filtrables por fechas, cliente, vendedor y estado.
- Documentar operaciones y respuestas con OpenAPI/Swagger.
- Usar códigos HTTP de forma consistente: `200`, `201`, `204`, `400`, `401`, `403`, `404`, `409` y `500`.

### Validación y errores

- Validar entradas en backend con Bean Validation y reglas de negocio en servicios.
- Definir excepciones propias para situaciones como recurso inexistente, stock insuficiente, duplicidad y venta no cancelable.
- Manejar errores de forma global con `@RestControllerAdvice` y una respuesta consistente que incluya fecha, estado, código de error, mensaje y ruta.
- Distinguir excepciones checked y unchecked. La importación CSV debe ejercitar errores de lectura como `IOException` y convertirlos en una respuesta REST entendible.
- Responder con `409 Conflict` para conflictos de negocio y no devolver `200 OK` para todos los casos.

### Seguridad

- API sin estado (*stateless*), protegida con Spring Security y JWT; no utilizar sesiones tradicionales para mantener la autenticación.
- `POST /api/auth/login` es público; los recursos protegidos requieren token.
- Aplicar autorización real por rol: administración para `ADMIN`, operaciones de venta para `VENDEDOR` o `ADMIN` y consultas según permisos.
- Diferenciar `401 Unauthorized` (credenciales/token ausentes o inválidos) de `403 Forbidden` (identidad válida sin permiso).

### JPA, Hibernate y SQL

- Practicar `@Entity`, `@Table`, `@Id`, `@GeneratedValue`, `@Column`, `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`, `@JoinColumn` y `@JoinTable`.
- Elegir `LAZY` o `EAGER` conscientemente. Evitar `EAGER` indiscriminado y explicar cuándo se cargan las relaciones.
- Reproducir un caso N+1 en desarrollo y documentar su solución usando, según convenga, `JOIN FETCH`, `EntityGraph` o proyección DTO.
- Escribir consultas SQL manuales con `INNER JOIN`, `LEFT JOIN` y `RIGHT JOIN`.
- Analizar índices para SKU, nombre de producto, correo de cliente, fecha/estado de venta y fecha de movimiento; justificar sus beneficios y costos.
- Resolver al menos un ejercicio de optimización: identificar consulta lenta, revisar joins, índices y datos recuperados, optimizar y comparar el plan de ejecución cuando esté disponible.
- Documentar cómo la confirmación de una venta demuestra atomicidad, consistencia, aislamiento y durabilidad (ACID).

## Frontend Angular

Organizar la aplicación por áreas funcionales: `auth`, `dashboard`, `usuarios`, `clientes`, `categorias`, `productos`, `inventario`, `ventas`, `reportes`, `core` y `shared`.

- Separar pantallas en componentes reutilizables; no concentrar toda la aplicación en un solo componente.
- Centralizar las llamadas HTTP en servicios como `AuthService`, `ProductoService`, `ClienteService`, `VentaService` e `InventarioService`.
- Consumir la API con `HttpClient` y Observables; practicar `map`, `tap`, `catchError`, `finalize`, `switchMap`, `debounceTime` y `distinctUntilChanged` donde sean útiles.
- El buscador de productos espera una pausa entre teclas, evita búsquedas duplicadas y cancela consultas anteriores cuando llega un nuevo término.
- Usar Reactive Forms en login, producto, cliente, usuario y venta. Agregar un formulario sencillo basado en plantilla para comparar ambos enfoques.
- Validar en la interfaz `required`, longitudes, correo, rangos numéricos y patrones. Estas validaciones mejoran la experiencia; el backend vuelve a validar reglas importantes.
- Implementar `AuthGuard` y `RoleGuard`; las rutas administrativas solo se muestran a roles autorizados. Los guards complementan, pero no sustituyen, la seguridad del backend.
- Crear un interceptor que agregue `Authorization: Bearer <token>` a solicitudes protegidas.
- Manejar respuestas `400`, `401`, `403`, `404`, `409` y `500`. Un `401` devuelve al login; un `403` informa que el usuario no tiene permiso.
- Mostrar alertas de stock bajo en Dashboard y permitir gestionar ventas con totales calculados por el backend.

## Ejercicios de Java y fundamentos

Integrar los conceptos en funcionalidades reales y explicar las decisiones en documentación breve:

- **Colecciones:** `List` para detalles y resultados ordenados, `Set` para roles sin duplicados y `Map` para agrupaciones de reportes (por ejemplo, categoría a total vendido).
- **POO:** encapsulación y composición en el dominio; interfaces e implementaciones para estrategias de descuento; polimorfismo en su selección; una clase abstracta justificada.
- **`==` y `equals()`:** incluir una demostración de la diferencia y aplicar `equals()`/`hashCode()` cuidadosamente, considerando las implicaciones de entidades JPA.
- **Lambdas y Stream API:** filtrar, transformar, ordenar y agrupar datos de reportes o convertir DTO cuando esto mejore la claridad. No sustituir automáticamente todo ciclo `for` por streams.
- **Excepciones:** comparar checked y unchecked, crear excepciones de negocio y manejar excepciones de archivo en la importación CSV.
- **Spring:** identificar Beans, alcance de responsabilidades e inyección de dependencias.
- **Microservicios:** documentar cómo podrían separarse ventas, catálogo e inventario en el futuro, qué datos y contratos compartirían y qué complejidad distribuida aparecería. Este análisis no cambia la implementación monolítica.

## Pruebas y calidad

Crear pruebas unitarias para reglas de servicios (cálculos, transiciones de estado, stock y permisos) y pruebas de integración para persistencia y endpoints. Cubrir casos correctos, validaciones, recursos inexistentes, duplicados, permisos insuficientes, stock insuficiente, rollback y doble cancelación. En Angular, probar componentes y servicios principales.

Las pruebas deben verificar comportamiento observable y reglas del dominio, no detalles internos sin valor. Mantener datos de prueba reproducibles y documentar cómo ejecutar cada conjunto.

## Git, Maven y Docker

- Usar Git con cambios pequeños y mensajes claros; trabajar con ramas para funcionalidades y correcciones.
- Mantener el proyecto Maven reproducible, con dependencias y perfiles de configuración separados para desarrollo y ejecución empaquetada.
- No guardar secretos ni credenciales en el repositorio. Proveer un archivo de ejemplo de variables de entorno.
- Empaquetar los componentes necesarios con Docker y levantar aplicación y PostgreSQL con Docker Compose.
- Documentar comandos de compilación, ejecución, pruebas y limpieza, además de puertos y variables requeridas.

## Scrum y organización del trabajo

Trabajar en iteraciones cortas. Mantener un Product Backlog priorizado, dividir historias en tareas de backend, frontend y datos, acordar criterios de aceptación y revisar el incremento al cierre de cada sprint.

Cada historia debe describir quién necesita la capacidad, para qué y cómo se comprobará. Registrar bloqueos, decisiones técnicas y retrospectivas breves para identificar qué mejorar en la siguiente iteración.

## Criterios de aceptación

- La aplicación puede levantarse localmente y con Docker Compose, conectada a PostgreSQL.
- Un usuario puede autenticarse y recibir un JWT; endpoints y pantallas respetan los roles.
- Se administran clientes, categorías y productos con validaciones, filtros y paginación.
- Cada ajuste de inventario deja un movimiento histórico y el Dashboard identifica stock bajo.
- Una venta obtiene precios y totales del backend, confirma stock de forma atómica y puede cancelarse una sola vez con restitución auditable.
- Los errores de API tienen formato estable y códigos HTTP adecuados.
- Swagger describe los endpoints principales; las pruebas cubren reglas críticas y fallos esperados.
- La interfaz Angular integra formularios, navegación protegida, interceptor, búsqueda reactiva y manejo de errores.
- La documentación explica las decisiones y ejercicios técnicos indicados en este README.

## Entregas sugeridas por etapas

1. **Base del proyecto:** repositorios Git, estructura Maven y Angular, conexión PostgreSQL y ejecución local.
2. **Catálogo:** categorías y productos, entidades JPA, DTO, mapper, CRUD, validaciones, filtros y paginación.
3. **Identidad:** usuarios, perfiles, roles, login, JWT, seguridad y guards.
4. **Clientes e inventario:** borrado lógico, movimientos, ajustes y alertas de stock bajo.
5. **Ventas:** carrito/registro, cálculo backend, confirmación transaccional, concurrencia y cancelación.
6. **Reportes y aprendizaje:** agrupaciones, lambdas/streams, CSV, SQL, N+1, índices y optimización.
7. **Calidad y entrega:** pruebas, Swagger, Docker Compose, revisión de criterios de aceptación y retrospectiva Scrum.

## Ejercicios de cierre

- Explica una petición desde Angular hasta PostgreSQL y de vuelta al usuario.
- Explica por qué una venta confirmada es transaccional y cómo se evita vender dos veces la última unidad.
- Compara `PUT` con `PATCH`, `401` con `403`, `LAZY` con `EAGER`, interfaz con clase abstracta y `==` con `equals()`.
- Muestra el antes y después de resolver N+1 y optimizar una consulta.
- Propón una evolución futura hacia microservicios, identificando costos y límites, sin perder de vista que NexusPOS se entrega como monolito.

## Estado

Proyecto de práctica en desarrollo. El alcance puede implementarse gradualmente; cada etapa debe quedar funcional y documentada antes de avanzar.
