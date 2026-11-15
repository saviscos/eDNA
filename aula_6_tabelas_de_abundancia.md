# Aula 6: Criando as Tabelas de Abundância (OTUs e ASVs)

Na aula anterior, nós criamos os nossos dois "dicionários" de biodiversidade: um usando a abordagem clássica (OTUs a 97%) e outro usando a abordagem moderna e exata (ASVs). 

No entanto, perdemos uma informação no meio do caminho quando misturamos tudo na Aula 3: não sabemos quantas cópias de cada OTU ou ASV vieram da amostra T10, da T12 ou da T16.

O objetivo de hoje é fazer o **Mapeamento de Leituras**. Vamos pegar todas as sequências originais e alinhá-las contra os nossos dois dicionários. O VSEARCH vai usar as etiquetas `sample=` que criamos para contar cada "hit" e montar nossas planilhas automaticamente.

---

### 1. Mapeamento para a Tabela de ASVs

Primeiro, vamos cruzar nossas leituras brutas misturadas com as nossas Variantes Exatas (ASVs). 
Certifique-se de estar na pasta `resultados_vsearch` e rode o comando:

```bash
vsearch --usearch_global todas_amostras_concatenadas.fastq \
        --db ASVs.fasta \
        --id 0.99 \
        --otutabout tabela_abundancia_ASVs.txt
```

---

### 2. Mapeamento para a Tabela de OTUs

Agora, faremos o exato mesmo processo, mas usando o nosso dicionário clássico de OTUs. Note que os parâmetros são idênticos, mudamos apenas o arquivo de referência (`--db`) e o nome da tabela de saída (`--otutabout`).

```bash
vsearch --usearch_global todas_amostras_concatenadas.fastq \
        --db OTUs_97.fasta \
        --id 0.97 \
        --otutabout tabela_abundancia_OTUs.txt
```

---

### 3. Visualizando e Comparando as Tabelas

As suas matrizes ecológicas estão prontas! Esses arquivos `.txt` podem ser abertos facilmente no Excel, no R, ou no Python para gerar gráficos e calcular a diversidade.

Para dar uma espiada rápida pelo terminal, vamos olhar as primeiras linhas da tabela de ASVs formatando as colunas com o comando `column -t`:

```bash
head -n 10 tabela_abundancia_ASVs.txt | column -t
```

*Você verá a primeira coluna listando `ASV_1`, `ASV_2`... e as colunas seguintes (T10, T12...) com a quantidade de DNA encontrado em cada amostra.*

Agora faça o mesmo para a tabela de OTUs:
```bash
head -n 10 tabela_abundancia_OTUs.txt | column -t
```
### 4. Alinhando com banco de dados:

Baixando o banco de dados:

```bash
pip install gdown
```

```bash
gdown "https://drive.google.com/file/d/17-DI1dU4BhxFxKX2mJlYyHCwH_lDnxeF/view"
```


### 4.1 Classifica as OTUs utilizando o MIDORI2 COI
vsearch --sintax OTUs_97.fasta \
        --db MIDORI2_LONGEST_NUC_GB273_CO1_SINTAX.fasta \
        --tabbedout Taxonomia_OTUs.tsv \
        --sintax_cutoff 0.8 \
        --strand both \
        --threads 4


### 4.2 Classifica as ASVs utilizando o MIDORI2 COI
vsearch --sintax ASVs.fasta \
        --db MIDORI2_LONGEST_NUC_GB273_CO1_SINTAX.fasta \
        --tabbedout Taxonomia_ASVs.tsv \
        --sintax_cutoff 0.8 \
        --strand both \
        --threads 4

# Parâmetros
--sintax	Seleciona o algoritmo de classificação taxonômica
--db	Indica o banco de referência MIDORI2
--tabbedout	Define o arquivo de resultados
--sintax_cutoff 0.8	Exige suporte bootstrap de pelo menos 80% para aceitar cada nível taxonômico
--strand both	Considera as duas orientações das sequências

