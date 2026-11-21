 Cuestionario - Fase 1

1. Con la analogía del edificio, explique qué es un contenedor y en qué se diferencia de una máquina virtual.
   
Un contenedor es como un apartamento dentro de un edificio: cada inquilino tiene sus propias paredes, muebles y privacidad, pero todos comparten los cimientos, las tuberías y la estructura principal (el kernel de Linux). En cambio, una máquina virtual es como construir una casa independiente desde cero en un terreno aparte, con su propio sistema operativo completo, cimientos y consumo pesado de recursos, mientras el contenedor arranca en segundos al reutilizar el kernel anfitrión.

3. En la Práctica 2 el mismo proceso tiene dos PID. ¿Qué namespace lo causa? ¿Cuáles fueron sus dos números?

Lo causa el PID namespace (espacio de nombres de identificadores de proceso), que aísla la tabla de procesos para que el contenedor crea que corre en un sistema independiente. Dentro del contenedor de order-api, el proceso Python tiene el PID 1 (el proceso principal), mientras que visto desde el sistema anfitrión (Ubuntu/WSL2) tiene otro identificador global, el PID 8939.

6. ¿Qué significan los dos números de su cpu.max? ¿Cuánta CPU tiene la API cada 100 ms?

En mi contenedor order-api, cpu.max tiene los valores `25000 100000`. El segundo número (100000 microsegundos) define el período total de medición, que equivale a 100 milisegundos, y el primer número (25000 microsegundos) es la cuota máxima de tiempo de procesador que puede consumir en ese lapso. Esto significa que la API tiene derecho a usar 25 ms de CPU cada 100 ms (es decir, 0.25 de un núcleo).

7. Según su calculos.py, ¿en qué llamada a /ram muere la API? ¿Por qué el exit code será 137?

Según mi `calculos.py`, la API sobrevive a las primeras 3 llamadas a `/ram` (llegando a ~65-69 MiB) y muere en la llamada número 4, donde supera el límite de 75 MiB (o 77 MiB) configurado en el cgroup. El código de salida es 137 porque se calcula sumando 128 + 9: el 128 indica que el proceso terminó por una señal del sistema y el 9 corresponde a SIGKILL, enviada abruptamente por el OOM Killer del kernel para proteger la estabilidad del sistema.

8. ¿Por qué memswap_limit debe ser igual a mem_limit? ¿Qué pasaría si pudiera usar swap?

Ambos límites deben ser iguales para deshabilitar por completo el uso de memoria swap (intercambio en disco) dentro del contenedor. Si el contenedor pudiera usar swap, al agotar su RAM física no moriría inmediatamente por OOM Killer, sino que empezaría a escribir páginas de memoria en el disco duro, degradando drásticamente el rendimiento y falseando la prueba de falla controlada.

9. ¿Qué es un descriptor de archivo? ¿Por qué cada /slow ocupa 2 y cuántas caben a la vez en su API?

Un descriptor de archivo (FD) es un número entero que el kernel asigna a cada recurso abierto por un proceso (archivos, conexiones de red o sockets). Cada petición a `/slow` ocupa 2 FDs porque abre un socket entrante desde Nginx hacia la API y un segundo socket saliente desde la API hacia Redis. Con un límite de 256 FDs y ~4-5 FDs base, en mi API caben máximo 125 o 126 peticiones simultáneas antes de fallar con error EMFILE (*Too many open files*).

10. ¿Qué diferencia hay entre SIGTERM y SIGKILL? ¿Por qué la API necesita un manejador de SIGTERM siendo el PID 1?

SIGTERM (señal 15) es una solicitud amable de cierre que permite al programa guardar datos, cerrar conexiones y terminar limpiamente, mientras que SIGKILL (señal 9) es una orden inmediata del kernel que mata el proceso sin darle oportunidad de reaccionar. En Linux, el kernel protege al PID 1 ignorando las señales SIGTERM por defecto, por lo que `app.py` necesita un manejador explícito para capturar esa señal y detenerse sin esperar los 10 segundos forzados de `docker stop`.


11. Describa el camino de curl localhost:8000/ hasta Redis (Sección 1.4), nombrando el puerto 8000, el bridge y los veth.

Al ejecutar `curl localhost:8000/`, la petición entra por el puerto 8000 mapeado en el host hacia el contenedor gateway (Nginx) a través de su interfaz virtual veth conectada al bridge de Docker (`br-6fd441e5d05f`). Nginx reenvía la solicitud por ese mismo bridge y otro par veth hacia el contenedor order-api, el cual finalmente consulta e incrementa el contador en el contenedor redis usando su respectiva interfaz veth en la misma red virtual.

12. ¿Qué diferencia hay entre un namespace y un cgroup? Dé un ejemplo de cada uno.

Un namespace controla qué puede ver un proceso (aislamiento de visibilidad), mientras que un cgroup (*Control Group*) controla cuánto puede consumir (límites de recursos físicos). Por ejemplo, el PID namespace hace que el contenedor solo vea sus propios procesos empezando desde el PID 1, mientras que el cgroup de memoria (`memory.max`) impide que order-api consuma más de 75 MiB de RAM.


13. ¿Qué diferencias hay entre el hardware de su equipo y lo que ve Linux (Tabla 1)? Si usa WSL2, ¿qué controla .wslconfig?

Mi equipo físico tiene 4 núcleos físicos y 8,0 GB de RAM DDR3 instalada (6,9 GB utilizables en Windows), mientras que Ubuntu en WSL2 ve los 4 núcleos pero con menor memoria RAM disponible asignada dinámicamente por el hipervisor. En WSL2, el archivo `.wslconfig` ubicado en la carpeta de usuario de Windows controla los recursos máximos globales (`memory`, `processors` y `swap`) que la máquina virtual ligera de Linux puede utilizar.
