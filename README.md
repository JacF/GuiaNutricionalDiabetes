# 🥗 Guía Nutricional – Control de Alimentación

Aplicación Android nativa que convierte un **plan de alimentación en PDF** (plan para diabetes, 4 páginas) en una guía digital interactiva: catálogo de alimentos, menú semanal, seguimiento diario de comidas y recordatorios. Funciona **100 % sin conexión** y guarda todos los datos solo en el dispositivo.

> ⚠️ **Aviso:** es una herramienta informativa de acompañamiento al plan nutricional. **No sustituye la consulta médica o nutricional.** El contenido proviene del documento del plan y no debe modificarse sin la validación de un profesional de nutrición.

---

## 📋 Tabla de contenidos
1. [Características](#-características)
2. [Pantallas y módulos](#-pantallas-y-módulos)
3. [Tecnologías](#-tecnologías)
4. [Arquitectura](#-arquitectura)
5. [Base de datos](#-base-de-datos)
6. [Estructura del proyecto](#-estructura-del-proyecto)
7. [Requisitos y compilación](#-requisitos-y-compilación)
8. [Pruebas](#-pruebas)
9. [Privacidad y seguridad](#-privacidad-y-seguridad)
10. [Fidelidad al documento original](#-fidelidad-al-documento-original)
11. [Estado y limitaciones](#-estado-y-limitaciones)
12. [Documentación adicional](#-documentación-adicional)

---

## ✨ Características

- 📴 **Offline first:** sin permiso de Internet, sin cuentas ni registro.
- 🗄️ **Base de datos SQLite precargada** (Room) con todo el plan: el usuario no introduce nada manualmente.
- 🚦 **Semáforo de alimentos:** verde (recomendado), amarillo (recomendado con cantidad o frecuencia a respetar) y rojo (no recomendado), siempre acompañado de **icono y texto**, no solo de color.
- 🔎 **Búsqueda y filtros** por nombre, categoría, recomendados, no recomendados y frecuencia.
- 📅 **Menú semanal** de 7 días × 5 tiempos de comida, con ingredientes y cantidades exactas del documento.
- ✅ **Seguimiento diario** por fecha (pendiente / completada / omitida) con resumen de 7 días.
- 🔔 **Recordatorios locales** por tiempo de comida, con hora editable y reprogramación tras reiniciar el teléfono.
- 👤 **Perfil opcional** guardado solo en el dispositivo (nunca se rellena con datos del PDF).
- 💾 **Copia de seguridad local** en JSON (exportar y restaurar).
- 🌗 **Tema** claro, oscuro o del sistema.
- 🧾 **Trazabilidad del documento:** cada dato indica si es impreso, manuscrito o **pendiente de verificación**.

---

## 📱 Pantallas y módulos

Navegación inferior con 5 secciones, más pantallas secundarias.

| Módulo | Qué ofrece |
|---|---|
| **Inicio** | Saludo, fecha, próximo tiempo de comida, menú de hoy, accesos rápidos a recomendados / no recomendados y a recomendaciones |
| **Alimentos** | Catálogo por 12 grupos, buscador, filtros y tarjetas con cantidad, frecuencia y observaciones |
| **Menú** | Selector de día y tarjetas desplegables con ingredientes y cantidades |
| **Seguimiento** | Línea de comidas del día, estado por comida, navegación a días anteriores y resumen de los últimos 7 días |
| **Perfil** | Nombre, edad, peso, estatura, objetivos y preferencias, con validación; borrado de datos con confirmación |
| **Recomendaciones** *(desde Inicio)* | Tarjetas con los hábitos del plan y tabla de equivalencias |
| **Configuración** *(desde Perfil)* | Tema, recordatorios, copia de seguridad, restablecer preferencias y acerca de |

### Grupos de alimentos
Lácteos · Panes integrales · Cereales integrales · Frutas · Verduras · Carnes rojas · Carnes blancas · Tubérculos · Grasas · Leguminosas · Bebidas · Frutos secos y semillas.

### Tiempos de comida
| Tiempo | Horario |
|---|---|
| Desayuno | 08:00 – 08:30 |
| Merienda de media mañana | 10:00 – 10:30 |
| Almuerzo | 12:30 – 13:00 |
| Merienda de media tarde | 15:30 – 16:00 |
| Cena | 18:00 – 18:30 |

### Recordatorios
- Un interruptor y una hora editable por cada tiempo de comida (desactivados por defecto).
- Notificación personalizada: «Hora de tu almuerzo» + «Hoy: *preparación del día*».
- Solicitud del permiso de notificaciones en Android 13+, con acceso directo a los ajustes si se deniega.
- Se reprograman al **reiniciar**, **actualizar la app** y **cambiar hora o zona horaria**, y al abrir la app.
- Usan **alarmas inexactas** (`setAndAllowWhileIdle`): no requieren el permiso de alarmas exactas; pueden demorar unos minutos con el teléfono en reposo profundo.

### Copia de seguridad
Un archivo JSON con perfil, seguimiento, recordatorios y tema. La restauración **valida todo el archivo antes de tocar la base** y se aplica de forma atómica: si el archivo no es válido, no se modifica ningún dato. El plan de alimentación no se incluye porque ya viene con la app.

---

## 🛠️ Tecnologías

| Área | Tecnología |
|---|---|
| Lenguaje | Kotlin 2.0.21 |
| UI | Jetpack Compose (BOM 2024.10.01) + Material Design 3 |
| Navegación | Navigation Compose 2.8.4 |
| Persistencia | SQLite mediante Room 2.6.1 (KSP) |
| Asincronía | Coroutines y Flow |
| Arquitectura | MVVM + Repository |
| Notificaciones | AlarmManager + BroadcastReceivers |
| Build | Gradle 8.9 (wrapper) · Android Gradle Plugin 8.7.3 |
| SDK | minSdk 26 (Android 8.0) · compileSdk / targetSdk 35 |
| Pruebas | JUnit 4 · AndroidX Test |

Sin dependencias de red ni servicios externos. Inyección de dependencias manual (sin Hilt).

---

## 🏗️ Arquitectura

```
Pantalla (Compose) ──► ViewModel (StateFlow) ──► Repository ──► DAO (Room) ──► SQLite
        ▲                                                            │
        └─────────────────── Flow (cambios en vivo) ─────────────────┘
```

- **Seed por script:** en la primera creación de la base, `SeedCallback` ejecuta `assets/seed_datos.sql` (un `INSERT` por línea). Después no se repite.
- **Clasificación fiel al PDF:** en la base solo existe `RECOMENDADO` / `NO_RECOMENDADO`. El amarillo es una regla de interfaz que no reclasifica nada.
- **Recordatorios resilientes:** el estado vive en la base y las alarmas del sistema se reconstruyen a partir de él.

Más detalle en [`ARQUITECTURA.md`](ARQUITECTURA.md).

---

## 🗃️ Base de datos

Archivo: `nutricion_diabetes.db` (creado por Room en el dispositivo).

| Tabla | Propósito |
|---|---|
| `categorias_alimentos` | Los 12 grupos de alimentos |
| `alimentos` | Alimentos con clasificación, cantidad, unidad, frecuencia, observaciones y origen del dato |
| `tiempos_comida` | Los 5 tiempos con su horario |
| `menus` | 35 comidas (7 días × 5 tiempos) |
| `menu_ingredientes` | Ingredientes y cantidades de cada comida (texto literal del documento) |
| `recomendaciones` | Recomendaciones nutricionales del plan |
| `equivalencias` | Equivalencias alimentarias del plan |
| `perfil_usuario` | Perfil único del usuario (vacío al inicio) |
| `seguimiento_comidas` | Estado por comida y fecha (único por comida + fecha) |
| `configuracion_recordatorios` | Hora y estado de cada recordatorio |
| `plan_version` | Versión del plan cargado (preparado para futuras versiones) |

**Datos iniciales:** 12 categorías · 78 alimentos (54 recomendados, 24 no recomendados) · 35 menús · 108 ingredientes · 9 recomendaciones · 6 equivalencias.

Incluye claves primarias y foráneas, índices, restricciones únicas y `CHECK` (estados, día de la semana, clasificación). Script equivalente: [`FASE2_schema_nutricion_diabetes.sql`](FASE2_schema_nutricion_diabetes.sql). Copia poblada de referencia: `base_datos/nutricion_diabetes_referencia.db`.

---

## 📁 Estructura del proyecto

```
GuiaNutricional/
├── build.gradle.kts            # Plugins del proyecto
├── settings.gradle.kts
├── gradle.properties
├── gradlew · gradlew.bat
├── gradle/wrapper/             # Gradle 8.9
├── app/
│   ├── build.gradle.kts        # Dependencias del módulo
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml
│       │   ├── assets/seed_datos.sql
│       │   ├── res/            # strings, themes, iconos vectoriales
│       │   └── java/com/guianutricional/app/
│       │       ├── data/
│       │       │   ├── entities/ · dao/ · database/
│       │       │   ├── repository/     # NutricionRepository, RespaldoManager
│       │       │   └── preferences/
│       │       ├── presentation/
│       │       │   ├── home/ · alimentos/ · menu/ · seguimiento/
│       │       │   ├── perfil/ · recomendaciones/ · configuracion/
│       │       │   ├── components/ · navigation/ · theme/
│       │       ├── notifications/      # Scheduler, receivers, notificador
│       │       └── utils/
│       ├── test/               # Tests unitarios (JVM)
│       └── androidTest/        # Tests instrumentados
├── base_datos/                 # Copia de referencia de la base
├── tests_sql/                  # Verificaciones ejecutables sin Android
└── *.md                        # Documentación
```

---

## 🚀 Requisitos y compilación

### Requisitos
- **JDK 17** (Gradle 8.9 y Kotlin 2.0.21 no funcionan con JDK 23 o superior).
- **Android SDK** con `platforms;android-35` y `build-tools;35.0.0`, o Android Studio (Ladybug 2024.2+).
- Dispositivo o emulador con Android 8.0 (API 26) o superior.

### Con Android Studio
1. *File ▸ Open* y elegir la carpeta del proyecto.
2. Esperar la sincronización de Gradle.
3. Ejecutar en un emulador o teléfono (▶ Run).

### Por línea de comandos
```bash
# Indicar dónde está el SDK (una sola vez)
echo "sdk.dir=$HOME/android-sdk" > local.properties

# APK de depuración
./gradlew assembleDebug
# → app/build/outputs/apk/debug/app-debug.apk
```

### APK firmado para distribuir
*Build ▸ Generate Signed App Bundle / APK ▸ APK*, crear un keystore y elegir la variante `release`. **Guarda el keystore y su contraseña**: sin ellos no podrás publicar actualizaciones.

### Instalar el APK en un teléfono
Copiar el archivo al teléfono, abrirlo y permitir «Instalar apps desconocidas» para la aplicación que lo abre.

Guía completa, incluida la instalación del SDK por consola: [`MANUAL_INSTALACION_Y_USO.md`](MANUAL_INSTALACION_Y_USO.md).

---

## 🧪 Pruebas

| Tipo | Contenido | Cómo ejecutarlo |
|---|---|---|
| Verificación SQL (99 comprobaciones) | Esquema, conteos, integridad, fidelidad al PDF, búsqueda, restricciones | `python3 tests_sql/verificar_seed.py` |
| Revisión estática | Paquetes, llaves e imports internos de los `.kt` | `python3 tests_sql/verificar_estatico.py` |
| Unitarias (22) | Semáforo, formato de cantidades, cálculo del próximo recordatorio, fechas | `./gradlew test` |
| Instrumentadas (16) | Seed, consultas, menú, seguimiento, persistencia al reabrir, respaldo | `./gradlew connectedAndroidTest` |

Además hay una **lista de verificación manual** (modo avión, reinicio con recordatorios, TalkBack, letra grande, tablet) en [`INFORME_PRUEBAS.md`](INFORME_PRUEBAS.md).

---

## 🔒 Privacidad y seguridad

- Sin permiso de Internet: **ningún dato sale del dispositivo**.
- Sin cuentas, contraseñas ni analítica.
- `allowBackup="false"`: los datos no se copian a la nube de Android.
- El perfil nace vacío; la información personal del PDF **no** se importa.
- La copia de seguridad es un archivo que el usuario guarda donde decida.
- Permisos declarados: solo notificaciones (`POST_NOTIFICATIONS`) y reinicio (`RECEIVE_BOOT_COMPLETED`).

---

## 📑 Fidelidad al documento original

- No se inventaron comidas ni se modificaron cantidades, horarios o equivalencias.
- Los ingredientes del menú conservan el **texto literal** del documento.
- Cada alimento y comida lleva un campo `origen_dato`: `IMPRESO`, `MANUSCRITO`, `OCR` o `PENDIENTE_VERIFICACION`.
- Los datos ambiguos se muestran con el aviso **«Pendiente de verificación»** hasta que un profesional los confirme (por ejemplo, anotaciones manuscritas ilegibles, la mayonesa del sábado en la cena, el atún).
- Lo que el PDF no indica (por ejemplo, la cantidad diaria de cereales) queda como «sin cantidad indicada».

El inventario completo de lo extraído está en [`FASE1_inventario_pdf.md`](FASE1_inventario_pdf.md).

---

## 🚧 Estado y limitaciones

**Estado de verificación:** el esquema y los datos están verificados con pruebas ejecutadas (SQL). El código Kotlin fue revisado de forma estática; los tests unitarios e instrumentados están escritos, pero su ejecución y las pruebas manuales en dispositivo están **pendientes** de confirmar en cada entorno.

**Limitaciones conocidas**
- Las imágenes de los alimentos son **emojis por categoría**; faltan ilustraciones propias.
- La búsqueda distingue tildes («pure» no encuentra «puré»).
- El menú se consulta por día de la semana; no hay calendario por fechas.
- No hay editor ni importador de planes dentro de la app: actualizar el plan requiere publicar una nueva versión (la arquitectura lo permite mediante migraciones; ver `ARQUITECTURA.md`).
- Un solo perfil y un solo plan.
- Datos del PDF pendientes de validar con el profesional de nutrición.

**Ideas futuras:** editor/importador de planes, varios perfiles, calendario por fechas, búsqueda sin distinción de tildes, ilustraciones propias, exportación del seguimiento a CSV.

---

## 📚 Documentación adicional

| Archivo | Contenido |
|---|---|
| [`FASE1_inventario_pdf.md`](FASE1_inventario_pdf.md) | Extracción del PDF: datos impresos, manuscritos y dudas a validar |
| [`FASE2_schema_nutricion_diabetes.sql`](FASE2_schema_nutricion_diabetes.sql) | Script SQL de creación de tablas |
| [`ARQUITECTURA.md`](ARQUITECTURA.md) | Capas, flujo de datos, decisiones y extensibilidad |
| [`MANUAL_INSTALACION_Y_USO.md`](MANUAL_INSTALACION_Y_USO.md) | Compilación, instalación y uso |
| [`INFORME_PRUEBAS.md`](INFORME_PRUEBAS.md) | Pruebas ejecutadas, escritas y lista de verificación manual |
| [`LEEME_COMPILAR.md`](LEEME_COMPILAR.md) | Resumen rápido de compilación y estado |

---

> Proyecto de uso personal. El contenido nutricional pertenece al plan entregado por el profesional de salud y debe interpretarse siempre bajo su supervisión.
