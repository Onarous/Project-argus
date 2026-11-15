# ARGUS — рабочий форк

Форк проекта [slopmaster1884/Project-argus](https://github.com/slopmaster1884/Project-argus) (ARGUS — домашняя нейросеть: ПК + телефоны, llama.cpp b12008, Rust/Tauri 2).

## Remotes
- `origin`   → https://github.com/Onarous/Project-argus.git  (мой форк, push сюда)
- `upstream` → https://github.com/slopmaster1884/Project-argus.git (оригинал)

## Состояние
- Локально: `main` @ `f9e8020` — форк на 3 коммита впереди `upstream/main`.
- Коммиты форка: `95b1998` (AGENTS.md), `87dcd4d` (0.3.5: фикс загрузки модели на ПК + LAN-нода),
  `f9e8020` (убрать `-ngl` + UMA-fallback в lan.rs).
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
- `-ngl 999` в `llama.rs:410` и `llama.rs:422` — GPU-хоста нет/NGPU не задан.
- llama.cpp `--fit` (`-fit on` по умолчанию) сам подбирает `-ngl`/`-c`/`-ts` под свободную память.

## Правило: НИКОГДА не передавать `-ngl` и `-ts` (0.3.5)
--fit сравнивает `n_gpu_layers` с дефолтом (-1) ЧИСЛЕННО (`common/fit.cpp:544`) => любое
явное значение (999, 27, 36) роняет fit с «n_gpu_layers already set by user».
Аналогично `-ts` = `tensor_split` => «tensor_split already set by user» (fit.cpp:547-557).
Abort срабатывает только когда модели НЕ хватает памяти (ранние выходы fit.cpp:407/437/448),
то есть ровно в проблемном сценарии. Без `-ngl`/`-ts` llama.cpp сам делит слои по свободной
памяти устройств (`llama-model.cpp:1658`, ветка `all_zero`).

## Проверенные на живой ноде (192.168.1.82:50052) результаты
- `--device Vulkan0,RPC0` без `-ngl`/`-ts`: fit отработал, abort'а нет, слои 0-13 на GPU,
  14-26 на RPC0, `model loaded`, ноутбук не ушёл в OOM.
- С `-ngl 27` и ctx 262144 (не влезает): abort + падение движка. Без `-ngl`: модель грузится.
- UMA-ноутбук (Vulkan-760M): rpc-server отдаёт 8151 MiB вместо 15279 MiB системной RAM =>
  `usable_mb` в `lan.rs` резервирует headroom 1024 MB под ОС.

## Сборка
- `desktop/`: `npm install` → `npm run build` (фронт), `cargo build` в `desktop/src-tauri`.
- Линтер/тесты Rust: `cargo test` (юнит-тесты в `split.rs`, `llama.rs`).

## Git-работа
- Правки в `main` форка, PR в `slopmaster1884/Project-argus` при необходимости.
- Перед синхронизацией: `git fetch upstream && git rebase upstream/main`.
