## 9. Classificação Taxonômica (Taxonomic Assignment) com SINTAX

### Introdução
Chegamos à última etapa de bioinformática no terminal! Nossas ASVs (ou OTUs) representam a diversidade biológica das amostras e já temos suas abundâncias na matriz gerada anteriormente, mas elas ainda são sequências "anônimas" (ex: `ASV_10`). 

A classificação taxonômica compara essas sequências contra um banco de dados de referência validado cientificamente para identificar a qual linhagem (Filo, Classe, Ordem, Família, Gênero, Espécie) cada uma pertence. Utilizaremos o algoritmo **SINTAX** (embutido no VSEARCH) contra o prestigiado banco de dados **MIDORI**, focado em marcadores mitocondriais eucariotos como o COI.

### 1. Download e Preparação do Banco de Dados MIDORI
O banco de dados MIDORI é disponibilizado no repositório Zenodo. Para a nossa análise, baixaremos a versão contendo apenas sequências únicas de COI em formato de nucleotídeos, já pré-formatada para o algoritmo SINTAX.

No seu terminal, execute os comandos abaixo:

```bash
# 1. Baixe o banco de dados compactado direto do Zenodo
wget https://zenodo.org/records/22685701/files/MIDORI2_UNIQ_NUC_GB273_CO1_SINTAX.fasta.gz

# 2. Descompacte o arquivo (isso extrairá o formato .fasta)
gunzip MIDORI2_UNIQ_NUC_GB273_CO1_SINTAX.fasta.gz

# 3. Converta o FASTA de texto para um banco binário otimizado (.udb) do VSEARCH
vsearch --makeudb_usearch MIDORI2_UNIQ_NUC_GB273_CO1_SINTAX.fasta \
    --output MIDORI2_UNIQ_NUC_GB273_CO1_SINTAX.udb
```
*Dica: Criar o arquivo `.udb` acelera drasticamente a etapa de busca e consome muito menos memória RAM do seu computador ou servidor.*

### 2. Classificando as Sequências
Com o nosso banco `.udb` pronto, a classificação é um processo direto e ultrarrápido. 

*(Nota: O comando abaixo usa as ASVs. Se você optou por OTUs, troque o arquivo de entrada para `otus_finais.fasta`)*:

```bash
vsearch --sintax asvs_finais_renomeadas.fasta \
    --db MIDORI2_UNIQ_NUC_GB273_CO1_SINTAX.udb \
    --tabbedout taxonomia_sintax.tsv \
    --strand both \
    --sintax_cutoff 0.8
```

**Entendendo os parâmetros-chave:**
*   `--sintax`: Invoca o algoritmo de classificação taxonômica preditiva.
*   `--tabbedout`: Gera o arquivo de saída em formato tabular (TSV).
*   `--strand both`: Verifica tanto a fita senso quanto a antissenso, garantindo que leituras invertidas não fiquem sem identificação.
*   `--sintax_cutoff 0.8`: Define o valor mínimo de confiança (bootstrap) para aceitar uma taxonomia. Um valor de 0.8 (80%) é o padrão rigoroso e amplamente aceito na ecologia molecular. Se a confiança para confirmar a Família for 85%, mas para o Gênero for apenas 40%, o SINTAX deixará o Gênero em branco, evitando identificações falsas.

### 3. Inspecionando o Resultado Final
Visualize sua taxonomia final com `more taxonomia_sintax.tsv`. 

A estrutura da tabela apresenta 4 colunas. A mais importante é a **quarta coluna**, que contém a taxonomia limpa e filtrada pelo seu *cutoff* de 80%. 

*Exemplo prático de interpretação:*
Se a `ASV_10` alcançou 100% de confiança até a Classe, 85% para a Ordem *Coleoptera*, mas baixa confiança para os níveis inferiores, a quarta coluna terminará prudentemente em `o:Coleoptera`, garantindo segurança e rigor científico na sua pesquisa.

### Próximos Passos: Indo para o R
Parabéns! Você concluiu a parte computacional estrutural de metabarcoding em Linux. Agora você tem os dois arquivos essenciais para a ecologia:
1.  **A Matriz de Abundância** (ex: `tabela_ASVs.txt`)
2.  **A Tabela de Taxonomia** (`taxonomia_sintax.tsv`)

Na próxima fase da disciplina, esses dados serão importados para o ambiente **R** (através de pacotes como o `phyloseq`) para gerar gráficos de barras de abundância relativa, calcular índices de diversidade (Shannon, Simpson) e plotar ordenações ecológicas (PCoA, NMDS).
