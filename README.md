# TP 3, UEFI: Entorno, desarrollo y análisis de seguridad

Trabajo práctico sobre la arquitectura UEFI como entorno pre-sistema operativo: exploración de la UEFI Shell, desarrollo de una aplicación nativa en C, análisis del formato PE/COFF con Ghidra y ejecución en hardware real.


## Entorno utilizado
| Componente | Versión |
|---|---|
| Sistema anfitrión | VM con LMDE (base Debian 13) |
| Emulador | QEMU 10.0 (`qemu-system-x86_64`) |
| Firmware | OVMF (distribución Debian de EDK II, UEFI 2.70) |
| Toolchain | gcc + gnu-efi + binutils |
| Ingeniería inversa | Ghidra 12.1.4 + OpenJDK 21 |

## Contenido
- [TP1: Exploración del entorno UEFI y la Shell](tp1-shell-uefi/)
- [TP2: Desarrollo, compilación y análisis de seguridad](tp2-app-uefi/)
- [TP3: Ejecución en hardware físico](tp3-bare-metal/)
- [Capturas de pantalla de todo el proceso](capturas/)

## Correcciones a la consigna
Durante el desarrollo encontramos algunos problemas en los comandos originales:
1. **QEMU sin disco:** el comando del 1.1 no monta ningún disco, así que `FS0:` no existe. Se agregó `-drive file=fat:rw:$HOME/uefi_security_lab,format=raw`.
2. **Literal de string:** `L "texto"` (con espacio) no compila. Se corrigió a `L"texto"`.
3. **Sección `.rodata` faltante:** el `objcopy` original no copia `.rodata`. Con la versión de gnu-efi de Debian 13, los strings quedan en esa sección, así que la app corría pero no imprimía nada. Se agregó `-j .rodata`.
