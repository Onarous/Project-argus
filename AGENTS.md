# ARGUS — рабочий форк

Форк проекта [slopmaster1884/Project-argus](https://github.com/slopmaster1884/Project-argus) (ARGUS — домашняя нейросеть: ПК + телефоны, llama.cpp b12008, Rust/Tauri 2).

## Remotes
- `origin`   → https://github.com/Onarous/Project-argus.git  (мой форк, push сюда)
- `upstream` → https://github.com/slopmaster1884/Project-argus.git (оригинал)

## Состояние
- Локально: `main` @ `c6c7f2a` — синхронизирован с `upstream/main`.
- Тег `v0.3.4` (`d92c1fb`) — на 3 коммита позади `main`.
- Ветки: `main`, `release-bin` (релизные сборки).

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

## Сборка
- `desktop/`: `npm install` → `npm run build` (фронт), `cargo build` в `desktop/src-tauri`.
- Линтер/тесты Rust: `cargo test` (юнит-тесты в `split.rs`, `llama.rs`).

## Git-работа
- Правки в `main` форка, PR в `slopmaster1884/Project-argus` при необходимости.
- Перед синхронизацией: `git fetch upstream && git rebase upstream/main`.
