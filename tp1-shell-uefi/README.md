# TP1: Exploración del entorno UEFI y la Shell

**Objetivo:** explorar cómo UEFI abstrae el hardware y gestiona la configuración antes de cargar el sistema operativo.

## 1.1 Arranque en el entorno virtual

Se usa QEMU con OVMF, una implementación abierta de UEFI basada en EDK II, como firmware de la máquina virtual. Se monta la carpeta de trabajo como disco FAT para tener un sistema de archivos accesible desde la shell:

```bash
qemu-system-x86_64 -m 512 -bios /usr/share/ovmf/OVMF.fd -net none \
  -drive file=fat:rw:$HOME/uefi_security_lab,format=raw
```

Al arrancar, el firmware cae en la **UEFI Interactive Shell v2.2** y muestra la *Mapping table*:
- **FS0:** la partición del disco con un sistema de archivos reconocido (FAT).
- **BLK0 / BLK2:** dispositivos de bloque crudos.

Cada uno se identifica por su **Device Path** (por ejemplo `PciRoot(0x0)/Pci(0x1,0x1)/Ata(0x0)/HD(1,MBR,...)`), no por un puerto fijo.

![Arranque de la UEFI Shell](img/1.1-arranque.png)

## 1.2 Exploración de dispositivos (handles y protocolos)

```
map
FS0:
ls
dh -b
```

`dh` vuelca la base de datos de handles. Cada handle es una entidad (un disco, un driver, la consola) y tiene asociados uno o más protocolos (interfaces identificadas por GUID). Algunos handles relevantes:

| Handle | Protocolos | Qué es |
|---|---|---|
| 9C | SimpleFileSystem, DiskIO, PartitionInfo, BlockIO, DevicePath | La partición FS0 |
| 9A / 9B | DiskIO, BlockIO, DevicePath (`/Ata(0x0)`) | El disco completo (BLK) |
| 92 | ConsoleOut, SimpleTextOut, GraphicsOutput | Salida de video |
| 98 | SerialIO (`Uart(115200,8,N,1)`) | Puerto serie |
| 9D | Shell, ShellParameters, LoadedImage(Shell) | La propia shell |

Un mismo dispositivo expone varios protocolos: la partición puede usarse como filesystem (`SimpleFileSystem`) o como bloques crudos (`BlockIO`).

![map](img/1.2-map.png)
![ls](img/1.2-ls.png)
![dh](img/1.2-dh.png)

### ❓ Pregunta 1
> ¿Cuál es la ventaja de seguridad y compatibilidad de este modelo frente al antiguo BIOS?

El BIOS Legacy accedía al hardware por puertos de I/O e interrupciones fijas (por ejemplo `INT 13h` para disco), atadas a una arquitectura concreta. UEFI, en cambio, expone el hardware a través de **protocolos estándar**: una aplicación que necesita leer archivos busca un handle con `SimpleFileSystem` y usa esa interfaz, sin saber si debajo hay SATA, USB, NVMe o un disco virtual. Esto da:
- **Compatibilidad:** el mismo binario funciona sobre hardware distinto, y agregar un tipo de dispositivo nuevo solo requiere un driver que instale el protocolo correspondiente.
- **Seguridad:** el acceso pasa por interfaces definidas y descubribles (identificadas por GUID) en lugar de escrituras directas a puertos, lo que permite controlar y auditar qué componente expone qué funcionalidad. Además, los drivers e imágenes pueden verificarse criptográficamente antes de cargarse (Secure Boot).

## 1.3 Variables globales (NVRAM)

```
dmpstore -b
dmpstore BootOrder
dmpstore Boot0000
bcfg boot dump
set TestSeguridad "Hola UEFI"
set -v TestVolatile "chau"
set
```

- `dmpstore` muestra las variables UEFI almacenadas en NVRAM, cada una con nombre, GUID y atributos.
- `bcfg boot dump` muestra las opciones de arranque de forma legible. Por ejemplo, **Boot0002 = "EFI Internal Shell"**, con `DevPath = Fv(...)/FvFile(7C04A583-9E3E-4F1C-AD65-E05268D0B4D1)`. Ese GUID coincide con el de la imagen de la shell vista en el handle 9D del `dh`.
- `set` crea variables de entorno de la shell. Con `-v` la variable es **volátil** y se pierde al reiniciar.

![BootOrder](img/1.3-bootorder.png)
![Boot####](img/1.3-boot0000.png)
![bcfg](img/1.3-bcfg.png)
![set](img/1.3-set.png)

### ❓ Pregunta 2
> Observando las variables Boot#### y BootOrder, ¿cómo determina el Boot Manager la secuencia de arranque?

En la fase BDS, el Boot Manager:
1. Lee la variable **`BootOrder`**: un arreglo de números de 16 bits que indica el orden de prioridad (por ejemplo `0000, 0001, 0002`).
2. Para cada número busca la variable **`Boot####`** correspondiente. Cada una contiene una descripción, atributos y un **Device Path** que apunta a la imagen `.efi` a ejecutar.
3. Intenta cargar esa imagen (`LoadImage` + `StartImage`). Si falla, pasa a la siguiente entrada de `BootOrder`.

En nuestro caso, `Boot0002` apunta a la shell embebida en el firmware volume de OVMF, que es la que terminó ejecutándose.

## 1.4 Footprinting de memoria y hardware

```
memmap -b
pci -b
drivers -b
```

**Mapa de memoria:** cada región tiene un tipo y atributos. Lo más relevante:
- Las regiones `BS_Code` / `BS_Data` tienen atributos `0x000000000000000F`.
- Las regiones `RT_Code` / `RT_Data` tienen `0x800000000000000F`. El bit 63 es **`EFI_MEMORY_RUNTIME`**: le indica al OS que esa memoria sigue en uso después de `ExitBootServices()`.
- Totales: `RT_Code` 256 páginas (1 MB), `RT_Data` 581 páginas (~2,3 MB), y unos 42 MB de `BS_*` que el OS libera al arrancar. Total: 511 MB, coherente con `-m 512`.

![memmap](img/1.4-memmap.png)

**PCI:** lista los dispositivos PCI emulados por QEMU (host bridge, controlador IDE/SATA, video, etc.).

![pci](img/1.4-pci.png)

**Drivers:** se ve el *UEFI Driver Model* en acción. Los drivers Virtio y NVMe están **cargados pero sin dispositivos** (`#D = -`), porque QEMU no emula ese hardware: el driver registra su `Driver Binding Protocol` pero nunca se conecta. Es la carga perezosa (*lazy activation*). En cambio, la cadena `SataController → AtaAtapiPassThru → AtaBus → DiskIo → Partition` sí tiene dispositivos asociados: es el camino hasta FS0.

![drivers](img/1.4-drivers.png)

### ❓ Pregunta 3
> ¿Por qué las regiones RuntimeServicesCode son un objetivo principal para los desarrolladores de malware (Bootkits)?

Porque es **código del firmware que sobrevive al arranque del sistema operativo**. Cuando el bootloader llama a `ExitBootServices()`, la memoria de Boot Services se libera, pero las regiones marcadas con `EFI_MEMORY_RUNTIME` se conservan: el OS las respeta y las mapea en su propio espacio de direcciones para poder invocar los Runtime Services (variables NVRAM, reloj, reset).

Si un bootkit inyecta o modifica código ahí:
- **Persiste** después de que el OS toma el control.
- **Se ejecuta con privilegios de kernel (ring 0)** cada vez que el OS llama a un Runtime Service, por ejemplo al leer una variable.
- Es **difícil de detectar** desde el OS, porque esa memoria es "del firmware" y el antivirus normalmente no la inspecciona.

(Nota: el SMM, o "ring -2", es un entorno distinto, aún más privilegiado, que vive en SMRAM. RuntimeServicesCode corre en el contexto del kernel.)
