# Previsão de Arrecadação Tributária Municipal — Aracaju

> Projeto da disciplina **Machine Learning – Prática e Projeto** · Prof. Fabio Santos
> Trilha: **Aprendizado Supervisionado** · Base para o TCC

Prever quanto o município de Aracaju vai arrecadar de **ISS, IPTU e ITBI** nos próximos meses, usando dados públicos do Tesouro Nacional (Siconfi), e medir o quanto essa previsão erra em comparação com a previsão oficial da prefeitura.

**Autora:** Fernanda Amaral de Souza

---

## Sumário

1. [Problema](#1-problema)
2. [Usuário](#2-usuário)
3. [Dados](#3-dados)
4. [Métrica de sucesso](#4-métrica-de-sucesso)
5. [Baseline](#5-baseline)
6. [Abordagem proposta](#6-abordagem-proposta)
7. [Riscos](#7-riscos)
8. [Ponte para o TCC](#8-ponte-para-o-tcc)
9. [Uso de IA](#9-uso-de-ia)

---

## 1. Problema

A Prefeitura de Aracaju estima a arrecadação de IPTU, ISS e ITBI para a **Lei Orçamentária Anual (LOA)** sem saber o quanto essa previsão costuma errar, o que pode levar a cortes de gastos no meio do ano ou a investimentos que deixam de ser planejados.

## 2. Usuário

O resultado é pensado para a **equipe de planejamento e orçamento da Secretaria da Fazenda de Aracaju**: as pessoas que, todo ano, precisam responder *"quanto vamos arrecadar?"* na hora de montar a LOA e que depois acompanham, mês a mês, se o número está se confirmando.

Hoje essa conta costuma ser feita de um jeito simples: pega-se o que entrou no ano anterior e corrige-se pela inflação. Funciona, mas ninguém sabe ao certo o quanto essa estimativa costuma errar.

Com o projeto, a equipe passa a ter **uma previsão para cada mês e cada tributo**, junto com uma noção clara da **margem de erro**. Isso dá mais segurança para definir metas e mais argumento para defender os números quando forem questionados.

## 3. Dados

| Fonte | Conteúdo | Acesso |
|---|---|---|
| **Siconfi – API de Dados Abertos** (Tesouro Nacional) | RREO, Anexo 03 — receita corrente líquida por tributo, Aracaju (`id_ente=2800308`), 2015 a 2026 | Livre, sem cadastro, JSON |
| **IBGE – SIDRA, Tabela 1737** | IPCA mensal, para deflacionar os valores | Livre, JSON |

**Endpoints testados:**

```text
# RREO Anexo 03 – Aracaju (exemplo: exercício 2015, 4º bimestre)
https://apidatalake.tesouro.gov.br/ords/siconfi/tt/rreo?an_exercicio=2015&nr_periodo=4&co_tipo_demonstrativo=RREO&no_anexo=RREO-Anexo%2003&id_ente=2800308

# IPCA – número-índice mensal
https://apisidra.ibge.gov.br/values/t/1737/n1/all/v/2266/p/all
```

- Cada consulta ao RREO traz os **últimos 12 meses** de arrecadação, separados por tributo, o que permite montar uma série mensal de IPTU, ISS e ITBI com cerca de **130 meses** de histórico.
- O IPCA é usado para trazer os valores a preços constantes — comparar reais de 2015 com reais de 2025 sem esse ajuste distorceria a análise.
- As duas bases são cruzadas pelo **mês de referência**.
- **Dados sensíveis:** nenhum. São dados públicos e agregados, sem informação pessoal de contribuintes.

## 4. Métrica de sucesso

**Métrica principal: WAPE** (*Weighted Absolute Percentage Error*)

$$
\text{WAPE} = \frac{\sum_t |y_t - \hat{y}_t|}{\sum_t |y_t|}
$$

Mostra o tamanho do erro em relação ao total arrecadado — na prática, responde algo como *"o modelo errou 8% do que entrou no ano"*. Foi escolhida porque é fácil de explicar para quem trabalha com orçamento e permite comparar tributos de tamanhos muito diferentes, como ISS e ITBI.

**Validação:** *backtesting* — treinar o modelo com dados até um certo ano e avaliar como ele teria se saído no ano seguinte, que já é conhecido.

**Critério de sucesso:** o modelo deve ter um WAPE **pelo menos 15% menor** que o da previsão oficial da prefeitura.

## 5. Baseline

Duas referências a serem superadas:

1. **Sazonal ingênuo (*seasonal naive*)** — cada mês arrecada o mesmo que o mesmo mês do ano anterior. Parece ingênuo, mas é difícil de bater, porque a arrecadação tem um padrão anual forte (ex.: pico do IPTU em janeiro e campanhas promovidas pela prefeitura).
2. **Previsão oficial da prefeitura** — informada ao Tesouro e disponível nos dados do RREO. É a comparação que dá sentido prático ao projeto: não basta superar uma regra simples, o objetivo é saber se o modelo faria melhor do que o número que o município realmente usou.

## 6. Abordagem proposta

Comparação entre dois caminhos:

| Família | Modelos | Como aprende |
|---|---|---|
| Estatística clássica de séries temporais | **SARIMA**, **ETS** | Tendência e sazonalidade mensal a partir do histórico |
| Machine Learning | **LightGBM** | Combina *features* como defasagens (*lags*) dos meses anteriores e o mês do ano |

- **Granularidade mensal:** todos os modelos preveem mês a mês. Com dados anuais haveria apenas ~10 observações; no nível mensal são ~120 (12 × 10). As previsões mensais são depois **somadas** para chegar ao total anual que interessa à LOA.
- **Separação dos dados:**
  - **2025** reservado como **conjunto de teste final**, avaliado uma única vez;
  - meses já publicados de **2026** usados como confirmação extra (*out-of-sample*).
- **Entrega:** previsão para **2027** junto com a margem de erro esperada.

## 7. Riscos

| Risco | Descrição |
|---|---|
| Mudança de formato | Nomes ou códigos de contas do RREO podem ter mudado entre 2015 e 2026, exigindo padronização cuidadosa |
| Anos atípicos | Recessão de 2015–2016 e pandemia de 2020 distorcem a série e podem confundir o modelo |
| Retificações | A prefeitura pode corrigir valores já enviados ao Tesouro |
| Previsão inicial x atualizada | É preciso confirmar se os dados trazem a previsão **inicial** da LOA, e não só a atualizada ao longo do ano — o que tornaria a comparação injusta |
| Histórico curto | Pouco histórico para séries temporais; a incerteza dos resultados precisa ser assumida e explicada |

## 8. Ponte para o TCC

O projeto continua no TCC, com a mesma base e o mesmo pipeline.

A prefeitura não faz uma previsão só: ela revisa o número ao longo do ano, conforme a arrecadação vai entrando. No TCC, a proposta é construir um modelo que:

- se **atualize a cada novo relatório bimestral**;
- entregue, em vez de um número único, uma **faixa de valores com nível de confiança** (ex.: *"entre R$ X e R$ Y, com 80% de chance"*).

**Pergunta de pesquisa:** essa previsão revisada erra menos do que as revisões oficiais? E a faixa de incerteza fica mais estreita com o passar do ano?

## 9. Uso de IA

O **Claude** (Anthropic) é usado como apoio durante o projeto: para tirar dúvidas, entender conceitos, revisar textos e apoiar em trechos de código mais complexos. As decisões, o código e as análises são da autora, e tudo o que vem da IA é conferido antes de entrar no trabalho.
