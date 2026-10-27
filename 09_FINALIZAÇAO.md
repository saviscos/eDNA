## 10. Mapeamento de Leituras (Read Mapping) e Geração da Matriz

### Introdução
Até o momento, nós processamos nossos dados para encontrar as sequências biológicas únicas representativas do nosso estudo (as nossas OTUs ou ASVs), filtrando todos os erros. No entanto, para fins ecológicos, não basta saber *quais* sequências existem; precisamos saber **quantas vezes cada sequência aparece em cada amostra**.

O mapeamento de leituras (*read mapping*) é o processo de pegar o nosso "banco de dados" de sequências limpas (`all.otus.fasta`) e comparar todas as nossas leituras originais (amostra por amostra) contra ele. O resultado final dessa etapa é a **Matriz de Abundância** (ou Tabela de OTUs/ASVs), que é o coração de qualquer análise de diversidade em softwares como o R.

### Padronizando a Nomenclatura (Relabeling)
Se você realizou a etapa de agrupamento (*Clustering* Guloso) na Parte 9 utilizando o parâmetro `--relabel OTU_`, as suas sequências já estão perfeitamente nomeadas (ex: `OTU_1`, `OTU_2`). 

*Nota: Caso você tenha optado por pular o agrupamento e trabalhar exclusivamente com ASVs (`asvs_finais.fasta`), suas sequências ainda estarão com o nome original do sequenciador Illumina. Antes de mapear, você deve renomeá-las usando o comando abaixo:*
```bash
vsearch --fastx_filter asvs_finais.fasta \
    --relabel ASV_ \
    --fastaout asvs_finais_renomeadas.fasta
```

### Executando o Mapeamento Global
Vamos usar o comando `--usearch_global` do VSEARCH. Ele pegará o arquivo contendo todas as leituras filtradas iniciais (que contêm as tags `sample=T10`, `sample=T11`, etc.) e fará o alinhamento contra o nosso arquivo de OTUs.

```bash
vsearch --usearch_global seqs_filtradas.fasta \
    --db all.otus.fasta \
    --id 0.97 \
    --otutabout all.otutab.txt
```

**Entendendo os parâmetros-chave:**
*   `--usearch_global`: O arquivo de consulta (*query*) contendo as leituras limpas de todas as amostras.
*   `--db`: O nosso arquivo de OTUs representativas.
*   `--id 0.97`: Exige uma identidade de 97% para que a leitura seja contada para aquela OTU (acompanhando o limite que usamos para criá-las). *Se estiver mapeando ASVs, use `--id 0.995` para garantir rigor absoluto.*
*   `--otutabout`: Instrui o VSEARCH a gerar a matriz de abundância formatada, cruzando as amostras nas colunas e as OTUs nas linhas.

Você pode inspecionar sua matriz final executando `more all.otutab.txt` no terminal.

---

## 11. Classificação Taxonômica (Taxonomic Assignment) com SINTAX

### Introdução
Chegamos à última etapa de bioinformática no terminal! Nossas OTUs representam a diversidade biológica das amostras e já temos suas abundâncias, mas elas ainda são sequências "anônimas" (ex: `OTU_10`). 

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

### 2. Classificando as OTUs
Com o nosso banco `.udb` pronto, a classificação é um processo direto e ultrarrápido:

```bash
vsearch --sintax all.otus.fasta \
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
Se a `OTU_10` alcançou 100% de confiança até a Classe, 85% para a Ordem *Coleoptera*, mas baixa confiança para os níveis inferiores, a quarta coluna terminará prudentemente em `o:Coleoptera`, garantindo segurança e rigor científico na sua pesquisa.

### Próximos Passos: Indo para o R
Parabéns! Você concluiu a parte computacional estrutural de metabarcoding em Linux. Agora você tem os dois arquivos essenciais para a ecologia:
1.  **A Matriz de Abundância** (`all.otutab.txt`)
2.  **A Tabela de Taxonomia** (`taxonomia_sintax.tsv`)

Na próxima fase da disciplina, esses dados serão importados para o ambiente **R** (através de pacotes como o `phyloseq`) para gerar gráficos de barras de abundância relativa, calcular índices de diversidade (Shannon, Simpson) e plotar ordenações ecológicas (PCoA, NMDS).