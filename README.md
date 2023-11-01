# GUIA-DE-EXAMEN Resumen de Ensamblador ARM en Raspberry Pi 

## Capítulo 3

### Memoria

Una computadora tiene una memoria donde .textse almacenan el código (en el ensamblador) y los datos, por lo que debe haber alguna forma de acceder a ellos desde el procesador. Una pequeña digresión aquí, en las arquitecturas 386 y x86-64, las instrucciones pueden acceder a los registros o a la memoria, por lo que podríamos sumar dos números, uno de los cuales está en la memoria. No puede hacer esto en ARM donde todos los operandos deben ser registros.

Cargar datos desde la memoria es un poco complicado porque necesitamos hablar de direcciones .

### Direcciones

Para acceder a los datos debemos darle un nombre. De lo contrario no podríamos referir qué dato queremos. Pero, por supuesto, una computadora no tiene un nombre diferente para cada dato que puede mantener en la memoria. Bueno, de hecho tiene un nombre para cada dato. Es la dirección . La dirección es un número, en ARM un número de 32 bits que identifica cada byte (son 8 bits) de la memoria.

![foto de cabecera](https://thinkingeek.com/wp-content/uploads/2013/01/memoria.png)

>La memoria es como una matriz de bytes donde cada byte tiene su propia dirección.
Al cargar o almacenar datos desde/hacia la memoria, necesitamos calcular una dirección. Esta dirección se puede calcular de muchas maneras.

### Datos
El ensamblador contiene código (llamado texto ) y datos. Las etiquetas en el ensamblador son solo nombres simbólicos de direcciones en su programa. Estas direcciones pueden referirse tanto a datos como a códigos. Hasta ahora hemos usado solo una etiqueta mainpara designar la dirección de nuestra mainfunción. Una etiqueta sólo indica una dirección, nunca su contenido.

Entonces, podemos definir datos y adjuntar alguna etiqueta a su dirección. Depende de nosotros, como programadores ensambladores, garantizar que el almacenamiento al que hace referencia la etiqueta tenga el tamaño y el valor adecuados.

Definamos una variable de 4 bytes e inicialicémosla a 3. Le daremos una etiqueta myvar1.
```
.balign 4
myvar1:
    .word 3
```
Hay dos nuevas directivas de ensamblador en el ejemplo anterior: .baligny .word. Cuando asencuentra una .baligndirectiva, garantiza que la siguiente dirección comenzará con un límite de 4 bytes. Es decir, la dirección del siguiente dato emitido (es decir, una instrucción pero también puede ser un dato) será múltiplo de 4 bytes. Esto es importante porque ARM impone algunas restricciones sobre las direcciones de los datos con los que puede trabajar. Esta directiva no hace nada si la dirección ya estaba alineada a 4. De lo contrario, la herramienta ensambladora emitirá algunos bytes de relleno , que el programa no utiliza en absoluto, por lo que se cumple la alineación solicitada. Es posible que podamos omitir esta directiva si todas las entidades emitidas por el ensamblador tienen 4 bytes de ancho (4 bytes son 32 bits), pero tan pronto como queramos utilizar datos de diferentes tamaños, esta directiva será obligatoria.

### Secciones
Los datos viven en la memoria como el código, pero debido a algunos tecnicismos prácticos, que ahora no nos importan mucho, generalmente se mantienen juntos en lo que se llama una sección de datos . .dataLa directiva le dice al ensamblador que emita las entidades en la sección de datos . Esa .textdirectiva que vimos en el primer capítulo hace algo similar para el código. Entonces pondremos datos después de una .datadirectiva y código después de .text.

Bien, recuperaremos nuestro ejemplo del Capítulo 2 y lo mejoraremos con algunos accesos a la memoria. Definiremos dos variables de 4 bytes myvar1y myvar2, inicializadas a 3 y 4 respectivamente. Cargaremos sus valores usando ldry realizaremos una suma. El código de error resultante debería ser 7, como el del capítulo 2

```ARM assembler
/* -- load01.s */

/* -- Data section */
.data

/* Ensure variable is 4-byte aligned */
.balign 4
/* Define storage for myvar1 */
myvar1:
    /* Contents of myvar1 is just 4 bytes containing value '3' */
    .word 3

/* Ensure variable is 4-byte aligned */
.balign 4
/* Define storage for myvar2 */
myvar2:
    /* Contents of myvar2 is just 4 bytes containing value '4' */
    .word 4

/* -- Code section */
.text

/* Ensure code is 4 byte aligned */
.balign 4
.global main
main:
    ldr r1, addr_of_myvar1 /* r1 ← &myvar1 */
    ldr r1, [r1]           /* r1 ← *r1 */
    ldr r2, addr_of_myvar2 /* r2 ← &myvar2 */
    ldr r2, [r2]           /* r2 ← *r2 */
    add r0, r1, r2         /* r0 ← r1 + r2 */
    bx lr

/* Labels needed to access data */
addr_of_myvar1 : .word myvar1
addr_of_myvar2 : .word myvar2
```
Bueno, cuando el ensamblador emita el código binario, .word myvar1no será la dirección de myvar1sino que será una reubicación . Una reubicación es la forma que utiliza el ensamblador para emitir una dirección, cuyo valor exacto se desconoce pero se sabrá cuando el programa esté vinculado (es decir, cuando se genere el ejecutable final). Es como decir bueno, no tengo idea de dónde estará realmente esta variable, el vinculador parcheará este valor más adelante . Entonces addr_of_myvar1se usará esto en su lugar. La dirección de addr_of_myvar1está en el mismo apartado .text. El vinculador parcheará ese valor durante la fase de vinculación (cuando se crea el ejecutable final y se sabe dónde se ubicarán definitivamente todas las entidades de nuestro programa en la memoria). Esta es la razón por la que se llama al enlazador (invocado internamente por gcc) ld. Significa Link eDitor.

Probablemente se esté preguntando por qué las dos cargas tienen una sintaxis diferente. El primero ldrutiliza la dirección simbólica de addr_of_myvar1la etiqueta. El segundo ldrutiliza el valor del registro como modo de direccionamiento . Entonces, en el segundo caso usamos el valor interno r1como dirección. En el primer caso, no sabemos realmente qué utiliza el ensamblador como modo de direccionamiento, por lo que lo ignoraremos por ahora.

El programa carga dos valores de 32 bits de myvar1y myvar2, que tenían valores iniciales 3 y 4, los suma y establece el resultado de la suma como el código de error del programa en el r0registro justo antes de salir main.

```
$ ./load01 ; echo $?
7
```

### Almacenar

Ahora tome el ejemplo anterior, pero en lugar de establecer los valores iniciales de myvar1y myvar2en 3 y 4 respectivamente, establezca ambos en 0. Reutilizaremos el código existente pero antepondremos algún ensamblador para almacenar un 3 y un 4 en las variables.

```ARM assembler
/* -- store01.s */

/* -- Data section */
.data

/* Ensure variable is 4-byte aligned */
.balign 4
/* Define storage for myvar1 */
myvar1:
    /* Contents of myvar1 is just '3' */
    .word 0

/* Ensure variable is 4-byte aligned */
.balign 4
/* Define storage for myvar2 */
myvar2:
    /* Contents of myvar2 is just '3' */
    .word 0

/* -- Code section */
.text

/* Ensure function section starts 4 byte aligned */
.balign 4
.global main
main:
    ldr r1, addr_of_myvar1 /* r1 ← &myvar1 */
    mov r3, #3             /* r3 ← 3 */
    str r3, [r1]           /* *r1 ← r3 */
    ldr r2, addr_of_myvar2 /* r2 ← &myvar2 */
    mov r3, #4             /* r3 ← 4 */
    str r3, [r2]           /* *r2 ← r3 */

    /* Same instructions as above */
    ldr r1, addr_of_myvar1 /* r1 ← &myvar1 */
    ldr r1, [r1]           /* r1 ← *r1 */
    ldr r2, addr_of_myvar2 /* r2 ← &myvar2 */
    ldr r2, [r2]           /* r2 ← *r2 */
    add r0, r1, r2
    bx lr

/* Labels needed to access data */
addr_of_myvar1 : .word myvar1
addr_of_myvar2 : .word myvar2
```
Tenga en cuenta una rareza en la strinstrucción: el operando de destino de la instrucción no es el primer operando . En cambio, el primer operando es el registro fuente y el segundo operando es el modo de direccionamiento.

## Capítulo 4
A medida que avancemos en el aprendizaje de los fundamentos del ensamblador ARM, nuestros ejemplos se harán más largos. Dado que es fácil cometer errores, creo que vale la pena aprender a utilizar GNU Debugger gdbpara depurar el ensamblador. Si desarrolla C/C++ en Linux y nunca lo usó gdb, la culpa es suya. Si lo sabe, gdbeste pequeño capítulo le explicará cómo depurar el ensamblador directamente.
## GBD
GBD generalmente se refiere a GDB (GNU Debugger), que es una herramienta de depuración de código fuente y ensamblador para programas escritos en varios lenguajes de programación, incluido el ensamblador ARM. GDB permite a los desarrolladores examinar y controlar el flujo de ejecución de un programa, inspeccionar variables, establecer puntos de interrupción y realizar muchas otras operaciones útiles para el desarrollo y la depuración de programas.

Cuando trabajas con ensamblador ARM utilizando GDB, puedes utilizar comandos específicos de GDB para examinar y manipular registros, memoria y otras estructuras de datos específicas de la arquitectura ARM.

Por ejemplo, para ver el valor del registro R0 en ARM ensamblado en GDB, puedes usar el siguiente comando:

```
info registers r0
```
Esto te mostrará el valor actual del registro R0 en el contexto de la ejecución del programa que estás depurando.

## Capítulo 5
### Derivación
Hasta ahora nuestros pequeños programas ensambladores ejecutan una instrucción tras otra. Si nuestro procesador ARM solo pudiera funcionar de esta manera, sería de uso limitado. No podría reaccionar a las condiciones existentes que pueden requerir diferentes secuencias de instrucciones. Este es el propósito de las instrucciones de rama .
### Un registro especial
En el capítulo 2 aprendimos que nuestro procesador ARM Raspberry Pi tiene 16 registros enteros de propósito general, al menos para registrarse r15. Este registro es muy especial, tan especial que también tiene otro nombre: pc. Es poco probable que lo veas usado ya r15que resulta confuso (aunque correcto desde el punto de vista de la arquitectura ARM). De ahora en adelante solo usaremos pcpara nombrarlo.

¿ Qué pc significa? pcsignifica contador de programa . Este nombre, cuyo origen se remonta a los albores de la informática, hoy en día significa poco o nada. En general el pcregistro (también llamado puntero de instrucción , en otras arquitecturas como 386 o x86_64) contiene la dirección de la siguiente instrucción que se va a ejecutar ip.

Cuando el procesador ARM ejecuta una instrucción, pueden suceder dos cosas al final de su ejecución. Si la instrucción no modifica pc(y la mayoría de las instrucciones no lo hacen), pcsimplemente se incrementa en 4 (como si lo hiciéramos nosotros add pc, pc, #4). ¿Por qué 4? Porque en ARM, las instrucciones tienen 32 bits de ancho, por lo que hay 4 bytes entre cada instrucción. Si la instrucción se modifica , se utiliza pcel nuevo valor de .pc

