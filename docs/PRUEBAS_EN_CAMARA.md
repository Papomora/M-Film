# Pruebas en la cámara (para Papo)

Antes de cualquier prueba:
- Batería **cargada al 100 %**.
- Una tarjeta SD **solo para pruebas** (8 GB o más), formateada **en la cámara**.
- Otra tarjeta normal sin Magic Lantern, para volver a la normalidad.
- Firmware Canon **1.0.1** (MENU → última pestaña amarilla → *Firmware Ver.*).
- Dial en **M**.

Si la cámara se congela o la luz de la tarjeta queda encendida o parpadea sin parar: **apaga, saca la batería 10 segundos**, pon la tarjeta normal y enciende. Guarda la tarjeta de pruebas tal cual y envía el archivo `ML/FILMROLL.LOG` y cualquier archivo de log o crash de la carpeta `ML` (si existe).

Envía siempre: fotos de la pantalla (con el celular), el archivo de log y una frase de lo que pasó.

---

## H0 — Magic Lantern normal (antes de M-Film)

1. Descarga el build oficial de ML para 200D (builds.magiclantern.fm/200D.html).
2. Copia `autoexec.bin`, `ML-SETUP.FIR` y la carpeta `ML` a la raíz de la tarjeta de pruebas.
3. MENU → *Firmware Ver.* → SET → sigue las instrucciones. Apaga al terminar.
4. Enciende. Abre el menú de ML (botón papelera o el que indique la pantalla).
5. Ve a *Debug → Free Memory*. **Foto de esa pantalla.**
6. Con ML apagado o con la tarjeta normal: configura *Calidad* = **RAW + L**, *Espacio de color* = **sRGB**. Toma 10 fotos variadas (día, interior con bombillo, noche con luces, una cara, cielo). Envía los `.CR2` y `.JPG`.

## H1 — Primera pantalla de M-Film

1. Formatea la tarjeta de pruebas en la cámara. Copia los archivos que te entregue Claude Code.
2. MENU → *Firmware Ver.* → SET (instala). Apaga y enciende.
3. ¿Arranca? ¿Ves la pantalla de M-Film? **Foto.**
4. Ve a *System → Screen colour* (carta de colores). **Foto bien de frente, sin reflejos.**
5. Medio disparo para salir. ¿Vuelve a la pantalla de Canon? Enciende y apaga 3 veces.
6. Envía fotos y `ML/FILMROLL.LOG`.

## H2 — Botones

1. Presiona **cada botón una vez, en este orden**, esperando 2 segundos entre cada uno: arriba, abajo, izquierda, derecha, SET, dial superior a la derecha (1 clic), dial a la izquierda (1 clic), Q, INFO, MENU, reproducción, papelera, medio disparo. Anota el orden si te equivocas.
2. Recorre la repisa, cambia de película, entra a Home y a System.
3. Envía `ML/FILMROLL.LOG` y una nota de qué botón no hizo nada o hizo algo raro.

## H3 — Un rollo completo

1. Calidad **RAW + L**, sRGB, dial en P, Av o M.
2. Carga un rollo de **12**. Dispara las 12.
3. ¿El contador baja? ¿Se bloquea el obturador al final? **Foto de la pantalla al final del rollo.**
4. Rebobina y carga otro rollo. Dispara 3, **apaga**, enciende: ¿sigue en el frame 3?
5. Envía log y fotos.

## H4 — Revelado en cámara

1. Termina un rollo de 12 y entra a *Darkroom*. Revela.
2. Anota cuánto tarda por foto (aprox.).
3. Mira las copias en *Prints*. **Foto de la hoja de contactos.**
4. Envía 3 JPEG revelados de la carpeta del revelado y el log.

## H5 — Uso real

Un día normal de fotos: dos rollos de 36. Anota cualquier congelamiento, `Err`, pantalla rara o batería que se gasta muy rápido. Envía logs.
