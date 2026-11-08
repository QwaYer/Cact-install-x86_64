# План работ: cact-install x86_64 (CactOS 3.0.0)

Форк от `2.0.0` (i686). `MAJOR` → 3 по правилу «`MAJOR` = смена архитектурного
поколения»: установщик перелинковывается под ABI ядра 3.0.0 и libc 3.0.0
(`CactKernel-x86_64`, `CactLibc-x86_64`, решения 5/9 ядра, решение 5 libc).

Это файл-план, а не обещание. Инвентарь снят по коду; при расхождении верить
коду.

## Что уже хорошо

cact-install — многолюдный C-проект (`src/*.c`: TUI, диск, ФС, GRUB, архив) без
ассемблера и прямых syscall: всё идёт через `clibc.so`. Архитектурно-зависимы
только сборка и скрипт линковки. zstd берётся из дерева ядра (сжат
`archive.c`), при этом `--gc-sections` его выбрасывает — поток установки его не
зовёт.

## Инвентарь 32-битных поверхностей

- `cross/i686-cact-clang.ini` — `-m32`, `cpu=i686`.
- `meson.build` — `ld -m elf_i386 -pie`; комментарий ссылается на i686-файл и
  `CactLibc-x86_32`.
- `link.ld` — `.rel.dyn`/`.rel.plt` (в x86_64 — `.rela.*`); нет `.data.rel.ro`.
  База `0x08000000` не двигается (решения 5/9 ядра).
- `meson_options.txt` — пять дефолтов `../*-x86_32` (`cactlib`, `lr_sbin`,
  `lr_lib`, `kern_build`, `zstd_dir`).

## Решения

1. **Тулчейн как у соседей**: `cross/x86_64-cact-clang.ini` (`-m64`),
   `ld -m elf_x86_64`, дефолты → `../*-x86_64` (в т. ч. `kern_build` и
   `zstd_dir` — в `CactKernel-x86_64`).
2. **База `0x08000000` не двигается** (решение 5 libc).
3. **Скрипт линковки под ELF64**: `.rela.dyn`/`.rela.plt` и `.data.rel.ro`,
   пара `.rel.*` остаётся против orphan-секций.

## Этапы

Порт — один этап: меняются только тулчейн и скрипт линковки; коммит один.

### Фаза 0 — тулчейн и скрипт линковки

**Milestone:** `ninja -C build-meson` собирает `cact-install`; `readelf -h` —
`ELF64`/`EM_X86_64`.

## Проверки

- **Кросс-сборка** — `meson setup build-meson --cross-file
  cross/x86_64-cact-clang.ini -Dcactlib=../CactLibc-x86_64 && ninja -C
  build-meson` (нужен собранный `CactLibc-x86_64/build-meson/start.o`/`clibc.so`;
  zstd — из `CactKernel-x86_64`).
- **Заголовок/релокации** — `readelf -h`/`readelf -r build-meson/cact-install`.
- **`stage`/`stage-media`** — копируют бинарь в `lr_sbin` и kernel+grub в
  `lr_lib`; запускать после сборки ядра и до упаковки cctkfs.
- **На железе/в QEMU** — за ядром, установщик стартует как
  `/usr/sbin/cact-install`.

## Открытые вопросы

- **`test/` — host-тесты** (`firmware_test.c`, `tui_wire_test.c`), собираются
  `tools/*.sh` на сборочной машине, не Meson'ом; отдельная поверка.
- **Образ — `ET_EXEC`** (как у прочих 2.0.0 PIE): ядро мапит по `PT_LOAD` без
  проверки `e_type`.
