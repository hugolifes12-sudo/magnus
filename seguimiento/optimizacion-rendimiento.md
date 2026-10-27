# Optimización y mejoras de rendimiento (app de escritorio + daemon)

## Objetivo

La app de escritorio se siente muy lenta en un equipo sin GPU NVIDIA (Intel Core Ultra 5 225U,
gráficos Intel integrados, 15.5 GB de RAM, Windows 11). Medir dónde se va el tiempo, arreglar
lo que más pese y dejar la app fluida en equipos modestos sin perder el diseño.

## Estado actual

- **Hecho:** revisión de código (solo lectura) y una primera medición. Ver *Hallazgos*.
- **En curso:** nada.
- **Sin empezar:** la medición completa (tarea 1) y todos los arreglos.

## Hallazgos

Cada hallazgo indica si está **medido** o es una **hipótesis** que sale de leer el código.
Ninguno está arreglado todavía.

### H1. Si el daemon no está corriendo, todo parece lento (medido)

Al revisar, ni `magnus.exe` ni `magnus_desktop.exe` estaban corriendo. Las peticiones a
`http://127.0.0.1:8420` fallaban a los ~2.0 s cada una: es lo que tarda Windows en dar por
rechazada una conexión local. Con `connectTimeout: 5 s` (`lib/core/network/dio_client.dart:12`),
cada pantalla puede esperar segundos antes de mostrar un error.

El daemon se cae si se cierra la ventana minimizada "Magnus daemon" que abre `Magnus.cmd`, o
si la app se abre directo desde `magnus_desktop.exe` en vez del acceso del menú Inicio.

Mejoras posibles:
- Que la app compruebe `/health` al arrancar y muestre un estado claro de "daemon apagado"
  con un botón para arrancarlo, en vez de esperar en cada pantalla.
- Que la app arranque el daemon ella misma (lo necesita también el MSIX para la Store),
  o que el daemon corra sin ventana de consola.
- Bajar `connectTimeout` a ~1–2 s para `127.0.0.1`: en local, si no conecta rápido, no va a
  conectar.

### H2. Animaciones que nunca paran (hipótesis)

Redibujan la pantalla a 60 fps aunque nadie toque nada:
- `AuroraBackdrop`, en `lib/shared/widgets/magnus_kit.dart:37` y `:45`: un `CustomPaint`
  con 3 manchas de degradado radial en bucle de 24 s. Se usa en `chat_page.dart`,
  `cloud_page.dart` y `dashboard_page.dart`. Tiene `RepaintBoundary`, así que no reconstruye
  widgets, pero sí repinta toda el área en cada frame.
- `SpotlightCard`, en `magnus_kit.dart:371` y `:387`: un giro en bucle de 6 s en cada
  tarjeta con `featured`.
- `magnus_kit.dart:309` (bucle de 800 ms) y `lib/shared/widgets/skeleton.dart:48`: hay que
  confirmar que solo corren mientras se ven.

Ya respetan "reducir movimiento" del sistema (`_reduceMotion`), pero eso viene apagado por
defecto.

Mejoras posibles: pausar las animaciones cuando la ventana no tiene foco o la página no está
visible; un ajuste "Rendimiento / efectos" en la app; o pintar la aurora una vez y dejarla
estática.

### H3. Desenfoque por tarjeta y ventana Acrylic (hipótesis, la más probable en gráficos Intel)

- `lib/shared/widgets/ui.dart:99`: cada tarjeta envuelve su contenido en
  `BackdropFilter(ImageFilter.blur(...))` cuando el tema tiene `glass: true`. Los temas
  Windows y Apple lo tienen activado (`lib/core/theme/magnus_theme.dart:222` y `:305`); el
  Material no (`:264`). Un desenfoque por tarjeta, encima de un fondo que se repinta cada
  frame (H2), obliga a recalcular todos los desenfoques en cada frame.
- `lib/main.dart:22`: `Window.setEffect(effect: WindowEffect.acrylic)`. El Acrylic de
  Windows también es caro en gráficos integrados.

Mejoras posibles: desactivar `glass` cuando la aurora está animando, o al revés; usar
`WindowEffect.mica`, que es mucho más barato que Acrylic en Windows 11, o ninguno; un modo
"ligero" automático cuando no hay GPU dedicada.

### H4. Sin GPU NVIDIA, el daemon dice que caben modelos de 0 B (medido)

`magnus platform` y `GET /platform` en este equipo devuelven `total_memory_gib: 0.0` y
`suggested_max_params_b: 0.0` (*"Tamaño sugerido (q4): hasta ~0.0 B"*). Sin GPU NVIDIA, el
perfil no usa la RAM del sistema (`core/platform.py`), así que el catálogo no recomienda nada,
aunque Ollama corre modelos pequeños en CPU.

Mejora posible: perfil de respaldo para CPU / GPU integrada que use parte de la RAM del
sistema y sugiera modelos de ~1–4 B.

### H5. Trabajo repetido en el daemon (hipótesis, poco impacto en este equipo)

- `detect_gpus()` (`core/hardware.py:187`) y `sample_system()` (`core/telemetry.py:158`)
  lanzan `nvidia-smi` en cada llamada cuando existe, y no guardan nada en caché. La página
  de recursos consulta `/system` cada 2 s (`system_cubit.dart:58`), y `/compatibility` con
  `target=auto` vuelve a detectar las GPUs en cada petición.
- En este equipo no hay `nvidia-smi` y `shutil.which` sale rápido, así que aquí no explica
  la lentitud. En equipos NVIDIA sí cuesta (~100–300 ms por llamada; esa cifra no está medida).

Mejora posible: cachear `detect_gpus()` unos segundos. La parte estática (modelo, VRAM
total) puede quedar en caché toda la vida del proceso.

### H6. El daemon congelado tarda en arrancar (hipótesis)

El instalador (`deploy/windows/build-installer.ps1`, rama `feat/windows-installer`) congela
el daemon con PyInstaller en modo carpeta e incluye todos los submódulos de `core`, `daemon`,
`cli`, `cloud`, `uvicorn` y `keyring`. No se ha medido cuánto tarda en responder `/health`
desde que se abre el acceso directo.

Mejora posible: medirlo; importar en diferido lo pesado (`cryptography`, `keyring`, RAG)
dentro de las funciones que lo usan; no cargar `cloud` en el daemon local.

## Tareas

| # | Tarea | Estado | Notas |
|---|---|---|---|
| 1 | Medir: tiempo hasta ver la ventana, hasta `/health` OK, CPU/GPU de la app en reposo 30 s, latencia de cada endpoint | Sin empezar | Con `flutter run --profile` y DevTools para ver frames |
| 2 | H1: estado "daemon apagado" en la app + `connectTimeout` corto | Sin empezar | |
| 3 | H2/H3: modo ligero (sin aurora animada, sin blur por tarjeta, Mica en vez de Acrylic) | Sin empezar | Comparar con la medición 1 antes y después |
| 4 | H4: perfil de CPU / GPU integrada en `core/platform.py` + tests | Sin empezar | |
| 5 | H5: caché de `detect_gpus()` | Sin empezar | |
| 6 | H6: medir y reducir el arranque del daemon congelado | Sin empezar | |
| 7 | Que la app arranque el daemon (necesario también para el MSIX de la Store) | Sin empezar | Decisión de diseño: hablarla antes |

## Decisiones y restricciones

- Medir antes y después de cada cambio. Ningún arreglo sin un número que lo respalde.
- No quitar el diseño: lo caro se apaga en modo ligero o en equipos sin GPU dedicada, y queda
  como opción en Ajustes.
- Cada arreglo va en su propio PR contra `HugoLifes/magnus`.

## Archivos / puntos de entrada

- App: `clients/desktop/lib/main.dart`, `lib/shared/widgets/magnus_kit.dart`,
  `lib/shared/widgets/ui.dart`, `lib/shared/widgets/skeleton.dart`,
  `lib/core/theme/magnus_theme.dart`, `lib/core/network/dio_client.dart`,
  `lib/features/resources/presentation/cubit/system_cubit.dart`
- Daemon: `core/platform.py`, `core/hardware.py`, `core/telemetry.py`, `daemon/app.py`
- Instalador: `deploy/windows/build-installer.ps1`, `deploy/windows/Magnus.cmd`
  (rama `feat/windows-installer`)

## Validación

- Comandos corridos:
  - `curl` con tiempos a `/health`, `/platform`, `/skills`, `/agents`, `/models`,
    `/models/local`, `/models/loaded`, `/catalog` y `/providers`.
  - `magnus platform`.
- Resultados:
  - Con el daemon apagado, todas las peticiones fallaron a los ~2.0 s.
  - Con el daemon arriba (en sesiones anteriores), `/health`, `/skills`, `/agents` y
    `/models/local` respondieron bien; no se cronometraron.
  - `platform` reporta 0.0 GiB y 0.0 B.
- Sin correr todavía: la medición de la tarea 1 y los tests (`pytest`, `flutter test`).

## Siguiente paso exacto

Tarea 1. Arrancar Magnus desde el menú Inicio, medir el tiempo hasta `/health` OK y el uso de
CPU de `magnus_desktop.exe` en reposo durante 30 s. Después, con
`flutter run --profile` en `clients/desktop`, abrir DevTools → Performance en el dashboard y
anotar los ms por frame con y sin `AuroraBackdrop` y `glass`.

## Log

- 2026-10-27: revisión inicial y primeras mediciones (H1, H4). Documento creado.
