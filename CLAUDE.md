# M-Film para Canon EOS 200D / Rebel SL2 — instrucciones del proyecto

Este proyecto porta **M-Film** (https://github.com/HNGSeries/M-Film, GPL-2.0) a la **Canon EOS 200D (Rebel SL2, Kiss X9), firmware Canon 1.0.1**, usando **magiclantern_simplified** (https://github.com/reticulatedpines/magiclantern_simplified), plataforma `200D.101`.

El dueño del proyecto es Papo (español, inglés básico). Habla con él en **español**, frases cortas. Código, comentarios, commits, issues y pull requests hacia los repositorios de origen van en **inglés**, siguiendo el estilo de cada repositorio.

Lee `PLAN.md` antes de cualquier trabajo. Ahí están las fases, los agentes y las puertas de control.

## Lo que ya sabemos (verificado leyendo el código, 2026-11-01)

- M-Film está separado en dos partes:
  - `ml-dev/filmos/` — las pantallas (Film OS). Dibujan en un lienzo propio de 720×480 de **8 bits indexados** y solo hablan con la cámara por las funciones `fos_port_*` declaradas al final de `ml-dev/filmos/fos.h`. Esta parte casi no cambia.
  - `ml-dev/fork/` — la capa de Magic Lantern: `fork/src/filmroll.c` (implementa `fos_port_*`) y `fork/apply_fork.py` (parchea ML por plataforma). Esta parte se rehace por cámara.
- `ml-dev/cameras.txt` marca `200D.101` como `no` porque ML es experimental en DIGIC 6+.
- **Pantalla en la 200D (el cambio más grande):** `platform/200D.101/features.h` define `FEATURE_VRAM_RGBA`. ML dibuja en un búfer indexado de 8 bits (`bmp_vram_indexed`) y una tarea (`refresh_yuv_from_rgb_task` en `src/bmp.c`) lo convierte a RGBA con `indexed2rgb()`. La tabla `indexed2rgbLUT` tiene solo `RGB_LUT_SIZE = 80` entradas (`src/bmp.h`); los índices ≥ 80 salen grises. M-Film usa 254 colores y los carga escribiendo Y/Cb/Cr en la paleta de Canon (`palette_ours()` / `palette_canon()` en `filmroll.c`). **Ese truco no existe en la 200D**: hay que dar a M-Film su propia tabla de 256 colores RGBA para la conversión y pedir el refresco tras cada `fos_port_present`.
- **Arranque y memoria:** la 200D es DIGIC 7, arranca por `src/boot-d678*.c`, carga ML dentro del pool AllocateMemory (`CONFIG_ALLOCATE_MEMORY_POOL`), `RESTARTSTART = 0x4d0000` en `platform/200D.101/Makefile`. El límite de tamaño de `autoexec.bin` para la 200D **no se conoce**; ver `ml-dev/MEMORY.md` para cómo se midió en otras cámaras. Existe la opción `--no-darkroom` (~60 KB menos).
- **Código:** la 200D ejecuta Thumb-2 (ARMv7). Cuidado con código que asuma ARM puro.
- **Revelado en cámara:** `fos_dev.c` elige matriz de color y nivel de blanco por `CONFIG_<cámara>` (datos de LibRaw `colordata.cpp`). La 200D no está en esa tabla. La 200D guarda CR2 (no CR3).
- ML 200D define `FEATURE_PICSTYLE`, `CONFIG_LIVEVIEW`, `CONFIG_PROP_REQUEST_CHANGE`, `FEATURE_INTERVALOMETER`, `FEATURE_GLOBAL_DRAW`, `CONFIG_NO_BFNT` (sin fuente en ROM), `CONFIG_NO_DISPLAY_CALLS`. Muchas funciones que existen en cámaras DIGIC 4/5 pueden faltar: **hay que auditar cada símbolo que usa `filmroll.c`** antes de portar.

## Reglas que no se rompen

1. **Nunca** escribir en la ROM ni en la flash de la cámara, ni tocar archivos de firmware Canon (`.FIR` de Canon). Solo se toca lo que va en la tarjeta.
2. **Nunca** cambiar propiedades de Canon (`prop_request_change`) que no estén en `platform/200D.101/property_whitelist.h`. Si hace falta una nueva, se para y se pregunta a Papo.
3. Todo cambio específico de la 200D va detrás de `#ifdef CONFIG_200D` o `#ifdef FEATURE_VRAM_RGBA`, o en su propio archivo de port. **La build de EOS M (`EOSM.202`) debe seguir compilando igual** después de cada cambio.
4. Nada se declara "funciona en la cámara" sin un reporte de Papo con fotos o el archivo `ML/FILMROLL.LOG`. Lo que compila o pasa en el simulador se marca como "compila" o "pasa en simulador", nunca como "funciona".
5. Compilar siempre con `FATAL_WARNINGS=y`. Cero errores y cero advertencias nuevas.
6. No subir a ningún repositorio: firmware de Canon, ROM dumps, fotos de Papo, LUTs sin licencia para compartir.
7. Mantener GPL-2.0 en todo lo derivado. Créditos a HNGSeries y al equipo de Magic Lantern.
8. Cada cambio pasa por el agente `code-reviewer` antes de llegar a Papo para probar en la cámara.
9. Si una puerta de control (gate) de `PLAN.md` falla, no se avanza a la siguiente fase. Se arregla o se pregunta.

## Entorno

- Linux o WSL2 (Ubuntu). Python 3. Toolchain `arm-none-eabi-gcc` (M-Film documenta gcc 13.2; ver `ml-dev/setup_wsl.sh`).
- Repos esperados:
  - `~/ml/magiclantern_simplified` (variable `ML_SRC` si está en otro sitio)
  - `~/ml/M-Film` (fork de Papo de HNGSeries/M-Film, rama `port-200d`)
- Build: `ml-dev/build_fork.sh 200D.101` (y `--no-darkroom`). Simulador: `make -C ml-dev/filmos/sim`.

## Cómo reportar a Papo

Al cerrar cada fase: qué se hizo (3–5 líneas), qué debe probar él en la cámara (pasos numerados de `docs/PRUEBAS_EN_CAMARA.md`), qué archivo debe copiar a la tarjeta y qué debe enviar de vuelta (fotos de la pantalla, `ML/FILMROLL.LOG`, logs de crash de ML si existen).
