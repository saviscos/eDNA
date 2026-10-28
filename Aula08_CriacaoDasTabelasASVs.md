## 8. Mapeamento Global: Gerando a Matriz de Abundância

### Introdução
Chegamos à última etapa do nosso pipeline de linha de comando. No momento, temos as nossas sequências biológicas definitivas e limpas de artefatos, mas perdemos a informação de quantas vezes elas aparecem em cada amostra individual.

Para resolver isso, faremos um **Mapeamento Global**. Vamos instruir o VSEARCH a pegar todas as leituras originais e alinhá-las contra o nosso arquivo limpo. O programa contará os "matches" e nos devolverá uma Matriz de Abundância (Tabela de ASVs/OTUs), pronta para ser importada no R (Phyloseq).

---

### Gerando a Tabela de ASVs (Caminho A)
Se você seguiu o Caminho de Alta Resolução, suas ASVs ainda possuem os cabeçalhos originais das leituras que as geraram. Antes de mapear, precisamos padronizar os nomes para `ASV_1`, `ASV_2`, etc.

```bash
# 1. Renomeia as ASVs no arquivo final
vsearch --fastx_filter asvs_finais.fasta \
    --fasta_width 0 \
    --relabel ASV_ \
    --fastaout asvs_finais_renomeadas.fasta

# 2. Mapeia as sequências originais contra as ASVs renomeadas
vsearch --usearch_global todas_amostras_concatenadas.fastq \
    --db asvs_finais_renomeadas.fasta \
    --id 0.99 \
    --otutabout tabela_ASVs.txt
```

---

### Gerando a Tabela de OTUs (Caminho B)
Se você seguiu o Caminho de Agrupamento clássico, os centróides já foram renomeados (para `OTU_1`, `OTU_2`) na etapa de clusterização. O nosso rigor de alinhamento para contagem deve ser de 97%.

```bash
# Mapeia as sequências originais contra as OTUs limpas
vsearch --usearch_global todas_amostras_concatenadas.fastq \
    --db otus_finais.fasta \
    --id 0.97 \
    --otutabout tabela_OTUs.txt
```

---

### Entendendo os Parâmetros do Mapeamento
*   `--usearch_global`: Invoca o algoritmo de alinhamento global (procura a melhor correspondência da leitura inteira contra a referência).
*   `--db`: Define qual será o nosso "banco de dados" de referência.
*   `--id`: Define a identidade mínima exigida para contabilizar uma leitura. (`0.99` para ASVs; `0.97` para OTUs).
*   `--otutabout`: O grande comando final! Ele condensa os resultados do alinhamento e estrutura automaticamente uma tabela no formato de texto (TSV), onde as linhas são as ASVs/OTUs, as colunas são as amostras, e os valores são as contagens (abundância).

### Verificando o Resultado
Para dar uma espiada em como a sua matriz ficou estruturada, visualize as primeiras linhas e colunas do arquivo gerado:

```bash
head -n 10 tabela_ASVs.txt
```

Você verá uma estrutura tabular clássica de ecologia. Com esta matriz e o seu arquivo FASTA final, o processamento no terminal Linux está concluído. Estamos prontos para iniciar a **Atribuição Taxonômica** e migrar para o **RStudio**!
