# AUDIT-200D: what M-Film needs from Magic Lantern on the Canon EOS 200D (FW 1.0.1, `200D.101`)

Audit date 2026-11-03. Auditor: `ml-auditor` (read-only; no source file was edited).

## Resumen para Papo (español)

- Revisé cada función y constante que usa M-Film (`filmroll.c`, `filmhook.c` y los parches de `apply_fork.py`) contra Magic Lantern para la 200D.
- **Buena noticia:** los parches de `apply_fork.py` encuentran todos sus puntos de anclaje en el ML de la 200D. Casi todas las funciones existen. El tamaño de ML cabe: ML normal usa 313,088 bytes de 524,288 disponibles.
- **Problemas que hay que arreglar antes de probar en la cámara:** (1) falta `get_ae_value()` y el enlace falla; (2) la paleta de Canon (`LCD_Palette`) en la 200D es una dirección falsa, no se puede tocar; (3) `fos_port_present` escribe en el búfer equivocado (RGBA en vez del indexado); (4) el botón MENU de la 200D no se conoce (ML pone 0); (5) ML bloquea el cambio de estilo de imagen, balance de blancos e ISO (solo permite 3 propiedades), así que `apply_film()` no hará nada sin tu permiso.
- La pantalla RGBA sí se puede resolver: hay un punto claro en `src/bmp.c` donde enchufar una tabla de 256 colores y hay que avisar al refresco después de cada `present`.
- Cosas que solo se sabrán en la cámara (marcadas "unknown"): si las propiedades de Canon (modo de disparo, LiveView, calidad) llegan bien, si ML reconoce la pantalla de disparo, y cuáles son los códigos de algunas teclas (MENU, soltar medio disparo).

## Conventions

- `S` = `/root/ml/ml-stock` (stock magiclantern_simplified, commit `ed3e7c0df`, `git status` clean). `P` = `S/platform/200D.101`. `F` = `ml-dev/fork/src/filmroll.c`. `AF` = `ml-dev/fork/apply_fork.py`.
- `nm` = `arm-none-eabi-nm S/platform/200D.101/build/magiclantern`. A stock (unpatched) 200D build is present in that tree (timestamp 2026-11-03 10:19); it gives link-level evidence ("T" = defined in ML, "A" = absolute address from `stubs.S`, "ABSENT" = not linked). Do NOT point `ML_SRC` at `ml-stock`: `build_fork.sh` runs `make clean` there and would delete that evidence build.
- Status values: **exists**, **missing stub**, **exists but different**, **unknown**. "exists" means present in the 200D build/headers; it does not mean verified on a camera (nothing in this audit was run on a camera).
- No firmware address is proposed anywhere in this document. Where an address would be needed, the row says "needs reversing" or "unknown".

Method: read `filmroll.c`, `filmroll.h`, `filmhook.c`, `filmos_mod.h`, `apply_fork.py`, `build_fork.sh`; checked every ML symbol in `S/src`, `S/platform/200D.101/{stubs.S,consts.h,features.h,internals.h,gui.h,property_whitelist.h,Makefile,function_overrides.c}` and in the stock 200D link map (`nm`). I did not compile M-Film for the 200D (not my task); warnings under `FATAL_WARNINGS=y` are therefore unknown.

---

## 1. Symbol table

### 1.1 Build flags and configuration

| # | symbol | used where | status | evidence | proposed handling |
|---|---|---|---|---|---|
| 1 | FEATURE_FILMROLL | AF:129 (added to features.h); gui-common.c / zebra.c / menu.c hooks | exists | `P/features.h` ends at line 74 with `#undef`; AF appends the define after `rstrip` | none; AF handles it |
| 2 | FEATURE_FILMOS_MODULE | AF:139; `filmroll.h:45`; menu.c patch | exists | same mechanism as #1 | none |
| 3 | FEATURE_VRAM_RGBA | not tested in F (F assumes a classic indexed vram); PLAN phase 2 | exists | `P/features.h:1` | guard palette code and `fos_port_present` with it (section 3) |
| 4 | FEATURE_GLOBAL_DRAW | zebra.c anchor (`get_global_draw`) | exists | `P/features.h:51`; `S/src/zebra.c:409-411` | none |
| 5 | CONFIG_200D / CONFIG_200D_101 / CONFIG_FW_VERSION | `fos_dev.c` per-camera tables (not F) | exists | `S/platform/Makefile:52-57` (`-DCONFIG_$(MODEL)=1`); `P/Makefile:75-76` | `fos_dev.c` has no `CONFIG_200D` branch (falls to EOS M defaults): darkroom, phase 5 only |
| 6 | CONFIG_EOSM | `F:145` (FOS_LV_ONLY), `fos_hud.c:271` (top plate) | exists | not defined on 200D (`P/features.h`, `P/internals.h`) so `FOS_LV_ONLY=0`, plate HUD compiled, film OS opens without LiveView | intended; note extra code size |
| 7 | CONFIG_DIGIC_VII / 78X / 678X | `struct fio_file` layout used at `F:796-804` | exists | `P/internals.h:198`; `S/src/internals-common.h:29,33`; `S/src/fio-ml.h:88-98` (78X layout: `mode` first, `name[]` after 24 bytes of other fields) | none |
| 8 | CONFIG_DIGIC_VI | `bmp.c` Ximr branch | exists | not defined on 200D, so `refresh_yuv_from_rgb` takes `winsys_sem` around `XimrExe` (`S/src/bmp.c:262-267`) | none |
| 9 | CONFIG_MODULES | `build_fork.sh --module` | exists | `S/platform/Makefile:29,235-243`; 21 `.mo` in `P/build/modules/` | none |
| 10 | CONFIG_ALLOCATE_MEMORY_POOL | boot path | exists | `P/internals.h:210`; `S/src/boot-d678-am.c:18` (`#error` if missing) | none |
| 11 | CONFIG_PROP_REQUEST_CHANGE | all `prop_request_change*` use in F | exists but different | defined `P/features.h:29`, but D678 also needs a per-property whitelist: `S/src/property.c:342-399`, `P/property_whitelist.h:267-275` (allows only PROP_ICU_AUTO_POWEROFF, PROP_REMOTE_SW1, PROP_REMOTE_SW2) | see rows 41-49: writes are silently dropped |
| 12 | FEATURE_PICSTYLE | `lens_set_contrast/...` | exists | `P/features.h:47` | none |
| 13 | CONFIG_NO_DEDICATED_MOVIE_MODE | `is_movie_mode()` (F:152) | exists but different | `P/internals.h:224`; `S/src/propvalues.c:152-166` makes movie mode = `lv_movie_select == LVMS_ENABLE_MOVIE` | confirm in the field (state dump, Phase 1.4) |
| 14 | CONFIG_NO_BFNT | ML text via `NotifyBox`; F loads own `FOSFONT.BIN` (F:1375) | exists | `P/internals.h:213`; `S/src/rbf_font.c:30,605,647` | none; M-Film fonts come from the card |
| 15 | CONFIG_BEEP | `beep()` / `beep_times()` (F:379, 407) | exists but different | not defined on 200D; `S/src/beep.c:15` guard, dummy stubs at the end of the file (`void beep(){}` ...). `nm`: `beep` 0x4ecf22, `beep_times` 0x4ecf26 (4 bytes apart) | beep is a silent no-op on 200D. Sound = not available; document it |
| 16 | CONFIG_MENU_WITH_AV | gui-common.c key chain | exists | not defined on 200D | none |

### 1.2 Tasks, time, memory

| # | symbol | used where | status | evidence | proposed handling |
|---|---|---|---|---|---|
| 17 | INIT_FUNC / TASK_CREATE / TASK_LOOP | F:1463, 1494-1495 | exists | `S/src/tasks.h:361,372,382`; same pattern used by `bmp.c:113` on 200D | none |
| 18 | task_create | F:606 (module: filmos_mod.h:92) | exists | `P/stubs.S:41`; `nm` A 0xdf008ce7 | none. Task count margin on 200D is unknown (`CONFIG_INCREASE_MAX_TASKS` not defined; `S/src/boot-d678-am.c:188-196`): two new tasks (`film_main`, `film_os`) |
| 19 | msleep | many | exists | `P/stubs.S:43`; `P/consts.h:95` MIN_MSLEEP 11 | none |
| 20 | get_ms_clock | F:128, 401, 408, ... | exists | `S/src/shoot.c:289`; `nm` T 0x4ea784 | none |
| 21 | ml_started | F:69, 1452 | exists | `S/src/init.c:59`; `nm` B | none |
| 22 | ml_shutdown_requested | F:654 | exists | `S/src/tasks.h:380`; `nm` B | none |
| 23 | CancelDateTimer | F:70, 532 | exists | `P/stubs.S:161`; `nm` A 0xe05f9db5 | none |
| 24 | malloc / free | F:1381, 1406, 827 | exists | `S/src/mem.h:89-90` (`__mem_malloc`, `__mem_free`; `nm` T) | none |
| 25 | tmp_malloc | F:820 (`fos_port_alloc`) | exists | `S/src/mem.h:105` | darkroom only needs large buffers; unknown how much is free (needs H0 "Free Memory" photo) |
| 26 | fio_malloc / fio_free | F:751, 244, 1391 | exists | `S/src/mem.h:110-111` (MEM_DMA) | none |
| 27 | CACHEABLE / UNCACHEABLE | F:760, 767-769 | exists but different | non-VxWorks macro uses bit `0x40000000` (`S/src/mem_defs.h:13-14`); not previously exercised by any 200D-specific code in this tree | unknown on D7: confirm by a read test at H1. Needed only for `fos_port_read` (card reads) |
| 28 | ml_reserved_mem / boot memory | build_fork.sh limit check | exists | `P/consts.h:184`; `S/src/boot-d678-am.c:184` | see section 4 |

### 1.3 Display, GUI, drawing

| # | symbol | used where | status | evidence | proposed handling |
|---|---|---|---|---|---|
| 29 | LCD_Palette | F:489, 507-508, 516-517, 635, 694-698 | exists but different | `P/stubs.S:418-420`: `NSTUB(0x56500000, LCD_Palette)`, comment "fake address in safe writable region, may not exist on 200D (doesn't seem to use indexed colour anywhere)". `nm`: A 0x56500000 | **do not read or write**. Guard `palette_ours`, `palette_canon`, the `shamem_read` probe (F:635) and the palette log (F:693-698) with `#ifndef FEATURE_VRAM_RGBA`; replace by the RGBA table (section 3) |
| 30 | EngDrvOut | F:507-508, 516-517 | exists but different | `S/src/debug.c:1944-1949`: reads `MEM(0xC0400008)` (a DIGIC 4/5 register; not known valid on DIGIC 7) then calls `_EngDrvOut`, which on 200D is an empty function (`P/function_overrides.c:138-165`; the real stub is commented out `P/stubs.S:115`). Stock `palette_disable` returns early on RGBA (`S/src/bmp.c:1379-1381`) so this path never runs in stock 200D ML | remove from RGBA build (same guard as #29) |
| 31 | shamem_read | F:635 | exists | `P/stubs.S:91` (`THUMB_FN(0x37b08, shamem_read)`); `nm` A 0x37b09 | only used on the palette probe: removed with #29 |
| 32 | COLOR_WHITE / COLOR_EMPTY / COLOR_BLACK / COLOR_RED / COLOR_DARK_RED / COLOR_GRAY(i) | F:489, 1015-1021 | exists | `S/src/bmp.h:346-370`; ML RGBA LUT covers 0..79 (`bmp.c:296-340`); `COLOR_GRAY(100)` = 79 | HUD colours stay inside the ML table; fine |
| 33 | BMPPITCH | F:683, 1064-1067, 1072 | exists but different | 960 (`S/src/bmp.h:169`), not 720; F already uses it as the pitch, good. The buffer returned by `bmp_vram()` already includes the 720x480 offset (`S/src/bmp.c:123-135`: `bmp_vram_indexed + BMP_HDMI_OFFSET`) | none |
| 34 | bmp_vram | F:1062, 1072 | exists | `S/src/bmp.c:119-140`; RGBA branch returns the malloc'd indexed buffer (518,400 B = 960x540 in `bmp_init`, `bmp.c:1472-1479`) | use this for all drawing |
| 35 | bmp_vram_real | F:677 (`fos_port_present`) | exists but different | `S/src/bmp.c:~70` returns `BMP_VRAM_START(bmp_vram_raw()) + BMP_HDMI_OFFSET`, and under RGBA `bmp_vram_raw()` is `rgb_vram_info->bitmap_data` (`S/src/bmp.h:76-79`): the real **32-bit BGRA** layer, not the indexed buffer | **bug on 200D**: `memcpy(vram + y*BMPPITCH, rows, 720)` writes 8-bit data into a 32-bit buffer. Use `bmp_vram()` and then request a refresh |
| 36 | DISPLAY_IS_ON | F:486, 654, 678 | exists | `P/consts.h:39` (`!(char)MEM(0x486d)`, "medium confidence") | verify in the state dump |
| 37 | display_is_on() | F:999 | exists | `S/src/gui-common.c:706`; `nm` T | same as #36 |
| 38 | display_idle() | F:152, 992, 1000, 1150, 1259, 1471 | exists but different | `S/src/shoot.c:83-93`: with LiveView off it needs `gui_state == GUISTATE_IDLE`, no menu, and the Canon dialog handler == `ShootOlcApp_handler` (stub `P/stubs.S:336`), or display off with GUI mode 0. LiveView branch calls `liveview_display_idle` (`nm` T) | unknown whether it is true on the 200D's normal shooting screen: state dump (Phase 1.4) decides |
| 39 | lv | F:152, 530, 541, ... | exists | `S/src/propvalues.h:37`; handler on `PROP_LV_ACTION` `propvalues.c:238-245` (generic ID 0x80050022, `property.h:67`) | unknown: the property ID itself is unverified on 200D (works well enough for stock LiveView menus) |
| 40 | is_movie_mode() | F:152, 156, 992, 1000 | exists but different | see #13; `nm` T 0x4e7340 | unknown behaviour until tested |
| 41 | shooting_mode / SHOOTMODE_AV | F:156, 1270 | exists | `S/src/propvalues.c:126-139` (PROP_SHOOTING_MODE_2 0x80000001); `S/src/property.h:235` (Av = 2) | unknown if 200D reports Av = 2 (wheel-as-aperture on manual lenses, F:1270) |
| 42 | CURRENT_GUI_MODE | F:156, 662, 668 | exists | `P/consts.h:16` (`*(int*)0x6624`, "high confidence") | none |
| 43 | GUIMODE_ML_MENU | F:529, 623, 662 | exists but different | `P/consts.h:24`: `(lv ? 0x4E : GUIMODE_PLAY)`. With LiveView off this is Canon PLAY mode (GUIMODE_PLAY = 2, `consts.h:17`): film OS on the non-LV shooting screen puts Canon in playback while it is up | same as ML's own menu behaviour on 200D; test at H1 that leaving does not leave Canon in playback |
| 44 | SetGUIRequestMode | F:529, 542, 623 | exists | `P/stubs.S:152`; `nm` A 0xe04a7ef5 | none |
| 45 | canon_gui_disable_front_buffer / canon_gui_enable_front_buffer | F:531, 544, 1115, 1130 | exists but different | `S/src/dialog_test.c:44-67`; works through `WINSYS_BMP_DIRTY_BIT_NEG` which on 200D is `MEM(0x56500000+0x30)` (`P/consts.h:107`, "wrong, no idea ... probably safe"). So it is a flag in a harmless place; it does **not** stop Canon drawing | on 200D Canon keeps drawing its GUI into the same RGBA layer; film OS covers it by overwriting opaque pixels. HUD "Canon's screen held" logic (F:1096-1134) will not work as on other DSLRs |
| 46 | clrscr | F:533, 545, 1116 | exists but different | `S/src/bmp.c:726`; under RGBA it clears the **indexed** buffer; pixels already converted to the RGBA layer stay until Canon redraws (the converter skips LUT value 0, `bmp.c:229-236`) | after clearing, call `redraw()` as ML does (`menu.c:5712-5718`) or add a full-refresh mode (section 3) |
| 47 | bmp_on / bmp_off | F:534, 547, 530, 541 | exists but different | `S/src/bmp.c:1406-1424`; `palette_disable` returns immediately on RGBA (`bmp.c:1379-1381`): these only flip flags | none |
| 48 | redraw | F:546 | exists | `nm` T 0x4f4bf8 (posts MLEV_REDRAW; `_redraw_do` T 0x4f50ac) | none |
| 49 | NotifyBox / NotifyBoxHide | F:172, 295, 527, 600 | exists | `S/src/notify_box.h:4-5`; `nm` T | draws via ML fonts into the indexed buffer; fine (limited to LUT < 80) |
| 50 | ml_refresh_display_needed | not used by F today (needed, section 3) | exists | `S/src/bmp.h:38`, `bmp.c:116`, set in `bmp_fill` (`bmp.c:585`), `rbf_font.c:372`, `menu.c:4878,5826`, `compositor.c:57`, `init.c:349`; `nm` B 0x514894 | set to 1 after every `fos_port_present` and HUD draw/clear |

### 1.4 Card, files

| # | symbol | used where | status | evidence | proposed handling |
|---|---|---|---|---|---|
| 51 | FIO_CreateFileOrAppend | F:134, 169 | exists | `S/src/fio-ml.c:680`; `nm` T | none |
| 52 | my_fprintf | F:137, 175 | exists | `S/src/fio-ml.h:188`; `nm` T 0x4d8370 | none |
| 53 | FIO_CloseFile | F:138, 177, 788 | exists | `P/stubs.S:130` | none |
| 54 | FIO_OpenFile / FIO_CreateFile | F:736 | exists | `S/src/fio-ml.c:370, 615`; stubs `P/stubs.S:125-126` | none |
| 55 | FIO_ReadFile / FIO_WriteFile | F:761-767, 778 | exists | `S/src/fio-ml.c:630, 658`; stubs `P/stubs.S:127-128` | none |
| 56 | FIO_SeekSkipFile | F:783 | exists | `P/stubs.S:129`; `S/src/fio-ml.h:160` returns `int64_t` | compare as `int64_t` (F:783 compares to `int`; fine) |
| 57 | FIO_FindFirstEx / FIO_FindNextEx / FIO_FindClose | F:797-808 | exists | `S/src/fio-ml.c:418`; stubs `P/stubs.S:133,135,136` | none |
| 58 | struct fio_file (.name, .mode), struct fio_dirent, FIO_MAX_PATH_LENGTH, ATTR_DIRECTORY, IS_ERROR | F:793-804 | exists | `S/src/fio-ml.h:81-98`, `fio-ml.h:84` (0x80), `mem.h:140` | none |
| 59 | FIO_CreateDirectory | F:814 | exists | `S/src/fio-ml.c:472`; stub `P/stubs.S:131` | none |
| 60 | is_file / is_dir | F:342, 813-815 | exists | `S/src/fio-ml.c:771, 777` | none |
| 61 | read_entire_file | F:208, 1375, 1400 | exists | `S/src/fio-ml.c:863` | none |
| 62 | FIO_GetFileSize_direct | `filmos_mod.h:45` | exists | `S/src/fio-ml.c:532`; stub `_FIO_GetFileSize` `P/stubs.S:137` | none |
| 63 | O_RDONLY / O_SYNC / SEEK_SET | F:736, 783 | exists | `S/src/fio-ml.h:53, 63, 69` | none |
| 64 | get_shooting_card (+ .drive_letter, .folder_number, .file_number) | F:162, 334, 347, 422, 435, 730 | exists | `S/src/fio-ml.c:60`; `S/src/fio-ml.h:11-23` | unknown: whether Canon's file/folder counters (`file_number`) update on the 200D (the roll logic depends on them); H3 test |
| 65 | get_dcim_dir_suffix | F:163 | exists | `S/src/fio-ml.c:325` | none |
| 66 | get_file_prefix | F:391 | exists | `S/src/fileprefix.c:22` | unknown: depends on prefix property on 200D; H3 test |
| 67 | LoadCalendarFromRTC / struct tm | F:182, 936 | exists | wrapper `P/function_overrides.c:60-63` calling stub `_LoadCalendarFromRTC` `P/stubs.S:354`; `S/src/dryos.h:173` | none |
| 68 | snprintf / vsnprintf / strchr / strncmp / atoi / memcpy / memset / strcat / strlen | F:122-139, many | exists | `nm` T/A (vsnprintf `P/stubs.S:198`) | none |

### 1.5 Camera state, lens, properties

| # | symbol | used where | status | evidence | proposed handling |
|---|---|---|---|---|---|
| 69 | prop_request_change_wait | F:262 | exists but different | `S/src/property.c:410-430`; calls `prop_request_change`, which on D678 returns without writing unless the ID is whitelisted (`property.c:381-393`). Then it waits the full timeout (1000 ms at F:262) for an ack that never comes | needs owner decision (CLAUDE.md rule 2). Until then `apply_film()` must not call it on 200D |
| 70 | PROP_PICTURE_STYLE | F:262 | exists but different | `S/src/property.h:289` (0x80000028, same ID for every camera but the 5DC); NOT in `prop_write_allow` (`P/property_whitelist.h:267-275`) | blocked; ask Papo before whitelisting |
| 71 | lens_set_contrast / sharpness / saturation / color_tone | F:266-275 | exists but different | `S/src/lens.c:2337-2358` macro writes `PROP_PICSTYLE_SETTINGS(i)` (24 bytes); needs `lens_info.picstyle` index from `get_prop_picstyle_index` (`S/src/picstyle.c:26`); `NUM_PICSTYLES 10` is "guess" (`P/consts.h:83`); property not whitelisted | blocked; also returns silently if `lens_info.picstyle == 0` |
| 72 | PS_MONO | F:270 | exists | defined by M-Film in `ml-dev/filmos/fos.h:32` (0x86), not by ML | none |
| 73 | lens_set_kelvin | F:278 | exists but different | `S/src/lens.c:1786-1805`: writes PROP_WB_MODE_LV, PROP_WB_KELVIN_LV, PROP_WB_MODE_PH, PROP_WB_KELVIN_PH (`property.h:256-261`) | blocked (not whitelisted) |
| 74 | lens_set_wbs_gm / lens_set_wbs_ba | F:279-280 | exists but different | `S/src/lens.c:2776, 2782`; PROP_WBS_GM / PROP_WBS_BA (`property.h:274-275`) | blocked |
| 75 | lens_set_rawiso | F:281 | exists but different | `S/src/lens.c:2740`; `prop_set_rawiso` writes PROP_ISO (`lens.c:2479-2485`, ID `property.h:148`); `FEATURE_EXPO_OVERRIDE` is not defined on 200D so no BV path | blocked |
| 76 | pic_quality / PICQ_RAW | F:291 | exists | `S/src/propvalues.c:88-90`, `property.h:494` (0x4060000) | unknown: whether 200D's RAW+JPEG value matches the mask (H3 test via state dump) |
| 77 | image_review_time | F:292 | exists | `S/src/propvalues.c:104`; `property.h:516` (0x02020006) | unknown on 200D |
| 78 | lens_info (.raw_aperture, .aperture, .name, .raw_shutter) | F:304, 313, 950, 1027 | exists | `S/src/lens.h:37-94` (name[32] line 40, raw_aperture 68, raw_shutter 69); `nm` D 0x50b2f0 | unknown: aperture/shutter values depend on generic props (works for lens ML menu). Manual-lens logic (`raw_aperture == 0`) unverified |
| 79 | lens_format_shutter | F:1032 | exists | `S/src/lens.c:583`; `nm` T | none |
| 80 | **get_ae_value** | F:1050, 1163 | **missing stub** | `S/src/lens.c:2868-2869` defines it only `#ifdef AE_VALUE`; 200D `consts.h` defines no `AE_VALUE`. `nm`: ABSENT. Declared unconditionally in `S/src/lens.h:753` | **link error.** Wrap both uses in `#ifdef AE_VALUE`, else `h.meter = 0` (needle hidden). Real value needs reversing; do not guess |
| 81 | job_state_ready_to_take_pic | F:1471 | exists | `S/src/lens.c:2021-2025` (PROP_LAST_JOB_STATE 0x80030012) | none |
| 82 | get_halfshutter_pressed | F:656, 1161 | exists | `S/src/debug.c:1846-1849`; `HALFSHUTTER_PRESSED` `P/consts.h:10` (0x5354, "high confidence") | none |
| 83 | camera_model | F:966 | exists | `S/src/propvalues.c:14-16, 54-57` (PROP_CAM_MODEL 0x2) | none |
| 84 | beep / beep_times | see #15 | exists but different | no-op | |
| 85 | powersave_prohibit / powersave_permit | F:913-914, 588 | exists | `S/src/powersave.c:19, 26` | none |
| 86 | idle_wakeup_reset_counters | F:632 | exists | `S/src/powersave.c:216` | none |
| 87 | fake_simple_button | F:591 | exists but different | `S/src/gui-common.c:693`; F:591 sends `BGMT_MENU`, which is 0 on 200D (`P/gui.h:124`) = a Canon event of param 0 | guard `#if BGMT_MENU` or drop `AFTER_CANON_MENU` on 200D |
| 88 | CONFIG_INT | F:73-100 | exists | `S/src/config.h:102` | none |
| 89 | module_translate_key / MODULE_KEY_PORTABLE | F:1311 | exists but different | `S/src/module.c:1209` (first match wins: `BGMT_MENU`=0 comes before `BGMT_JOY_CENTER`=0, so a Canon event with param 0 is read as MENU, line order `module.c:1210-1216, 1223`); `module.h:111` | see section 2 |
| 90 | event->param / event->arg / struct event | F:1230, 1311, 1334 | exists | `S/src/gui.c:67-80`: only `event->type == 0` events reach the key chain | touch events never arrive |
| 91 | handle_module_keys | AF anchor | exists | `S/src/gui-common.c:450-451` (weak alias) and real `module.c` (`nm` T) | none |

### 1.6 Module API (only for `--module`)

| # | symbol | used where | status | evidence | proposed handling |
|---|---|---|---|---|---|
| 92 | MODULE_FUNCTION / MODULE_SYMBOL | `filmhook.c:16-18` | exists | `S/src/module.h:333-355` | none |
| 93 | MODULE_CONFIGS_START / MODULE_CONFIG / MODULE_CONFIGS_END, `module_config_t`, `__module_config_filmos` | `filmos_mod.h:8-35, 61` | unknown | macros live in `module.h`, not opened; not exercised unless `--module` | verify at module build |
| 94 | get_config_dir | `filmos_mod.h:44, 46` | exists | `S/src/config.c:611`; `nm` T | none |
| 95 | MODULE_INFO_START / MODULE_INIT / MODULE_DEINIT | `filmos_mod.h:118-121` | exists | used by stock 200D modules (`P/build/modules/*.mo`) | none |
| 96 | symbols the module imports (`fos_*` pulls libm/libgcc) | `build_fork.sh:75-88` | unknown | check is done by script against `P/build/modules/200D_101.sym` (file exists) | can only be answered by building the module |

### 1.7 Anchors patched by `apply_fork.py`

All anchors were checked textually against `S` (read-only).

| # | anchor | AF line | status | evidence | proposed handling |
|---|---|---|---|---|---|
| 97 | `    if (handle_module_keys(event) == 0) return 0;` in `src/gui-common.c` | AF:39-45 | exists | `S/src/gui-common.c:508` (one occurrence, inside `handle_common_events_by_feature`, after Q equivalence and long-press handling) | none |
| 98 | `#include <zebra.h>` in gui-common.c | AF:46-49 | exists | `S/src/gui-common.c:10` | none |
| 99 | `int get_global_draw() // menu setting, or off if \n{\n` in zebra.c | AF:52-58 | exists | `S/src/zebra.c:409-410` | none. Side effect: `zebra_should_run()` (`zebra.c:3618-3624`) becomes false whenever film mode is built in, which selects the "skip transparent pixels" refresh path (section 3) |
| 100 | `#include "zebra.h"` in zebra.c | AF:59-62 | exists | `S/src/zebra.c:25` | none |
| 101 | `static void menu_open() \n{ \n    if (menu_shown)\n        return;\n` in menu.c (note trailing spaces) | AF:66-78 | exists | `S/src/menu.c:5649-5653` | none. Built-in mode means ML's menu can never open on 200D (see blocker B7) |
| 102 | `#include "powersave.h"` in menu.c | AF:79-82 | exists | `S/src/menu.c:41` | none |
| 103 | `    img_name \\\n    lua \\\n` in `modules/Makefile` | AF:85-88 | exists | `S/modules/Makefile:48-49` | none |
| 104 | `#if defined(CONFIG_6D) \|\| defined(CONFIG_100D) \|\| defined(CONFIG_70D)` in `src/boot-d45-am.c` | AF:101-108 | exists | `S/src/boot-d45-am.c:127` (applies regardless of platform; not used by 200D) | none |
| 105 | `include ../Makefile` anchor in platform Makefile | AF:171-175 | exists | `P/Makefile:103`; the existing `ML_SRC_EXTRA_OBJS += .../function_overrides.o` line (`P/Makefile:99`) does not match the removal filter (`filmroll`, `fos_`, `filmhook`) so it survives | none |
| 106 | `platform/200D.101/modules.included` | AF:182-188 | exists | has `bench dual_iso file_man mlv_lite`, no `lua` | none |
| 107 | platform `features.h` append | AF:127-131 | exists | last line `#undef CONFIG_AUTOBACKUP_ROM` | none |

---

## 2. Key codes: `P/gui.h` versus what M-Film uses

M-Film reads portable `MODULE_KEY_*` through `module_translate_key()` (`S/src/module.c:1209-1246`), which maps a Canon `BGMT_*` code to a portable one, and in two places raw `BGMT_*` (`F:591`, `F:1314-1323`). A key that the camera header leaves undefined becomes `-1` and is never produced (`module.c:1090-1180`). Events reach M-Film only if `event->type == 0` (`S/src/gui.c:70-72`); touch events are not passed. Hook order: `handle_Q_button_equiv` → `handle_longpress_events` → `handle_filmroll_keys` (`gui-common.c:490-508`).

| M-Film use (filmroll.c) | BGMT source | 200D value (`P/gui.h`) | status | note |
|---|---|---|---|---|
| MODULE_KEY_PRESS_HALFSHUTTER (1188, 1262, 1302) | BGMT_PRESS_HALFSHUTTER | 0x47 (line 146) | exists | |
| MODULE_KEY_UNPRESS_HALFSHUTTER (1193, 1262) | BGMT_UNPRESS_HALFSHUTTER | 0x48 derived `+1` (`S/src/gui-common.h:30`); `gui.h:147` carries "FIXME just this one button is defined in another file" | unknown | resolve by logging events |
| MODULE_KEY_PRESS_FULLSHUTTER / UNPRESS (1189, 1194, 1302) | BGMT_PRESS/UNPRESS_FULLSHUTTER | 0x49 / 0x4A derived (`gui-common.h:31-32`) | unknown | |
| MODULE_KEY_REC (1190) | BGMT_REC | not defined | missing stub | never fires; abort still works via half shutter |
| MODULE_KEY_WHEEL_LEFT / RIGHT (1197-1198, 1271) | BGMT_WHEEL_LEFT / RIGHT | 2 / 3 (158-159) | exists | event may be grouped: `event->arg` (F:1230, 1334) |
| MODULE_KEY_WHEEL_UP / DOWN (1199-1200) | BGMT_WHEEL_UP / DOWN | 77 / 88, "wrong (this wheel doesn't exist on 200D)" (160-162) | exists but different | dummy IDs; harmless unless Canon uses 77/88 for something real (unknown) |
| MODULE_KEY_PRESS_LEFT / RIGHT / UP / DOWN (1202-1205, 1273) | BGMT_PRESS_* | 0x28 / 0x26 / 0x2a / 0x2c (111-118) | exists | |
| MODULE_KEY_UNPRESS_UDLR (1206) | BGMT_UNPRESS_UDLR | 0x15 (136) | exists | |
| BGMT_UNPRESS_LEFT/RIGHT/UP/DOWN (1314-1323, `#ifdef`) | per-direction release | 0x29 / 0x27 / 0x2b / 0x2d (112-118) | exists | both per-direction and UDLR releases are defined: unknown which one the 200D sends, M-Film pushes both kinds of release (harmless but verify) |
| MODULE_KEY_PRESS_SET (1213, 1285, 1295) | BGMT_PRESS_SET | 0x4 (119) | exists | |
| MODULE_KEY_UNPRESS_SET (1216) | BGMT_UNPRESS_SET | 0x5 (120) | exists | |
| MODULE_KEY_Q (1214) | BGMT_Q | 0x24 (139) = BGMT_PRESS_AV (141) | exists but different | "use AV for ML submenus": on the 200D the portable Q key is the **Av/exposure-comp button**, not the Q button. `BGMT_Q_SET` 0x1d (138) is separate |
| MODULE_KEY_JOY_CENTER (1215) | BGMT_JOY_CENTER | 0x0 "wrong, avoiding conflicting definition" (154) | exists but different | no joystick on 200D; value 0 collides with MENU |
| MODULE_KEY_MENU (1217) | BGMT_MENU | 0x0 "very unsure, 0x62 maybe?" (124) | unknown | `module_translate_key` maps a Canon event with param 0 to MENU. Note 0x62 is `GMT_OLC_INFO_CHANGED` (173), not obviously a MENU press. Resolve with the event spy (`FEATURE_SHOW_GUI_EVENTS`, `P/features.h:39`) or the Phase 1.4 key log. Without a real MENU code, "MENU" cannot be used inside the film OS |
| MODULE_KEY_INFO (1218) | BGMT_INFO | 0x7 (125) | exists | |
| MODULE_KEY_PLAY (1219, 1251) | BGMT_PLAY | 0xb (122) | exists | |
| MODULE_KEY_TRASH (1220, 1239) | BGMT_TRASH | 0xd (123) | exists | this is also ML's own "open menu" key on a long press (`S/src/menu.c:6240-6250`); M-Film opens Home on press |
| MODULE_KEY_TOUCH_2_FINGER (1221, 1239) | BGMT_TOUCH_2_FINGER | not defined | missing stub | the SL2 has a touch screen but ML 200D delivers no touch events (`gui.c:70-72`) |
| `fake_simple_button(BGMT_MENU)` (591) | BGMT_MENU | 0 | exists but different | would send Canon event 0; guard it |

Buttons the 200D has that M-Film does not use (available for Phase 3): `BGMT_LV` 0x1e, `BGMT_FUNC` 0x12, `BGMT_PICSTYLE` 0x13, `BGMT_PRESS_DISP` 0x8 / 0x9, zoom in 0xe/0xf and zoom out 0x10/0x11, `BGMT_PRESS_AV` 0x24 / 0x25, `BGMT_Q_SET` 0x1d.

---

## 3. FEATURE_VRAM_RGBA display path

### 3.1 What the stock code does (`S/src/bmp.c`, `bmp.h`)

- ML draws into an 8-bit indexed buffer `bmp_vram_indexed`: `malloc(BMP_VRAM_SIZE)` = 960 x 540 = 518,400 bytes, set to `COLOR_TRANSPARENT_BLACK` (3) (`bmp.c:1472-1479`). `bmp_vram()` returns it plus `BMP_HDMI_OFFSET` (`bmp.c:123-135`), so a 720x480 canvas with pitch 960 (`BMPPITCH`, `bmp.h:169`). Everything in M-Film that draws through `bmp_vram()` (HUD, plate) already lands in the right buffer.
- `indexed2rgbLUT` is **`static`**, 80 entries, `RGB_LUT_SIZE 80` (`bmp.h:81`, `bmp.c:296-340`). Index 0 and 3 are `0x00000000` (transparent). Indices ≥ 80 return `indexed2rgbLUT[4]` = `0xffa33800`, a brownish orange, not grey (`bmp.c:343-353`). The `PLAN.md`/`CLAUDE.md` sentence "los índices ≥ 80 salen grises" is slightly off.
- Conversion `refresh_yuv_from_rgb()` (`bmp.c:194-270`): copies **all** 960x540 pixels, one call to `indexed2rgb()` per pixel (non-inline, not yet benchmarked: "SJE FIXME benchmark this loop"), into `rgb_vram_info->bitmap_data` (Canon's 32-bit BGRA layer, `_rgb_vram_info` `P/stubs.S:402`). Then `take_semaphore(winsys_sem)` + `XimrExe(XIMR_CONTEXT)` + `give_semaphore` (`bmp.c:262-267`, because `CONFIG_DIGIC_VI` is not defined on DIGIC 7). `XIMR_CONTEXT` is `0xa09a0` (`P/consts.h:220`); `XimrExe` stub `P/stubs.S:359`; `winsys_sem` `P/stubs.S:217`.
- Two branches (`bmp.c:206-253`): if `zebra_should_run()`, every pixel is written (so ML can erase). Otherwise a pixel is skipped when the test `(rgb && 0xff000000) == 0` is true. That test uses `&&` (logical), not `&`, so it is true only when `rgb == 0` exactly: a LUT value with alpha 0 but non-zero colour (for example `0x00000001`) **is** copied and therefore erases. Transparent index 0 / 3 never erases.
- When it runs: task `redraw_task` (`TASK_CREATE`, priority 0x1e, stack 0x1000, `bmp.c:113`; loop `bmp.c:278-293`): every 50 ms (max 20 fps), only if `ml_refresh_display_needed && !ml_shutdown_requested && DISPLAY_IS_ON`. The flag is set by ML's own primitives (`bmp_fill` `bmp.c:585`, `rbf_font.c:372`, menu code, compositor). **Direct memcpy into `bmp_vram()` (what M-Film does) does not set it.** The ML menu forces an immediate `refresh_yuv_from_rgb()` in `menu_redraw_flood` (`menu.c:5598-5601`).
- `bmp_on` / `bmp_off` only set flags; `palette_disable()` returns at once under RGBA (`bmp.c:1379-1381`).
- `screenshot.c:375` also calls `indexed2rgb()`.

### 3.2 Interaction with M-Film

| M-Film piece | what happens on 200D today |
|---|---|
| `palette_ours()` / `palette_canon()` (F:484-519) | write `EngDrvOut(LCD_Palette[...])` against a fake address and a no-op `_EngDrvOut`; they never change what the screen shows. They also read `LCD_Palette[3*COLOR_WHITE+2]` to learn the opaque flag and white luma. Must be replaced, not just disabled |
| `fos_port_sleep` probe (F:635-640) | `shamem_read(LCD_Palette[3*200])` reads fake memory at `0x56500000 + 3*200*4` and then uses that value as the address for `shamem_read`: a read of an arbitrary address, likely a data abort. Remove |
| `fos_port_palette()` / `fos_port_photo()` / `fos_port_set(FS_SCR_*)` (F:686-713, 893-897) | all end in `palette_ours()`: they are the right places to build the 256-entry RGBA table |
| `fos_port_present()` (F:675-684) | writes to `bmp_vram_real()`, the BGRA layer (bug, #35). And never requests a refresh |
| `fos_screen_yuv(...)` / `fos_screen_yuv_rgb(...)` (called F:498, 504) | return Y/Cb/Cr entries for a LUT. For RGBA an RGB variant is needed (the "Screen colour" page: Canon vs Rec.709, saturation, warmth, brightness, would apply to RGB). Needs a new function in `fos_gfx.c`; unknown cost |
| HUD / plate (`fos_hud_draw`, `fos_hud_clear`, `fos_hud_plate_clear`, F:1062-1075, 1141) | draw into `bmp_vram()` (right buffer) but no refresh flag; clearing writes index 0, which is skipped by the converter, so the HUD stays on the Canon layer until Canon repaints |
| `film_ui_close()` (F:538-548) | `clrscr(); redraw();` mirrors `menu_close` (`menu.c:5712-5718`): the Canon GUI is repainted by `redraw()`; fine |
| `filmroll_overlays_active()` returning 1 | forces `zebra_should_run()` false, so the converter is always in "skip transparent" mode while film mode is built in |

### 3.3 Hook points for a 256-entry table (no firmware addresses needed)

Preferred option: patch `bmp.c` from `apply_fork.py`, `#ifdef FEATURE_FILMROLL`, anchors present in stock:

1. `bmp.c:296` `static uint32_t indexed2rgbLUT[RGB_LUT_SIZE]` and `bmp.c:343` `uint32_t indexed2rgb(uint8_t color)`. Add a pointer `const uint32_t *fos_rgb_lut` (NULL = ML's own 80-entry behaviour). `fos_port_palette()` fills a static `uint32_t fos_lut[256]` (entry = `0xFF000000 | rgb`, entry 0 = `0`, plus one "erase" entry such as `0x00000001`) and sets the pointer; `film_ui_close()` sets it to NULL. Cheap and keeps the HUD (ML colours) unchanged outside the film OS.
2. Better for speed: in the two copy loops (`bmp.c:206-253`) take `const uint32_t *lut = fos_rgb_lut` once and index directly, avoiding 518,400 function calls per refresh. Dirty rows: copy only `y0..y1` (rows 30..510 of the 960x540 buffer) instead of all 540 rows.
3. Refresh request: after `fos_port_present` and after every HUD draw/clear, `ml_refresh_display_needed = 1` (declared `bmp.h:38`). For animations the 50 ms sleep caps the rate at 20 fps; if that is too low, call `refresh_yuv_from_rgb()` directly from the film OS task as `menu_redraw_flood` does (`menu.c:5598-5601`), under the same conditions the task checks.
4. Erasing: either add `|| fos_full_refresh` to the `if(zebra_should_run())` test at `bmp.c:~207` (full copy, ML erases), or use a non-zero alpha-0 LUT entry for "clear" (works with the existing `&&` quirk, but depends on that quirk not being "fixed").
5. In module mode the table and the flag live in the core (`filmhook.c` gets one more shim function) because `bmp.c` is core code.

Alternative without touching `bmp.c`: convert in `fos_port_present` directly into `rgb_vram_info->bitmap_data` and call `XimrExe`. Not recommended: ML's `redraw_task` would later overwrite it from the indexed buffer whenever any ML primitive sets the flag (NotifyBox, any `bmp_printf`).

Costs unknown (need a benchmark on the camera): conversion time for 960x540 at 1 call/pixel; cache effects (the RGBA layer is written by CPU and read by the Ximr unit; stock code does no cache clean there, `marv_dcache_clean` is available `P/stubs.S:369` but unused by `refresh_yuv_from_rgb`).

---

## 4. Boot path

- 200D uses `ML_BOOT_OBJ = boot-d678-am.o` (`P/Makefile:92`), not `boot-d678.c`. `RESTARTSTART = 0x4d0000` (`P/Makefile:89`, aligned so the MMU tables before it pack well). `boot-d678.c` is the older scheme that reserves `_bss_end - user_mem_start` (`S/src/boot-d678.c:216-223, 246-247`); it is not built for 200D.
- `boot-d678-am.c` requires `CONFIG_MMU_REMAP` (`P/features.h:56`) and explicit `CONFIG_ALLOCATE_MEMORY_POOL` (`P/internals.h:210`, `boot-d678-am.c:10-19`). It sanity-checks `RESTARTSTART` against the AllocMem start: `RESTARTSTART < orig_AM_start || RESTARTSTART > orig_AM_start + 0x2000000` fails (`boot-d678-am.c:174-182`). `PTR_ALLOC_MEM_START 0xe00412c8` (`P/consts.h:183`) is the location that holds the AllocateMemory start; the comment there gives it as `0x46c000`.
- **What limits `autoexec.bin`:** ML is copied to `RESTARTSTART` and the AllocateMemory start is moved to `RESTARTSTART + ALLOC_MEM_STOLEN` = `0x4d0000 + 0x80000` = `0x550000` (`P/consts.h:184`; `S/src/patch_mmu.c:708-725`; `boot-d678-am.c:184`: `ml_reserved_mem = ALLOC_MEM_STOLEN`). So code + data + **bss** (`_bss_end - RESTARTSTART`) must fit in **524,288 bytes**. There is no build-time assertion (`S/src/magiclantern.lds.S` has none: only `_bss_end = .` at line 123); overflow would be silent heap corruption, so `build_fork.sh` must enforce it.
- **Measured on the stock 200D build in `S`:** `_text_start` 0x4d0000, `_bss_start` 0x50e900, `_bss_end` 0x51c700 → **313,088 bytes** used (`nm`); `arm-none-eabi-size`: text 219,431, data 36,809, bss 56,832; `autoexec.bin` 259,584 bytes. Headroom to the 512 KB region: **211,200 bytes**.
- M-Film adds (from `ml-dev/MEMORY.md:71-78`): the film OS is about 180 KB of code and data built in, of which the darkroom is about 60 KB, so `--no-darkroom` should add roughly 120 KB on DIGIC 4/5. That is an estimate from other cameras (gcc 13.2, different CPU mode: Thumb-2 on 200D instead of ARM, so smaller code is likely). On this arithmetic even the full build would fit (313,088 + ~180,000 = ~493,000 < 524,288) but with little spare, and extra bss (`keyq`, `photo_pal`, `recent`, `hud_pal`, the 256-entry table) is not counted. **unknown until built**; measure with `build_fork.sh` and write the figure into `ml-dev/MEMORY.md` (Phase 1.3).
- `cameras.txt:36` has `-` for the 200D limit, so `build_fork.sh` prints "no limit known" (line ~76). A hard ceiling exists in code (524,288); suggest putting it in the column as a fail-safe (not "known to start", which only the camera can prove).
- `ALLOC_MEM_STOLEN` can be raised in `P/consts.h` (more ML space, less AllocMem pool). Effect and alignment constraints for D7 are **unknown** (needs a camera test; the pool is ~9 MB per the comment at `P/consts.h:178-183`). Not recommended before an H0 "Free Memory" photo.
- Other boot facts: Canon's bootloader loads `autoexec.bin` at `AUTOEXEC_BASE 0x800000` (`P/Makefile:81-82`); whether it limits the *file* size is **unknown**.
- **`--module` alternative:** `build_fork.sh` supports it (lines 17-19, 70-111): `apply_fork.py --module` puts only `filmhook` in autoexec (`AF:156-157`, ~190 KB less per MEMORY.md) and builds `ML/modules/filmos.mo` per camera (`ml-dev/fork/module/Makefile`). Facts for 200D: modules are compiled `-march=armv5te` (ARM state; `S/modules/Makefile:20,30`) with `-mlong-calls -DMODULE` (`modules/Makefile:158-163`) while autoexec is `armv7-a -mthumb -mlong-calls` (`P/Makefile:96-97`); `readelf -A mlv_lite.mo` shows `Tag_CPU_arch: v5TE`. The module loader needs `P/build/modules/200D_101.sym` (present; 35,284-byte `magiclantern.sym` too). Module memory comes from Canon's heap after boot; free size on the 200D is unknown (needs H0). With `FEATURE_FILMOS_MODULE`, `menu_open` falls back to ML's menu if the module is not loaded (`AF:66-78`): that makes `--module` the safer first camera test. One gotcha in `build_fork.sh:96-100`: it compares module symbols against `200D_101.sym`; any `fos_*` import ML does not export fails the build.

---

## 5. Thumb-2 risks

Searched `filmroll.c`, `filmhook.c`, `filmos_mod.h`, `filmos/fos*.c|h` for `asm`, `__asm`, `naked`, hard-coded function pointers, `MEM(`, cache calls, absolute addresses.

1. **No inline assembly, no naked functions, no hard-coded firmware addresses or casts of integers to function pointers** in `filmroll.c`, `filmhook.c`, `filmos_mod.h`, `fos*.c`. Constants matching `0x[0-9a-f]{6,}` in `fos_gfx.c` are colours; `fos_dev.c` ones are file-format values. Result: no ARM-only code written by hand.
2. 200D builds with `-march=armv7-a -mthumb -mlong-calls` (`P/Makefile:96-97`), so `filmroll.c` and all `fos_*.c` compile as Thumb-2 in built-in mode. In `--module` mode the same files compile as **armv5te ARM state** (`modules/Makefile:20`). Behaviour differences to expect: speed, and alignment code generation (below).
3. `fos_dev.c:844-873, 862`: `typedef uint32_t __attribute__((may_alias)) word;` plus 4-word unrolled reads (`next_ff`). With `-mthumb armv7-a` GCC may emit `LDM/LDRD` for such loads; those fault on unaligned addresses (plain `LDR` is fine). Only in the darkroom (not in `--no-darkroom`). **unknown**: whether the pointer is always 4-aligned. Resolve by `-mno-unaligned-access` for `fos_dev.c` or by an alignment assertion.
4. Floating point: `fos_ui.c` (springs, easing), `fos_dev.c` (matrices) use `float`. 200D has no `-mfpu` flag in the platform CFLAGS (`S/Makefile.globals:56-81` has none), so it is soft-float via libgcc/libm (`ML_LIBC` includes `gcc-libgcc.a newlib-libm.a`, `Makefile.globals:54`). Slower, not a correctness risk. In module mode `fork/module/Makefile` links `fos-libm.a` (errno remapped to `fos_errno`).
5. Function pointers passed to firmware: `task_create(..., ui_task, ...)` (F:606), `TASK_CREATE(filmroll_task ...)` (F:1495) are normal C pointers (Thumb bit set by the compiler, same pattern as `bmp.c:113`). Pointers into the core from a module (`MODULE_FUNCTION`, `filmhook.c:16-18`) resolve to ARM functions (LSB 0) called from Thumb code via `blx reg` on ARMv7, which interworks correctly; relies on `-mlong-calls` in the module build (`modules/Makefile:158-163`) so no direct `BL`/`B` crosses states. **unknown at runtime** (Thumb-tail-call `B` would not switch state); check by disassembling `filmos.mo` for `b`/`bl` to undefined symbols.
6. Callback `fos_port_list(dir, fn, ctx)` (F:791) is plain C.
7. `fio_file` layout differs between DIGIC generations (`fio-ml.h:88-98`); F:796-804 uses `.name`/`.mode` through the compiler's struct, so safe as long as `internals.h` is included (done at `F:32-36` in module mode).
8. `CONFIG_CRASH_LOG` is on but "no stack unwinding, since existing code assumes ARM, not Thumb" (`P/features.h:68-71`): a crash in M-Film code will produce a poor log. Plan Phase 1.4 logging (`FILMROLL.LOG`) matters more on this camera.
9. `fos_port_read()` (F:758-774): uses uncacheable bounce buffer through `UNCACHEABLE()` (#27). DMA coherence on D7 not exercised by other 200D code in the tree: **unknown**.

---

## 6. Blockers for Phase 1 (`build_fork.sh 200D.101 --no-darkroom`, gate G1)

"Compile/link" blockers must be fixed for G1. "Safety" blockers do not stop the build but must be closed before any camera test (they are in the PLAN's Phase 1 step 2: "stubs or #ifdef for everything marked missing").

| # | type | item (rows) | what is wrong | what resolves it |
|---|---|---|---|---|
| B1 | compile/link | `get_ae_value` (#80) | missing stub: `AE_VALUE` is not defined for 200D; `nm` ABSENT; M-Film calls it at F:1050 and F:1163 | `#ifdef AE_VALUE` around both uses (meter needle off on 200D). Finding a real meter variable needs reversing; do not guess |
| B2 | safety (and probable compile warning) | `LCD_Palette`, `EngDrvOut`, `shamem_read` (#29-31) | fake palette address `0x56500000`; `palette_ours`, `palette_canon`, the `fos_port_sleep` probe and the palette log would read/write fake memory (the probe reads an arbitrary address taken from fake memory) | guard with `#ifndef FEATURE_VRAM_RGBA`; with the guard, `palette_probe` and `ui_last_palette` become unused: avoid `-Werror` unused-variable errors |
| B3 | safety | `fos_port_present` / `bmp_vram_real` (#35) | draws into the 32-bit BGRA layer | use `bmp_vram()` and set `ml_refresh_display_needed = 1` (section 3.3 item 3) |
| B4 | safety / owner decision | `prop_request_change_wait`, `PROP_PICTURE_STYLE`, `lens_set_*` (#69-75) | D678 whitelist allows only 3 properties; `apply_film()` silently does nothing and stalls ~1 s per call | `#ifdef CONFIG_200D` skip `apply_film`, or **ask Papo** before adding properties to `P/property_whitelist.h` (CLAUDE.md rule 2); each added ID needs a check of meaning/length on the 200D |
| B5 | safety | `BGMT_MENU` = 0 (#87, section 2) | `fake_simple_button(BGMT_MENU)` (F:591) sends event 0; `module_translate_key` reads any param-0 event as MENU | `#if BGMT_MENU` guard around F:591; real MENU code found by event spy / Phase 1.4 log |
| B6 | unknown, resolves only on camera | `display_idle` / `is_movie_mode` / `lv` / `shooting_mode` (#38-41), `pic_quality`, `image_review_time` (#76-77), `get_shooting_card().file_number` (#64) | property IDs are generic and unverified on 200D; roll counting and `shooting_screen()` depend on them | Phase 1.4 state dump (`DBG_STATE`, F:155) written to `ML/FILMROLL.LOG`; confirmed at H1/H3 |
| B7 | safety | `menu_open` patch in built-in mode (#101) | with the film OS built in, ML's menu never opens (`AF:66-78`): if the film OS does not come up on the 200D there is no ML menu to turn it off | first camera test with `--module` (menu falls back to ML when the module is not loaded), or an escape (for example keep ML's menu reachable when `filmroll.debug` is on) |
| B8 | unknown, resolves at build | boot size (section 4) | no limit in `cameras.txt`; margin to the 524,288-byte region is ~211 KB for everything M-Film adds | run `build_fork.sh` and record `_bss_end - RESTARTSTART`; compare with 524,288 |
| B9 | unknown, resolves at build | warnings under `FATAL_WARNINGS=y` | this audit did not compile M-Film for 200D | `build-verifier` must run it; items likely to warn: unused statics after B2, `get_ae_value` implicit declaration if wrapped wrongly |
| B10 | unknown, resolves at build | module mode (#93, #96) | only relevant if `--module` is chosen; `module.h` config macros not opened; import check against `200D_101.sym` | run `build_fork.sh 200D.101 --no-darkroom --module` |
| B11 | safety | `canon_gui_disable_front_buffer` and HUD "Canon's screen held" (#45) | on 200D it does not stop Canon drawing; HUD/plate may flicker or leave residue | accept for Phase 1 (no HUD claim); revisit with Phase 2 refresh design (3.3 item 4) |
| B12 | informational | `beep`/`beep_times` (#15) | silent no-ops (`CONFIG_BEEP` undefined) | none; document that the 200D has no beep, only LED blink where available |

No blocker was found in the `apply_fork.py` anchors, in the Makefile integration, or in the card/file API.

### Counts (107 rows in section 1)

| status | rows |
|---|---|
| exists | 81 |
| exists but different | 23 |
| missing stub | 1 (`get_ae_value`) |
| unknown | 2 (#93 module config macros, #96 module imports vs `200D_101.sym`) |

Of the 81 "exists" rows, 9 (#18, 25, 39, 41, 64, 66, 76, 77, 78) are symbols that link but whose behaviour on a real 200D is unverified (generic property IDs, task margin, card counters, free memory). Four of the "exists but different" rows (#27, 38, 40, 43) also carry open behaviour questions. Section 2 (19 rows) adds 3 key-code rows marked unknown (UNPRESS_HALFSHUTTER, PRESS/UNPRESS_FULLSHUTTER, MENU) and 2 missing (REC, TOUCH_2_FINGER).
