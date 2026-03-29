# ALU de 8 bits 

## Introdução

Uma **ALU (Unidade Lógica e Aritmética)** é o componente central de qualquer processador, responsável por executar operações matemáticas e lógicas sobre dados binários. Este projeto consiste no desenvolvimento de uma ALU de 8 bits construída no simulador de circuitos digitais **Digital**, utilizando arquivos `.dig` para representar cada componente do circuito.

A ALU opera com palavras de **8 bits** e suporta as seguintes operações:

- **Operações aritméticas:** soma, subtração, multiplicação e divisão
- **Shifts lógicos:** deslocamento tanto para esquerda quanto para a direita
- **Portas lógicas:** NAND e XOR

Não foi utilizado nenhum componente pronto de operação aritimética ao longo desse projeto.

---

## Vídeo Explicativo

Assista ao vídeo para uma explicação sobre o funcionamento da ALU, a estrutura dos circuitos e as decisões de projeto tomadas durante o desenvolvimento.

> **Vídeo:** [Clique aqui para assistir](https://www.youtube.com/watch?v=GGGXcRowNjk)

---

## Operações Aritméticas

### Somador de 1 bit - `soma1bit.dig`

O `soma1bit.dig` é o fundamental para a ALU. Nele existe um somador completo, que soma dois bits levando em conta um carry de entrada.

**Entradas:**
- `A` — primeiro operando (1 bit)
- `B` — segundo operando (1 bit)
- `Cin` — carry-in (carry de entrada, 1 bit)

**Saídas:**
- `S` — resultado da soma (1 bit)
- `Cout` — carry-out (carry de saída, 1 bit)

**Lógica implementada:**

```
S    = A XOR B XOR Cin   (porta XOR de 3 entradas)
Cout = (A AND B) OR (A AND Cin) OR (B AND Cin)   (3 portas AND + 1 porta OR de 3 entradas)
```

O circuito utiliza **Tunnels** para organização interna, evitando cruzamento excessivo de fios e tornando o layout mais legível.

<!-- Imagem do circuito soma1bit.dig -->
![Somador de 1 bit](/assets/soma1bits.png)

---

### Somador de 8 bits - `soma8bits.dig`

O `soma8bits.dig` implementa 8 instâncias do `soma1bit.dig` em cascata.

**Entradas:**
- `A[7:0]` — primeiro operando (8 bits)
- `B[7:0]` — segundo operando (8 bits)
- `Cin` — carry-in inicial (1 bit)

**Saídas:**
- `Out[7:0]` — resultado da soma (8 bits)
- `Cout` — carry-out final (1 bit)

**Funcionamento:**

Splitters são utilizados para separar os barramentos de 8 bits em bits individuais, que são então conectados às entradas de cada instância do `soma1bit.dig`. O Cout de cada estágio alimenta diretamente o Cin do estágio seguinte, propagando o carry da posição menos significativa até a mais significativa.

![Somador de 8 bits](/assets/soma8bits.png)

---

### Inversor por Complemento de 2 - `inversor_comp.dig`

O `inversor_comp.dig` é um componente auxiliar utilizado pelo subtrator. Ele recebe um valor de 8 bits e produz o seu **complemento de 2**, que corresponde à negação aritmética do número.

**Entrada:**
- `A[7:0]` — valor original (8 bits)

**Saída:**
- `Ainv[7:0]` — complemento de 2 de A (8 bits)

**Processo:**

1. Todos os 8 bits de `A` são invertidos individualmente por **8 portas NOT** (complemento de 1)
2. O resultado é somado a `1` utilizando o `soma8bits.dig` com `Cin = 1`

Esse processo garante que `A + Ainv = 0` (em aritmética de 8 bits), propriedade fundamental do complemento de 2.

![Inversor por Complemento de 2](/assets/inversor_comp.png)

---

### Subtrator de 8 bits - `subtrator8bits.dig`

O `subtrator8bits.dig` implementa a operação `A - B` utilizando complemento de 2.

**Entradas:**
- `A[7:0]` (8 bits)
- `B[7:0]` (8 bits)

**Saídas:**
- `Out[7:0]` — resultado da subtração 8 bits, valor absoluto (módulo)
- `Cout` — indica o sinal do resultado, positivo ou negativo.

**Funcionamento:**

- Um `inversor_comp.dig` é usado para calcular `-B` (complemento de 2 de B)
- O `soma8bits.dig` realiza a soma `A + (-B)`
- Um segundo `inversor_comp.dig` inverte o resultado para obter o valor absoluto quando o resultado é negativo
- Um **Multiplexer de 8 bits** seleciona entre o resultado direto e o resultado invertido, com base no sinal
- Um **LED indicador -/+** acende quando o resultado é negativo (`Cout = 0` indica resultado negativo)

![Subtrator de 8 bits](/assets/subtrator8bits.png)

---

### Produto Parcial - `produto_parcial.dig`

O `produto_parcial.dig` é a base da multiplicação. Calcula o produto parcial entre o multiplicando e um único bit do multiplicador.

**Entradas:**
- `A[7:0]` — multiplicando (8 bits)
- `ni` — bit `i` do multiplicador (1 bit)

**Saída:**
- `PP[7:0]` — produto parcial (8 bits)

**Funcionamento:**

Cada bit de `A` é multiplicado (operação AND) pelo bit `ni` por meio de 8 portas AND. Se `ni = 0`, todos os bits do produto parcial serão `0`; se `ni = 1`, o produto parcial é igual ao próprio `A`, 0 x qualquer coisa = 0, 1 x qualquer coisa = qualquer coisa.

![Produto Parcial](/assets/produto_parcial.png)

---

### Multiplicador de 8 bits - `multiplicador8bits.dig`

O `multiplicador8bits.dig` implementa a **multiplicação por somas sucessivas de produtos parciais**, seguindo o algoritmo clássico de multiplicação binária.

**Entradas:**
- `A[7:0]` — multiplicando (8 bits)
- `B[7:0]` — multiplicador (8 bits)

**Saída:**
- Resultado de 16 bits, sendo a soma de "S7-" e "S7+" respectivamente LSB e MSB.

**Funcionamento:**

São utilizadas 8 instâncias do `produto_parcial.dig`, uma para cada bit de `B`. Cada produto parcial é deslocado de acordo com a posição do bit correspondente em `B` e é somado ao acumulador por meio de somadores ligados. O resultado final é a soma de todos os produtos parciais deslocados.

![Multiplicador de 8 bits](/assets/multiplicador8bits.png)

---

### Etapa de Divisão - `etapa_divisao.dig`

O `etapa_divisao.dig` representa uma primeira parte do algoritmo de divisão por subtração sucessiva. Ele determina se o divisor (shiftado) cabe no resto parcial atual, ou seja, verifcando se o número é divisivel ou não.

**Entradas:**
- `R_in[7:0]` — resto parcial de entrada (8 bits)
- `N_sh[7:0]` — numerador deslocado (8 bits)

**Saídas:**
- `R_out[7:0]` — resto atualizado (8 bits)
- `Q_bit` — bit do quociente (1 bit)

**Funcionamento:**

O `subtrator8bits.dig` calcula `R_in - N_sh`. Se o resultado for positivo (`Cout = 1`), o **Multiplexer** seleciona o novo resto (resultado da subtração) e `Q_bit = 1`, caso contrário, o resto permanece igual a `R_in` e `Q_bit = 0`.

![Etapa de Divisão](/assets/etapa_divisao.png)

--- 

### Divisor de 8 bits - `divisor8bits.dig`

O `divisor8bits.dig` implementa a divisão completa por subtração sucessiva, seguindo o algoritmo clássico de divisão binária.

**Entradas:**
- `A[7:0]` — dividendo (8 bits)
- `B[7:0]` — divisor (8 bits)

**Saída:**
- Quociente de 8 bits

**Funcionamento:**

O dividendo `A` é separado bit a bit por meio de um splitter. Também `etapa_divisao.dig` é colocado em sequência, cada uma processando um bit do quociente. A cada etapa, o algoritmo verifica se o divisor cabe no resto parcial atual, gerando o bit correspondente do quociente e atualizando o resto para a próxima etapa.

![Divisor de 8 bits](/assets/divisor8bits.png)

---

## Shifts Lógicos

### Shift Lógico à Esquerda de 8 bits - `shift_esquerda8bits.dig`

O `shift_esquerda8bits.dig` desloca todos os bits do operando uma posição para a esquerda, sendo a mesma coisa que multiplicar por 2.

**Entrada:**
- `A[7:0]` — operando (8 bits, exibido em binário)

**Saída:**
- `Out[7:0]` — resultado do deslocamento (8 bits, exibido em binário)

**Implementação:**

Um splitter separa os 8 bits do operando. O recabeamento é feito diretamente, então cada `bit[i]` é conectado à posição `bit[i+1]` da saída. A posição menos significativa (`bit 0`) recebe um valor nulo, e o bit mais significativo (`bit 7`) é descartado.

![Shift Lógico à Esquerda](/assets/shift_esquerda8bits.png)

---

### Shift Lógico à Direita de 8 bits - `shift_direita8bits.dig`

O `shift_direita8bits.dig` desloca todos os bits do operando uma 1 posição para a direita, sendo a mesma coisa que uma divisão por 2.

**Entrada:**
- `A[7:0]` — operando (8 bits, exibido em binário)

**Saída:**
- `Out[7:0]` — resultado do deslocamento (8 bits, exibido em binário)

**Implementação:**

Um splitter separa os 8 bits do operando. O recabeamento é feito diretamente, então cada `bit[i]` é conectado à posição `bit[i-1]` da saída. A posição mais significativa (`bit 7`) recebe um valor nulo, e o bit menos significativo (`bit 0`) é descartado.

![Shift Lógico à Direita](/assets/shift_direita8bits.png)

---

## Portas Lógicas em 8 bits

### NAND de 8 bits

A operação NAND opera bit a bit sobre os dois operandos de 8 bits, produzindo o complemento do AND para cada par de bits correspondente.

**Resultado:** `NOT(A AND B)` para cada par de bits `A[i]` e `B[i]`

A porta NAND é uma porta lógica universal, onde qualquer função booleana pode ser implementada utilizando apenas portas NAND. Essa propriedade a torna especialmente relevante no contexto de projeto de circuitos digitais.

> **Obs:** Esta operação é implementada diretamente na ALU utilizando as primitivas de porta NAND disponíveis no simulador Digital, sem necessidade de um arquivo `.dig` separado. Portanto, sua demonstração estará presente apenas na imagem da ALU completa.

---

### XOR de 8 bits

A operação XOR opera bit a bit sobre os dois operandos de 8 bits, produzindo `1` quando os bits correspondentes são diferentes e `0` quando são iguais.

**Resultado:** `A XOR B` para cada par de bits `A[i]` e `B[i]`

> **Obs:** Esta operação é implementada diretamente na ALU utilizando as primitivas de porta XOR disponíveis no simulador Digital, sem necessidade de um arquivo `.dig` separado. Portanto, sua demonstração estará presente apenas na imagem da ALU completa.


---

## ALU de 8 bits Completa - `ALU8bits.dig`

O `ALU8bits.dig` é o circuito principal do projeto, a ALU de 8 bits completamente integrada e funcional. Reúne todos os componentes desenvolvidos anteriormente em um único circuito, expondo uma interface unificada com entradas, saídas e um sinal de controle de operação.

**Entradas:**
- `AC[7:0]` — acumulador, primeiro operando (8 bits)
- `N[7:0]` — segundo operando (8 bits)
- `Opcode[2:0]` — seletor de operação (3 bits, mostra qual operação será executada)

**Saídas:**
- `AC_out[7:0]` — resultado principal da operação selecionada (8 bits)
- `MQ_out[7:0]` — resultado auxiliar: 8 bits mais significativos na multiplicação, quociente na divisão, e zero nas demais operações

**Operações suportadas que são selecionadas via `Opcode`:**

| Código `Opcode` | Operação              | Componente utilizado         | `AC_out`                  | `MQ_out`                  |
|:---------------:|:---------------------:|:----------------------------:|:-------------------------:|:-------------------------:|
| `000`           | Soma                  | `soma8bits.dig`              | Resultado (8 bits)        | `0`                       |
| `001`           | Subtração             | `subtrator8bits.dig`         | Resultado (8 bits)        | `0`                       |
| `010`           | Multiplicação         | `multiplicador8bits.dig`     | 8 bits menos significativos | 8 bits mais significativos |
| `011`           | Divisão               | `divisor8bits.dig`           | Resto                     | Quociente                 |
| `100`           | Shift lógico esquerda | `shift_esquerda8bits.dig`    | Resultado (8 bits)        | `0`                       |
| `101`           | Shift lógico direita  | `shift_direita8bits.dig`     | Resultado (8 bits)        | `0`                       |
| `110`           | NAND                  | Primitiva do simulador       | Resultado (8 bits)        | `0`                       |
| `111`           | XOR                   | Primitiva do simulador       | Resultado (8 bits)        | `0`                       |

**Funcionamento:**

A ALU recebe dois operandos de entrada, `AC[7:0]` e `N[7:0]`, juntamente com o sinal de controle `Opcode`, o qual define a operação que será utilizada. Todos os módulos da ALU operam separadamente, as operações de soma, subtração, multiplicação, divisão, deslocamentos lógicos, NAND e XOR, são executadas ao mesmo tempo a partir dos mesmos valores de entrada, independentemente de qual será a saída final. Cada uma dessas operações gera seu próprio resultado de forma independente, ficando todos disponíveis ao mesmo tempo para seleção.

No caso da multiplicação, como o resultado possui 16 bits, ele é dividido em duas partes, que são elas, os 8 bits menos significativos (LSB) são enviados para `AC_out` e os 8 bits mais significativos (MSB) são enviados para `MQ_out`. Na divisão, algo parecido acontece, mas com significados diferentes. O resto da operação é enviado para `AC_out` e o quociente é enviado para `MQ_out`. Para as outras operações da ALU, que geram apenas 8 bits, o valor de `MQ_out` é definido como zero.

Para lidar com isso, a ALU utiliza dois multiplexadores controlados pelo mesmo `Opcode`, que é um responsável por selecionar o valor de `AC_out` entre todas as operações disponíveis, e outro responsável por selecionar o valor de `MQ_out`, que será relevante apenas nos casos de multiplicação e divisão. Dessa forma, embora todas as operações sejam calculadas ao mesmo tempo, apenas a saída correspondente à operação selecionada pelo `Opcode` é propagada para as saídas finais, garantindo o funcionamento correto e eficiente da ALU.

![ALU de 8 bits Completa](/assets/ALU8bits.png)

---

## Hierarquia de Componentes

A seguir, a hierarquia completa de dependências entre os circuitos da ALU:

- **ALU de 8 bits**
  - Soma (`soma8bits.dig`)
    - `soma1bit.dig` (×8)
  - Subtração (`subtrator8bits.dig`)
    - `inversor_comp.dig` (×2)
      - `soma8bits.dig`
        - `soma1bit.dig` (×8)
    - `soma8bits.dig`
      - `soma1bit.dig` (×8)
  - Multiplicação (`multiplicador8bits.dig`)
    - `produto_parcial.dig` (×8)
    - `soma8bits.dig` (somadores encadeados)
      - `soma1bit.dig` (×8)
  - Divisão (`divisor8bits.dig`)
    - `etapa_divisao.dig`
      - `subtrator8bits.dig`
    - `passo_resto_div.dig` (×múltiplos)
      - `subtrator8bits.dig`
  - Shift Esquerda (`shift_esquerda8bits.dig`)
  - Shift Direita (`shift_direita8bits.dig`)
  - NAND 8 bits (primitiva do simulador)
  - XOR 8 bits (primitiva do simulador)

---

## Conclusão

Este projeto demonstra como uma **ALU de 8 bits completa** pode ser construída a partir de componentes simples, utilizando exclusivamente portas lógicas básicas e o simulador Digital. A estratégia escolhida partindo do somador de 1 bit até as operações de multiplicação e divisão, permitiu o desenvolvimento modular e incremental de cada funcionalidade, facilitando a verificação e o reuso de componentes.

O projeto evidencia um princípio fundamental da computação digital. Somadores constroem subtratores, subtratores constroem divisores, e produtos parciais constroem multiplicadores, e tudo isso a partir de portas AND, OR, NOT e XOR.
