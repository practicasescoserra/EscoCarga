# EscoCarga — Propuesta de arquitectura (v0.1, para revisión)

> Estado: **propuesta**. No se ha generado código de la aplicación. Todo lo marcado
> con ⚠️ **VERIFICAR** debe confirmarse con documentación oficial de Epson y con las
> impresoras físicas antes de implementarse de forma definitiva.

---

## 1. Análisis de requerimientos

### 1.1 Lo que el MVP debe resolver

| # | Requerimiento | Implicación técnica |
|---|---------------|---------------------|
| 1 | Android + iOS | Flutter; lógica 100 % Dart; nativo solo si la impresora lo exige |
| 2 | Capturar *Solicitud de Carga Directa RED* (ADM-GER-SCD-001) | Modelo de datos fiel al formato; sin campos inventados |
| 3 | Validar datos | Reglas de validación en la capa de dominio, no en los widgets |
| 4 | Generar reporte | Transformación `Incidencia → Documento` independiente del protocolo |
| 5 | Imprimir por Wi‑Fi en TM‑T20III / TM‑T20IV | Socket TCP + ESC/POS (puerto 9100 a verificar) |
| 6 | Verificar conexión y reintentar | Consulta de estado previa + cola de trabajos de impresión |
| 7 | Conservar información y reimprimir | Persistencia local (SQLite) **antes** de imprimir |

### 1.2 Principio rector: guardar ≠ imprimir

```
[Capturar] → [Validar] → [GUARDAR incidencia (id único)] → [Crear trabajo de impresión] → [Imprimir]
                                   ▲                                    │
                                   └──── la incidencia nunca se pierde ─┘  (si falla: reintentar el
                                                                            MISMO trabajo, no crear
                                                                            otra incidencia)
```

- La incidencia se guarda con un **UUID generado en el dispositivo** antes de cualquier intento de impresión.
- Cada intento de impresión es un registro `PrintJob` ligado a ese UUID. Reintentar = nuevo intento del mismo
  trabajo; **nunca** se crea una incidencia nueva → se evita la duplicidad.
- Reimprimir (días después) = nuevo `PrintJob` sobre la misma incidencia.

### 1.3 Riesgos detectados

1. **Capacidades de red de las impresoras no homogéneas.** Las TM‑T20III y TM‑T20IV se venden en variantes con
   distintas interfaces (USB, serie, Ethernet, Wi‑Fi). La interfaz Wi‑Fi puede ser integrada o un módulo
   adicional según el modelo/región. ⚠️ **VERIFICAR** el número de modelo exacto (etiqueta inferior, p. ej.
   `M267D`/`M267E`…) de cada impresora de las estaciones e imprimir su *hoja de estado* (status sheet) para
   conocer IP, MAC, puertos y protocolos habilitados.
2. **Puerto 9100 (RAW/“Port 9100”)** es el mecanismo habitual de las interfaces de red Epson, pero puede estar
   deshabilitado en la configuración (EpsonNet Config / Web Config). ⚠️ **VERIFICAR**.
3. **Caracteres en español** (ñ, á, é, ¿…): requieren seleccionar una tabla de caracteres ESC/POS
   (p. ej. PC850 o WPC1252 con `ESC t n`) y codificar el texto con esa misma tabla. ⚠️ **VERIFICAR** qué tablas
   trae cada firmware.
4. **iOS y la red local**: desde iOS 14 el sistema pide permiso de “Red local”; el descubrimiento por
   broadcast/multicast propio requiere un *entitlement* que Apple aprueba manualmente (ver §5.4).
5. **Confirmación de impresión**: TCP confirma que los bytes llegaron a la impresora, no que el papel salió.
   Se mitiga consultando estado en tiempo real (papel, tapa, error) antes y después de enviar (⚠️ **VERIFICAR**
   que la interfaz de red responda a comandos de estado `DLE EOT` por el puerto 9100).

---

## 2. Decisiones técnicas propuestas (y por qué)

| Tema | Propuesta | Alternativas | Motivo |
|------|-----------|--------------|--------|
| Lenguaje / framework | Flutter (stable) + Dart 3 | — | Requerimiento |
| Comunicación impresora | `dart:io` `Socket` (sin paquete) | Epson ePOS SDK nativo vía Platform Channels | Funciona igual en Android e iOS, cero dependencias, control total de errores y timeouts |
| Comandos ESC/POS | **Generador propio, pequeño** (≈200 líneas: texto, negrita, alineación, tamaño, corte, tabla de caracteres) | `esc_pos_utils_plus` u otros | Solo usamos ~10 comandos; los paquetes ESC/POS de la comunidad tienen mantenimiento irregular. Evita dependencia crítica |
| Persistencia | **SQLite** vía `drift` (o `sqflite` si se prefiere algo más simple) | Hive / Isar / SharedPreferences | Datos relacionales (incidencias ↔ trabajos de impresión ↔ impresoras), consultas por fecha/estado, migraciones. `shared_preferences` solo para preferencias triviales |
| Estado / DI | `flutter_riverpod` | Provider, Bloc, `ChangeNotifier` puro | Inyección de dependencias + estado testeable con una sola librería |
| Navegación | `go_router` | Navigator 2 manual | Oficial del equipo Flutter; opcional — con 5 pantallas también basta `Navigator` |
| Descubrimiento | Escaneo de subred TCP:9100 (Dart puro) + IP manual + última IP conocida | Epson SDK Discovery, mDNS | El escaneo no requiere entitlements especiales en iOS; ver §5 |
| Fotos de INE/licencia | **Fuera del MVP** (pendiente de definir almacenamiento) | `image_picker` | Decisión pendiente §9 del contexto |

**Dependencias del MVP (propuesta mínima):** `flutter_riverpod`, `drift` + `sqlite3_flutter_libs`
(+ `drift_dev`/`build_runner` solo en desarrollo), `uuid`, `intl` (fechas/moneda en es‑MX), `go_router` (opcional).
Nada más hasta que haga falta.

---

## 3. Estructura inicial del proyecto

Arquitectura por **features** con capas internas (presentation / domain / data). Los módulos de impresión
quedan en `core/printing` porque son infraestructura reutilizable por futuros tipos de incidencia.

```
escocarga/
├── lib/
│   ├── main.dart
│   ├── app/
│   │   ├── app.dart                    # MaterialApp, tema, router
│   │   ├── router.dart
│   │   └── theme/
│   │       ├── app_colors.dart         # #098D45 verde, #081845 azul marino
│   │       └── app_theme.dart
│   │
│   ├── core/
│   │   ├── database/                   # drift: AppDatabase, tablas, migraciones
│   │   ├── utils/                      # formateo de fechas, moneda, Result<T>
│   │   └── printing/                   # ── MÓDULOS DE IMPRESIÓN ──
│   │       ├── document/               # (3) Documento abstracto independiente del protocolo
│   │       │   ├── print_document.dart     # líneas, títulos, pares clave-valor, separadores, firmas
│   │       │   └── document_layout.dart    # ajuste de texto a N columnas (48 col @ 80 mm, a verificar)
│   │       ├── escpos/                 # (6) Generación de comandos ESC/POS
│   │       │   ├── escpos_commands.dart    # constantes ESC/GS/DLE
│   │       │   ├── escpos_encoder.dart     # PrintDocument → List<int>
│   │       │   └── charset_encoder.dart    # UTF-8 → PC850/WPC1252
│   │       ├── transport/              # (5) Comunicación
│   │       │   ├── printer_transport.dart  # interfaz abstracta
│   │       │   └── tcp_transport.dart      # dart:io Socket, timeouts, errores tipados
│   │       ├── discovery/              # (4) Descubrimiento
│   │       │   ├── printer_discovery.dart  # interfaz
│   │       │   └── subnet_scan_discovery.dart
│   │       ├── status/
│   │       │   └── printer_status.dart     # parseo de DLE EOT (papel, tapa, error)
│   │       └── print_service.dart      # orquesta: verificar → enviar → confirmar → registrar
│   │
│   └── features/
│       ├── incidents/                  # (2) Gestión de incidencias
│       │   ├── domain/
│       │   │   ├── incident.dart           # entidad (Solicitud de Carga Directa RED)
│       │   │   ├── enums.dart              # Puesto, Producto, TipoChip
│       │   │   ├── incident_validator.dart
│       │   │   └── incident_repository.dart    # interfaz
│       │   ├── data/
│       │   │   └── incident_repository_drift.dart
│       │   ├── report/
│       │   │   └── scd_report_builder.dart # Incident → PrintDocument (formato ADM-GER-SCD-001)
│       │   └── presentation/
│       │       ├── incident_form_screen.dart
│       │       ├── incident_form_controller.dart
│       │       ├── incident_review_screen.dart
│       │       ├── incident_list_screen.dart   # historial + reimpresión
│       │       └── widgets/
│       ├── printers/                   # configuración de impresoras
│       │   ├── domain/printer_config.dart
│       │   ├── data/printer_repository_drift.dart
│       │   └── presentation/ (lista, alta/edición, “buscar en red”, “imprimir prueba”)
│       └── stations/                   # estación activa (local en el MVP)
│
├── test/                               # unit tests: validador, encoder ESC/POS, report builder, print service con transporte falso
├── tools/
│   └── print_probe.dart                # script Dart de línea de comandos para la prueba con impresoras reales
└── docs/
```

**Regla de dependencias:** `presentation → domain ← data`. `features/incidents/report` conoce
`core/printing/document`, pero **no** conoce ESC/POS ni TCP. Cambiar de protocolo (p. ej. ePOS‑Print XML o el
SDK de Epson) = nueva implementación de `PrinterTransport`/encoder, sin tocar formularios ni reportes.

---

## 4. Arquitectura del formulario de incidencias

### 4.1 Modelo de dominio (fiel al formato ADM-GER-SCD-001)

```dart
enum PuestoSolicitante { gerente, cajero }
enum TipoProducto { regularMagna, premiumPremier, diesel }
enum TipoChip { vehicularPegado, conductorLlavero }

class Incident {
  final String id;                 // UUID local (mejora digital: idempotencia)
  final DateTime fechaSolicitud;
  final String estacion;
  final String solicitanteNombre;
  final PuestoSolicitante solicitantePuesto;
  final String cliente;
  final String numeroCuentaCliente;
  final String vehiculoMarcaModelo;
  final String vehiculoPlacas;
  final String conductorNombre;
  final TipoProducto producto;
  final TipoChip tipoChip;
  final int montoCentavos;         // dinero en enteros, nunca double
  final int kilometraje;
  final String motivo;
  final String? ejecutivoRedNombre; // aparece en la sección de firmas
  final DateTime createdAt;        // mejora digital: auditoría
}
```

Campos marcados como “mejora digital” (`id`, `createdAt`) son internos y se justifican por la idempotencia y
trazabilidad. **No** se agrega folio consecutivo hasta que se decida (§9 del contexto).

### 4.2 Flujo de pantallas

```
Inicio (estación activa + botón “Nueva solicitud” + historial)
  └─ Formulario (una sola pantalla con secciones plegables o 3 pasos:
        1. Solicitud  2. Vehículo/Conductor/Carga  3. Motivo)
       └─ Revisión (vista previa del ticket)
            └─ [Guardar e imprimir]  → estado de impresión (Imprimiendo / Impreso / Error → Reintentar)
```

### 4.3 Decisiones de UX para captura rápida

- Selección única con **chips/segmented buttons** (puesto, producto, tipo de chip) en vez de dropdowns.
- Teclados adecuados: numérico para monto/km, mayúsculas automáticas en placas.
- Prellenado: fecha = hoy, estación = la activa, último solicitante usado.
- Validación al salir del campo + resumen al intentar continuar; el botón no se habilita hasta que es válido.
- Borrador automático: si la app se cierra a mitad de la captura, el formulario se recupera.

### 4.4 Validación (en `incident_validator.dart`, testeable sin UI)

Obligatorios: todos los campos del formato salvo los que se definan como opcionales. Reglas propuestas
(a confirmar): monto > 0; kilometraje ≥ 0; placas 5–8 caracteres alfanuméricos; motivo con longitud mínima.

---

## 5. Arquitectura del módulo de impresión Epson

### 5.1 Capas

```
Incident ──► ScdReportBuilder ──► PrintDocument ──► EscPosEncoder ──► bytes
                                                                       │
PrinterConfig (ip, puerto, ancho, charset) ──► PrintService ──► TcpTransport ──► TM-T20III / TM-T20IV
                                                    │
                                                    └─► PrintJobRepository (pendiente / enviado / error / intentos)
```

### 5.2 `PrintService` — algoritmo

1. Resolver impresora de la estación (IP guardada).
2. **Conectar** con timeout corto (≈3 s). Si falla → intentar redescubrir (§5.3) → si cambia la IP, actualizarla.
3. **Consultar estado** (`DLE EOT 1..4`): sin papel / tapa abierta / error → mensaje claro al usuario, no enviar.
4. Enviar bytes (inicializar `ESC @`, tabla de caracteres, contenido, avance y corte `GS V`).
5. Consultar estado de nuevo; marcar `PrintJob` como `enviado` o `error` con causa.
6. Errores tipados: `PrinterUnreachable`, `ConnectionTimeout`, `PaperOut`, `CoverOpen`, `PrinterError`,
   `UnknownError` → cada uno con mensaje en español y acción (“Reintentar”, “Cambiar impresora”).

### 5.3 Identificación y descubrimiento (estrategia híbrida)

| Orden | Mecanismo | Android | iOS |
|-------|-----------|---------|-----|
| 1 | Última IP conocida / reserva DHCP | ✅ | ✅ |
| 2 | IP manual | ✅ | ✅ |
| 3 | **Escaneo de subred** /24 al puerto 9100 (conexiones TCP en paralelo, ~254 hosts, 1–3 s) | ✅ | ✅ (solo requiere permiso de Red local) |
| 4 (futuro) | Discovery de Epson (SDK nativo) o mDNS/Bonjour | Requiere `MulticastLock` | Requiere entitlement de multicast y/o `NSBonjourServices` |

- La IP propia del teléfono se obtiene con `NetworkInterface.list()` (`dart:io`), sin plugins.
- **MAC**: no es obtenible de forma fiable desde una conexión TCP en iOS ni en Android moderno. Se capturará
  **manualmente** desde la hoja de estado de la impresora o, si más adelante se usa el SDK/discovery de Epson,
  desde su respuesta. Servirá para confirmar que la impresora encontrada en el escaneo es la correcta
  (cuando el mecanismo lo permita), nunca para conectar.
- Recomendación operativa fuerte: **reserva DHCP en el router de cada estación**. El escaneo es el plan B.

### 5.4 Diferencias Android / iOS relevantes

| Tema | Android | iOS |
|------|---------|-----|
| Permiso de red | `INTERNET` en el manifest (no requiere aprobación del usuario) | Desde iOS 14: diálogo “Red local” la primera vez; se necesita `NSLocalNetworkUsageDescription` en `Info.plist`. Si el usuario lo niega, **ninguna** conexión a la impresora funciona → la app debe detectarlo y guiar a Ajustes |
| Broadcast / multicast (descubrimiento Epson, mDNS propio) | `CHANGE_WIFI_MULTICAST_STATE` + `MulticastLock` | Entitlement `com.apple.developer.networking.multicast`, solicitado a Apple y aprobado manualmente. Bonjour con `NSBonjourServices` no requiere ese entitlement |
| Datos móviles vs Wi‑Fi | Si el Wi‑Fi “no tiene internet”, Android puede preferir datos móviles; las IPs privadas siguen ruteándose por Wi‑Fi normalmente, pero conviene probarlo | Comportamiento similar (“Wi‑Fi Assist”) |
| HTTP sin TLS (solo si se usa ePOS‑Print/Web Config) | Requiere `network_security_config` | Requiere excepción ATS |
| Segundo plano | La impresión ocurre en primer plano; no se requiere servicio | Igual |

### 5.5 Diseño del ticket

- Papel 80 mm (≈48 columnas en fuente A — ⚠️ **VERIFICAR** con la impresora; 58 mm es configurable en la TM‑T20).
- Encabezado: “GRUPO ESCOSERRA”, título “SOLICITUD DE CARGA DIRECTA RED”, código ADM-GER-SCD-001.
- Secciones en el mismo orden del formato físico; motivo con ajuste de línea.
- **Firmas**: en el MVP se imprimen líneas en blanco con el nombre debajo (gerente/cajero, conductor, ejecutivo
  RED), igual que el formato físico. Las firmas digitales quedan pendientes.
- Pie: nota de anexar INE/licencia; identificador corto de la incidencia (primeros caracteres del UUID — mejora
  digital, a aprobar) y, en reimpresiones, la leyenda “REIMPRESIÓN” (mejora digital, a aprobar).

---

## 6. Plan incremental propuesto

| Fase | Entregable | Depende de |
|------|------------|------------|
| **0. Prueba de impresoras** | `tools/print_probe.dart`: conecta por TCP a IP:9100, consulta estado, imprime ticket de prueba con acentos/ñ y corte. Se ejecuta desde una laptop **o** desde una app Flutter mínima en un teléfono | Acceso a una TM‑T20III y una TM‑T20IV |
| 1. Esqueleto | Proyecto Flutter, tema corporativo, navegación, base de datos vacía | — |
| 2. Formulario | Modelo, validador (con tests), pantallas de captura y revisión, guardado local | 1 |
| 3. Reporte | `PrintDocument` + `ScdReportBuilder` + vista previa en pantalla (tests) | 2 |
| 4. Impresión | Encoder ESC/POS, `TcpTransport`, `PrintService`, cola de trabajos y reintentos | 0, 3 |
| 5. Impresoras | Alta/edición, IP manual, escaneo de subred, impresión de prueba | 4 |
| 6. Historial | Lista de incidencias, detalle, reimpresión | 2, 4 |

Recomiendo empezar por la **Fase 0 y la Fase 1 en paralelo**: la Fase 0 despeja el mayor riesgo técnico.

---

## 7. Preguntas abiertas para el equipo

Ver la sección de dudas en la conversación / issue correspondiente. Se resumen en:

1. Modelos exactos y hoja de estado de cada impresora.
2. ¿Ancho de papel 80 mm en todas las estaciones?
3. ¿Número de copias por incidencia (una para estación, otra para el conductor, otra para RED)?
4. ¿Quién captura el número de cuenta y el nombre del ejecutivo RED?
5. ¿Un teléfono por estación (dispositivo de la empresa) o teléfonos personales?
6. ¿Catálogo de estaciones fijo en el MVP o configurable desde la app?
7. ¿Se aprueban el identificador corto y la leyenda “REIMPRESIÓN” en el ticket?
