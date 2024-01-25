---
title: "Entendiendo los descriptores de archivos de Linux: Profundizando en '2>&1' y la redirección"
date: 2024-01-22T11:50:00-06:00
---

Probablemente hayas visto esta sintaxis antes:

```bash
$ command > file 2>&1
```

Esto redirige la salida estándar y la salida estándar de error de `command` al archivo `file`. ¿Pero qué significa? ¿Qué son los descriptores de archivos? ¿Qué es la salida estándar? Descubrámoslo.

## Descriptores de archivos

> Un descriptor de archivo es la abstracción en Unix para un flujo de entrada/salida abierto: un archivo, una conexión de red, una pipe (un canal de comunicación entre procesos), una terminal, etc.[^1]

[^1]: [File Descriptors - Harvard CS61](https://cs61.seas.harvard.edu/site/ref/file-descriptors/)

Cada proceso normalmente tiene 3 descriptores de archivo que están abiertos por defecto y que se heredan del proceso padre (generalmente la shell)

| Valor entero | Nombre            | Constante simbólica de `<unistd.h>` | Flujo de archivo de `<stdio.h>` |
|--------------|-------------------|------------------------------------|--------------------------------|
| 0            | Entrada estándar  | `STDIN_FILENO`                     | `stdin`                        |
| 1            | Salida estándar   | `STDOUT_FILENO`                    | `stdout`                       |
| 2            | Error estándar    | `STDERR_FILENO`                    | `stderr`                       |

Fuente: [File Descriptor - Wikipedia](https://en.wikipedia.org/wiki/File_descriptor)

Cuando un proceso abre un archivo (recuerda que todo en Unix es un archivo, incluyendo dispositivos como la terminal, sockets, pipes, etc.), el kernel asigna un descriptor de archivo a este. Este descriptor de archivo es un número que identifica de manera única al archivo para el proceso.

Internamente, el kernel mantiene una tabla de descriptores de archivo para cada proceso. Esta tabla se llama la tabla de descriptores de archivo. Cada entrada en la tabla contiene información sobre el archivo, como la posición dentro del archivo, los indicadores de estado del archivo, etc.

Cuando un proceso abre un archivo, el kernel devuelve el descriptor de archivo disponible más bajo. Esto significa que el primer archivo abierto por un proceso tendrá el descriptor de archivo 3, el segundo archivo tendrá el descriptor de archivo 4 y así sucesivamente.

Mira el siguiente programa en C:

```c
#include <stdio.h>

int main()
{
    fprintf(stdout, "Estoy escribiendo a stdout\n");
    fprintf(stderr, "Estoy escribiendo a stderr\n");
}
```

El programa imprime dos líneas, una a la salida estándar y otra al error estándar. Vamos a compilarlo y a ejecutarlo:

```bash
$ gcc -o print-fd print-fd.c
$ ./print-fd
Estoy escribiendo a stdout
Estoy escribiendo a stderr
```

Ambas líneas se imprimen en la terminal. Pero, ¿por qué?

## Inspeccionando los descriptores de archivos de un proceso

Para inspeccionar los descriptores de archivo de un proceso, podemos usar el sistema de archivos [`/proc`](https://docs.kernel.org/filesystems/proc.html). Este sistema de archivos es un sistema virtual que proporciona información sobre el sistema y los procesos que se ejecutan en él. Usualmente se monta en `/proc`.

Contiene un directorio para cada proceso en ejecución en el sistema, que es el ID del proceso. Por ejemplo, el directorio para el proceso actual es `/proc/self`.

Cada directorio contiene mucha información sobre el proceso, pero nos interesa el directorio `fd`. Este directorio contiene un enlace simbólico para cada descriptor de archivo abierto por el proceso. El nombre del enlace simbólico es el número del descriptor de archivo y el enlace es el archivo al que el descriptor está apuntando.

```bash
$ ls -l /proc/self/fd
total 0
lrwx------. 1 sebastian sebastian 64 Jan 22 19:28 0 -> /dev/pts/0
lrwx------. 1 sebastian sebastian 64 Jan 22 19:28 1 -> /dev/pts/0
lrwx------. 1 sebastian sebastian 64 Jan 22 19:28 2 -> /dev/pts/0
lr-x------. 1 sebastian sebastian 64 Jan 22 19:28 3 -> /proc/2645/fd
```

Como puedes ver, bash tiene 3 descriptores de archivo abiertos por defecto: 0, 1 y 2. Todos ellos están apuntando al mismo archivo: `/dev/pts/0`. Esto es la terminal donde está ejecutándose el proceso.

## ¿Qué significa 2>&1?

Ahora que sabemos qué son los descriptores de archivo, podemos entender qué significa la sintaxis `2>&1`.

```bash
$ ./print-fd > file.txt 2>&1
```

Esto redirige la salida estándar de `./print-fd` a `file.txt` y redirige el error estándar (descriptor de archivo 2) de `./print-fd` al mismo lugar que la salida estándar (descriptor de archivo 1).

Veamos otros ejemplos:

```bash
$ # Redireccionar el error estándar a /dev/null
$ ./print-fd 2> /dev/null
Estoy escribiendo a stdout

$ # Redireccionar la salida estándar a /dev/null y el error estándar a la terminal actual
$ ./print-fd > /dev/null 2> $(tty)
Estoy escribiendo a stderr
```

## Cómo funcionan las redirecciones

Con esta información, podemos entender cómo funcionan las redirecciones. Veamos el siguiente script de comandos de bash:

```bash
#!/bin/bash
echo "hello" > /tmp/1234
```

Este comando redirige la salida estándar de `echo` al archivo `/tmp/1234`. Veamos qué sucede cuando lo ejecutamos e inspeccionamos los syscalls utilizando `strace`[^2]:

[^2]: [strace](https://strace.io/)

```bash
strace -f -e trace=write,dup2,read,openat ./test.sh
```

> **Nota:** `strace` es una herramienta que nos permite inspeccionar los syscalls que un proceso está realizando. Es muy útil para depurar y entender cómo trabajan los programas. Estamos limitando la salida para mostrar solo los syscalls en los que estamos interesados.

```
...
read(3, "# test.sh\n\n#!/bin/bash\necho \"hel"..., 80) = 48
dup2(3, 255)                            = 255
read(255, "# test.sh\n\n#!/bin/bash\necho \"hel"..., 48) = 48
openat(AT_FDCWD, "/tmp/1234", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
dup2(3, 1)                              = 1
write(1, "hello\n", 6)                  = 6
...
+++ salió con 0 +++
```

1. La primera línea de la salida es la syscall `read`. Está utilizando el descriptor de archivo 3 para leer el script e intentando leer 80 caracteres, pero solo lee 48. Esto se debe a que el script solo tiene 48 caracteres de longitud.
2. Las siguientes 2 líneas son la syscall `dup2`. Está duplicando el descriptor de archivo 3 al descriptor de archivo 255. Esto se debe a que la siguiente syscall también está utilizando el descriptor de archivo 3, por lo que es necesario duplicarlo a otro descriptor de archivo para evitar cerrarlo.
3. A continuación, está la syscall `openat`. Está abriendo el archivo `/tmp/1234` con los indicadores `O_WRONLY|O_CREAT|O_TRUNC`. Esto significa que el archivo se abrirá en modo solo escritura, se creará si no existe, y se truncará a 0 de longitud si existe. Devuelve el descriptor de archivo 3, que era el descriptor de archivo que se liberó con la syscall `dup2` anterior.
4. La siguiente syscall `dup2` está duplicando el descriptor de archivo 3 al descriptor de archivo 1. Esto es porque queremos redirigir la salida estándar de `echo` al archivo `/tmp/1234` (Recuerda que la salida estándar es el descriptor de archivo 1).
5. La siguiente syscall es la syscall `write`. Está escribiendo la cadena `hello\n` al descriptor de archivo 1, que es el archivo `/tmp/1234`.

## Implementando redirecciones desde cero

Entender el proceso que bash sigue para implementar redirecciones es muy útil, pero ¿qué pasa si queremos implementar redirecciones desde cero? ¿Cómo podemos hacerlo?

Para hacer esto, podemos seguir este proceso:

1. Hacer fork al proceso
2. En el proceso hijo, abrir los archivos a redirigir utilizando `open`
3. Utilizando la syscall [dup2](https://man7.org/linux/man-pages/man2/dup.2.html), redirigir la salida estándar y el error estándar a los archivos
4. Usar la syscall [execvp](https://man7.org/linux/man-pages/man3/exec.3p.html) para reemplazar el proceso actual con el nuevo programa.

Veamos cómo funciona esto en la práctica. El siguiente programa en C redirige la salida estándar y el error estándar del nuevo programa a los archivos `/out.log` y `/error.log` respectivamente.

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(int argc, char * argv[]) {
  pid_t pid;
  int status;

  // Hacer fork al proceso
  pid = fork();
  if (pid == -1) {
    perror("fork");
    return 1;
  }

  if (pid == 0) {
    // Estamos en el proceso hijo

    // Abrir el archivo de error, con permisos de solo escritura y crearlo si no existe
    int newerr = open("/error.log", O_WRONLY | O_CREAT, 0666);
    if (newerr == -1) {
      perror("open");
      return 1;
    }

    // Abrir el archivo de salida, con permisos de solo escritura y crearlo si no existe
    int newout = open("/out.log", O_WRONLY | O_CREAT, 0666);
    if (newout == -1) {
      perror("open");
      return 1;
    }

    // Hacer que el descriptor de archivo de stderr apunte al archivo de error
    if (dup2(newerr, STDERR_FILENO) == -1) {
      perror("dup2");
      return 1;
    }

    // Hacer que el descriptor de archivo de stdout apunte al archivo de salida
    if (dup2(newout, STDOUT_FILENO) == -1) {
      perror("dup2");
      return 1;
    }

    // Reemplazar el proceso actual con el programa print-fd
    // Observa que estamos reemplazando el proceso del hijo del fork, no el proceso original
    char * newargv[] = {
      "/print-fd",
      NULL
    };
    execvp(newargv[0], newargv);

    // execvp solo retorna si hay un error
    perror("execvp");
    return 1;
  } else {
    // Estamos en el proceso padre
    // Esperar que el hijo termine
    if (waitpid(pid, & status, 0) == -1) {
      perror("waitpid");
      return 1;
    }

    // Verificar el estado de salida del hijo
    if (WIFEXITED(status)) {
      printf("El proceso hijo salió con el estado %d\n", WEXITSTATUS(status));
    } else {
      printf("El proceso hijo no salió limpiamente\n");
    }
  }

  return 0;
}
```

Ahora, vamos a compilarlo y a ejecutarlo:

```bash
$ gcc -o redirect redirect.c
$ ./redirect
El proceso hijo salió con el estado 0
$ tail -n +1 /*.log
==> /error.log <==
Estoy escribiendo a stderr

==> /out.log <==
Estoy escribiendo a stdout
```

## Conclusión

En este post hemos aprendido qué son los descriptores de archivo, cómo funcionan y cómo implementar redirecciones desde cero. Espero que hayas encontrado útil esta publicación y hayas aprendido algo nuevo.
