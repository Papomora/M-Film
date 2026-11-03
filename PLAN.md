# Plan de desarrollo — M-Film en la Canon SL2 (200D)

Meta: llevar la 200D en `ml-dev/cameras.txt` de `no` a `testing` y luego a `working`, con repisa de películas, rollo, visor (HUD) y revelado en cámara, sin romper ninguna otra cámara.

Cada fase termina en una **puerta (gate)**. Las puertas marcadas **[CÁMARA]** solo las puede pasar Papo probando en su SL2. Claude Code prepara la build, los pasos y los logs; Papo prueba y devuelve evidencia.

## Agentes (definidos en `.claude/agents/`)

| Agente | Rol | Escribe código |
|---|---|---|
| `ml-auditor` | Lee ML y M-Film, encuentra qué falta en la 200D. Solo lectura. | No |
| `port-engineer` | Capa de port: `filmroll.c`, `apply_fork.py`, `cameras.txt`, teclas, estado de la cámara. | Sí |
| `display-engineer` | Ruta de pantalla RGBA: paleta de 256 colores, refresco, rendimiento. | Sí |
| `darkroom-engineer` | Revelado: CR2 de la 200D en `fos_dev.c`, matriz de color, memoria. | Sí |
| `build-verifier` | Compila 200D y EOS M, mide tamaño de arranque, corre simulador y pruebas. | No (solo scripts de prueba) |
| `code-reviewer` | Revisa cada cambio sin haberlo escrito. Seguridad, regresiones, reglas de `CLAUDE.md`. | No |

Orquestación: la sesión principal reparte trabajo, nunca deja que un agente revise su propio código, y corre `build-verifier` + `code-reviewer` antes de cada puerta. Los agentes que tocan archivos distintos pueden trabajar en paralelo (usar `isolation: worktree` cuando dos agentes escriben a la vez).

---

## Fase 0 — Entorno y auditoría (sin cámara)

Agentes: `build-verifier`, `ml-auditor` (en paralelo).

1. Instalar toolchain (seguir `ml-dev/setup_wsl.sh`), clonar ambos repos, crear rama `port-200d` en el fork de M-Film.
2. Compilar **ML stock** para `200D.101` y **M-Film para `EOSM.202`**. Ambas deben salir limpias: prueba de que el entorno sirve.
3. Compilar el simulador (`make -C ml-dev/filmos/sim`) y generar las vistas previas.
4. `ml-auditor` produce `docs/AUDIT-200D.md`:
   - Cada función, variable, propiedad, constante y `CONFIG_*`/`FEATURE_*` que usa `ml-dev/fork/src/filmroll.c` (y lo que `apply_fork.py` parchea en `src/gui-common.c`, `src/zebra.c`, `src/menu.c`), con su estado en la 200D: **existe / falta stub / existe pero distinto / desconocido**, citando archivo y línea.
   - Los códigos de tecla de la 200D (`platform/200D.101/gui.h`) frente a los que usa M-Film (`MODULE_KEY_*`).
   - Riesgos de Thumb-2 y de la ruta `FEATURE_VRAM_RGBA`.

**Puerta G0:** ML stock 200D y M-Film EOS M compilan con cero errores; `AUDIT-200D.md` sin filas "desconocido" que bloqueen la Fase 1.

**Puerta H0 [CÁMARA]:** Papo instala el build oficial de ML para 200D (firmware 1.0.1), confirma que arranca y envía: foto de *Debug → Free Memory*, `autoexec.bin` usado, y 5 a 10 fotos **RAW + JPEG L** de su SL2 (en sRGB) para las pruebas del revelado. Ver `docs/PRUEBAS_EN_CAMARA.md`, prueba H0.

## Fase 1 — Integración de la build (sin cámara)

Agentes: `port-engineer`, luego `build-verifier`, `code-reviewer`.

1. `apply_fork.py` acepta `200D.101`: `#define FEATURE_FILMROLL` en `features.h`, objetos en el `Makefile` de la plataforma.
2. Stubs o `#ifdef` para todo lo marcado "falta" en la auditoría, sin inventar direcciones de firmware: si una función no tiene stub conocido, se desactiva esa función de M-Film en la 200D y se documenta.
3. Primero `--no-darkroom`. Medir y anotar el tamaño de arranque (`_bss_end - RESTARTSTART`).
4. Agregar modo de diagnóstico: con `filmroll.debug` encendido, escribir en `ML/FILMROLL.LOG` cada tecla recibida, el modo de pantalla y la memoria libre.

**Puerta G1:** `build_fork.sh 200D.101 --no-darkroom` y `build_fork.sh EOSM.202` compilan con `FATAL_WARNINGS=y`; tamaño anotado en `ml-dev/MEMORY.md`; revisión aprobada.

## Fase 2 — Pantalla RGBA

Agentes: `display-engineer`, luego `build-verifier`, `code-reviewer`.

1. Bajo `FEATURE_VRAM_RGBA`: tabla de 256 entradas RGBA propia de M-Film, usada por la conversión de `src/bmp.c` mientras el Film OS está en pantalla, y la tabla original de ML (80 entradas) cuando no. Ningún cambio de comportamiento para cámaras sin `FEATURE_VRAM_RGBA`.
2. `fos_port_palette()` llena la tabla desde `fos_pal`; `palette_canon()` la devuelve a ML. Nada de escribir en registros de paleta de Canon en la 200D.
3. `fos_port_present()` marca el refresco (`ml_refresh_display_needed` o equivalente) para que la tarea de redibujo convierta solo cuando hay cambios.
4. Medir en el simulador o con un benchmark el costo del bucle de conversión 720×480; si es lento, convertir solo las filas sucias (`y0..y1`).
5. Pantalla de prueba: *System → Screen colour* con la carta de colores.

**Puerta G2:** compila, revisión aprobada, simulador sin cambios en las vistas previas de EOS M.

**Puerta H1 [CÁMARA]:** arranca, aparece el Film OS, la carta de colores se ve correcta (foto), salir y volver a la pantalla de Canon funciona. Prueba H1.

## Fase 3 — Botones y navegación

Agentes: `port-engineer`, `build-verifier`, `code-reviewer`.

1. Con el `FILMROLL.LOG` de H1 (teclas reales), mapear los botones de la 200D: cruceta, SET, dial superior, MENU, INFO, Q, reproducción, papelera, medio disparo. Medio disparo = salir (como en M-Film). Pantalla táctil solo si la auditoría muestra eventos disponibles.
2. Repisa, Home y menús completos.

**Puerta H2 [CÁMARA]:** recorrer repisa, cambiar de película, Home, System, salir con medio disparo. Prueba H2.

## Fase 4 — Rollo, disparo y HUD

Agentes: `port-engineer`, `build-verifier`, `code-reviewer`.

1. Detección de pantalla de disparo con LiveView apagado y encendido (`shooting_screen()`), numeración de archivos, tarjeta SD, objetivo, apertura, ISO de la película, bloqueo al terminar el rollo, sonido/LED si existen en la 200D.
2. Todo lo que no tenga soporte en ML 200D se desactiva con `#ifdef CONFIG_200D` y se anota en la documentación.

**Puerta H3 [CÁMARA]:** cargar rollo de 12, disparar 12 (RAW + JPEG L), ver contador y bloqueo, rebobinar, cargar otro, apagar y encender a mitad de rollo (el rollo debe conservarse). Prueba H3.

## Fase 5 — Revelado en cámara

Agentes: `darkroom-engineer`, `build-verifier`, `code-reviewer`.

1. Con los CR2 de H0: leer dimensiones, bordes, slices y nivel de negro del archivo; agregar `CONFIG_200D` a las tablas de `fos_dev.c` (matriz y nivel de blanco tomados de LibRaw `colordata.cpp`, entrada de la EOS 200D / Rebel SL2 / Kiss X9, citando versión).
2. `ml-dev/filmos/sim/devtest.c` revela los CR2 en el PC: cero muestras erróneas y comparación visual contra LibRaw/dcraw.
3. Build completa (con revelado). Si supera el límite de arranque medido, usar `--no-darkroom` en la 200D y dejar el revelado a la app de escritorio, o mover el Film OS a módulo cargado tras el arranque (ver final de `MEMORY.md`); preguntar a Papo antes de elegir.

**Puerta G5:** `devtest` limpio con todos los CR2 de Papo; revisión aprobada.

**Puerta H4 [CÁMARA]:** revelar en la cámara un rollo de 12; enviar 3 JPEG revelados y el log. Prueba H4.

## Fase 6 — Endurecimiento y publicación

Agentes: todos; `code-reviewer` hace una revisión final completa.

1. Pruebas largas: rollo de 36, dos rollos seguidos, batería baja, sacar tarjeta con la cámara apagada, apagar durante el revelado.
2. `cameras.txt`: `200D.101 testing` (y `working` solo con H1–H4 aprobadas), regenerar tablas con `python3 ml-dev/tools/cameras.py`, actualizar `PORTING.md` con la ruta RGBA (sirve para otras DIGIC 6/7).
3. Pull request en inglés a `HNGSeries/M-Film` con lo probado, firmware y evidencia. Si se tocó ML (`src/bmp.c`), proponer el cambio también al equipo de Magic Lantern.

**Puerta H5 [CÁMARA]:** sesión real de un día de uso sin congelamientos. Si hay `Err 70` o congelamiento, sacar batería, guardar logs, volver a la fase que corresponda.

---

## Reglas de avance

- Si una puerta falla: diagnosticar con los logs, arreglar, volver a pasar `build-verifier` y `code-reviewer`, repetir la prueba. No saltar fases.
- Las puertas [CÁMARA] se esperan: Claude Code no continúa a la siguiente fase sin el reporte de Papo.
- Cada fase deja un commit por agente con mensaje claro en inglés, y una nota en `docs/BITACORA.md` (español) con qué se hizo, qué se probó y qué falta.
