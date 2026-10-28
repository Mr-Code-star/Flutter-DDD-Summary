# Guía de Arquitectura: Flutter con Domain-Driven Design (DDD)

Este proyecto implementa **Domain-Driven Design (DDD)** combinado con **Clean Architecture** y el patrón **BLoC** para la gestión de estado, usando **GetIt** para la inyección de dependencias.

El objetivo es separar estrictamente las **reglas de negocio** (dominio) de los **detalles técnicos** (UI, base de datos, APIs), facilitando la escalabilidad, el trabajo en equipo y las pruebas unitarias.

## Tabla de contenidos

1. [Estructura de carpetas](#1-estructura-de-carpetas)
2. [Capas y sus reglas](#2-capas-y-sus-reglas)
3. [Flujo de datos](#3-flujo-de-datos)
4. [Inyección de dependencias](#4-inyección-de-dependencias-getit)
5. [Cómo agregar una nueva funcionalidad](#5-cómo-agregar-una-nueva-funcionalidad)
6. [Convenciones y buenas prácticas](#6-convenciones-y-buenas-prácticas)

---

## 1. Estructura de carpetas

El código se organiza por **Bounded Contexts** (Contextos Delimitados). Actualmente existe el contexto `catalog`; los nuevos módulos (por ejemplo `users`, `checkout`) deben replicar esta misma estructura.

```text
lib/
├── catalog/                  # Bounded Context: Catálogo de productos
│   ├── api/                  # Puntos de entrada externos (ej: CatalogApi)
│   ├── application/          # Orquestadores y fachadas (ej: CatalogFacadeService)
│   ├── domain/               # Corazón del negocio (entidades, interfaces, lógica)
│   ├── infrastructure/       # Implementaciones técnicas (repositorios, data providers)
│   └── presentation/         # UI y gestión de estado (screens, widgets, BLoCs)
│
├── common/                   # Código compartido entre todos los contextos
│   ├── config/               # Configuraciones globales
│   ├── exceptions/           # Excepciones personalizadas (ServerException, etc.)
│   ├── helpers/              # Funciones de ayuda
│   ├── platform/             # Servicios del sistema (ej: Connectivity)
│   ├── utils/                # Utilidades (formateadores, validadores)
│   └── widgets/              # Widgets genéricos compartidos
│
├── injections.dart           # Configuración de GetIt (Service Locator)
└── main.dart                 # Punto de entrada de la aplicación
```

---

## 2. Capas y sus reglas

| Capa | Responsabilidad | Depende de |
|------|-----------------|------------|
| **Domain** | Reglas de negocio puras | Nada (Dart puro) |
| **Application** | Orquestar casos de uso | Domain |
| **Infrastructure** | Detalles técnicos (red, caché) | Domain |
| **Presentation** | UI y estado | Application |
| **Common** | Código compartido entre contextos | Nada |

### 2.1 Domain (Dominio)

> **Regla de oro:** esta capa **NO** debe depender de Flutter, paquetes de UI ni bases de datos. Es código Dart puro.

| Elemento | Archivo de ejemplo | Descripción |
|----------|--------------------|-------------|
| Entidades | `product.dart` | Objetos de negocio con identidad propia. Usan `equatable` para comparación por valor. |
| Interfaces / Contratos | `product_interface.dart` | Definen **qué** se puede hacer, no **cómo** (ej: `fetchProducts()`, `getProduct()`). |
| Lógica de dominio | `catalog_logic.dart`, `get_official_product_manufacturer.dart` | Reglas de negocio puras. |

### 2.2 Application (Aplicación)

Actúa como puente entre la presentación y el dominio. **No** contiene lógica de negocio compleja; solo orquesta.

| Elemento | Archivo de ejemplo | Descripción |
|----------|--------------------|-------------|
| Fachadas | `catalog_facade_service.dart` | Expone los casos de uso a la capa de presentación. Usa el repositorio del dominio y aplica la lógica de dominio (mixin `CatalogLogic`). |
| API | `index.dart` | Punto de entrada para que otras partes de la app o módulos externos consuman este contexto. **Nunca** se accede directamente al dominio o a la infraestructura. |

### 2.3 Infrastructure (Infraestructura)

Aquí viven los detalles técnicos. Implementa las interfaces del dominio.

| Elemento | Archivo de ejemplo | Descripción |
|----------|--------------------|-------------|
| Repositorios | `product_repository.dart` | Implementa `ProductInterface`. Decide de dónde obtener los datos (red o caché local) según la conectividad. |
| Data Providers | `product_local_data_provider.dart`, `product_remote_data_provider.dart` | Se comunican con la API REST o con la base de datos local (SQLite/Hive). |
| Modelos | `product_model.dart` | Extienden las entidades del dominio y añaden la serialización (JSON). |

### 2.4 Presentation (Presentación)

Todo lo relacionado con Flutter, la UI y la reacción a eventos del usuario.

| Elemento | Archivo de ejemplo | Descripción |
|----------|--------------------|-------------|
| BLoC | `catalog_list_bloc.dart` | Recibe eventos, llama a la fachada (`CatalogFacadeService`) y emite estados. |
| Screens | `catalog_list_screen.dart` | Widgets que escuchan el estado del BLoC (`BlocBuilder`) y renderizan la UI. |
| Widgets | `catalog_name.dart` | Componentes visuales específicos de este contexto. |

### 2.5 Common (Común)

Cualquier utilidad, excepción o widget que se use en **dos o más contextos** (`catalog`, `users`, etc.) debe ir aquí para evitar acoplamiento entre contextos.

---

## 3. Flujo de datos

El flujo siempre va en **una sola dirección**:

```text
[UI / Screen]
     │  (dispara un evento)
     ▼
[BLoC]
     │  (llama al caso de uso)
     ▼
[Application / Facade]
     │  (orquesta y aplica lógica)
     ▼
[Domain / Interface]
     ▲  (implementada por)
     │
[Infrastructure / Repository]
     │  (¿hay internet?)
     ├── Sí ──► [Remote Data Provider] ──► API
     └── No ──► [Local Data Provider]  ──► Caché
```

---

## 4. Inyección de dependencias (GetIt)

Se usa **GetIt** como Service Locator. Toda la configuración se encuentra en `injections.dart`.

> **Regla:** ninguna clase debe instanciar sus dependencias directamente con `NombreClase()`. Siempre deben ser inyectadas.

| Tipo de clase | Método de registro | Ciclo de vida |
|---------------|--------------------|---------------|
| BLoCs | `registerFactory` | Se crean y destruyen por pantalla. |
| Servicios, Fachadas y Repositorios | `registerLazySingleton` | Se instancian solo cuando se necesitan y viven toda la app. |

---

## 5. Cómo agregar una nueva funcionalidad

**Ejemplo:** agregar "Buscar producto por nombre".

1. **Domain**
   Agregar en `product_interface.dart`:
   ```dart
   Future<List<Product>> searchProducts(String query);
   ```

2. **Infrastructure**
   Implementar el método en `product_repository.dart` y en los Data Providers (local y remoto).

3. **Application**
   Agregar el método en `catalog_facade_service.dart` que llame al repositorio.

4. **Presentation**
   - Crear el nuevo **Event** y **State** en el BLoC.
   - Añadir la lógica de manejo del evento en el BLoC (`mapEventToState`).
   - En la UI, añadir un `TextField` que dispare el evento al BLoC.

---

## 6. Convenciones y buenas prácticas

### Nomenclatura

| Elemento | Estilo | Ejemplo |
|----------|--------|---------|
| Archivos | `snake_case.dart` | `catalog_list_bloc.dart` |
| Clases | `PascalCase` | `CatalogFacadeService` |
| Variables y métodos | `camelCase` | `fetchProducts` |

### Reglas de código

- **Excepciones:** usar las excepciones personalizadas de `common/exceptions`. Nunca exponer excepciones nativas (como `SocketException`) a la capa de presentación.
- **Manejo de errores en UI:** los BLoCs deben emitir estados de error específicos (ej: `CatalogListErrorState`) en lugar de lanzar excepciones que rompan la app.
- **No saltar capas:** la UI nunca debe llamar directamente a un `DataProvider` ni a un `Repository`. Siempre debe pasar por el BLoC y luego por la Facade.
- **Testing:** gracias a las interfaces en `domain`, los repositorios y BLoCs pueden mockearse fácilmente en pruebas unitarias.

Os mando el link de mas información como documentación diagramas esto es un resumen también encontraran el código fuente.

[Mas Informacion](https://github.com/mhadaily/flutter-architecture-ddd/tree/master)
