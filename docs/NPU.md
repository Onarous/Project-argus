# NPU на ноутбуке: что получилось и почему это не Argus-нода

> **Статус:** эксперимент вне Argus. Бэкенд собирается и работает на этом ноутбуке (ускорение prefill в 1.9×),
> но **в Argus он не подключается** — см. «Почему нельзя подключить к Argus».
> Проверено 2026-11-15 на ноутбуке: AMD Ryzen 5 8640HS (Phoenix, Radeon 760M), Kali Linux, kernel 7.1.5.

## Коротко

| | |
|---|---|
| Что | бэкенд [FireBurn/ggml-xdna1](https://github.com/FireBurn/ggml-xdna1) — GGML-бэкенд под XDNA1/AIE2 |
| Как работает | напрямую через DRM-драйвер `amdxdna` на `/dev/accel/accel0`, **без XRT** |
| NPU в ноутбуке | XDNA1 / AIE2, PCI ID `1022:1502`, 4 колонки × 4 compute tiles = 16 ядер |
| Что ускоряет | **prefill** (обработку промпта) |
| Что не ускоряет | **decode** (генерацию) — по дизайну |
| В Argus | **не подключается**, три независимых барьера |

## Замеры (это ноутбук, 8 потоков)

`llama-bench`, gemma-2-2b-it-Q4_K_M, сервер llama.cpp b12008.
Вулкан отключён через `GGML_VK_VISIBLE_DEVICES=-1`, иначе baseline считался бы на 760M, а не на CPU.

| Тест | CPU | CPU + NPU | Разница |
|---|---|---|---|
| pp512 (prefill) | 151.91 t/s | 288.57 t/s | **1.90×** |
| pp128 (prefill) | 123.73 t/s | 211.43 t/s | **1.71×** |
| tg64 (decode) | 22.67 t/s | 23.33 t/s | без разницы |

Decode не ускоряется, и это ожидаемо: NPU через shim даёт ~13.5 GB/s против ~65 GB/s DDR5 у CPU,
а decode — это один токен за шаг с одной колонки. Ускоряется только обработка промпта.

Бэкенд сам отклоняет формы, где NPU медленнее CPU (`XDNA1_PROFILE=1`):

```
[xdna1] declining   1024x2304   N=512  : 409 GMAC/s predicted, below the 550 GMAC/s CPU reference
[xdna1] declining   2304x2048   N=512  : 439 GMAC/s predicted, below the 550 GMAC/s CPU reference
[xdna1] plan   2048x2304   N=512   -> 4 shards x 4 cores (16 tiles), Mw=512 Kp=2304 nt=2, 571 GMAC/s predicted
```

## Почему нельзя подключить к Argus

Argus раздаёт модель между устройствами только через RPC-протокол llama.cpp (`--rpc` → `ggml-rpc-server`).
NPU в этом пути не участвует, и мешают три независимых барьера — каждый по отдельности достаточен:

**1. rpc-server физически отбрасывает ACCEL-устройства.** `tools/rpc/rpc-server.cpp:196`:

```cpp
if (dev_type != GGML_BACKEND_DEVICE_TYPE_CPU && dev_type != GGML_BACKEND_DEVICE_TYPE_ACCEL) {
    devices.push_back(dev);
}
```

Тип `ACCEL` — это и есть NPU. Он выкидывается, и rpc-server отдаёт только CPU.
Проверено: в логе rpc-server есть только `CPU: AMD Ryzen 5 8640HS`, никакого XDNA1.

**2. В llama.cpp b12008 нет бэкенда для NPU.** В `ggml/src` из ACCEL-бэкендов есть только `ggml-vulkan`.
Нет ни `ggml-xdna`, ни `ggml-amdxdna` — даже если бы rpc-server захотел отдать NPU,
на стороне ПК нет кода, который умеет с ним говорить.

**3. AMD не поддерживает LLM на NPU этого поколения.** Ryzen AI Software 1.8,
Supported Processor Configurations: для Ryzen AI 7000/8000 (Phoenix/Hawk Point) все NPU-режимы
помечены ✗ — поддержка только у серии 300. Linux-релиз 1.8: «supports STX and KRK platforms».

Вывод: NPU — это отдельный проект, а не Argus-нода. Подключить его к Argus нельзя,
и «правильной настройкой» это не лечится.

## Как собрать (если захочется повторить)

### 1. Требования ноутбука

```bash
ls -l /dev/accel/          # accel0, crw-rw----+ root render
lsmod | grep amdxdna       # модуль amdxdna загружен
ulimit -l                 # должен быть unlimited: драйвер пинит 64 МБ device heap
```

`ulimit -l` по умолчанию 8 МБ — тогда каждое выделение падает с `mmap DEV_HEAP: Resource temporarily unavailable`,
это переживает перезагрузку драйвера и молчит в логах, поэтому читается как «сломанное железо».
На Kali `ulimit -l` уже unlimited; на других дистрибутивах надо поднять в `limits.conf`.

### 2. Инструменты сборки

Нужен cmake >= 3.16 и компилятор. Без root:

```bash
pipx install cmake
pipx install ninja
export PATH="$HOME/.local/bin:$PATH"
```

### 3. Исходники

```bash
git clone --depth 1 https://github.com/FireBurn/ggml-xdna1.git
git clone --depth 1 --branch b12008 https://github.com/ggml-org/llama.cpp.git
```

Ядра AIE2 **не нужно компилировать**: они синтезируются в процессе из встроенного шаблона
`src/xdna1-template.h`. mlir-aie / XRT не требуются — в `ldd` плагина только libstdc++, libm,
libgcc и libc.

### 4. Патч под b12008 (обязательный)

Плагин написан под llama.cpp `169e4a7ff` (август 2026). В b12008 в
`ggml_backend_buffer_type_i` добавили **два** поля: `alloc_buffer_n` и `get_alloc_size_n`.
Инициализатор структуры в плагине позиционный, поэтому `get_alignment` встаёт в слот
`alloc_buffer_n` и компилятор ругается на несовместимость указателей:

```
error: invalid conversion from ‘size_t (*)(ggml_backend_buffer_type_t)’
       to ‘ggml_backend_buffer* (*)(ggml_backend_buffer_type_t, ggml_tensor**, int)’
```

Оба поля опциональны (NULL → дефолтная реализация upstream), поэтому правка минимальная —
в `src/ggml-xdna1.cpp`:

```cpp
static ggml_backend_buffer_type xdna1_buft = {
    /* .iface = */ {
        /* .get_name       = */ xdna1_buft_name,
        /* .alloc_buffer   = */ xdna1_buft_alloc,
        /* .alloc_buffer_n = */ NULL,      // + новое поле в b12008
        /* .get_alignment  = */ xdna1_buft_align,
        /* .get_max_size   = */ NULL,
        /* .get_alloc_size = */ NULL,
        /* .get_alloc_size_n = */ NULL,    // + новое поле в b12008
        /* .is_host        = */ xdna1_buft_is_host,
    },
    /* .device  = */ NULL,
    /* .context = */ NULL,
};
```

### 5. Сборка

```bash
cd ggml-xdna1
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_SOURCE_DIR=$HOME/work/llama.cpp/ggml
cmake --build build -j$(nproc)
# -> build/lib/libggml-xdna1.so
```

`GGML_SOURCE_DIR` обязателен: плагину нужны `ggml-backend-impl.h` из дерева исходников,
а не установленные заголовки. `ggml_backend_i` между `169e4a7ff` и b12008 не менялся —
расходится только `ggml_backend_buffer_type_i`.

### 6. Запуск

```bash
export GGML_BACKEND_PATH=/path/to/libggml-xdna1.so
export LD_LIBRARY_PATH=/path/to/llama.cpp/build/bin:$LD_LIBRARY_PATH

# бенчмарк (--device не нужен только для llama-bench)
llama-bench -m gemma-2-2b-it-Q4_K_M.gguf -p 512 -n 0 -r 3 -t 8

# сервер: --device XDNA1 обязателен, иначе плагин не откроется и всё пойдёт на CPU
llama-server -m gemma-2-2b-it-Q4_K_M.gguf --device XDNA1 -ngl 0 \
    --host 127.0.0.1 --port 8080 -c 4096 -t 8
```

Два требования, которые легко пропустить:

- **`--device XDNA1`** нужен для всего, кроме `llama-bench`. Внеплагиновые бэкенды грузятся только
  через `ggml_backend_load_all()`, а до `llama-cli`/`llama-server` это доходит только через разбор
  списка устройств — без `--device` плагин просто не откроется, и всё молча уйдёт на CPU.
- **`-ngl 0`**: веса остаются в системной памяти. Это ускоритель, дополняющий CPU, а не цель
  offload: он объявляет только свой 64 МБ device heap, и просьба положить слои на него ломает загрузку.

Признак успеха — строка в stderr:

```
[xdna1] ready: 4 hardware contexts, kernels synthesised on demand, min_batch=64, ...
```

Если её нет, плагин не загрузился (а скорость будет выглядеть как обычный CPU-прогон,
а не как ошибка). Диагностика: `XDNA1_PROFILE=1`.

## Что это значит для Argus

Ничего отрицательного: Argus на ноутбуке использует его как обычную RPC-ноду (Vulkan-760M),
и это работает. NPU — отдельный эксперимент, и его результаты можно использовать, только если
в rpc-server появится поддержка ACCEL-устройств и в llama.cpp — бэкенд для NPU.

Практическая ценность для проекта сейчас: знать, что на Ryzen 7000/8000 NPU недоступен для Argus
и не тратить на него время.

## Осторожно с памятью (UMA)

На ноутбуке с UMA графика (760M) и NPU делят одну системную RAM. Одновременно запущенные
`rpc-server` (Argus-нода) и XDNA-`llama-server` оставляют ноутбуку мало свободной памяти,
и проверки загрузки модели получаются нерепрезентативными. Перед такими проверками
XDNA-сервер надо останавливать.

В `dmesg` при этом идут сообщения `amdgpu "Not enough memory for command submission!"` —
это ожидаемо для UMA и само по себе не ошибка.