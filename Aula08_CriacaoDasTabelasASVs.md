## 8. ASVs, OTUs e Algoritmos de Agrupamento (Clustering)

### Introdução

Nas seções anteriores, processamos nossos dados brutos até obtermos um conjunto de sequências únicas sem erros: as **ASVs** (*Amplicon Sequence Variants*). Idealmente, essas ASVs representam todos os haplótipos (variantes biológicas verdadeiras) presentes nas nossas amostras. Elas englobam tanto a variação intraespecífica (dentro da mesma espécie) quanto interespecífica (entre espécies diferentes).

Nesta seção, vamos examinar maneiras de agrupar essas ASVs em Unidades Taxonômicas Operacionais (**OTUs**). Posteriormente, juntaremos nossas sequências biológicas finais com os dados de cada amostra para gerar a nossa **Tabela de Abundância** (a matriz de local x espécie).

### ASVs ou OTUs?

Nos primeiros estudos, as sequências eram sempre agrupadas em OTUs (análogas a espécies) para esconder erros de PCR e remover a variação intraespecífica. Com métodos modernos de filtragem (como o *denoising*), a taxa de erro das ASVs caiu tanto que podemos analisar os dados no nível do haplótipo com confiança.

* **ASVs:** Resolução máxima (nível de linhagem/população). Mais sensíveis a erros residuais.
* **OTUs:** Agrupam variações e criam unidades análogas a espécies, sacrificando a diversidade intraespecífica real.

Se optarmos por trabalhar com OTUs, precisamos de algoritmos para agrupá-las. Abaixo, exploramos as três principais abordagens.

### 1. Clusterização "Gulosa" (*Greedy Clustering* - VSEARCH / UPARSE)

É o método conceitualmente mais simples, rápido e amplamente utilizado na bioinformática clássica de amplicons.

**Como funciona:** Ele ordena as ASVs da mais abundante para a menos abundante. A mais comum vira o "centróide" (sequência representativa) da primeira OTU. A próxima ASV é comparada; se ela tiver, por exemplo, 97% de similaridade com o centróide, ela entra no grupo. Se não, ela forma uma nova OTU.

Para realizar esta clusterização a 97% utilizando as ASVs limpas (sem quimeras) que geramos na aula passada, execute o seguinte comando:

```bash
# Agrupa as ASVs em OTUs com 97% de similaridade
vsearch --cluster_size asvs_finais.fasta \
    --threads 4 \
    --id 0.97 \
    --strand plus \
    --sizein \
    --sizeout \
    --fasta_width 0 \
    --uc all.clustered.uc \
    --relabel OTU_ \
    --centroids all.otus.fasta \
    --otutabout all.otutab.txt
```

**Entendendo os parâmetros principais:**
* `--id 0.97`: O famoso limite taxonômico de 97% de similaridade. Permite até 3% de divergência entre a ASV e o centróide.
* `--strand plus`: Compara apenas na fita direta (nós já orientamos e cortamos as sequências corretamente nas etapas de primer).
* `--uc all.clustered.uc`: Gera um arquivo de rastreabilidade, mostrando exatamente qual ASV foi englobada dentro de qual OTU.
* `--relabel OTU_`: Renomeia os cabeçalhos. Em vez de sequências aleatórias, elas se chamarão `OTU_1`, `OTU_2`, etc.
* `--centroids`: Salva as sequências representativas (o "DNA oficial") de cada OTU.
* `--otutabout`: Um bônus do VSEARCH! Ele já gera uma Tabela de Abundância preliminar em formato de texto para essas OTUs (embora o mapeamento rigoroso com as *reads* originais seja recomendado no próximo passo).

### 2. Delimitação Baseada em Ligação / Redes (*Linkage-based* - Swarm)

O programa **Swarm** utiliza uma filosofia completamente diferente. Em vez de usar um limite global de porcentagem (como os 97%), ele usa um limite *local* baseado no **número absoluto de diferenças** (mutações) entre as sequências.

**Como funciona:** Ele constrói uma rede onde cada ASV é um nó. Se a diferença entre duas ASVs for de apenas 1 nucleotídeo (parâmetro `-d 1`), elas são ligadas. Ele vai caminhando por essa rede unindo sequências até encontrar um "vale" estrutural, separando os grupos de ASVs onde ocorrem mais diferenças. O Swarm não é afetado pela abundância inicial e não força sequências a ficarem num raio arbitrário, mas costuma gerar um número muito maior de OTUs.

### 3. Clusterização Bayesiana (*Bayesian clustering* - CROP)

O software **CROP** utiliza uma abordagem puramente estatística baseada em Cadeias de Markov de Monte Carlo (MCMC) para encontrar a probabilidade natural das sequências pertencerem à mesma espécie biológica.

**Desvantagens:** Enquanto o VSEARCH usa comparações rápidas de texto, o CROP é incrivelmente pesado e computacionalmente lento. Além disso, algoritmos bayesianos não leem tags de abundância (`;size=N`) facilmente, exigindo que você "des-desreplique" seus dados (inserindo as cópias repetidas de volta) antes de rodar. Por isso, ele caiu em desuso em pipelines de alto rendimento.

### Conclusão: Qual escolher?

Para a vasta maioria dos pipelines contemporâneos (e para as nossas aulas), manter a resolução ao nível de **ASVs** é o padrão-ouro. Se houver a necessidade de colapsar para o nível genérico (OTU), a **Clusterização Gulosa (VSEARCH 97%)** ensinada acima oferece o melhor equilíbrio entre velocidade, reprodutibilidade e padronização com a literatura.