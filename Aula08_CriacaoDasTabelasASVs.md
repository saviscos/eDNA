## 8. Mapeamento Global: Gerando a Matriz de Abundância

### Introdução
Chegamos à última etapa do nosso pipeline de linha de comando. No momento, temos as nossas sequências biológicas definitivas e limpas de artefatos (em `asvs_finais.fasta` ou `otus_finais.fasta`), mas perdemos a informação de quantas vezes elas aparecem em cada amostra individual.

Para resolver isso, faremos um **Mapeamento Global**. Vamos instruir o VSEARCH a pegar todas as leituras originais (aquelas que etiquetamos com o nome da amostra lá atrás) e alinhá-las contra o nosso arquivo limpo. O programa contará os "matches" e nos devolverá uma Matriz de Abundância (Tabela de ASVs/OTUs), pronta para ser importada no R (Phyloseq).

---

### Gerando a Tabela de ASVs
Se você seguiu o Caminho A (Alta Resolução) na etapa anterior, usaremos as ASVs como nosso banco de dados de referência. Como as ASVs representam variantes exatas, exigimos um rigor altíssimo (99%) no alinhamento para que a leitura seja contada.

Execute o comando:

```bash
# Mapeia as sequências originais contra as ASVs limpas
vsearch --usearch_global todas_amostras_concatenadas.fastq \
    --db asvs_finais.fasta \
    --id 0.99 \
    --otutabout tabela_ASVs.txt
```

---

### Gerando a Tabela de OTUs
Se você seguiu o Caminho B (Agrupamento clássico), usaremos os centróides das OTUs como referência. Como as OTUs englobam uma variação de até 3%, o nosso rigor de alinhamento para contagem deve ser de 97%.

Execute o comando:

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
*   `--db`: Define qual será o nosso "banco de dados" de referência (nossas ASVs ou OTUs limpas).
*   `--id`: Define a identidade mínima exigida para contabilizar uma leitura. (`0.99` para ASVs; `0.97` para OTUs).
*   `--otutabout`: O grande comando final! Ele condensa os resultados do alinhamento e estrutura automaticamente uma tabela no formato de texto (TSV), onde as linhas são as ASVs/OTUs, as colunas são as amostras, e os valores são as contagens (abundância).

### Verificando o Resultado
Para dar uma espiada em como a sua matriz ficou estruturada, visualize as primeiras linhas e colunas do arquivo gerado:

```bash
head -n 5 tabela_ASVs.txt
```

Você verá uma estrutura tabular clássica de ecologia. Com esta matriz e o seu arquivo FASTA final (`asvs_finais.fasta`), o processamento no terminal Linux está concluído. Estamos prontos para migrar para a interface gráfica do RStudio e iniciar as análises de diversidade ecológica com o pacote **Phyloseq**!
