## 6. Desreplicação (Dereplication)

### Introdução
Nosso arquivo FASTA filtrado contém todas as leituras potencialmente válidas para o nosso projeto. Agora que já utilizamos e descartamos os dados de qualidade (escores Phred), podemos percorrer todas essas leituras e agrupar as que são exatamente idênticas. 

Esta etapa é chamada de **desreplicação**, pois "descartamos" as leituras replicadas, mantendo apenas as sequências únicas, mas registrando cuidadosamente o número de vezes que cada sequência única apareceu no conjunto de dados (sua abundância).

A partir de agora, o objetivo principal é analisar essas sequências únicas e filtrar aquelas que podem ser errôneas (artefatos). Não há necessidade de realizar processos computacionalmente intensivos (como detecção de quimeras) em milhares de leituras idênticas; é muito mais inteligente e rápido desreplicar, reduzir o volume de dados e trabalhar apenas com as sequências únicas.

### Executando a Desreplicação e Removendo Singletons
Faremos a desreplicação utilizando o VSEARCH. Neste passo, aplicaremos um filtro biológico muito importante: a remoção de *singletons* (sequências que aparecem apenas uma única vez em todo o conjunto de dados). Em estudos de eDNA, *singletons* são majoritariamente erros de PCR ou de sequenciamento.

Certifique-se de estar no diretório contendo o seu arquivo FASTA filtrado da etapa anterior e execute o seguinte script:

```bash
# Desreplica as sequências e remove singletons (tamanho mínimo = 2)
vsearch --derep_fulllength seqs_filtradas.fasta \
    --minuniquesize 2 \
    --sizein \
    --sizeout \
    --fasta_width 0 \
    --uc all.derep.uc \
    --output all.derep.fasta
```

**Entendendo os parâmetros-chave:**

* `--derep_fulllength`: Compara as sequências em todo o seu comprimento para agrupá-las.
* `--minuniquesize 2`: **Filtro de singletons.** Descarta qualquer sequência que tenha abundância igual a 1.
* `--sizein` e `--sizeout`: O `--sizeout` adiciona a anotação `;size=N;` no cabeçalho de cada sequência no FASTA de saída, onde `N` é o número de vezes que a sequência original apareceu. O `--sizein` permite que o programa leia tamanhos anteriores, se existirem.
* `--fasta_width 0`: Mantém a sequência inteira na mesma linha, sem quebras (ideal para manipular no terminal).
* `--uc`: Gera um arquivo de rastreabilidade, registrando quais leituras originais foram agrupadas em cada sequência única.

### Exercício

1. Use o comando `head all.derep.fasta` para ver como os cabeçalhos das sequências estão formatados. Note a anotação de tamanho (ex: `size=45;`).
2. Use o `grep -c "^>" all.derep.fasta` para contar quantas sequências únicas restaram. Compare esse número com a quantidade de sequências no arquivo antes da desreplicação. A redução de volume de dados deve ser massiva.

### O que são essas sequências agora?
Neste ponto, temos um conjunto de sequências de amplicon únicas recuperadas do nosso projeto. Esta etapa gerou o que chamamos de **Variantes de Sequência de Amplicon (ASVs - *Amplicon Sequence Variants*)**. 

Elas representam putativamente todas as variantes únicas do nosso marcador genético na comunidade amostrada (os nossos haplótipos). No entanto, sabemos que ainda é muito provável que existam erros estruturais complexos presentes nestas ASVs (como sequências quiméricas formadas durante a PCR). As próximas etapas focarão em identificar e remover esses últimos artefatos para garantir que nossas ASVs sejam representações biológicas verdadeiras.