---
layout: post
title: "Soma de Números Float: Por que não é tão simples quanto parece?"
date: 2025-05-08
excerpt_separator: <!--more-->
---

Você já escreveu algo como isso?

```python
print(0.1 + 0.2)  # Resultado: 0.30000000000000004
```

E pensou: **“Como assim? Isso não deveria dar 0.3?”**

<!--more-->
---

Isso acontece em **todas as linguagens modernas**:

### JavaScript

```javascript
console.log(0.1 + 0.2);  // 0.30000000000000004
```

### Java

```java
System.out.println(0.1 + 0.2);  // 0.30000000000000004
```

### C

```c
#include <stdio.h>

int main() {
    printf("%.17f\n", 0.1 + 0.2);  // 0.30000000000000004
    return 0;
}
```
---
## Parte 1: Como um computador soma

Em um sistema eletrônico, existem apenas dois números, 0 (adotando a tensão de 0v) e 1 (Adotando 5v para TTL
ou 3.3v ), um somador binário basicamente soma dois bits e leva um bit para frente, basicamente um "Vai um"

| A | B | S (A⊕B) | Cout (A∧B) |
| - | - | ------- | ---------- |
| 0 | 0 | 0       | 0          |
| 0 | 1 | 1       | 0          |
| 1 | 0 | 1       | 0          |
| 1 | 1 | 0       | 1          |

O circuito gerado basicamente é:


<div style="text-align: center;">
  <img src="/assets/images/meio_somador.png" alt="Meio Somador" style="width: 60%;">
</div>

Com isso, podemos cascatear o "Vai um" ou como chamamos de _Carry out_ e assim, compomos o somador completo, de tabela:

| A | B | Cin | S (A⊕B⊕Cin) | Cout |
| - | - | --- | ----------- | ---- |
| 0 | 0 | 0   | 0           | 0    |
| 0 | 1 | 0   | 1           | 0    |
| 1 | 0 | 1   | 0           | 1    |
| 1 | 1 | 1   | 1           | 1    |

Com representação lógica:

<div style="text-align: center;">
  <img src="/assets/images/somador_completo.png" alt="Somador Completo" style="width: 60%;">
</div>

E em nível de silício, tudo isso é construído com transistores, como neste exemplo em tecnologia CMOS:

<div style="text-align: center;">
  <img src="/assets/images/somador_completo_CMOS.png" alt="Somador Completo em CMOS" style="width: 60%;">
</div>

Em uma linguagem descritiva como o VHDL, podemos descrever o processo de soma de 4 bits de forma simples

```vhdl
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;

entity somador_4bits is
    Port (
        A     : in  STD_LOGIC_VECTOR (3 downto 0);
        B     : in  STD_LOGIC_VECTOR (3 downto 0);
        Cin   : in  STD_LOGIC;
        S     : out STD_LOGIC_VECTOR (3 downto 0);
        Cout  : out STD_LOGIC
    );
end somador_4bits;

architecture Structural of somador_4bits is

    -- Declara o componente
    component somador_1bit
        Port (
            A     : in  STD_LOGIC;
            B     : in  STD_LOGIC;
            Cin   : in  STD_LOGIC;
            S     : out STD_LOGIC;
            Cout  : out STD_LOGIC
        );
    end component;

    -- Sinais internos para os "carry"
    signal c1, c2, c3 : STD_LOGIC;

begin

    -- Instância do bit 0
    U0: somador_1bit port map(
        A => A(0),
        B => B(0),
        Cin => Cin,
        S => S(0),
        Cout => c1
    );

    -- Instância do bit 1
    U1: somador_1bit port map(
        A => A(1),
        B => B(1),
        Cin => c1,
        S => S(1),
        Cout => c2
    );

    -- Instância do bit 2
    U2: somador_1bit port map(
        A => A(2),
        B => B(2),
        Cin => c2,
        S => S(2),
        Cout => c3
    );

    -- Instância do bit 3
    U3: somador_1bit port map(
        A => A(3),
        B => B(3),
        Cin => c3,
        S => S(3),
        Cout => Cout
    );

end Structural;
```

<div style="text-align: center;">
  <img src="/assets/images/somador_em_cascata.png" alt="Somador Completo em CMOS" style="width: 60%;">
</div>


Com essa estrutura, é possível cascatear múltiplos somadores para operações com 8, 16 ou 32 bits. E é justamente a partir desse princípio que a soma de números `float` se torna possível — mas com complexidades adicionais.

---

## Parte 2: Como o computador representa números float

Um número `float` é representado usando o padrão **IEEE 754**, que divide um número em três partes:

| Parte | Bits | Descrição                     |
|-------|------|-------------------------------|
| S     | 1    | Sinal (0 = positivo, 1 = negativo) |
| E     | 8    | Expoente (com viés)           |
| M     | 23   | Mantissa ou fração (significand) |

### Exemplo: 0.1 em IEEE 754 (32 bits)

- 0.1 decimal não pode ser representado exata e finitamente em binário (assim como 1/3 não pode em decimal).
- Ele vira uma dízima binária:  
  `0.000110011001100...`

O computador **trunca** essa dízima para caber nos 23 bits da mantissa. Isso já gera um pequeno erro **logo na representação**.

---

## Parte 3: A soma de floats no nível eletrônico (hardware)

A soma de dois números `float` é feita em um **ULA-UPF (Unidade de Ponto Flutuante)** do processador.

### Etapas básicas da soma:

1. **Alinhar os expoentes:**  
   O número com o menor expoente tem sua mantissa deslocada à direita para igualar os expoentes.
2. **Somar as mantissas:**  
   A operação binária é feita, bit a bit, como em números inteiros.
3. **Normalizar o resultado:**  
   O resultado pode ter que ser ajustado (com shift e arredondamento).
4. **Arredondar e armazenar:**  
   O resultado final pode ser arredondado novamente por falta de espaço nos bits da mantissa.

Essas etapas são implementadas por meio de **circuitos digitais combinacionais** (multiplexadores, somadores, shifters, registradores, etc.), controlados por uma **unidade de controle lógica** dentro da CPU.

> ⚠️ Mesmo antes da soma, já temos erro na representação. Depois da soma, o erro se acumula ou muda.

Em computadores anigos, seja IBMs, Amigas ou mesmo Atari STs, para se calcular o número de ponto flutuante era necessário um processador a parte, uma FPU (ou intel 80x87 ou Motorola 68882) já que seus processadores apenas calculavam numeros inteiros

```asm

section .data
    a dd 0.1       ; float (32 bits)
    b dd 0.2
    resultado dd 0.0

section .text
    global _start

_start:
    fld dword [a]      ; carrega a em ST(0)
    fadd dword [b]     ; ST(0) = ST(0) + b
    fstp dword [resultado] ; salva ST(0) em resultado e remove da pilha

    ;(...)
```    
Em 1991, a Intel revolucionou com a introdução no mercado do 486dx, um processador que não dependia de uma FPU x87, isso se sussedeu com o pentium, que em suas primeiras unidades, teve problemas com calculos de pontos flutuantes devido a um erro de arquitetura.

---

## Parte 3: Como evitar erros com float

### ✅ Em Python (com `decimal`)

```python
from decimal import Decimal

a = Decimal('0.1')
b = Decimal('0.2')
print(a + b == Decimal('0.3'))  # True
```

### ✅ Em Java (com `BigDecimal`)

```java
import java.math.BigDecimal;

BigDecimal a = new BigDecimal("0.1");
BigDecimal b = new BigDecimal("0.2");
System.out.println(a.add(b).equals(new BigDecimal("0.3")));  // true
```

### ✅ Em JavaScript (biblioteca `decimal.js`)

```javascript
const Decimal = require('decimal.js');

let a = new Decimal('0.1');
let b = new Decimal('0.2');
console.log(a.plus(b).equals(new Decimal('0.3')));  // true
```

---

## Parte 4: Alternativas práticas

1. **Evite comparações diretas com floats:**  
   Sempre use uma margem de erro (`epsilon`):

   ```python
   abs((0.1 + 0.2) - 0.3) < 1e-9
   ```

2. **Trabalhe com inteiros sempre que possível:**  
   Em sistemas financeiros, armazene valores em centavos:

   ```python
   preco = 1099  # em centavos
   ```

3. **Use bibliotecas de precisão arbitrária**  
   Quase toda linguagem moderna tem suporte a isso.

---

## Conclusão

A soma de números `float` é **precisa o suficiente para a maioria dos casos**, mas não é perfeita. Compreender como ela funciona nos bastidores — tanto no nível de bits quanto no nível eletrônico — ajuda a prevenir erros em sistemas críticos, como bancários, científicos ou de engenharia.