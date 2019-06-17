# Orga2 - Segundo Parcial

| Hexa | Binario| | Hexa | Binario | | Hexa | Binario | | Hexa | Binario |
|------|--------|-|------|---------|-|------|---------|-|------|---------|
| 0    | 0000   | | 4    | 0100    | | 8    | 1000    | | C    | 1100    |
| 1    | 0001   | | 5    | 0101    | | 9    | 1001    | | D    | 1101    | 
| 2    | 0010   | | 6    | 0110    | | A    | 1010    | | E    | 1110    |
| 3    | 0011   | | 7    | 0111    | | B    | 1011    | | F    | 1111    |

|Power|Exact Value      |Approx value|Bytes   |Hex de a bytes|
|-----|      ---:       |------------|---:    |     ---:     |
|7    |              128|            |        |              |
|8    |              256|            |  1 byte|            80|
|10   |             1024| 1 thousand |  1 KB  |         04 00|
|12   |             4096|            |  4 KB  |         10 00|
|16   |           65,536|            | 64 KB  |      01 00 00|
|20   |        1,048,576| 1 million  |  1 MB  |      10 00 00|
|30   |    1,073,741,824| 1 billion  |  1 GB  |   40 00 00 00|
|32   |    4,294,967,296|            |  4 GB  |01 00 00 00 00| 
|40   |1,099,511,627,776| 1 trillion |  1 TB  |              |



## Mapa de memoria

Recordar que límite es tamaño - 1.

En las entradas de GDT tenemos 20 bits en total para escribir el límite, entonces el número más grande que podemos escribir, con G=0 en la GDT entry, es 2^20 - 1 = 1MB - 1 = 0xFFFFF.

En cambio, si en la GDTe ponemos G=1, el límite se interpreta como cantidad de páginas de 4KB. De esta forma el tamaño máximo de segmento que podemos describir, pasa a ser de 1MB x 4KB = 4GB.

El límite es relativo a la base. Es decir que si tenemos una base de 1GB y un límite de 1,5GB - 1, el tamaño *no* es de 0,5GB, sino de 1,5GB.

Errores posibles:
* Limite
* Bit presente (da un Page Fault)
* Nivel de privi (General Protection Fault)
* Type (como querer leer código solo ejecutable) (GP Fault)

### Registros de segmento

CS: Code segment
DS, ES, GS, FS: Data segment
SS: Stack segment

### Registros implícitos

```mov eax, [ds : ecx]```

```call cs : eax```

ret usa CS 

push y pop usan SS


Memory operands using esp or ebp as the base register default to SS

[//]: # (Más info: https://unix.stackexchange.com/a/109654)

### Direccionamiento

#### 16 bits

![dir_en_16_bits](https://i.imgur.com/eLrQhtq.png)

En modo real, los registros tienen solo 16 bits, así que solo se puede direccionar 64 k (2^16 = 65.536 bits ~= 64 KB)

En x86 Real-Mode Memory, la dir física es de 20 bits y se calcula:

PhysicalAddress = Segment * 16 + Offset

tenés 16 bits y necesitás 20 bits (para direccionar 1mb) (ta falta 1 nibble, entonces lo shifteas 1 nibble a la izq)

Because of the way the segment address and offset are added, a single linear address can be mapped to up to 2^12 = 4096 distinct segment:offset pairs.

#### 32 bits

Cómo pasar de dire virtual/lógica a dire lineal:
![unidad_de_seg](https://i.imgur.com/gEa9Nl2.png)

Cómo interpretar el selector de segmento:
![info del selector de segmento](https://i.imgur.com/DME8z1P.png)

Si no está activada la paginación, las direcciones lineales son iguales a las direcciones físicas (identity mapping)


![descriptor_de_segmento](https://i.imgur.com/kJ0lykk.png)

|Atributo          |Significado                                                                                    |Tam en bits|Valor default      |
|------------------|-----------------------------------------------------------------------------------------------|-----------|-------------------|
|l (large)         |64 bits? (0 = 32 bits)                                                                         |    1      | 0                 |
|avl               |Disponible                                                                                     |    1      | 0                 |
|base              |Base segmento                                                                                  |   32      | -                 |
|d/b (default/big) |Operation size (0 = 16 bit segment, 1= 32)                                                     |    1      | 1                 |
|dpl               |Descriptor privilege level (kernel = 00, user = 11)                                            |    2      | -                 |
|g                 |Granularity (0 = 1 byte, 1 = 4 KB)                                                             |    1      | 1 (páginas de 4KB)|
|limit (g = 0)     |Límite segmento (tamaño en bytes - 1): lineal - base + bytes a leer\|escribir\|ejecutar - 1    |   20      | -                 |
|limit (g = 1)     |Límite segmento (tamaño en páginas - 1):  \[(lineal + cant_de_bytes - base) / 4K\] - 1         |   20      | -                 |
|p                 |Presente                                                                                       |    1      | 1                 |
|s (system)        |Tipo descriptor (codigo/datos = 1, tss = 0)                                                    |    1      | -                 |
|type              |Tipo segmento                                                                                  |    4      | -                 |


![tipos de selectores de segmentos](https://i.imgur.com/pGzOWoT.png)

A verificar, pero: pareciera que expand-down es que el segmento empieza en limit+1 y termina en base.


## Paginacion

![big_picture](https://i.imgur.com/axjHSc2.png)

La unidad de paginación recibe una dir lineal y devuelve una física:




![dir_lineal](https://i.imgur.com/GD1nK36.png)

En paginación, para las direcciones base, siempre (tanto en el cr3, como en las pd y pt entries) tomamos los primeros 20 bits y los shifteamos 12 a la izquierda (para que sea una dirección de 32 bits).

Cuando tenemos una dirección física y nos piden leer/escribir/ejecutar una determinada cantidad de bytes, tenemos que fijarnos si no va a empezar a leer/escribir/ejecutar en una página y seguir en otra. Esto lo hacemos viendo la dirección física: los últimos 3 nibbles (chars hexa) son el offset dentro de la página, y los anteriores indican en qué página estamos (las páginas están alineadas a 4KB). Si nos pasamos del offset 0xFFF, hay que seguir en otra página (creando su PTe correspondiente).

Dice noit:
> el address space lineal esta dado por las tablas, en orden
> de cada entrada
> si arrancas a escribir en una direccion lineal y te quedas sin pagina vas a seguir escribiendo en la pagina apuntada por la entrada que sigue
> y si es la ultima entrada de esa PTE es la primera > entrada de la siguiente entrada de la PDE
> y asi

El rango de índices en las Page Directories y Page Tables van del 0x000 al 0x3FF (son 1024 posibles índices). Los PD y PT ocupan 1 page de 4KB cada uno.

Los índices de pd y pt, se multiplican por 4, ya que eso es lo que ocupa cada pd/pt entry (4 bytes = 32 bits).

Los últimos 12 bits de la dire lineal, son el offset a byte dentro de la página. No hace falta multiplicarlo por nada.


![pd_entry](https://i.imgur.com/2iYWOTh.png)

![pt_entry](https://i.imgur.com/BMgrTHa.png)

(Page table attribute index no importa)




### Control registers:
**CR0**: bit 0 de modo protegido (PE), bit 31 de paginación (PG)

CR1: Reservado 

CR2: Page Fault Linear Address (PFLA): When a page fault occurs, the address the program attempted to access is stored in the CR2 register.

**CR3**: The upper 20 bits of CR3 become the **page directory base register** (PDBR), which stores the physical address of the first page directory entry. If the PCIDE bit in CR4 is set, the lowest 12 bits are used for the process-context identifier (PCID)

CR4: Used in protected mode to control operations such as virtual-8086 support (PAE), enabling I/O breakpoints, page size extension (tamaño pages) and machine-check exceptions. Bit 9 (OSFXSR) activa SSE.


### Des/mapear página

```c
void mmu_mapPage(uint32_t virtual, uint32_t cr3, uint32_t phy, uint32_t attrs) {
  int32_t pdedir = PDE_INDEX(virtual); // Índice del PD donde queremos mapear la página
  int32_t ptedir = PTE_INDEX(virtual); // Índice del PT dónde queremos mapear la página

  pd_entry* pdbase = (pd_entry*) cr3; // puntero a la base del PD
  pd_entry* pdt    = &pdbase[pdedir];

  if (pdt->p == 0) { // si la entrada de la PD no está presente, pide una página para la PT y la setea:
    pt_entry* ptableDir = pdt->table_base_addr
                          ? (pt_entry*) (pdt->table_base_addr << 12)
                          : (pt_entry*) mmu_nextFreeKernelPage();
    pdt->p               = PG_PRESENT;
    pdt->r_w             = PG_READ_WRITE;
    pdt->u_s             = (virtual < 0x3FFFFF ? 0 : PG_USER);
    pdt->table_base_addr = ((uintptr_t) ptableDir) >> 12; // tiene que ser de 20 bits
    pdt->pwt,pcd,a,zero,ps,g,avl = 0;
  }

  // seteamos la entrada de la tabla apuntada por la dire virtual recibida por param:
  pt_entry* pte = &( (pt_entry*)(((uintptr_t) pdbase[pdedir].table_base_addr)<<12) )[ptedir]; // lo pasamos a 32bits para poder hacer el cast a pt_entry
  pte->p              = PG_PRESENT;
  pte->r_w            = ((attrs & 2) >> 1); // queremos el bit 1 de attrs
  pte->u_s            = ((attrs & 4) >> 2); // queremos el bit 2 de attrs
  pte->page_base_addr = phy >> 12;
  pte->pwt,pcd,a,zero,pat,g,avl = 0;

  // invalidamos la cache de traducciones:
  tlbflush();
}
```

```c
uint32_t mmu_unmapPage(uint32_t virtual, uint32_t cr3) {
  // Índice del PD donde queremos mapear la página:
  int32_t pdedir = PDE_INDEX(virtual);
  // Índice del PT dónde queremos mapear la página:
  int32_t ptedir = PTE_INDEX(virtual);
  // Puntero a la base del PD:
  pd_entry *pdbase = (pd_entry*) cr3;
  // Entrada del PD indicada por la dirección virtual:
  pd_entry pdtEnt = pdbase[pdedir];
  // No está mapeada, nada que hacer:
  if (pdtEnt.p == 0) { return 0; }

  pt_entry *pt = (pt_entry*) (pdtEnt.table_base_addr << 12);
  // No está mapeada, nada que hacer:
  if (pt[ptedir].p == 0){ return 0; }
  // Desmapeo:
  pt[ptedir].p = 0;
  // Fuerzo a recargar la información de página:
  tlbflush();
  return 0;
}
```

---

## Permisos

### En segmentación

![bit_p_limites](https://i.imgur.com/6KZyTgh.png)


| Sigla| Significado               | Explicación                                                                               | 
|------|---------------------------|-------------------------------------------------------------------------------------------|
| DPL  | Descriptor Privilege Level| El nivel de privilegio del descriptor de segmento seleccionado por la dir lógica en la GDT|
| CPL  | Current Privilege Level   | El DPL en la GDT del descriptor de segmento de código que estoy ejecutando (el seleccionado por CS)|
| RPL  | Requested Privilege Level | El nivel con el que pretendo acceder al segmento (vía dirección lógica)                   |
| EPL  | Effective Privilege Level | El menos privilegiado entre CPL y RPL (el máximo numérico)                                |

#### Segmento de datos

Puedo acceder si el efectivo es igualmente o más privilegiado que el de la dir lógica en la GDT (EPL ≤ DPL).

#### Segmento de código

##### Non-conforming

  Para acceder al segmento, el efectivo tiene que ser igual al del segmento (EPL = DPL).

##### Conforming

> Intel made has the favor of introducing also Conforming Code Segment (CCS) that can be accessed by less privileged applications (in case the kernel need to share some code without elevating the privilege of the caller).
[Fuente](https://stackoverflow.com/questions/31074904/privilege-level-checking-when-accessing-code-segment)

  Para acceder al segmento, el segmento seleccionado por la dir lógica tiene que ser igual o más privilegiado que el código que estoy ejecutando (para que tareas puedan acceder a servicios del sistema sin tener que elevar los privilegios de la tarea, por ejemplo) (DPL ≤ CPL).

#### Chequeo de bits de tipo

1. Chequeo el bit de sistema (S) en el descriptor de segmento (que vive en la GDT)

2. Chequeo los bits de tipo del segmento:

    * Los datos siempre se pueden leer, el código solo si tiene lectura habilitada.

    * El código nunca se puede escribir, los datos solo si tienen escritura habilitada.

    * Los datos no se pueden ejecutar.

### En paginación

Hay que combinar los atributos de las entradas de PD y PT indicadas por la dirección lineal.

1. Reviso que las entradas de PD y PT estén presentes.

2. Chequeo permisos de acceso:

    * Si la PDe o la de PTe tienen Supervisor (U/S = 0), la página solo es accesible por supervisor.
    * Si ambas entradas tienen User, la página es accesible por user y por supervisor.
    * Solo si ambas entradas tienen Read/Write es posible escribir en la página (R/W=1).

En las PDe conviene poner siempre U/S=1 y R/W=1, y manejar los permisos más finamente en las PTe.


### Sintaxis PDe/PTe (3 bits más bajos)

|U/S|R/W| P |Hex| Privilege |
|---|---|---|---| --------- |
|  0|  1|  1|0x3| supervisor|
|  1|  1|  1|0x7| user      |

### En tareas

Para los descriptores de TSS (que viven en la GDT), ponemos el bit S=0 (sistema, no code/data).


* P = 1
* CPL &leq; DPL (en la GDT)
* B = 0 (no se puede saltar a una tarea que esté busy)

Si algo de esto no se cumple: GP Fault


### En interrupciones

* P = 1
* CPL &leq; DPL (de la IDT)
* Si es de hardware: DPL (de la IDT) = 0

Si algo de esto no se cumple: GP Fault

El DPL dice quién puede llamar a la interrupción. No con qué nivel se ejecuta la ISR.

Noit tip a descifrar: Las entradas de la IDT tienen dos DPL, uno el que esta dentro de su selector de segmento que indica con que privilegio se pretende ejecutar la rutina (Generalmente va lo mismo que el cs del kernel), y otra dentro de la entrada misma que indica el privilegio necesario para llamar a dicha interrupción, este ultimo es el que discrimina que es una syscall de que no lo es.

## Interrupciones

Hay un IDTR con el puntero a IDT y su límite.

![esquema_IDT](https://i.imgur.com/lXoRySs.png)

![esquema_Gates](https://i.imgur.com/vNYEVsO.png)



El trap gate es como el interrupt, pero no deshabilita interrupciones.

El task gate produce un cambio de tarea, con todo lo que eso implica (cambio de TR, cambio de CR3/CPD...), y tampoco deshabilita interrupciones.

Interrupciones y traps te guardan el contexto de la tarea interrumpida automaticamente. En las task gates hay que guardar a mano. Puede ser más barato si solo nos interesa guardar una parte del contexto.

Traps son más de software, ints son más de hardware. Estas últimas te deshabilitan interrupciones (eflags.if=0), las primeras no.

El selector de segmento de int o trap va a ser de código nivel 0 o nivel 3 respectivamente. De la GDT sacamos la base y al sumarle el offset de la IDT entry, sabemos dónde empezar a ejecutar. gdt[N].base ⊕ idt[M].límite debe ser menor a gdt[N].límite.

Para manejar una interrupción debemos:

1. Preservar los registros que vayamos a romper (la interrupción debe ser transparente)
2. Comunicar al PIC que ya se atendió la interrupción: **call pic_finish1**
3. Realizar la tarea correspondiente a la interrupción
4. Restaurar los registros
5. Retornar de la interrupción (**iret**)

![stack_en_interrupciones_fixed](https://i.imgur.com/jKcQ32b.png)

Si soy una tarea de nivel 3 y me llega una interrupción, va a haber cambio de privilegios.

El ESP de nivel 3 se va a pushear en la pila de nivel 0 de la tarea (que está apuntada por TSS.esp0).

El ESP del pushad del ISR, es el de la interrupción (nivel 0).

```asm
; pseudo-código para obtener esp de nivel 3:
xor  eax,  eax ; limpio eax
str   ax       ;  ax = índice descriptor de tss de la tarea cuyo esp3 busco (`str ax` pondría el de la tarea actual)
sgdt ebx       ; ebx = puntero a gdt
mov  ebx, [ebx + eax * size_gdte] ; ebx = gdt[idx_tss]
mov  ebx, [ebx + offset_base]     ; ebx = gdt[idx_tss].base = tss[tarea_buscada]
mov  ebx, [ebx + offset_esp0]     ; ebx = tss[tarea_buscada].esp0
mov  ebx, [ebx + 4 * cant_pushes] ; ebx = esp_nivel3
cant_pushes  EQU 4                ; creemos
```

Si busco el esp3 de la misma tarea que fue interrumpida, sigo en el mismo contexto, entonces es `mov eax, [esp + 4 * cant_pushes]`.

Quizás haya que sumarle los registros de `pushad` a `cant_pushes`.


### Reloj (32)

```asm
global _isr32

_isr32:
  pushad ; push EAX, ECX, EDX, EBX, ESP*, EBP, ESI, EDI
    call pic_finish1
    call sched_nextTask ; ax = nextTask

    str cx ; store tr: cx = tr
    cmp ax, cx ; if (tr != nextTask)
    je .fin ; este chequeo es para no saltar sí misma (que está busy)
      mov [selector], ax
      jmp far [offset]
    .fin:
  popad ; pop EDI, ESI, EBP, (ESP), EBX, EDX, ECX, EAX
iret

; *: el ESP es el anterior a hacer el primer push
```


### Teclado (33)

* Leemos del teclado a través del puerto 0x60 (**in al, 0x60**).
* Obtenemos un scan code (make code cuando se aprieta, break code cuando se suelta; break code = make code + 0x80)

* Llamar a pic_finish1


```asm

extern isr_kb_c
global _isr33:
_isr33:
  ; Resto 0x82. El scancode numérico más chico, corresponde a '1'.
  ; Chequeo bounds: res < 0 || res >= 10 no hago nada.
  ; Leo el valor de la tabla (1..9 U 0, en ese orden).
  ; Imprimo '0' + valor.
  pushad
  pushfd ; en el tp no hacía falta, pero más vale ponerlo

  xor eax, eax
  in al, 0x60

  push eax ; pusheo el param de isr_kb_c
  call isr_kb_c
  add esp, 4 ; restauro pila

  call pic_finish1
  popfd
  popad
  
iret
```

```c

void isr_kb_c(uint8_t scancode) {

  if (scancode == 0x15) {
    /* Toggle modo debug.
     * Durante pantalla de debug no se atienden interrupciones.
     */
    debug_on = !debug_on;
    uint8_t color_cartel_debug = debug_on != 0 ? 0x0F : 0x00;
    print("MODO DEBUG: ON", 50, 49, color_cartel_debug);
  }

  if (scancode >= 0x82 && scancode < 0x8C) {
    scancode -= 0x82;
    scancode +=    1;
    scancode %=   10;
    print_dec(scancode, 1, VIDEO_COLS - 1, 0, C_FG_WHITE | C_BG_BLACK);
  }

  // Coloquio:
  // Setea la var global
  // Después en housekeeping nos encargamos de matar la tarea y resetear la var
  if (scancode == 0x1E) { // A
    matar_A = 1;
  }
  if (scancode == 0x30) { // B
    matar_B = 1;
  }
}
```





## Tareas

Una tarea está compuesta por:
1. Espacio de ejecución:
    * Segmento de código.
    * Segmento de datos/pila (uno o varios).

2. Segmento de estado (TSS):
   * Almacena el estado de la tarea (su contexto) para poder reanudarla desde el mismo lugar. Debe estar descripta en la GDT del mismo modo que se describen los segmentos de código y datos


### TSS

![TSS](https://i.imgur.com/ZnI5cB4.png)

**EFLAGS**: Por defecto: 0x00000002. Interrupciones habilitadas: 0x00000202

![encontrando_tss_tarea_actual](https://i.imgur.com/hGTRk1G.png)

### Cambio de tarea

1) Ejecutamos jmp 0x20:0 (4to índice GDT). Lo que sigue lo hace automáticamente el CPU
2) Se busca el descriptor en gdt[4]
3) Como es un cambio de tarea, se lee el TR
4) Se busca la TSS apuntada por el TR
5) Se guarda el contexto actual
6) Se busca TSS apuntado por descriptor de tarea a ejecutar (gdt[4])
7) Se carga el contexto nuevo
8) Se actualiza el TR
9) Se continúa la ejecución desde el nuevo cs:eip

Siempre hay que tener una tarea inicial para proveer una TSS en donde el procesador pueda guardar el contexto actual.

### Convención 32 bits

* Devuelve en EAX.

* Los parámetros se pasan por la pila (se pushean de derecha a izquierda). Después del call hay que popearlos (o hacer add esp, cant_params\*4). No se pueden pushear/popear cosas que no sean de 32 bits.

* Se preservan EBX, ESI y EDI.

* La pila está alineada a 4 bytes (no hace falta chequearlo, queda siempre así).



### Saltar a tarea

```asm
offset: dd 0
selector: dw 0

mov [selector], ax ; ax porque mide una word
jmp far [offset] ; porque jmp far lee 48 bits
```


## TODO

* [ ] Parciales

* [ ] Achicar algunas imágenes que están re grandes. Para que ocupe menos hojas el resumen (está ocupando 23 carillas).

### Consultas

[//]: # (¿se puede setear g=1 en un descriptor de segmento si estamos en modo real? es decir, ¿podemos direccionar más de 1mb de memoria?)

Preguntar si en el 3) hay solapamiento en 2GB. Es decir, si se solapan el segmento de código nivel 0 con el segmento de datos nivel 3. Si hay que restarle 1 byte al tamaño.

También en el 3), preguntar por qué solaparías los segmentos así.

[//]: # (Ej. 4\) Suponiendo que se construye un esquema de paginación donde ninguna página utiliza identity mapping, ¿es posible activar paginación bajo estas condiciones?)

[//]: # (Ej. 6\) a\) si ya la hice solo lectura en segmentación, ¿hace falta que setee r/w=0 también en la PTe?)
