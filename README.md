# Deployment de VMs en Proxmox



> #### Que tipo de imagen es mas conveniente de utilizar para desplegar VMs en Proxmox.
>



## Introducción

Para desplegar máquinas virtuales en Proxmox, la elección más conveniente depende de tu prioridad: **rendimiento** o **flexibilidad**. No hay una opción única para todos los casos, sino un equilibrio entre dos formatos principales.

Estas son las dos opciones que debes considerar y cuándo usar cada una:

| Característica        | **RAW**                                                      | **QCOW2**                                                    |
| :-------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **Rendimiento**       | **Máximo** (menor sobrecarga)                                | Ligeramente inferior al RAW debido a la sobrecarga por metadatos |
| **Aprovisionamiento** | **Delgado (Thin)** solo si el backend lo soporta (ej. ZFS, LVM-thin) | **Delgado (Thin)** nativo, el archivo crece a medida que se escribe datos |
| **Snapshots**         | Depende del **backend** de almacenamiento (ej. ZFS, Ceph)    | **Nativo**, permite crear snapshots internos fácilmente      |
| **Compresión**        | No (depende del backend como ZFS)                            | **Sí**, compresión nativa para ahorrar espacio               |
| **Mejor caso de uso** | **Cargas de producción** donde el rendimiento es crítico, usando un backend que gestione las funcionalidades avanzadas (ZFS, LVM-thin) | **Entornos de desarrollo, pruebas y flexibilidad**, o cuando el backend no ofrece características como snapshots o thin provisioning (ej. almacenamiento en directorio) |

### 📊 Análisis en detalle

- **RAW: La apuesta por el rendimiento puro**
  Este es el formato de disco más simple y directo. Actúa como una representación "cruda" del disco, lo que se traduce en el **mejor rendimiento posible** porque tiene muy poca sobrecarga . Es la opción ideal para cargas de trabajo de producción que exigen el máximo rendimiento de E/S.
  Sin embargo, debes tener en cuenta que **RAW es un formato "tonto"**. No ofrece características avanzadas como snapshots o thin provisioning por sí mismo. Para obtenerlas, tu **backend de almacenamiento** (como ZFS, LVM-thin o Ceph) debe proporcionarlas. En estos casos, obtienes lo mejor de ambos mundos: el rendimiento de RAW con las funcionalidades modernas de tu sistema de archivos .
- **QCOW2: La opción flexible y rica en funciones**
  Este es el formato estrella de QEMU/KVM y ofrece un conjunto de características muy atractivas .
  - **Thin provisioning:** El archivo de disco solo ocupa el espacio que realmente está utilizando la VM, lo que ahorra espacio en el almacenamiento .
  - **Snapshots nativos:** Puedes crear puntos de restauración de la VM de forma sencilla y eficiente sin depender de la capa de almacenamiento .
  - **Compresión:** Puedes comprimir el disco para ahorrar aún más espacio, aunque a costa de un poco de rendimiento .
    La contrapartida es un pequeño **sacrificio de rendimiento** en comparación con RAW, debido a la sobrecarga de gestionar estas características . Históricamente esta diferencia era notable, pero con las mejoras recientes en QCOW2 (como la función de sub-clústeres), esta brecha se ha reducido considerablemente, especialmente en sistemas con almacenamiento rápido como SSDs o NVMe .

### 💡 Recomendación final

- **Para entornos de producción exigentes:** elige **RAW** combinado con un backend de almacenamiento potente como **ZFS** o **LVM-thin**. Así obtendrás el máximo rendimiento sin renunciar a las funcionalidades modernas .
- **Para entornos de desarrollo, pruebas o si necesitas máxima flexibilidad:** elige **QCOW2**. Te dará una gran libertad para gestionar snapshots, ahorrar espacio y moverte entre diferentes almacenamientos sin problemas .
- **Para migraciones desde VMware o Hyper-V:** formatos como **VMDK** o **VHD** son útiles para la **importación**, pero se recomienda **convertirlos a QCOW2 o RAW** para un uso productivo en Proxmox .

Espero que esta guía te ayude a tomar la mejor decisión para tus VMs. Si quieres profundizar en algún aspecto en particular, no dudes en preguntar.



##  Elección del tipo de instalador 

Al momento de elegir la descarga de una imagen para instalar un sistema operativo en un PC/laptop o bare-metal host, es común descargar una `*.iso`  o `*.img` . 

Sin embargo, que cambia si  se trata de desplegar  una maquina virtual en la nube o en algún sistema de virtualizacion basado en un hipervisor?



#### ISO o Cloud images? 

---


> **Supongamos que quiero instalar Alma Linux en Proxmox. **



Para desplegar AlmaLinux en Proxmox, la opción más recomendada es utilizar la **Cloud Image**, combinada con **Cloud-Init**. Este método es el estándar para entornos de nube y automatización, ofreciendo una eficiencia muy superior a la instalación tradicional mediante el ISO de instalación .

Aquí tienes una comparación para que la decisión sea más clara:

| Característica           | **Cloud Image (con Cloud-Init)**                             | **ISO de Instalación**                                       |
| :----------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **Proceso**              | Importar una imagen de disco pre-instalada (`.qcow2`) y configurar los parámetros con Cloud-Init . | Ejecutar el instalador paso a paso, seleccionando idioma, particiones, zona horaria, etc. |
| **Tiempo de despliegue** | **Muy rápido** (de 1 a 2 minutos o menos). Un clon de una plantilla puede arrancar y estar listo en 15-25 segundos . | **Lento** (10-20 minutos) solo para la instalación base, sin contar la configuración post-instalación . |
| **Automatización**       | **Altamente automatizable**. Permite definir usuarios, claves SSH, red y scripts en la primera ejecución . | **Manual**, requiere intervención humana para cada paso o soluciones complejas de automatización como Kickstart. |
| **Consistencia**         | **Alta**. Cada nueva VM es un clon de una plantilla base que mantiene la misma configuración, lo que reduce errores y desviaciones . | **Baja**. Es fácil que las configuraciones varíen ligeramente si el proceso no es idéntico. |
| **Mantenimiento**        | **Muy sencillo**. Basta con descargar la nueva imagen oficial de AlmaLinux y reconstruir la plantilla . | **Complicado**. Para actualizar la "imagen base" hay que repetir todo el proceso de instalación desde cero. |
| **Ideal para**           | **Producción y desarrollo**, especialmente si se necesitan desplegar varias VMs de forma rápida, repetible y estandarizada . | **Casos muy específicos**, como instalaciones con particionado complejo, paquetes no estándar desde el inicio, o cuando se necesita un único servidor y el tiempo no es crítico. |

### 💡 Conclusión y recomendación

Para un uso profesional o eficiente, la **Cloud Image es la opción claramente superior**. Te permite crear una **plantilla (template)** que luego puedes clonar en segundos, inyectando automáticamente la configuración única de cada VM (hostname, IP, usuario, clave SSH) en el primer arranque .

Este es el mismo enfoque que utilizan los grandes proveedores de nube y te ahorrará una cantidad considerable de tiempo y esfuerzo a largo plazo.

Dado que tu intención es instalar AlmaLinux, te sugiero que busques una guía específica para descargar su imagen Cloud (`AlmaLinux-*-GenericCloud-latest.x86_64.qcow2`) y configurarla como plantilla en Proxmox. Este flujo de trabajo es el que te dará la mayor ventaja.

Espero que esta comparación te ayude a elegir. Si quieres, puedo darte más detalles sobre los pasos concretos para descargar la imagen y crear la plantilla.



## Acerca del método de instalacion 

Te voy a guiar paso a paso para instalar AlmaLinux en Proxmox usando una Cloud Image. Este método es rápido, eficiente y te permitirá desplegar nuevas máquinas virtuales en segundos usando la función de plantillas de Proxmox .

El proceso se realiza completamente desde la línea de comandos (shell) de tu servidor Proxmox.

### 🛠️ Prerrequisitos

Antes de empezar, asegúrate de tener:

- Acceso SSH o al shell web de tu servidor Proxmox como usuario `root` o con privilegios `sudo`.
- Una conexión a internet para descargar la imagen.
- Suficiente espacio libre en tu almacenamiento (unos 20 GB recomendados) .

### 📥 Paso 1: Descargar la Cloud Image de AlmaLinux

Primero, accede al shell de tu servidor Proxmox y descarga la imagen oficial de AlmaLinux. La URL para AlmaLinux 9 es la siguiente :


```bash
wget https://repo.almalinux.org/almalinux/10/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2
```


> **Nota:** Si necesitas AlmaLinux 9, cambia el `10` por `9` en la URL .

### 🖥️ Paso 2: Crear una Máquina Virtual (VM) Base

Ahora, crearemos una nueva VM que servirá como base para nuestra plantilla. Es importante **no iniciarla** después de crearla.


```bash
qm create 9000 --name "almalinux-10-template" --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0

# Importante!
qm set 9000 --cpu host
```


- `9000`: Es el ID de la VM. Usar un número alto como `9000` es una buena práctica para identificar plantillas .
- `--name`: El nombre que le darás a la plantilla.
- `--memory`: Memoria RAM en MB.
- `--cores`: Número de núcleos de CPU.
- `--net0`: Configura la interfaz de red. `virtio` es el driver más eficiente y `bridge=vmbr0` asume que tu bridge principal se llama `vmbr0` . Cámbialo si es necesario.

### 💾 Paso 3: Importar y Configurar el Disco

1. **Importar el disco:** Importa la imagen `.qcow2` descargada al almacenamiento de tu VM. Reemplaza `local-lvm` con el nombre de tu almacenamiento (puede ser `local`, `local-zfs`, etc.) .


   ```bash
   qm disk import 9000 AlmaLinux-9-GenericCloud-latest.x86_64.qcow2 local-lvm
   ```

   

2. **Configurar el disco SCSI:** Asigna el disco importado a la VM con el controlador SCSI adecuado.

   ```bash
   qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
   ```
   
3. **Redimensionar el disco (Opcional):** Las Cloud Images suelen ser pequeñas (2-10 GB). Es conveniente aumentarlas.


   ```bash
   qm disk resize 9000 scsi0 32G
   ```


4. **Establecer orden de arranque:** Asegura que la VM arranque desde el disco SCSI.


   ```bash
   qm set 9000 --boot order=scsi0
   ```

   

### ☁️ Paso 4: Configurar Cloud-Init

Cloud-Init es la herramienta que permite personalizar la VM en su primer arranque (usuario, contraseña, red, etc.) .

1. **Añadir el disco de Cloud-Init:** Añade un pequeño disco para que Proxmox pueda inyectar la configuración.


   ```bash
   qm set 9000 --ide2 local-lvm:cloudinit
   ```

   

2. **Configurar la Consola Serial:** Esto es necesario para que la consola de la VM funcione correctamente con las Cloud Images .

   
   ```bash
   qm set 9000 --serial0 socket --vga serial0
   ```

   

3. **Configurar los parámetros de Cloud-Init:** Define el usuario, la contraseña (o mejor, una clave SSH) y la red. Esto es lo que se aplicará a cada clon.

   - **Definir usuario y contraseña:**

   
     ```bash
     qm set 9000 --ciuser admin
     qm set 9000 --cipassword "TuContraseñaSegura"
     ```
   

   - **Inyectar clave SSH (Recomendado):**


     ```bash
     qm set 9000 --sshkeys ~/.ssh/authorized_keys
     ```


​     

     (Asegúrate de que el archivo `~/.ssh/authorized_keys` exista en tu nodo Proxmox).

   - **Configurar IP (DHCP):**


     ```bash
     qm set 9000 --ipconfig0 ip=dhcp
     ```


​     

   - **Configurar IP (Estática):**



     ```bash
     qm set 9000 --ipconfig0 ip=192.168.1.100/24,gw=192.168.1.1
     qm set 9000 --nameserver 8.8.8.8
     ```


​     

### 🎯 Paso 5: Convertir a Plantilla

El último paso es convertir tu VM configurada en una plantilla. Este es el momento mágico que te permitirá crear clones en segundos .


```bash
qm template 9000
```



Una vez hecho esto, el ID `9000` aparecerá como una plantilla en la interfaz web de Proxmox.

### 🚀 Paso 6: Desplegar Nuevas VMs desde la Plantilla

Ahora ya puedes crear nuevas máquinas virtuales instantáneamente a partir de esta plantilla.

- **Clonar la plantilla:**



  ```bash
  qm clone 9000 101 --name "mi-servidor-web" --full
  ```

  

  - `9000`: ID de tu plantilla.
  - `101`: ID para la nueva VM.
  - `--full`: Crea un clon completo e independiente de la plantilla . Si omites esta opción, se creará un "clon enlazado" que ahorra espacio pero depende de la plantilla original.

- **Personalizar la nueva VM antes del primer arranque:** Puedes sobrescribir la configuración de Cloud-Init para esta VM específica, por ejemplo, para darle una IP estática diferente.

  
  ```bash
  qm set 101 --ipconfig0 ip=192.168.1.101/24,gw=192.168.1.1
  ```

  

- **Iniciar la VM:**

  

  ```bash
  qm start 101
  ```

  

Tras el arranque, la VM tendrá la configuración que definiste en los pasos de Cloud-Init (usuario, clave SSH, IP), y podrás acceder a ella inmediatamente sin necesidad de una instalación manual.

> **Un consejo sobre el espacio `/boot`**: Ten en cuenta que algunas versiones de las Cloud Images de AlmaLinux tienen una partición `/boot` pequeña (por ejemplo, 512 MB), lo que puede causar problemas al actualizar el kernel si se acumulan demasiadas versiones . Desde AlmaLinux 9.4 y 8.10, el tamaño se ha aumentado a 1 GB, pero si usas una versión anterior y te encuentras con este problema, puedes ajustar el parámetro `installonly_limit=2` en el archivo `/etc/dnf/dnf.conf` de la VM clonada para limitar el número de kernels antiguos .



## Problemas de espacio de almacenamiento

> #### Que significa esta advertencia durante la creación de una plantilla?
>



```bash 
> qm disk resize 9001 scsi0 40G
```



```plaintext
  Size of logical volume pve/vm-9001-disk-0 changed from 10.00 GiB (2560 extents) to 40.00 GiB (10240 extents).
  WARNING: You have not turned on protection against thin pools running out of space.
  WARNING: Set activation/thin_pool_autoextend_threshold below 100 to trigger automatic extension of thin pools before they get full.
  Logical volume pve/vm-9001-disk-0 successfully resized.
  WARNING: Sum of all thin volume sizes (<365.52 GiB) exceeds the size of thin pool pve/data and the size of whole volume group (237.47 GiB).
```



Este mensaje es **una advertencia muy importante** que no debes ignorar. Te está alertando sobre un problema de **espacio en tu almacenamiento LVM-thin** en Proxmox.



 ⚠️     [Ver el desglose detallado](section/diskspace-warning.md)



## Estrategia de almacenamiento

La imagenes ISOs, backups, CT templates no se almacenan  en el disco del sistema Proxmox, mapeado  en `local-lvm`.  

Para reducir las operaciones de **I/O**, se ha configurado un disco secundario mapeado en la partición `/pool` dejando asi en `local-lvm`, solamente las imágenes de discos de  **VM Disks**  y **CT Volumes**.

 

​     💡   [Ver la estructura de almacenamiento](section/storage-planning.md)



## Recomendaciones con una VM Template



**Escenario:**

1. Se necesita crear 3 VMs a partir de la VM template.
2.  Se quiere regenerar la VM template y almacenar la  imagenes en el disco secundario  montado en `/pool`.?



Por supuesto!. Para crear 3 VMs a partir de una plantilla, lo recomendable es usar **clones completos (full clone)** para asegurar su independencia, y puedes **regenerar tu plantilla para que apunte a `/pool`** para separar su disco de los de las VMs que ya tienes.

### 1) Recomendación para crear 3 VMs a partir de la plantilla

Para crear 3 nuevas VMs a partir de tu plantilla existente, la recomendación es:

- **Usa clonación completa (`Full Clone`)** desde la interfaz web de Proxmox o con el comando `qm clone` con la opción `--full` . Esto crea una copia independiente de la plantilla, que no depende de ella para su funcionamiento .
- **Aplica Cloud-Init para personalizarlas**: Al clonar desde la plantilla, puedes configurar el **usuario, contraseña y dirección IP** mediante Cloud-Init para cada nueva VM, sin necesidad de modificar la plantilla base .

Desde la línea de comandos, el proceso sería así:



```bash
# Clonar la plantilla 9000 para crear 3 VMs independientes
qm clone 9000 101 --name "almalinux-01" --full
qm clone 9000 102 --name "almalinux-02" --full
qm clone 9000 103 --name "almalinux-03" --full

# Personalizar cada clon con Cloud-Init (ejemplo para la 101)
qm set 101 --ciuser admin --cipassword "TuContraseñaSegura" --ipconfig0 ip=dhcp
qm start 101
```



> **Nota**: Puedes usar `--ipconfig0 ip=192.168.1.101/24,gw=192.168.1.1` para IP estática.

### 2) Regenerar la VM template para apuntar a `/pool`

**Sí, es posible mover el disco de tu plantilla a `/pool`**. La forma más directa es **mover el disco de la VM antes de convertirla a plantilla**.

Desde la interfaz web de Proxmox puedes mover el disco de una VM a otro almacenamiento fácilmente, y el formato recomendado para discos de VM es **QEMU image format (qcow2)** para aprovechar el aprovisionamiento ligero y las instantáneas .

#### Pasos para mover y regenerar tu plantilla en `/pool`

1. **Clona la plantilla actual** para tener una copia de trabajo. Por ejemplo, clona la plantilla 9000 en una nueva VM (ej. 9001) .
2. **Inicia la VM clonada y limpia datos sensibles** (opcional). Elimina historial, logs y configura tu usuario con Cloud-Init .
3. **Apaga la VM** y mueve su disco a `/pool`:
   - Desde el menú contextual de la VM, selecciona **Disco duro (scsi0)** → **Acciones de disco** → **Mover almacenamiento** .
   - Elige tu almacenamiento **`pool`** y el formato **`qcow2`** .
   - Marca **"Eliminar el disco de origen"** para liberar espacio .
4. **Convierte la VM en plantilla**:
   - Una vez movido el disco, haz clic derecho sobre la VM y selecciona **"Convertir en plantilla"** .
5. **Reemplaza la plantilla antigua** (opcional). Puedes eliminar la plantilla original (ID 9000) y reemplazarla por la nueva (ej. 9001) .

> **Importante**: No se puede convertir a plantilla una VM con discos en almacenamientos que no soporten imágenes de VM (`images`) . Asegúrate de que `/pool` esté configurado para `images` en Proxmox .

Siguiendo estos pasos, tendrás tus 3 VMs nuevas y una plantilla regenerada en `/pool`, manteniendo `local-lvm` libre para otros discos de VM.

---

## Ejecución de comandos QEMU Guest Agent 

**Sintaxis:**

```bash
 qm guest cmd <vmid> <command>
```

```plainttext
 <command>: <fsfreeze-freeze | fsfreeze-status | fsfreeze-thaw | fstrim | get-fsinfo | get-host-name | get-memory-block-info
       | get-memory-blocks | get-osinfo | get-time | get-timezone | get-users | get-vcpus | info | network-get-interfaces | ping |
       shutdown | suspend-disk | suspend-hybrid | suspend-ram>
```

### Listar estadísticas de  la interfaces de red  ( Ver IP DHCP)

```bash
qm guest cmd 9100 network-get-interfaces [ | grep ip-address ]
```



---



## 💻 Crear Snapshot desde la Línea de Comandos (Vía SSH al Host Proxmox)

1. Conéctate por SSH a tu servidor Proxmox.

2. Lista las VMs para encontrar el **VMID** (primera columna):

  

   ``` bash
   qm list
   ```

   

3. Ejecuta el comando de snapshot. Por ejemplo, para la VM con ID `100`:

  

   ```bash
   qm snapshot 100 pre-practica --description "Estado limpio antes de practicar LVM"
   ```

   

   - `100` es el VMID.
   - `pre-practica` es el nombre del snapshot.
   - `--description` es opcional pero muy útil.
   - **No añadas** `--vmstate` para un snapshot rápido de solo disco (esto equivale a no marcar "Include RAM" en la web).

### ⏪ Cómo Usarlo (El Rescate)

Si algo sale mal (un error en `/etc/fstab` que impide arrancar, un servicio que no levanta, etc.):

1. **Desde la Web**: Ve a `Snapshots`, selecciona el snapshot que tomaste, pulsa **Rollback** y confirma. La VM se detendrá, revertirá su disco al estado guardado y arrancará de nuevo.

2. **Desde CLI**:

   

   ```bash
   qm rollback 100 pre-practica
   ```

   

**Nota Importante**: Un snapshot en Proxmox **no es un backup**. Vive en el mismo almacenamiento que la VM. Es perfecto para "deshacer" cambios rápidos durante tus prácticas, pero no te protege si el disco del servidor Proxmox falla.



## Para listar los snapshots de una máquina virtual (VM)

 En Proxmox, la forma más directa es usando la línea de comandos a través de SSH en el nodo Proxmox.

###  `qm listsnapshot`

El comando `qm listsnapshot` es la herramienta estándar para esta tarea. Necesitas conocer el **ID de la VM** (por ejemplo, 100).



```bash
qm listsnapshot <vmid>
```



Por ejemplo, para la VM 101:



```bash
qm listsnapshot 101
```



La salida mostrará una estructura de árbol con la jerarquía de los snapshots, indicando cuál es el estado actual (`current`)















