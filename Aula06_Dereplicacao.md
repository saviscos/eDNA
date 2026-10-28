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

O que o `--derep_fulllength` faz?

Ele pega aquele arquivo gigante com milhares de leituras filtradas e procura todas as sequências de DNA que são 100% idênticas do primeiro ao último par de bases.
Em vez de o computador carregar o peso de analisar 10.000 cópias da exata mesma sequência, ele guarda apenas uma cópia dela e anota no cabeçalho quantas vezes ela apareceu (usando o comando --sizeout, que escreve size=10000; no nome).

O parâmetro --minuniquesize 2 diz: "Se uma sequência apareceu apenas uma única vez em todo o estudo (singleton), jogue fora". Isso é feito porque sequências que aparecem apenas 1 vez costumam ser erros de PCR ou de sequenciamento, e não organismos reais.
