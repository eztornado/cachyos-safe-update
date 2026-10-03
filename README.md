# cachyos-safe-update

Actualiza CachyOS (y cualquier Arch-based) **solo si hay espacio de sobra**, tras calcular cuánto va a consumir de verdad la actualización — que es bastante más de lo que pacman anuncia.

## Por qué existe

Si actualizas con el disco justo, pacman puede quedarse sin espacio *a mitad de la transacción*: paquetes a medias, bases de datos corruptas y un sistema que quizá no arranca. Y es que pacman te dice cuánto ocupa la **descarga**, pero la actualización real consume más:

- **Coexistencia de versiones**: durante la transacción están en disco la versión vieja y la nueva a la vez.
- **Initramfs**: cada kernel regenera su initramfs (~150–250 MB por kernel; con 3–4 kernels instalados son cientos de MB extra).
- **Snapshots de snapper**: si usas snap-pac, cada actualización crea snapshots pre/post que duplican por COW los bloques modificados.
- **Caché**: los `.pkg.tar.zst` descargados se quedan en `/var/cache/pacman/pkg` además de lo instalado.

Este script exige un margen holgado antes de tocar nada: si no lo hay, te dice qué limpiar y no actualiza.

## Qué hace

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

## Instalación

```bash
git clone https://github.com/eztornado/cachyos-safe-update.git
sudo install -Dm755 cachyos-safe-update/cachyos-safe-update /usr/local/bin/cachyos-safe-update
```

o, si prefieres no usar sudo:

```bash
install -Dm755 cachyos-safe-update/cachyos-safe-update ~/.local/bin/cachyos-safe-update
```

(`~/.local/bin` debe estar en tu `PATH`.)

## Uso

```bash
cachyos-safe-update          # limpia, calcula y actualiza si hay espacio suficiente
cachyos-safe-update --check  # igual, pero sin actualizar nada (solo muestra el cálculo)
```

Si algo falla (sin espacio, sin red), el script aborta **antes** de instalar nada y te dice qué hacer.

## Requisitos

- Bash, sudo, pacman y util-linux (`numfmt`), `paccache` (viene con pacman).
- `paru` es opcional: sin él solo se actualizan los repos oficiales.

## Ajustes

Las constantes de la cabecera del script:

| Variable | Valor por defecto | Qué controla |
|---|---|---|
| `FACTOR` | `2` | Margen de seguridad sobre lo estimado |
| `MIN_LIBRE_GB` | `10` | Espacio libre mínimo, aunque la actualización sea pequeña |
| `AUR_RESERVA_GB` | `2` | Presupuesto por paquete AUR pendiente |

## Notas

- No lo ejecutes como root: usa sudo internamente y paru se niega a correr como root.
- Al ser `pacman -Sy` + `-Su` en la misma ejecución (refresco de bases de datos y actualización en la misma sesión), no hay riesgo de *partial upgrade*.
- Probado en CachyOS con btrfs + snapper y pacman 7 con `DownloadUser = alpm`. En otras configuraciones de Arch debería funcionar igual, pero sin garantías.
