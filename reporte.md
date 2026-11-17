# **Planificación de procesos: estudio de** 

# **MLFQ y puesta a punto de XV6**

Universidad del Valle  
Escuela de Ingeniería de Sistemas y Computación

**Integrantes:**  
Margareth Sophya Gamboa Izquierdo  
2518629-3743

Sara Michelle González Posso  
2519548-3743

**Profesor:**  
JOHN ALEXANDER SANABRIA ORDOÑEZ 

**Fecha:**  
Noviembre 2026

![][image1]

1. **Preparar el repositorio y el entorno**

***1.1. Versión de Linux del sistema operativo***  
sara@Sara:\~\$ lsb\_release \-a  
No LSB modules are available.  
Distributor ID: Ubuntu  
Description:    Ubuntu 26.04.1 LTS  
Release: 26.04  
Codename: resolute  
sara@Sara:\~\$

***1.2. Identificador del commit (Commit Hash)***  
sara@Sara:\~\$ cd xv6-riscv  
sara@Sara:\~/xv6-riscv\$ git rev-parse HEAD  
fa2ddcb7ca87a77af579380a6eafd5aff163b44f

***1.3 Herramientas y versiones de compilación***  
sara@Sara:\~\$ riscv64-linux-gnu-gcc \--version  
riscv64-linux-gnu-gcc (Ubuntu 15.2.0-16ubuntu1) 15.2.0  
Copyright (C) 2025 Free Software Foundation, Inc.  
This is free software; see the source for copying conditions.  There is NO  
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

sara@Sara:\~\$ qemu-system-riscv64 \--version  
QEMU emulator version 10.2.1 (Debian 1:10.2.1+ds-1ubuntu3.3)  
Copyright (c) 2003-2025 Fabrice Bellard and the QEMU Project developers  
sara@Sara:\~\$ make \--version  
GNU Make 4.4.1  
Built for x86\_64-pc-linux-gnu  
Copyright (C) 1988-2023 Free Software Foundation, Inc.  
License GPLv3+: GNU GPL version 3 or later \<https\://gnu.org/licenses/gpl.html\>  
This is free software: you are free to change and redistribute it.  
There is NO WARRANTY, to the extent permitted by law.  
sara@Sara:\~\$ git \--version  
git version 2.53.0  
sara@Sara:\~\$

2. **Compilar y ejecutar XV6**

***2.1. Ejecución del sistema con una sola CPU virtual***  
Se compiló el código fuente y se ejecutó el emulador QEMU configurando explícitamente el uso de un solo núcleo de procesamiento (CPUS=1), tal como lo requiere la práctica para aislar el comportamiento del planificador.

sara@Sara:\~\$ cd xv6-riscv  
sara@Sara:\~/xv6-riscv\$ make qemu CPUS=1  
qemu-system-riscv64 \-machine virt \-bios none \-kernel kernel/kernel \-m 128M \-smp 1 \-nographic \-global virtio-mmio.force-legacy=false \-drive file=fs.img,if=none,format=raw,id=x0 \-device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

init: starting sh

***2.2. Verificación de operatividad con comandos básicos***  
Una vez iniciado XV6, se comprobó el funcionamiento del shell mediante la ejecución del comando *ls*, que permitió visualizar los archivos disponibles (salieron, por ejemplo, lo que es el README, cat, echo, etc.) en el sistema de archivos. Además, se utilizó el comando *cat README* para verificar la lectura de archivos de texto, y efectivamente salió el contenido del README

3. **Investigar el planificador original**

***3.1. ¿Qué algoritmo o política de planificación utiliza XV6 por defecto?***  
**Round Robin (Algoritmo):** Método de planificación por turnos. los procesos se organizan en cola según orden de llegada, se selecciona el primer proceso y se le concede la CPU durante un máximo de un quantum, si el proceso termina antes de agotar el quantum sale de la cola, el planificador cede el turno al siguiente proceso y así hasta que todos los procesos terminan.

**codigos:** 

* **usertrap():**

// give up the CPU if this is a timer interrupt.  
if (which\_dev \== 2\)  
yield();  
perdon es que por alguna razon tenia saltos de linea muy grandes, entonces borre el formato xd

* **kerneltrap():** 

// give up the CPU if this is a timer interrupt.  
if (which\_dev \== 2 && myproc() \!= 0\)  
  yield();

* **grep the README &:**

\$ grep the README &  
Version 6 \$( v6).  xv6 loosely follows the structure and style of v6,  
We are also grateful for the bug reports and patches contributed by  
riscv64-softmmu.  Once they are installed, and in your shell  
grep the README &  
Vers\$ion  6 (v6).  xv6 loosely follows the structure and style of v6,  
We are also grateful for the bug reports and patches contributed by  
riscv64-softmmu.  Once they are installed, and in your shell  
grep the README &  
Ve\$rs ion 6 (v6).  xv6 loosely follows the structure and style of v6,  
We are also grateful for the bug reports and patches contributed by  
riscv64-softmmu.  Once they are installed, and in your shell  
grep the README &  
\$V ersion 6 (v6).  xv6 loosely follows the structure and style of v6,  
We are also grateful for the bug reports and patches contributed by  
riscv64-softmmu.  Once they are installed, and in your shell

Las dos funciones anteriores tienen la ruta  kernel/trap.c. En ambas funciones aparece un condicional if que verifica si la interrupción fue generada por el reloj (*which\_dev \== 2*). Cuando esto ocurre, se llama a *yield()* para cederla CPU y permitir que otro proceso pueda ejecutarse. Esto se relaciona con el funcionamiento de Round Robin, ya que xv6 recorre los procesos de forma circular y les asigna turnos de ejecución. Este comportamiento se puede observar al ejecutar varios comandos en segundo plano en QEMU, cuyas salidas se intercalan durante la ejecución.

***3.2. ¿En qué archivo y función se implementa la operación del algoritmo de planificación donde se hace la selección del siguiente proceso a entrar a la CPU?***

**Ruta de archivo:** kernel/proc.c.   
**Función:** scheduler(void)   
**Fragmento:**  
void  
scheduler(void)  
{  
  struct proc \*p;  
  struct cpu \*c \= mycpu();

  c-\>proc \= 0;  
//Se ejecuta un ciclo infinito, dentro se recorre por medio del bucle la tabla global.   
for (;;) {  
    ...  
    int found \= 0;  
    for (p \= proc; p \< \&proc\[NPROC\]; p++) {  
//Por cada entrada,  se adquiere un candado y se evalúa la condición.  
      acquire(\&p-\>lock);  
      if (p-\>state \== RUNNABLE) {  
        p-\>state \= RUNNING;  
//El estado cambia de RUNNABLE → RUNNING, se asigna al núcleo actual y se transfiere el control de la CPU llamando a swtch.  
        c-\>proc \= p;  
        swtch(\&c-\>context, \&p-\>context);  
        ...  
        c-\>proc \= 0;  
        found \= 1;  
      }  
      release(\&p-\>lock);  
    }  
    ...  
  }  
}

***3.3. ¿Cómo se relaciona esa selección del proceso con los estados de los procesos (el proceso que está en la CPU y el proceso que va a entrar a la CPU) y con los eventos que permiten ceder o recuperar la CPU?***  
Se relaciona con los estados en los que se encuentran los procesos, ya que el proceso que está utilizando la CPU debe cederla mediante algún evento, como podría ser una interrupción o una llamada al sistema, o alguna otra cosa. Esto permite que su estado cambie de *RUNNING* a *RUNNABLE*, según corresponda. Luego, el planificador busca otro proceso que se encuentre en estado *RUNNABLE* para asignarle la CPU, cambiando su estado a *RUNNING*. De esta manera, se realiza el cambio entre procesos y se permite que varios de ellos compartan el uso del procesador.

Se puede ver, a modo de ejemplo, como si fuera una fila de fichas de dominó, donde cada ficha (proceso) espera su turno de pie (RUNNABLE) para ser empujada por lo que vendría siendo el planificador. Cuando le toca el turno, cae en ejecución (RUNNING) y toma el control de la mesa. Cuando termina su impulso,  le cede el turno a la siguiente ficha en la fila, permitiendo que el ciclo continúe de manera ordenada.

Esto se respalda con el código fuente del *kernel*, en la función de *scheduler(void)*:

p-\>state \= RUNNING;  
        c-\>proc \= p;  
        swtch(\&c-\>context, \&p-\>context);

También puede observarse este comportamiento en *yield(void)* y en *sleep(void)*.

***3.4. ¿Cómo se incorpora este código del algoritmo de planificación a la versión final del sistema operativo durante el proceso de compilación y construcción de XV6?***  
Se incorpora durante el proceso de compilación de XV6 a través del archivo *Makefile*. Para hacerlo, el código fuente, es decir el *proc.c,* se compila junto con los demás componentes del sistema, como lo es el planificador y demás funciones relacionadas con el manejo de procesos. Después, en el proceso se enlazan los archivos objeto y generan la imagen que se ejecuta del sistema operativo, que puede cargarse en el emulador QEMU para que funcione.

K=kernel  
U=user

OBJS \= \\  
  \$K/entry.o \\  
  (otros componentes)  
  \$K/virtio\_disk.o

4. **Describir MLFQ y explorar su incorporación**

***4.1. Descripción inicial de MLFQ***   
El algoritmo MLFQ (Multi-Level Feedback Queue) es un método de planificación de procesos que utiliza varias colas con diferentes niveles de prioridad. Los procesos se organizan en estas colas y el planificador selecciona primero aquellos que se encuentran en los niveles de mayor prioridad. Cada cola puede tener una cantidad de tiempo asignada para la ejecución de los procesos (a esto se le llama quantum), que puede variar dependiendo de su nivel.

Una de sus principales características es que permite modificar la prioridad de los procesos según su comportamiento durante la ejecución. Por ejemplo, cuando un proceso consume todo el tiempo asignado sin ceder la CPU, puede ser trasladado a una cola de menor prioridad. Por otro lado, los procesos que suelen ceder la CPU antes de agotar su tiempo, como aquellos que realizan operaciones de entrada y salida, pueden conservar una prioridad más alta.

Además, el planificador debe permitir que los procesos de menor prioridad también puedan ejecutarse, evitando que permanezcan esperando indefinidamente. Para esto, se pueden utilizar mecanismos como el aumento periódico de prioridades (que se le llama originalmente como priority boost), que permite devolver los procesos a niveles superiores después de cierto tiempo.

Entre las dudas que nos surgen sobre este algoritmo están los criterios exactos para cambiar un proceso de una cola a otra, el tiempo que debe permanecer en cada nivel de prioridad, la forma en que se determina el quantum de cada cola y cómo se evita que los procesos de menor prioridad sufran de inanición (starvation).

Pero en general, entendemos que MLFQ busca mejorar la distribución del tiempo de CPU mediante la asignación dinámica de prioridades, teniendo en cuenta el comportamiento de los procesos y permitiendo una mejor respuesta para aquellos que requieren atención frecuente.

***4.2. Prompt 1***  
Actúa como tutor socrático de una pareja de estudiantes de cuarto semestre de Ingeniería de Sistemas. Conocemos C++ y hemos comenzado a explorar XV6 para RISC-V.

Queremos identificar y justificar qué se necesitaría para incorporar MLFQ a XV6. En esta entrega no vamos a implementarlo.

**Nuestra descripción inicial del algoritmo, escrita antes de esta conversación:**

El algoritmo MLFQ (Multi-Level Feedback Queue) es un método de planificación de procesos que utiliza varias colas con diferentes niveles de prioridad. Los procesos se organizan en estas colas y el planificador selecciona primero aquellos que se encuentran en los niveles de mayor prioridad. Cada cola puede tener una cantidad de tiempo asignada para la ejecución de los procesos (a esto se le llama quantum), que puede variar dependiendo de su nivel.

Una de sus principales características es que permite modificar la prioridad de los procesos según su comportamiento durante la ejecución. Por ejemplo, cuando un proceso consume todo el tiempo asignado sin ceder la CPU, puede ser trasladado a una cola de menor prioridad. Por otro lado, los procesos que suelen ceder la CPU antes de agotar su tiempo, como aquellos que realizan operaciones de entrada y salida, pueden conservar una prioridad más alta.

Además, el planificador debe permitir que los procesos de menor prioridad también puedan ejecutarse, evitando que permanezcan esperando indefinidamente. Para esto, se pueden utilizar mecanismos como el aumento periódico de prioridades (que se le llama originalmente como priority boost), que permite devolver los procesos a niveles superiores después de cierto tiempo.

Entre las dudas que nos surgen sobre este algoritmo están los criterios exactos para cambiar un proceso de una cola a otra, el tiempo que debe permanecer en cada nivel de prioridad, la forma en que se determina el quantum de cada cola y cómo se evita que los procesos de menor prioridad sufran de inanición (starvation).

Pero en general, entendemos que MLFQ busca mejorar la distribución del tiempo de CPU mediante la asignación dinámica de prioridades, teniendo en cuenta el comportamiento de los procesos y permitiendo una mejor respuesta para aquellos que requieren atención frecuente.

**Lo que averiguamos del planificador original y de su incorporación al sistema compilado:**

1. Algoritmo original: XV6 utiliza por defecto una política de planificación Round Robin sin niveles de prioridad ni múltiples colas, recorriendo secuencialmente el arreglo de procesos.  
2. Selección y cambio de contexto: En el archivo kernel/proc.c, la función scheduler() recorre la tabla proc (cuya estructura struct proc está definida en kernel/proc.h) buscando procesos con estado RUNNABLE. Cuando encuentra uno, cambia su estado a RUNNING, lo asigna a la CPU actual con c-\>proc \= p y realiza el cambio de contexto mediante la función swtch() definida en kernel/swtch.S.  
3. Eventos de liberación de CPU: Un proceso abandona el estado RUNNING por dos vías principales en kernel/proc.c: de forma voluntaria mediante sleep(), pasando a estado SLEEPING al esperar E/S u otro evento; o de forma forzosa mediante yield(), pasando de nuevo a RUNNABLE, función invocada desde el manejo de interrupciones de reloj usertrap() y kerneltrap() en kernel/trap.c, cuando se agota su quantum de tiempo.  
4. Compilación e integración: El código del planificador (kernel/proc.c) se integra al sistema operativo mediante el archivo Makefile en la raíz del proyecto, donde se incluye kernel/proc.o dentro de la lista OBJS. Al ejecutar make, el compilador compila kernel/proc.c a kernel/proc.o y el enlazador lo une con el resto de los módulos para generar la imagen ejecutables kernel que carga QEMU.

Reglas de interacción:

1\. Formula una sola pregunta principal por turno y espera nuestra respuesta. No presentes una lista completa de cambios ni una arquitectura resuelta.  
2\. Empieza examinando nuestra descripción de MLFQ. Ayúdanos a detectar reglas ambiguas, ausentes o contradictorias. No sustituyas nuestra descripción por una versión terminada ni elijas silenciosamente una variante del algoritmo.  
3\. Pídenos que propongamos y justifiquemos las decisiones que faltan. Después, ayúdanos a comprobarlas con ejemplos pequeños de procesos y decisiones de planificación que nosotros debamos resolver.  
4\. Guíanos para relacionar cada regla con XV6. Antes de sugerir una modificación, pregúntanos qué comportamiento necesitamos y qué evidencia del código respalda nuestra propuesta.  
5\. No supongas que tienes acceso a nuestro repositorio. Pide los fragmentos necesarios y distingue lo observado de lo que todavía sea una hipótesis. No inventes archivos, funciones o comportamientos de nuestra versión.  
6\. No entregues código de implementación, parches, una lista cerrada de archivos por modificar ni pseudocódigo que resuelva el planificador. Revisa nuestras propuestas y señala problemas mediante preguntas y pistas graduales. Puedes explicar conceptos previos con ejemplos mínimos.  
7\. Si nos bloqueamos, reduce la dificultad de la pregunta y ofrece una pista concreta. Si pedimos la solución directa, conserva el modo socrático.

El objetivo es que nosotros logremos:

A. Redactar una descripción revisada y coherente de la variante de MLFQ considerada.  
B. Explicar qué cambios necesitaría XV6 para soportarla, indicando para cada uno su propósito, su relación con las reglas y la evidencia del código que lo justifica.  
C. Proponer escenarios para comprobar una futura implementación del algoritmo MLFQ en el sistema operativo XV6. Pídenos que expliquemos qué procesos participarían, qué harían, qué regla comprobaríamos y qué comportamiento esperaríamos observar. No nos entregues los escenarios resueltos. En esta tarea solo los describiremos; no vamos a programarlos ni ejecutarlos.

Mantén el análisis dentro del alcance de una propuesta: no necesitamos implementar MLFQ ni construir un selector de planificadores. Ayúdanos a registrar las cuestiones que queden pendientes de investigación.

Al finalizar, pídenos que redactemos nuestras conclusiones. Comprueba su consistencia mediante una última pregunta aplicada a un caso concreto, sin escribir por nosotros el reporte.

Comienza con una pregunta que permita comprobar cómo entendemos una regla de nuestra descripción inicial de MLFQ.

***4.3. Propuesta de cambios en XV6***  
Para incorporar el algoritmo MLFQ en XV6 proponemos las siguientes modificaciones:

1) En kernel/proc.h (struct proc), agregar *int priority* para los niveles de cola e *int ticks\_used* para contar los ticks consumidos en cada nivel, lo cual atiende la necesidad de almacenar el nivel jerárquico y el historial de uso de CPU de cada proceso, ya que la estructura original no contempla prioridades.  
2) En kernel/proc.c (*allocproc()*), poner cada proceso nuevo en la mayor prioridad inicializando *p-\>priority \= 0* para la mayor prioridad y *p-\>ticks\_used \= 0* al crear un proceso nuevo, así se garantiza que las tareas recién creadas o interactivas reciban respuesta inmediata.  
3) En kernel/proc.c (*scheduler()*), organizar los procesos por prioridad en lugar de ser circular y no tener jerarquías, recorriendo primero los procesos *RUNNABLE* de colas superiores para que tengas preferencia sobre las inferiores.  
4) En kernel/trap.c (*usertrap()* y *kerneltrap()*) junto con kernel/proc.c (*yield()*), contabilizar los ticks en cada interrupción de reloj para rebajar la prioridad (*p-\>priority++*) cuando se agote el *quantum* o umbral asignado y aplicar un *priority boost* constante que devuelva todos los procesos a *priority \= 0*, atendiendo la necesidad de castigar procesos intensivos en CPU y evitar la inanición (*starvation*).

***4.4. Evidencia del diálogo socrático con la IA***  
**Intercambio 1: Definición de cómo se contabiliza el tiempo de CPU (Hipótesis)**  
**Pregunta de la IA:**  
"Fíjense en la frase de su descripción: “cuando un proceso consume todo el tiempo asignado sin ceder la CPU”. Comparen dos lecturas con el mismo proceso A (9 ticks de CPU, se duerme, despierta, otros 9 ticks...): Lectura 1: el tiempo se cuenta por turno. Lectura 2: el tiempo se acumula mientras A permanezca en ese nivel, aunque se duerma entre medias. ¿Cuál de las dos lecturas corresponde a lo que ustedes quieren que haga su MLFQ, y por qué?"

**Nuestra respuesta:**  
"La lectura 2, porque el tiempo de CPU utilizado por un proceso se acumula mientras permanezca en el mismo nivel de prioridad, incluso si cede la CPU o pasa momentáneamente al estado de SLEEPING. Así un proceso que realiza pausas breves para operaciones de E/S no puede evitar la reducción de prioridad solo cediendo el proceso antes de completar el quantum."

**¿Qué demuestra?** El planteamiento de una hipótesis de diseño para prevenir que procesos intensivos en CPU engañen al planificador, cediendo el procesador justo antes de acabar su quantum.

**Intercambio 2: Detección y corrección de una contradicción con procesos interactivos (Corrección)**  
**Pregunta de la IA:**  
"Su texto dice que los procesos que ceden la CPU antes de agotar su tiempo, “como aquellos que realizan operaciones de entrada y salida, pueden conservar una prioridad más alta”. Ahora apliquen la lectura 2 a este proceso B, que es claramente interactivo (corre 1 tick, se duerme esperando una tecla...). Con la lectura 2, ¿qué le ocurre a B con el paso del tiempo? ¿Eso es coherente con la frase de su descripción?"

**Nuestra respuesta:**  
"Creemos que, con esta regla, el proceso B también acumularía tiempo de CPU y después de varias ejecuciones, podría bajar de prioridad, aunque sea interactivo... Y pues esto contradice lo que dijimos anteriormente. Para evitarlo, no basta con contar el tiempo total de CPU, también habría que tener en cuenta cuánto tiempo pasa el proceso ejecutándose y cuánto tiempo permanece esperando. Así que A, que utiliza casi todo el tiempo disponible, podría bajar de nivel, y B, que se detiene, podría quedarse con su prioridad actual."

**¿Qué demuestra?** Autocorrección de una falla lógica en la regla inicial al comprobar qué ocurría con un proceso interactivo real, descubriendo la necesidad de evaluar la proporción entre tiempo de ejecución y tiempo de espera.

**Intercambio 3: Comprobación con ventana de tiempo y decisión de parámetros pendientes (Comprobación)**  
**Pregunta de la IA:**  
"¿Su descripción revisada debe fijar un valor concreto para la ventana y el umbral, o puede dejarlos como parámetros por justificar? Si los fijan, ¿con qué criterio elegirían esos números? Si los dejan abiertos, ¿cómo los registrarían como cuestión pendiente? Piensen en el proceso C (5 ticks corriendo, 1 durmiendo), que da 5/6 (\~83%). Si cambiaran el umbral de 80% a 85%, ¿qué le pasaría a C?"

**Nuestra respuesta:**  
"Sería mejor que la ventana y el % queden como parámetros por definir, porque todavía no tenemos un criterio para elegir sus valores. Tomando tu ejemplo, si el umbral cambia de 80 a 85%, entonces el proceso C, que utiliza aproximadamente 83% de la CPU, cambiaría su comportamiento \[conservando su nivel\]. Ya que su comportamiento depende de esos valores, sería necesario justificarlos antes de establecerlos."

**¿Qué demuestra?** Comprobación numérica de la sensibilidad del algoritmo ante parámetros arbitrarios y registro formal de una cuestión pendiente de investigación para el diseño final.

***4.5. Escenarios de comprobación futura***  
**Escenario \#1:**  
Procesos a participar: Un proceso (P1) que se ejecute continuamente sin llamadas al sistema, y otro proceso (P2) que a diferencia del anterior ejecute en rafagas.

¿Qué harían?: Ambos se iniciarán al mismo tiempo en la cola de máxima prioridad, P1 hace un uso continuo del CPU, mientras que P2 la usa en rafagas breve cediendo voluntariamente la CPU.

Reglas que deseamos comprobar: Regla 3 y Regla 1\.

Comportamiento Esperado: Que P1 cada que se consuman sus quantum de tiempo baje de nivel, mientras que P2 inicie cada vez en la máxima prioridad, interrumpiendo a P1.

**Escenarios \#2:**  
Procesos a participar: Dos procesos intensivos en CPU (P3 y P4) que realizan cálculos continuos sin ceder el procesador, y un proceso interactivo (P5) que realiza pausas frecuentes por operaciones de entrada/salida.

¿Qué harían?: P3 y P4 se inician primero y consumen sus quantums hasta bajar a la cola de menor prioridad, donde compiten por el procesador. Luego, P5 ingresa a la cola de mayor prioridad y se ejecuta de forma intermitente, mientras los otros procesos esperan. Después de un intervalo de tiempo determinado, se activa el *priority boost*.

Reglas que queremos comprobar: Regla 2 y Regla 5

Comportamiento esperado: Cuando P5 no está utilizando la CPU, P3 y P4 se alternan el procesador mediante Round Robin. Al activarse el priority boost, ambos regresan a la cola de mayor prioridad, lo que les permite competir nuevamente por el procesador y evita que permanezcan indefinidamente en las colas inferiores.

**Escenario \#3:**  
Procesos a participar: Dos procesos (P6 y P7) que comienzan su proceso al mismo tiempo y en el mismo nivel de prioridad, con *quantum* definido.

¿Qué harían?: Ambos compiten por el uso de la CPU. Sin embargo, P6 realiza cálculos continuos y consume todo el tiempo asignado en ese nivel (Regla 4a), mientras que P7 ejecuta tareas cortas y cede CPU mediante operaciones E/S antes de agotar su *quantum* (Regla 4b).

Reglas que queremos comprobar: Regla 4a y Regla 4b

Comportamiento esperado: Se espera que la ejecución de P6 cumpla con la regla 4a y se reduzca su nivel de prioridad. En cambio, P7 conservará su nivel de prioridad al ceder la CPU antes de tiempo y volver a estar disponible en estado RUNNABLE.

5. **Conclusiones y reflexiones individuales**

***Reflexión \#1 (Margareth Gamboa):*** En la realización de este trabajo, he logrado entender mejor el funcionamiento de MLFQ, pero siendo más específica el cómo usa la sucesión de procesos, que se realizan en un tiempo limitado, y que este suele manejar la complejidad de estados para equilibrar el tiempo de respuesta. También comprobé al analizar ciertos tramos del código, el como un planificador sencillo no tiene historial ni prioridades, para mayor simplicidad, mientras que si se hiciera con MLFQ se manejaría un mayor nivel de complejidad, y pude darme cuenta de mis errores en cómo creía que funcionaba XV6.

***Reflexión \#2 (Sara González):*** Durante el desarrollo de este trabajo comprendí mejor cómo un sistema operativo organiza los procesos y les asigna tiempo de CPU según sus necesidades, buscando mantener un equilibrio entre el rendimiento y el tiempo de respuesta. Yo entiendo el funcionamiento de MLFQ como una carrera de relevos, donde cada proceso aprovecha su turno en la CPU y cede la barra para que otro pueda continuar. También el diálogo socrático con la IA me ayudó a identificar algunos aspectos que no tenía del todo claros, como la diferencia entre contar el tiempo de CPU por turno o acumularlo, y cómo esto puede afectar el comportamiento del planificador. 

Mi aporte se centró especialmente en la configuración del repositorio y el entorno de ejecución (Partes 1 y 2), y por supuesto en la revisión del planificador original de XV6, así como en el diálogo socrático y la elaboración de propuestas y escenarios de prueba para MLFQ, que fue un trabajo en conjunto. Como aspecto pendiente, me gustaría aprender más sobre cómo se eligen los parámetros del algoritmo, como la duración de los *quantums* y el intervalo del aumento periódico de prioridad. 

***Conclusiones:***  
A partir del método de planificación de XV6 y del análisis de cómo incorporar MLFQ, comprendimos que la planificación de procesos debe mantener un equilibrio entre el funcionamiento del sistema y el tiempo de respuesta. El planificador original de XV6 utiliza *Round Robin* sin manejar prioridades ni llevar un registro del uso de CPU, lo que permite entender cómo se alterna la ejecución de los procesos. Por otro lado, MLFQ permite asignar diferentes prioridades según el comportamiento de cada proceso, diferenciando aquellos que necesitan realizar operaciones de entrada y salida de los que utilizan la CPU durante más tiempo.

También identificamos que adaptar XV6 a MLFQ requiere modificar algunas partes importantes del sistema, como la estructura *struct pro*c en *kernel/proc*. y funciones como *allocproc()*, *scheduler()* y el manejo de las interrupciones del reloj en *kernel/trap.c.* Por último, el análisis nos permitió reconocer que el funcionamiento de MLFQ depende de varios parámetros, como la duración de los *quantums*, la forma de contabilizar el tiempo de CPU y los intervalos del *priority boost*. Estos valores deben elegirse con cuidado para evitar que los procesos interactivos pierdan prioridad o que los procesos de menor prioridad tengan que esperar indefinidamente.

[image1]: <data:image/png;base64,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>