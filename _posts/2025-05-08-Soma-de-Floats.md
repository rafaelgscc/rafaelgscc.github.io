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

Em sistemas eletrônicos digitais, a informação é codificada em dois estados de tensão, representados pelos bits 0 e 1. A operação fundamental de adição é realizada por circuitos lógicos chamados somadores binários. Um meio somador combina dois bits, gerando a soma (S) e o "vai um" (Carry out - Cout):

| A | B | S (A⊕B) | Cout (A∧B) |
| - | - | ------- | ---------- |
| 0 | 0 | 0       | 0          |
| 0 | 1 | 1       | 0          |
| 1 | 0 | 1       | 0          |
| 1 | 1 | 0       | 1          |

Este circuito é implementado com portas lógicas:

<div style="text-align: center;">
  <img src="/assets/images/meio_somador.png" alt="Meio Somador" style="width: 60%;">
</div>

Para somar números com múltiplos bits, utilizamos o somador completo, que incorpora o "vai um" da etapa anterior (Cin):

| A | B | Cin | S (A⊕B⊕Cin) | Cout |
| - | - | --- | ----------- | ---- |
| 0 | 0 | 0   | 0           | 0    |
| 0 | 1 | 0   | 1           | 0    |
| 1 | 0 | 1   | 0           | 1    |
| 1 | 1 | 1   | 1           | 1    |

Com sua representação lógica:

<div style="text-align: center;">
  <img src="/assets/images/somador_completo.png" alt="Somador Completo" style="width: 60%;">
</div>

Em nível de silício, transistores CMOS implementam essas funções:

<div style="text-align: center;">
  <img src="/assets/images/somador_completo_CMOS.png" alt="Somador Completo em CMOS" style="width: 60%;">
</div>

A linguagem de descrição de hardware VHDL permite descrever um somador de 4 bits de forma estruturada, conectando quatro somadores de 1 bit em cascata:

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


Essa arquitetura em cascata permite a construção de somadores para números com um número maior de bits. A soma de números de ponto flutuante, embora mais complexa, também se baseia nesses princípios fundamentais da aritmética binária.

---

## Parte 2: A representação traiçoeira dos números float

Os computadores representam números de ponto flutuante utilizando o padrão **IEEE 754**. Este padrão define um formato que divide um número em três partes:

| Parte | Bits | Descrição                     |
|-------|------|-------------------------------|
| S     | 1    | Sinal (0 = positivo, 1 = negativo) |
| E     | 8    | Expoente (com viés)           |
| M     | 23   | Mantissa ou fração (significand) |

**O problema da representação: 0.1 em binário**

A dificuldade começa porque alguns números decimais que possuem uma representação finita, como 0.1, não podem ser expressos de forma exata e finita na base binária. Vamos ver porquê:

<span class="math-inline">0.1_{10} = ?_2\$
Multiplicando sucessivamente a parte fracionária por 2:
-   \$0.1 \times 2 = 0.2\$ (inteiro: 0)
-   \$0.2 \times 2 = 0.4\$ (inteiro: 0)
-   \$0.4 \times 2 = 0.8\$ (inteiro: 0)
-   \$0.8 \times 2 = 1.6\$ (inteiro: 1)
-   \$0.6 \times 2 = 1.2\$ (inteiro: 1)
-   \$0.2 \times 2 = 0.4\$ (inteiro: 0)  \<- Repetição do padrão
Assim, \$0.1_{10} = 0.000110011001100..._2\$, uma dízima binária infinita, assim como \$1/3\$ é uma dízima decimal infinita (\$0.333...\).

O computador, com seus 23 bits limitados para a mantissa, precisa truncar essa representação, introduzindo uma pequena imprecisão já na forma como o número é armazenado.

O código VHDL a seguir ilustra de forma simplificada as etapas de uma soma de ponto flutuante seguindo o padrão IEEE 754:

```vhdl
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.NUMERIC_STD.ALL;

entity ieee754_Somador is
    Port (
        A       : in  STD_LOGIC_VECTOR(31 downto 0);
        B       : in  STD_LOGIC_VECTOR(31 downto 0);
        RESULT  : out STD_LOGIC_VECTOR(31 downto 0)
    );
end ieee754_somador;

architecture Behavioral of ieee754_somador is

    -- Declaração de sinais com inicialização para evitar latches invisíveis
    signal sign_a, sign_b        : STD_LOGIC := '0';
    signal exp_a, exp_b          : unsigned(7 downto 0) := (others => '0');
    signal man_a, man_b          : unsigned(23 downto 0) := (others => '0');
    signal exp_diff              : integer := 0;
    signal aligned_man_a, aligned_man_b : unsigned(24 downto 0) := (others => '0');
    signal result_sign           : STD_LOGIC := '0';
    signal result_exp            : unsigned(7 downto 0) := (others => '0');
    signal sum_man               : unsigned(25 downto 0) := (others => '0');
    signal result_man            : unsigned(24 downto 0) := (others => '0');
    signal normalized_man        : unsigned(22 downto 0) := (others => '0');
    signal final_exp             : unsigned(7 downto 0) := (others => '0');

begin

    process(A, B)
    begin
        -- Inicialização explícita no início do processo
        sign_a <= '0';
        sign_b <= '0';
        exp_a  <= (others => '0');
        exp_b  <= (others => '0');
        man_a  <= (others => '0');
        man_b  <= (others => '0');
        exp_diff <= 0;
        aligned_man_a <= (others => '0');
        aligned_man_b <= (others => '0');
        result_sign <= '0';
        result_exp <= (others => '0');
        sum_man <= (others => '0');
        result_man <= (others => '0');
        normalized_man <= (others => '0');
        final_exp <= (others => '0');

        -- Verificar valores indefinidos
        if (A /= A or B /= B) then
            report "Warning: Entrada contém valores indefinidos!" severity warning;
            RESULT <= (others => '0'); -- Retorna 0 se houver metavalores
        else
            -- Separar campos
            sign_a <= A(31);
            sign_b <= B(31);
            exp_a  <= unsigned(A(30 downto 23));
            exp_b  <= unsigned(B(30 downto 23));

            -- Acrescentar bit oculto (1) na mantissa
            man_a  <= "1" & unsigned(A(22 downto 0));
            man_b  <= "1" & unsigned(B(22 downto 0));

            -- Alinhar expoentes garantindo que sejam válidos
            if exp_a > exp_b then
                exp_diff <= to_integer(exp_a - exp_b);
                aligned_man_a <= resize(man_a, 25);
                aligned_man_b <= shift_right(resize(man_b, 25), exp_diff);
                result_exp <= exp_a;
            elsif exp_b > exp_a then
                exp_diff <= to_integer(exp_b - exp_a);
                aligned_man_a <= shift_right(resize(man_a, 25), exp_diff);
                aligned_man_b <= resize(man_b, 25);
                result_exp <= exp_b;
            else
                exp_diff <= 0;
                aligned_man_a <= resize(man_a, 25);
                aligned_man_b <= resize(man_b, 25);
                result_exp <= exp_a;
            end if;

            -- Soma
            sum_man <= ("0" & aligned_man_a) + ("0" & aligned_man_b);

            -- Normalização
            if sum_man(25) = '1' then
                result_man <= shift_right(sum_man(25 downto 1), 1);
                final_exp <= result_exp + 1;
            else
                result_man <= sum_man(24 downto 0);
                final_exp <= result_exp;
            end if;

            normalized_man <= result_man(22 downto 0);

            -- Saída final
            result_sign <= '0'; -- Apenas resultados positivos por enquanto
            RESULT <= result_sign & std_logic_vector(final_exp) & std_logic_vector(normalized_man);
        end if;
    end process;

end Behavioral;
```
Este código RTL (Register-Transfer Level) descreve as operações de alinhamento de expoentes, soma das mantissas (de forma simplificada para números positivos), e normalização do resultado. A unidade de ponto flutuante (ULA-UPF) dentro do processador executa essas etapas através de circuitos combinacionais e sequenciais complexos.

<div style="text-align: center;">
  <img src="/assets/images/RTL.png" alt="RTL do circuito" style="width: 60%;">
</div>

Este circuito é a base do tratamento de soma de pontos flutuantes em um computador.

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

Em processadores mais antigos, como os utilizados em sistemas como IBM PC, Amiga e Atari ST, o cálculo de ponto flutuante era frequentemente realizado por um coprocessador matemático separado (FPU - Floating Point Unit), como o Intel 80x87 ou o Motorola 68882, pois as CPUs principais eram otimizadas para operações inteiras. A comunicação entre a CPU e a FPU envolvia instruções específicas em assembly, como demonstrado abaixo:

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
A integração da FPU diretamente no chip da CPU, como ocorreu com o Intel 486DX em 1991, marcou um avanço significativo no desempenho do processamento de ponto flutuante. Curiosamente, as primeiras gerações do Pentium enfrentaram um notório problema de precisão em seus cálculos de ponto flutuante, conhecido como "bug da divisão do Pentium", demonstrando a complexidade de implementar corretamente essas operações em hardware.

---

## Parte 4: Navegando pelas armadilhas do ponto flutuante

Para aplicações onde a precisão decimal é fundamental, como sistemas financeiros (cálculo de juros, transações monetárias), criptomoedas (rastreamento de saldos e transações) ou simulações científicas de alta precisão (cálculo de trajetórias, modelagem molecular), a imprecisão inerente aos números float pode levar a erros significativos. Nessas situações, é crucial adotar estratégias para mitigar esses problemas.

Linguagens de alto nível oferecem mecanismos para trabalhar com maior precisão, geralmente através de bibliotecas que implementam aritmética decimal. Essas bibliotecas representam os números decimais de forma diferente, frequentemente como inteiros com um expoente implícito, permitindo cálculos exatos.

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

### ✅ Em C não temos uma biblioteca padrão, mas usa-se a `GMP (GNU Multiple Precision Arithmetic Library)`

```c
#include <stdio.h>
#include <gmp.h>

int main() {
    mpf_t a, b, resultado;
    mpf_set_default_prec(256); // Define a precisão

    mpf_init(a);
    mpf_init(b);
    mpf_init(resultado);

    mpf_set_str(a, "123.456", 10);
    mpf_set_str(b, "789.123", 10);

    mpf_add(resultado, a, b); // Realiza a soma

    gmp_printf("Resultado: %.10Ff\n", resultado); // Saída com 10 casas decimais

    mpf_clear(a);
    mpf_clear(b);
    mpf_clear(resultado);
    return 0;
}
```

---

## Parte 5: Alternativas práticas

1. **Evite comparações diretas com floats:**  
   Sempre use uma margem de erro (`epsilon`):

   ```python
   abs((0.1 + 0.2) - 0.3) < 1e-9
   ```

2. **Trabalhe com inteiros sempre que possível:**  
   Em sistemas financeiros, armazene valores em centavos:

   ```csharp
   preco = 1099  // em centavos
   ```

3. **Use bibliotecas de precisão arbitrária**  
   Quase toda linguagem moderna tem suporte a isso.

---

## Conclusão

A soma de números `float` é **precisa o suficiente para a maioria dos casos**, mas não é perfeita. Compreender como ela funciona nos bastidores — tanto no nível de bits quanto no nível eletrônico — ajuda a prevenir erros em sistemas críticos, como bancários, científicos ou de engenharia.