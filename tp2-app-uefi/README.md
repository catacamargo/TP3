# TP2: Desarrollo, compilación y análisis de seguridad

**Objetivo:** crear una aplicación UEFI nativa en C, entender el formato PE/COFF y analizar cómo un descompilador interpreta los opcodes.

## 2.1 Desarrollo de la aplicación

El código está en [`aplicacion.c`](aplicacion.c). La app:
1. Declara un arreglo `volatile` con el byte `0xCC` (opcode de `INT3`, la instrucción de breakpoint).
2. Limpia la pantalla con `ConOut->ClearScreen`.
3. Imprime un mensaje con `ConOut->OutputString`.
4. Si `code[0] == 0xCC`, imprime un segundo mensaje.
5. Espera 5 segundos con `BootServices->Stall`.

Todas las llamadas pasan por `uefi_call_wrapper`, que convierte la convención de llamadas de Linux (System V) a la de Microsoft x64, que es la que usa UEFI.

### ❓ Pregunta 4
> ¿Por qué utilizamos `SystemTable->ConOut->OutputString` en lugar de `printf`?

Porque en el entorno pre-OS **no existe un sistema operativo ni una libc**. `printf` depende de la libc, que a su vez usa syscalls del kernel (`write` sobre stdout), y nada de eso existe en UEFI. Lo único que recibe la aplicación es el `ImageHandle` y la **`EFI_SYSTEM_TABLE`**, que contiene los punteros a todos los servicios del firmware. La salida de texto se hace a través del protocolo `SIMPLE_TEXT_OUTPUT` (`ConOut`).

Esto se confirma en el análisis del binario: el *Import Directory* del PE está vacío. La app no importa ninguna biblioteca y todo le llega por la System Table en tiempo de ejecución.

## 2.2 Compilación a PE/COFF

La compilación está en [`build.sh`](build.sh) y tiene tres pasos:
1. **`gcc`:** compila a objeto, con flags específicos para un entorno *freestanding* (sin libc, sin stack protector, sin red zone, `wchar_t` de 16 bits).
2. **`ld`:** linkea con el linker script y el `crt0` de gnu-efi y genera un `.so` (ELF).
3. **`objcopy`:** convierte el ELF a **PE/COFF** (`--target=efi-app-x86_64`).

> ⚠️ **Corrección:** el `objcopy` de la consigna no incluye la sección `.rodata`. Con la versión de gnu-efi de Debian 13, los strings constantes quedan en esa sección, así que **la app corría pero no imprimía nada**: limpiaba la pantalla, esperaba 5 segundos y volvía. En Ghidra se veía que el string se cargaba desde `0xe000` (`LEA RDX,[0xe000]`), pero esa sección no existía en el `.efi`. Se agregó `-j .rodata` y se solucionó.

![Compilación](img/2.2-compilacion.png)

## 2.3 Análisis de metadatos y decompilación

### Metadatos

```bash
file aplicacion.efi
readelf -h aplicacion.efi
readelf -h aplicacion.so
objdump -x aplicacion.efi
```

- `file`: **PE32+ executable for EFI (application), x86-64**. Es formato Windows aunque se compiló en Linux.
- `readelf` sobre el `.efi` falla ("no es un fichero ELF"); sobre el `.so` sí funciona (magic `7f 45 4c 46` = `\x7fELF`). Evidencia de la conversión que hizo `objcopy`.
- Campos del header PE relevantes:

| Campo | Valor | Significado |
|---|---|---|
| Magic | `0x020b` | PE32+ (64 bits) |
| Subsystem | `0x0a` | EFI Application (un driver sería `0x0b`/`0x0c`) |
| AddressOfEntryPoint | `0x3000` | Punto de entrada |
| ImageBase | `0x0` | Sin dirección fija: es relocatable |
| Base Relocation Dir. | `0xa000` | Tabla de *fix-ups* que aplica `LoadImage()` |
| Import Directory | vacío | No importa bibliotecas |
| Security Directory | vacío | **No está firmado**, por eso en bare metal hay que desactivar Secure Boot |


### Ghidra

Al importar el `.efi`, Ghidra lo reconoce como PE y muestra los headers `MZ` (`4d 5a`) y `PE` (`50 45`). El análisis arranca en `entry` (`0x3000`), que **no es `efi_main`**: es el `crt0` de gnu-efi, que llama a `_relocate()` (aplica las relocaciones) y después a `_entry()`, que llama a `efi_main`.

En `efi_main` se observa:
- Las llamadas aparecen como `efi_call1()` / `efi_call2()`: son los *thunks* de `uefi_call_wrapper` (el número indica la cantidad de argumentos). Ghidra no los resuelve, por eso no muestra los argumentos.
- **El `if` desaparece del pseudocódigo.** Ghidra ignora el `volatile`: ve que `local_9` se escribe con `0xCC` y deduce que la condición siempre es verdadera, así que la elimina o la muestra como `if (true)`. Sin embargo, la comparación **sí está en el binario**:

```
00003039  c6 45 ff cc   MOV    byte ptr [RBP + local_9], 0xcc
00003088  0f b6 45 ff   MOVZX  EAX, byte ptr [RBP + local_9]
0000308c  3c cc         CMP    AL, 0xcc      → Ghidra: CMP AL, -52
0000308e  75 29         JNZ    LAB_000030b9
```


### ❓ Pregunta 5
> En el pseudocódigo de Ghidra, la condición `0xCC` suele aparecer como `-52`. ¿A qué se debe y por qué importa en ciberseguridad?

**Por qué:** `0xCC` es `11001100` en binario. Como entero sin signo vale 204, pero si se interpreta como `char` **con signo** (complemento a 2) vale 204 − 256 = **−52**. El byte es el mismo (en el opcode sigue apareciendo `3c cc`); lo que cambia es el tipo que le asigna el decompilador. Ghidra, sin información de tipos, asume `char` con signo.

**Por qué importa:** `0xCC` es el opcode de `INT3`, el breakpoint por software. Malware y protecciones anti-debug buscan este byte para detectar si están siendo depurados, y los analistas lo buscan para encontrar esa lógica. Si el decompilador lo muestra como `-52`, una búsqueda de `0xCC` o de `204` en el pseudocódigo no lo encuentra. Moraleja: **el pseudocódigo es una interpretación**. En análisis de seguridad hay que contrastarlo con los bytes y el assembly reales. Lo mismo pasó con el `if` eliminado: el decompilador "optimizó" una verificación que en el binario existe.

## 2.4 Ejecución en QEMU

```
FS0:
ls
aplicacion.efi
```

La pantalla se limpia, se imprimen los dos mensajes y, tras 5 segundos, vuelve el control a la shell (la app retorna `EFI_SUCCESS`).
