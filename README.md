# cachyos-safe-update

[English](#english) · [Español](#español) · [日本語](#日本語)

---

## English

Update CachyOS (or any Arch-based distro) **only when there's plenty of free space**, after calculating how much the update will *really* consume — considerably more than pacman reports.

### Why it exists

If you update with a nearly-full disk, pacman can run out of space *mid-transaction*: half-written packages, corrupted databases, and a system that may no longer boot. The catch is that pacman only tells you the **download** size, while the real update consumes more:

- **Version coexistence**: during the transaction, the old and new versions of each package are on disk at the same time.
- **Initramfs**: every kernel regenerates its initramfs (~150–250 MB per kernel; with 3–4 kernels installed that's hundreds of MB extra).
- **snapper snapshots**: if you use snap-pac, every update creates pre/post snapshots that duplicate modified blocks via COW.
- **Cache**: the downloaded `.pkg.tar.zst` files stay in `/var/cache/pacman/pkg` on top of what gets installed.

This script demands a generous margin before touching anything: if there isn't one, it tells you what to clean and does not update.

### What it does

1. **Cleans caches** (before measuring, so freed space counts toward the check):
   - `paccache -rk1` — keeps only 1 version of each package in pacman's cache.
   - `paccache -ruk0` — removes cached versions of uninstalled packages.
   - `paru -Sc` — cleans the AUR build cache (if you have paru).
   - Deletes the `download-*` leftovers of interrupted transactions, which besides wasting space **break pacman 7's sandboxed downloader** (`DownloadUser = alpm`) with the cryptic `Error reading fd N` error.
2. **Calculates the space needed**:
   - Download: what pacman announces for the pending packages.
   - Install: the installed size of each package (read from `pacman -Si`).
   - Demands **twice** that sum, plus **2 GB per pending AUR package** (its size can't be known in advance), with an **absolute minimum of 10 GB free**.
3. **Updates only if there's room**: `pacman -Su` for repos and `paru -Su --aur` for AUR. If not, it shows concrete cleanup commands and aborts.

### Installation

```bash
git clone https://github.com/eztornado/cachyos-safe-update.git
sudo install -Dm755 cachyos-safe-update/cachyos-safe-update /usr/local/bin/cachyos-safe-update
```

or, if you'd rather not use sudo:

```bash
install -Dm755 cachyos-safe-update/cachyos-safe-update ~/.local/bin/cachyos-safe-update
```

(`~/.local/bin` must be in your `PATH`.)

### Usage

```bash
cachyos-safe-update          # cleans, calculates and updates if there's enough space
cachyos-safe-update --check  # same, but updates nothing (only shows the calculation)
```

If anything fails (no space, no network), the script aborts **before** installing anything and tells you what to do.

### Requirements

- Bash, sudo, pacman, util-linux (`numfmt`) and `paccache` (ships with pacman).
- `paru` is optional: without it, only the official repos get updated.

### Settings

The constants at the top of the script:

| Variable | Default | What it controls |
|---|---|---|
| `FACTOR` | `2` | Safety margin over the estimate |
| `MIN_LIBRE_GB` | `10` | Minimum free space, even for a small update |
| `AUR_RESERVA_GB` | `2` | Budget per pending AUR package |

### Notes

- Don't run it as root: it uses sudo internally, and paru refuses to run as root.
- Since it's `pacman -Sy` + `-Su` in the same run (database refresh and upgrade in the same session), there's no partial-upgrade risk.
- Tested on CachyOS with btrfs + snapper and pacman 7 with `DownloadUser = alpm`. It should work on other Arch setups too, but no guarantees.

---

## Español

Actualiza CachyOS (y cualquier Arch-based) **solo si hay espacio de sobra**, tras calcular cuánto va a consumir de verdad la actualización — que es bastante más de lo que pacman anuncia.

### Por qué existe

Si actualizas con el disco justo, pacman puede quedarse sin espacio *a mitad de la transacción*: paquetes a medias, bases de datos corruptas y un sistema que quizá no arranca. Y es que pacman te dice cuánto ocupa la **descarga**, pero la actualización real consume más:

- **Coexistencia de versiones**: durante la transacción están en disco la versión vieja y la nueva a la vez.
- **Initramfs**: cada kernel regenera su initramfs (~150–250 MB por kernel; con 3–4 kernels instalados son cientos de MB extra).
- **Snapshots de snapper**: si usas snap-pac, cada actualización crea snapshots pre/post que duplican por COW los bloques modificados.
- **Caché**: los `.pkg.tar.zst` descargados se quedan en `/var/cache/pacman/pkg` además de lo instalado.

Este script exige un margen holgado antes de tocar nada: si no lo hay, te dice qué limpiar y no actualiza.

### Qué hace

1. **Limpia cachés** (antes de medir, para que lo liberado cuente):
   - `paccache -rk1` — deja solo 1 versión de cada paquete en la caché de pacman.
   - `paccache -ruk0` — borra las versiones de paquetes ya desinstalados.
   - `paru -Sc` — limpia la caché de builds de AUR (si tienes paru).
   - Borra los restos `download-*` de transacciones interrumpidas, que además de ocupar **rompen el downloader con sandbox de pacman 7** (`DownloadUser = alpm`) con el críptico error `Error reading fd N`.
2. **Calcula el espacio necesario**:
   - Descarga: lo que anuncia pacman para los paquetes pendientes.
   - Instalación: tamaño instalado de cada paquete (leído de `pacman -Si`).
   - Exige **el doble** de esa suma, más **2 GB por cada paquete de AUR** pendiente (su tamaño no se puede saber por adelantado), con un **mínimo absoluto de 10 GB libres**.
3. **Actualiza si hay de sobra**: `pacman -Su` para repos y `paru -Su --aur` para AUR. Si no, muestra comandos concretos para liberar espacio y aborta.

### Instalación

```bash
git clone https://github.com/eztornado/cachyos-safe-update.git
sudo install -Dm755 cachyos-safe-update/cachyos-safe-update /usr/local/bin/cachyos-safe-update
```

o, si prefieres no usar sudo:

```bash
install -Dm755 cachyos-safe-update/cachyos-safe-update ~/.local/bin/cachyos-safe-update
```

(`~/.local/bin` debe estar en tu `PATH`.)

### Uso

```bash
cachyos-safe-update          # limpia, calcula y actualiza si hay espacio suficiente
cachyos-safe-update --check  # igual, pero sin actualizar nada (solo muestra el cálculo)
```

Si algo falla (sin espacio, sin red), el script aborta **antes** de instalar nada y te dice qué hacer.

### Requisitos

- Bash, sudo, pacman y util-linux (`numfmt`), `paccache` (viene con pacman).
- `paru` es opcional: sin él solo se actualizan los repos oficiales.

### Ajustes

Las constantes de la cabecera del script:

| Variable | Valor por defecto | Qué controla |
|---|---|---|
| `FACTOR` | `2` | Margen de seguridad sobre lo estimado |
| `MIN_LIBRE_GB` | `10` | Espacio libre mínimo, aunque la actualización sea pequeña |
| `AUR_RESERVA_GB` | `2` | Presupuesto por paquete AUR pendiente |

### Notas

- No lo ejecutes como root: usa sudo internamente y paru se niega a correr como root.
- Al ser `pacman -Sy` + `-Su` en la misma ejecución (refresco de bases de datos y actualización en la misma sesión), no hay riesgo de *partial upgrade*.
- Probado en CachyOS con btrfs + snapper y pacman 7 con `DownloadUser = alpm`. En otras configuraciones de Arch debería funcionar igual, pero sin garantías.

---

## 日本語

CachyOS（および Arch 系ディストロ）を、アップデートが**実際に消費する容量を計算した上で、十分な空き容量がある場合にだけ**更新します。pacman が表示するサイズよりかなり多く消費されるためです。

### なぜ存在するのか

空き容量がぎりぎりの状態で更新すると、pacman はトランザクションの途中で容量不足に陥ることがあります：途中まで書き込まれたパッケージ、破損したデータベース、最悪の場合起動しなくなるシステムです。pacman が表示するのは**ダウンロード**サイズだけで、実際の更新はそれ以上消費します：

- **バージョンの共存**：トランザクション中は旧バージョンと新バージョンが同時にディスク上に存在します。
- **initramfs**：カーネルごとに initramfs を再生成します（1 カーネルあたり約 150〜250 MB。3〜4 個のカーネルが入っていれば数百 MB の追加になります）。
- **snapper のスナップショット**：snap-pac を使っている場合、更新のたびに pre/post スナップショットが作られ、COW によって変更ブロックが複製されます。
- **キャッシュ**：ダウンロードした `.pkg.tar.zst` は、インストール分とは別に `/var/cache/pacman/pkg` に残ります。

このスクリプトは、何かをする前に十分なマージンを要求します。マージンがなければ、何を掃除すればいいかを表示して更新を中止します。

### 何をするか

1. **キャッシュを掃除します**（計測の前に実行し、解放された容量を計算に反映させます）：
   - `paccache -rk1` — pacman のキャッシュで各パッケージのバージョンを 1 つだけ残します。
   - `paccache -ruk0` — アンインストール済みパッケージのキャッシュを削除します。
   - `paru -Sc` — AUR のビルドキャッシュを掃除します（paru がある場合）。
   - 中断されたトランザクションが残した `download-*` を削除します。これは容量を浪費するだけでなく、**pacman 7 のサンドボックスダウンローダー**（`DownloadUser = alpm`）を壊し、謎の `Error reading fd N` エラーを吐かせます。
2. **必要な容量を計算します**：
   - ダウンロード：pacman が保留中のパッケージについて表示するサイズ。
   - インストール：各パッケージのインストール後サイズ（`pacman -Si` から取得）。
   - その合計の **2 倍**、さらに保留中の AUR パッケージ 1 個につき **2 GB**（事前にサイズを知る方法がないため）を要求し、**最低でも 10 GB の空き**を確保します。
3. **十分な空きがあれば更新します**：リポジトリは `pacman -Su`、AUR は `paru -Su --aur`。なければ具体的な掃除コマンドを表示して中止します。

### インストール

```bash
git clone https://github.com/eztornado/cachyos-safe-update.git
sudo install -Dm755 cachyos-safe-update/cachyos-safe-update /usr/local/bin/cachyos-safe-update
```

sudo を使いたくない場合：

```bash
install -Dm755 cachyos-safe-update/cachyos-safe-update ~/.local/bin/cachyos-safe-update
```

（`~/.local/bin` が `PATH` に含まれている必要があります。）

### 使い方

```bash
cachyos-safe-update          # 掃除し、計算し、空き容量が十分なら更新する
cachyos-safe-update --check  # 同じだが何も更新しない（計算結果の表示のみ）
```

何かが失敗した場合（容量不足、ネットワーク断など）、スクリプトは**何もインストールする前に**中止し、対処法を表示します。

### 要件

- Bash、sudo、pacman、util-linux（`numfmt`）、`paccache`（pacman に同梱）。
- `paru` は任意：なければ公式リポジトリのみ更新されます。

### 設定

スクリプト冒頭の定数：

| 変数 | デフォルト | 説明 |
|---|---|---|
| `FACTOR` | `2` | 推定値に対する安全マージン |
| `MIN_LIBRE_GB` | `10` | 更新が小さくても確保する最低空き容量 |
| `AUR_RESERVA_GB` | `2` | 保留中の AUR パッケージ 1 個あたりの予算 |

### 注意

- root で実行しないでください：内部で sudo を使うほか、paru は root での実行を拒否します。
- `pacman -Sy` と `-Su` を同じ実行内で行うため（データベースの更新とアップグレードが同一セッション内）、partial upgrade の危険はありません。
- btrfs + snapper、`DownloadUser = alpm` の pacman 7 という CachyOS 環境で動作確認済み。他の Arch 構成でも動くはずですが、保証はありません。
