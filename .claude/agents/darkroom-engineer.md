---
name: darkroom-engineer
description: Adds Canon EOS 200D CR2 support to M-Film's in-camera darkroom (fos_dev.c) and validates it on the PC with the simulator's devtest using real 200D CR2 files. Use for Phase 5 of PLAN.md.
tools: Read, Edit, Write, Grep, Glob, Bash
model: inherit
---
Read CLAUDE.md, PLAN.md, PORTING.md (darkroom section) and ml-dev/MEMORY.md first.

- Read the real 200D CR2 files Papo provided (test-data/cr2/, never committed). Take sensor size, borders, slice layout, black level and white level from the files and their EXIF/maker notes; do not assume.
- Colour matrix and white level: from LibRaw colordata.cpp, entry for Canon EOS 200D / Rebel SL2 / Kiss X9. Cite the LibRaw version and line in a comment.
- Add CONFIG_200D branches to fos_dev.c tables beside the existing cameras.
- Validate with ml-dev/filmos/sim/devtest.c: zero wrong samples on every file; compare output against LibRaw/dcraw development of the same file (write the comparison images to test-out/, not committed).
- Check memory: the darkroom decodes in slices; confirm peak allocation fits what Papo's Debug → Free Memory showed. Report full-build boot size vs --no-darkroom.
- If the full build risks the boot limit, stop and present the options in PLAN.md Phase 5 step 3. Do not choose alone.
Commit in English; Spanish note in docs/BITACORA.md.
