# ARGUS — рабочий форк

Форк проекта [slopmaster1884/Project-argus](https://github.com/slopmaster1884/Project-argus) (ARGUS — домашняя нейросеть: ПК + телефоны, llama.cpp b12008, Rust/Tauri 2).

## Remotes
- `origin`   → https://github.com/Onarous/Project-argus.git  (мой форк, push сюда)
- `upstream` → https://github.com/slopmaster1884/Project-argus.git (оригинал)

## Состояние
- Локально: `main` — форк на 5 коммитов впереди `upstream/main`.
- Коммиты форка: `95b1998` (AGENTS.md), `87dcd4d` (0.3.5: фикс загрузки модели на ПК + LAN-нода),
  `f9e8020` (убрать `-ngl` + UMA-fallback в lan.rs), `4b76c12` (docs), `0d90214` (версия 0.3.5).
- Тег `v0.3.4` (`d92c1fb`) — базовая точка upstream.
- Ветки: `main`, `release-bin` (релизные сборки).
- Тесты: `cargo test` — 43 passed / 0 failed.

## Структура
- `desktop/` — Tauri 2 приложение: `src/` (React+TS фронт), `src-tauri/src/` (Rust-бэкенд).
  Ключевые модули: `llama.rs` (аргументы llama-server), `split.rs` (раскладка слоёв по устройствам),
  `lan.rs` (LAN-ноды по RPC), `cluster.rs`, `gpu.rs`, `settings.rs`, `state.rs`.
- `android/` — Argus Node (APK) для телефонов.
- `scripts/` — `argus-node-linux.sh` (нода для Linux/RPi: rpc-server + UDP-маяк 50053).
- `docs/`, `.github/`.

## Ключевые константы
- RPC протокол major = 8 (ggml-rpc.h @ llama.cpp b12008), `lan.rs:23`.
- `llama-server` порт 8080 (внутри), RPC-ноды на 50052, UDP-маяк 50053.
- llama.cpp `--fit` (`-fit on` по умолчанию) сам подбирает `-ngl`/`-c`/`-ts` под свободную память.
  Argus не должен передавать `-ngl`/`-ts` сам (см. правило ниже).

## Правило: НИКОГДА не передавать `-ngl` и `-ts` (0.3.5)
--fit сравнивает `n_gpu_layers` с дефолтом (-1) ЧИСЛЕННО (`common/fit.cpp:544`) => любое
явное значение (999, 27, 36) роняет fit с «n_gpu_layers already set by user».
Аналогично `-ts` = `tensor_split` => «tensor_split already set by user» (fit.cpp:547-557).
Проверка abort стоит ПОСЛЕ ранних выходов fit.cpp:407/437/448 («no changes needed» / «the entire
model can be fit by reducing context»), поэтому abort срабатывает только когда модели реально
не хватает памяти и fit вынужден пересчитывать раскладку. Без `-ngl`/`-ts` llama.cpp сам делит
слои по свободной памяти устройств (`llama-model.cpp:1658`, ветка `all_zero`).
CPU-устройство нельзя передать в `--device` (`parse_device_list` его отбрасывает), поэтому
`cpu_layers` в плане Argus — информационная величина: реальную раскладку делает llama.cpp.

## Проверенные на живой ноде (192.168.1.82:50052) результаты
- `--device Vulkan0,RPC0` без `-ngl`/`-ts`: fit отработал, abort'а нет, слои 0-13 на RPC0
  (ноутбук), 14-26 на Vulkan0 (GPU ПК), `model loaded`, ноутбук не ушёл в OOM.
- Abort воспроизводится только когда fit вынужден пересчитывать раскладку, т.е. модели реально
  не хватает памяти. На ctx 262144 модель ВЛЕЗАЕТ: fit выходит по раннему выходу
  «targets for free memory can be met on all devices, no changes needed» (fit.cpp:448) и до
  проверки abort (строка 544) не доходит — потому `-ngl 27` там грузится штатно.
  Правильный перебор: ctx 1048576 (не влезает по KV-кэшу):
    `-ngl 27`  => «n_gpu_layers already set by user to 27, abort», движок падает;
    без `-ngl` => fit успешен, дальше честная ошибка аллокации KV в RAM ПК (без abort'а fit).
- UMA-ноутбук (Vulkan-760M): rpc-server отдаёт 8151 MiB вместо 15279 MiB системной RAM =>
  `usable_mb` в `lan.rs` резервирует headroom 1024 MB под ОС.

## Сборка
- `desktop/`: `npm install` → `npm run build` (фронт), `cargo build` в `desktop/src-tauri`.
- Линтер/тесты Rust: `cargo test` (юнит-тесты в `split.rs`, `llama.rs`).

## Git-работа
- Правки в `main` форка, PR в `slopmaster1884/Project-argus` при необходимости.
- Перед синхронизацией: `git fetch upstream && git rebase upstream/main`.
