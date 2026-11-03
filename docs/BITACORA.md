# Bitácora del port a la 200D

## 2026-11-03 — Fase 0 (entorno y auditoría) — puerta G0 aprobada

**Qué se hizo**
- Ubuntu 24.04 en WSL**1** (`Ubuntu-MFilm`, guardado en `D:\WSL\Ubuntu-MFilm`). WSL2 no se puede: la virtualización está apagada en el BIOS. WSL1 compila bien, solo más lento.
- Fork `Papomora/M-Film`, rama `port-200d`. Magic Lantern en el commit `ed3e7c0` (el mismo que usa M-Film). Copia limpia para consultar en `/root/ml/ml-stock`.
- Compilan: ML normal para 200D (313 088 bytes al arrancar, de 524 288 disponibles) y M-Film para EOS M (720 416 de 727 072). Simulador compilado y vistas previas guardadas como línea base en `/root/ml/phase0/baseline_previews`.
- Auditoría completa en `docs/AUDIT-200D.md`. Números y detalles de las builds en `docs/VERIFICACION.md`.

**Qué se probó**
- Solo compilación y simulador en el PC. Nada se ha probado en la cámara.

**Qué falta / decisiones pendientes**
- Prueba H0 en la cámara (Papo).
- Decisión de Papo (bloqueador B4 de la auditoría): ML en la 200D solo deja cambiar 3 propiedades. Sin ampliar esa lista, la película no puede cambiar estilo de imagen, balance de blancos ni ISO. Ampliarla es decisión de Papo (regla 2 de CLAUDE.md).
- Fase 1: arreglar B1 (`get_ae_value`), B2 (paleta falsa), B3 (búfer de `present`), B5 (tecla MENU = 0); empezar con `--no-darkroom` y probar también `--module`.
