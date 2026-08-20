# Análise do simulador de investimentos da ArenaCash

Documento de leitura para quem **não programa**. Descreve, em português comum, como o
simulador da ArenaCash decide qual investimento indicar para cada pessoa.

- **Arquivo analisado:** `simulador.js` (238 linhas) e o formulário `simulador.html`
- **Repositório:** github.com/Product-Arena/arena-cash (público)
- **Data da análise:** 20 de agosto de 2026

> A ArenaCash e os produtos citados são fictícios, mas a lógica descrita aqui é a que
> roda de verdade quando alguém usa o simulador.

---

## 1. O que o simulador faz, em um parágrafo

O simulador faz **três perguntas** — idade, salário mensal e qual perfil de investidor a
pessoa se considera (conservador, moderado ou arrojado) — e devolve **um** produto
recomendado, junto com um valor sugerido de investimento mensal e uma lista de motivos.
O valor sugerido é sempre 15% do salário, sem exceção. A escolha do produto acontece em
dois momentos: primeiro o sistema verifica se a pessoa passa por duas regras de corte
que ignoram tudo o mais; se passar, ele soma uma nota de idade com uma nota de perfil,
chega a uma pontuação de 2 a 6, e cada pontuação corresponde a um produto fixo da
prateleira. Não há nenhuma pergunta sobre dívidas, dependentes, gastos, objetivos ou
sobre quanto a pessoa já tem guardado.

---

## 2. Os produtos da prateleira

São cinco produtos, do mais seguro para o mais arriscado. Antes da tabela, o significado
dos termos que aparecem nela:

- **CDI** — o "juro de referência" do mercado brasileiro. Render "102% do CDI" significa
  render um pouco acima dessa referência; "95% do CDI", um pouco abaixo.
- **Liquidez diária** — dá para sacar o dinheiro em qualquer dia útil.
- **Carência** — tempo mínimo em que o dinheiro fica preso.
- **Isento de IR** — não desconta Imposto de Renda do que rendeu.
- **Deságio** — perda que se leva ao sacar antes do prazo combinado.
- **D+30** — o dinheiro pedido hoje cai na conta 30 dias depois.

| Produto | Rendimento | Prazo | Risco | Em uma frase |
|---|---|---|---|---|
| **CDB Reserva** | 102% do CDI | Liquidez diária | Muito baixo | Rende todo dia útil e sai na hora que precisar. |
| **LCI Isenta** | 95% do CDI, isento de IR | 1 ano de carência | Baixo | Rende menos no papel, mas não paga imposto. |
| **CDB Progressivo** | 112% do CDI | 2 anos | Baixo | Rende mais quanto mais tempo ficar parado. |
| **CDB Longo Prazo** | 124% do CDI | 5 anos | Médio | O melhor rendimento garantido, com o dinheiro preso por mais tempo. |
| **Arena Quant** | Variável, **sem garantia** | Resgate em D+30 | Alto | Pode render bem acima da referência — e pode fechar o ano no negativo. |

Repare que os produtos não estão ordenados só por risco: eles também estão ordenados por
**quanto tempo o dinheiro fica preso**. Essa segunda ordem é importante para entender os
casos-limite da seção 5.

---

## 3. Como a pontuação é calculada, passo a passo

### Passo 1 — quanto a pessoa vai investir por mês

O sistema pega **15% do salário informado** e arredonda para o real mais próximo. Esse é
o "aporte". Ele não muda por idade nem por perfil.

| Salário informado | Aporte sugerido |
|---|---|
| R$ 2.000 | R$ 300 |
| R$ 6.000 | R$ 900 |
| R$ 20.000 | R$ 3.000 |

### Passo 2 — a idade vira uma nota

A ideia é estimar quanto tempo falta até a pessoa provavelmente precisar do dinheiro.
Quanto mais nova, mais tempo — e, no raciocínio do sistema, mais capacidade de atravessar
períodos ruins.

| Idade | Nota | Como o sistema chama |
|---|---|---|
| Menos de 30 anos | **3** | horizonte longo |
| De 30 a 49 anos | **2** | horizonte médio |
| 50 anos ou mais | **1** | horizonte curto |

### Passo 3 — o perfil declarado vira uma nota

| Perfil escolhido | Nota |
|---|---|
| Arrojado | **3** |
| Moderado | **2** |
| Conservador | **1** |

### Passo 4 — as duas notas são somadas

A pontuação final vai de **2** (mais cauteloso possível) a **6** (mais arrojado possível).

**Ponto central para entender todo o resto:** as duas notas têm exatamente o mesmo peso e
são simplesmente somadas. Isso significa que **um ponto de juventude cancela um ponto de
cautela declarada**. Uma pessoa jovem e conservadora recebe a mesma pontuação de uma
pessoa mais velha e arrojada.

### Passo 5 — a pontuação vira um produto

| Pontuação | Produto indicado |
|---|---|
| 2 | CDB Reserva |
| 3 | LCI Isenta |
| 4 | CDB Progressivo |
| 5 | CDB Longo Prazo |
| 6 | Arena Quant *(se o aporte permitir — ver seção 4)* |

### Todas as combinações possíveis

Só existem nove combinações de idade e perfil. Esta é a tabela completa:

| Idade | Perfil | Pontos | Produto indicado |
|---|---|---|---|
| Menos de 30 | Arrojado | 6 | Arena Quant *(ou CDB Longo Prazo, se o aporte for baixo)* |
| Menos de 30 | Moderado | 5 | CDB Longo Prazo |
| Menos de 30 | Conservador | 4 | CDB Progressivo |
| 30 a 49 | Arrojado | 5 | CDB Longo Prazo |
| 30 a 49 | Moderado | 4 | CDB Progressivo |
| 30 a 49 | Conservador | 3 | LCI Isenta |
| 50 ou mais | Arrojado | 4 | CDB Progressivo |
| 50 ou mais | Moderado | 3 | LCI Isenta |
| 50 ou mais | Conservador | 2 | CDB Reserva |

Duas leituras que saltam dessa tabela:

- **O Arena Quant só é alcançável por uma única combinação:** ter menos de 30 anos *e* se
  declarar arrojado. Nenhuma outra pessoa chega lá, por mais dinheiro que tenha.
- **Quem tem 50 anos ou mais e se declara conservador está permanentemente no piso.** Faz
  2 pontos, o mínimo, e nunca alcança nem a LCI — que é de risco Baixo e isenta de imposto.

---

## 4. As regras que têm precedência sobre a pontuação

Existem duas regras que passam por cima da pontuação. Elas olham **apenas o valor do
aporte** — nunca a idade nem o perfil.

### Regra 1 — reserva antes de tudo

> **Se o aporte for menor que R$ 100 por mês, a indicação é CDB Reserva, ponto final.**

Essa regra é verificada **antes** de qualquer outra coisa e ignora completamente idade e
perfil. A justificativa registrada no próprio código é que quem ainda não consegue guardar
R$ 100 por mês não deveria travar dinheiro em produto de prazo longo. O código também
registra que essa é a regra que mais gera reclamação de cliente jovem e arrojado — e a que
mais evita que alguém precise sacar antes do prazo e perder dinheiro no caminho.

**Quando ela dispara:** com salário de até R$ 600 por mês (15% de R$ 600 = R$ 90). A
partir de R$ 700, o aporte passa de R$ 100 e a regra deixa de valer. Como esse patamar
está bem abaixo do salário mínimo, na prática essa regra quase nunca deve ser acionada.

### Regra 2 — o piso do Arena Quant

> **O Arena Quant só é indicado se o aporte for de pelo menos R$ 500 por mês.**

Quem faz a pontuação máxima (6) mas não alcança esse valor não recebe o Quant: a indicação
"desce um degrau" e vira **CDB Longo Prazo**.

**Quando ela dispara:** com salário abaixo de R$ 3.400 (15% de R$ 3.300 = R$ 495, que não
alcança o piso; 15% de R$ 3.400 = R$ 510, que alcança).

**Atenção a uma consequência não óbvia:** o produto para onde a pessoa é desviada prende o
dinheiro por **5 anos** — o prazo mais longo de toda a prateleira. Ou seja, a regra reduz o
risco de perder dinheiro, mas aumenta bastante o risco de ficar sem acesso a ele. Para
alguém que não consegue comprometer R$ 500 por mês, ficar cinco anos sem poder sacar pode
ser o problema maior.

### O que essas regras **não** verificam

Nenhuma das duas pergunta se a pessoa **já tem** uma reserva de emergência. A descrição do
CDB Reserva diz que é "onde a reserva de emergência deve ficar antes de qualquer outra
coisa", mas o sistema nunca checa isso diretamente — ele usa o tamanho do aporte como
substituto. O resultado é que alguém com salário alto e nenhuma economia pode ser mandado
direto para um produto de 5 anos ou para o Arena Quant, sem que a pergunta sobre reserva
seja feita uma única vez.

---

## 5. Casos-limite, com exemplos

Todos os exemplos abaixo usam valores que o formulário realmente aceita. O formulário só
permite idade de 18 a 100 anos e salário em múltiplos de R$ 100.

### 5.1 Um aumento de salário troca "garantido" por "pode dar prejuízo"

| Pessoa | Resultado |
|---|---|
| 26 anos, arrojado, **R$ 3.300** | CDB Longo Prazo — risco Médio, rendimento garantido |
| 26 anos, arrojado, **R$ 3.400** | Arena Quant — risco Alto, sem garantia nenhuma |

**Por que parece contraditório:** cem reais a mais no salário atravessam o piso de R$ 500
de aporte e mudam a indicação de renda fixa garantida para uma estratégia que pode fechar
o ano no negativo. A pessoa não mudou de opinião sobre risco — ela só passou a ganhar um
pouco mais. E a mudança vai na direção oposta da intuição: ganhar mais deveria ampliar as
opções, não empurrar automaticamente para o produto mais arriscado da casa.

### 5.2 A tela mostra "6 de 6" e entrega o produto de quem fez 5

**Exemplo:** 26 anos, arrojado, salário de R$ 3.000 (aporte de R$ 450).

O resultado exibido diz, com todas as letras, *"Pontuação de risco calculada: 6 de 6"* — e
logo abaixo apresenta o CDB Longo Prazo, que é o produto correspondente à pontuação 5.

**Por que parece contraditório:** a pontuação mostrada na tela é a bruta, calculada antes
de a Regra 2 entrar em ação. O cliente vê a nota máxima e recebe outra coisa, sem que a
tela explique que a nota exibida não é a que valeu. Há uma linha de justificativa
mencionando o piso de R$ 500, mas o número grande continua contando outra história.

Vale notar que o mesmo acontece na Regra 1: alguém de 25 anos e arrojado com salário de
R$ 600 vê *"6 de 6"* na tela e recebe o CDB Reserva, o produto mais conservador de todos.

### 5.3 Um aniversário muda o produto inteiro

| Pessoa | Resultado |
|---|---|
| **29 anos**, arrojado, R$ 6.000 | Arena Quant — risco Alto |
| **30 anos**, arrojado, R$ 6.000 | CDB Longo Prazo — risco Médio |

**Por que parece contraditório:** é o maior salto do sistema e acontece de um dia para o
outro. As faixas de idade são cortes secos, então alguém que refizer a simulação no dia do
aniversário recebe uma recomendação diferente sem que nada na vida financeira tenha
mudado. O mesmo degrau existe na virada dos 49 para os 50 anos.

### 5.4 Declarar-se conservador e sair com o dinheiro preso por mais tempo

| Pessoa | Resultado |
|---|---|
| 25 anos, **conservador**, R$ 4.000 | CDB Progressivo — **2 anos** de carência |
| 55 anos, **moderado**, R$ 4.000 | LCI Isenta — **1 ano** de carência |

**Por que parece contraditório:** quem se declarou conservador fica com o dinheiro travado
pelo dobro do tempo de quem se declarou moderado. A juventude vale 3 pontos e compensa com
folga o único ponto do perfil declarado. Na prática, a idade fala mais alto que a resposta
que o cliente deu — e ele não tem como perceber isso pela tela.

### 5.5 Ganhar muito bem e receber o produto de entrada

**Exemplo:** 60 anos, conservador, salário de R$ 20.000 (aporte de R$ 3.000 por mês).

O resultado é o **CDB Reserva** — exatamente o mesmo produto indicado a quem consegue
guardar R$ 90 por mês.

**Por que parece contraditório:** alguém disposto a aportar R$ 3.000 mensais razoavelmente
espera algo desenhado para esse volume. E, como já apontado, essa pessoa nunca alcançará
outro produto: 50 anos ou mais somado a conservador dá 2 pontos, o mínimo possível, não
importa quanto dinheiro ela traga.

### 5.6 Formulário em branco não avisa nada

Se algum dos três campos ficar vazio — ou se o salário for zero — o botão simplesmente não
faz nada. Nenhuma mensagem de erro aparece, nenhum aviso é exibido. Para o cliente, parece
que o site travou.

### 5.7 Observação técnica menor

Se o campo de perfil receber um valor diferente de "arrojado" ou "moderado", a pessoa é
tratada como conservadora. Pela tela isso não acontece, porque a lista oferece apenas as
três opções válidas — vale registrar apenas caso o simulador venha a ser alimentado por
outro caminho no futuro.

---

## 6. Perguntas para levar ao time de produto

**Sobre a pontuação exibida**

1. A tela mostra "Pontuação de risco calculada: 6 de 6" mesmo quando uma regra de corte já
   mudou o produto. Devemos exibir a pontuação efetiva, esconder o número nesses casos, ou
   deixar explícito que a nota foi substituída por uma regra?

**Sobre o peso da idade**

2. Idade e perfil declarado têm exatamente o mesmo peso e são somados. Isso é intencional?
   Uma pessoa de 25 anos que responde "conservador" recebe carência de 2 anos, mais longa
   que a de uma pessoa de 55 anos que respondeu "moderado". Estamos confortáveis em deixar
   a idade sobrepor a resposta do cliente?
3. As faixas de idade são cortes secos. Uma pessoa recebe uma recomendação aos 29 anos e
   outra aos 30, sem nada mais ter mudado. Faz sentido suavizar essa transição?

**Sobre a Regra 2 (piso do Arena Quant)**

4. Quando o cliente não alcança o piso de R$ 500, a indicação cai no CDB Longo Prazo — o
   produto com o prazo **mais longo** da prateleira. Se a intenção é proteger quem tem
   menos folga, faz sentido enviá-lo para o produto do qual é mais difícil sair? Um
   destino mais líquido não serviria melhor?
5. O Arena Quant só é alcançável por quem tem menos de 30 anos e se declara arrojado.
   Essa exclusividade é uma decisão de produto deliberada ou um efeito colateral da conta?

**Sobre a reserva de emergência**

6. O sistema nunca pergunta se a pessoa já tem reserva. Usamos o aporte mensal como
   substituto dessa informação. Alguém com salário alto e nenhuma economia pode ser
   mandado direto ao Arena Quant. Devemos incluir essa pergunta no formulário?

**Sobre os dados coletados**

7. O aporte é sempre 15% do salário, sem considerar dívidas, dependentes ou gastos fixos.
   Duas pessoas com o mesmo salário recebem a mesma sugestão, mesmo que uma esteja no
   cheque especial. Esse número deveria ser ajustável pelo cliente?
8. A Regra 1 (aporte mínimo de R$ 100) só dispara para salários de até R$ 600 por mês,
   valor bem abaixo do salário mínimo. O código a descreve como a regra que mais gera
   reclamação — o piso está calibrado no valor certo, ou foi definido em outro contexto?

**Sobre o teto dos clientes mais velhos**

9. Quem tem 50 anos ou mais e se declara conservador sempre recebe o CDB Reserva, mesmo
   aportando R$ 3.000 por mês, e nunca alcança a LCI Isenta. Temos uma oferta pensada para
   esse cliente?

**Sobre a experiência**

10. O formulário incompleto não dá nenhum retorno visual. Podemos incluir uma mensagem de
    erro?
