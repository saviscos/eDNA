## 7. A Grande Bifurcação: ASVs vs. OTUs

### Introdução
Após a etapa de desreplicação (onde removemos os *singletons* e mantivemos apenas sequências únicas), chegamos a um momento de decisão crucial no nosso pipeline. Historicamente, a ecologia molecular trabalhou agrupando sequências parecidas em **OTUs** (Unidades Taxonômicas Operacionais). Hoje, com algoritmos mais precisos, a tendência é refinar os dados até obtermos sequências 100% exatas, chamadas de **ASVs** (Variantes de Sequência de Amplicon).

Na sua pesquisa real, você escolherá apenas **um** desses caminhos. Aqui, vamos demonstrar como executar ambos.

---

### CAMINHO A: Abordagem por ASVs (Alta Resolução)
Neste caminho, assumimos que as variantes reais existem e que sequências muito parecidas com elas (mas em baixa abundância) são apenas "ruídos" (erros de sequenciamento ou PCR). O objetivo é limpar esse ruído mantendo a resolução de 1 único nucleotídeo de diferença.

#### A1. Denoising (Redução de Ruído com UNOISE3)
O algoritmo UNOISE compara ASVs raras com abundantes e descarta as que são derivadas de erros sistemáticos, mantendo apenas as variantes biológicas verdadeiras.

```
vsearch --cluster_unoise all.derep.fasta \
    --minsize 4 \
    --unoise_alpha 2 \
    --sizein \
    --sizeout \
    --centroids all.denoised.fasta
```
**O que este comando faz:**
*   `--cluster_unoise`: Invoca o algoritmo de correção de erros.
*   `--minsize 4`: Descarta variantes que apareceram menos de 4 vezes no estudo todo (filtro de segurança).
*   `--unoise_alpha 2`: Controla a rigidez do algoritmo ao decidir se uma sequência rara é erro de uma abundante.
*   `--sizein` / `--sizeout`: Instrui o VSEARCH a ler e manter as informações de abundância (`size=N`) nos cabeçalhos, algo crucial para a remoção de quimeras mais à frente.
*   `--centroids`: Salva as ASVs "verdadeiras" no arquivo de saída.

#### A2. Filtragem de Erros Pontuais (Tradução Virtual)
Como o COI é um gene que produz uma proteína funcional, podemos traduzir o DNA virtualmente. Se a sequência tiver erros graves de PCR (como inserções que quebram a leitura) ou for um pseudogene, ela gerará *stop codons* e será descartada.

```
filtertranslate -i all.denoised.fasta \
    -t 5 \
    -y separate \
    -o all.translated
```
**O que este comando faz:**
*   `--translate 5`: Usa a tabela de código genético nº 5 (Mitocondrial de Invertebrados).
*   `--yes separate`: Separa as sequências aprovadas (`pass`) das reprovadas (`fail`).

#### A3. Remoção de Quimeras (*Chimera Removal*)
Quimeras são "monstros de Frankenstein" do DNA: sequências híbridas formadas artificialmente durante a PCR por duas metades de organismos diferentes.

```
vsearch --uchime_denovo all.translated_pass.fasta \
    --sizein \
    --sizeout \
    --fasta_width 0 \
    --nonchimeras asvs_finais.fasta
```
**O que este comando faz:**
*   `--uchime_denovo`: Busca quimeras comparando as sequências do próprio dataset, assumindo que as sequências "pais" são sempre mais abundantes que as quimeras.
*   `--nonchimeras`: O arquivo final (`asvs_finais.fasta`), contendo o seu tesouro biológico limpo de artefatos.

---

### CAMINHO B: Abordagem por OTUs (Agrupamento Clássico)
Neste caminho, agrupamos (clusterizamos) todas as sequências que tenham, por exemplo, 97% de semelhança entre si. A sequência mais abundante do grupo se torna o "representante" (Centróide) daquela OTU. É uma abordagem mais permissiva, excelente para quando o marcador genético varia muito dentro de uma mesma espécie.

#### B1. Agrupamento (*Clustering* a 97%)

```
vsearch --cluster_size all.derep.fasta \
    --id 0.97 \
    --sizein \
    --sizeout \
    --relabel OTU_ \
    --centroids all.otus_com_quimeras.fasta
```
**O que este comando faz:**
*   `--cluster_size`: Agrupa as sequências do maior para o menor (ordem de abundância).
*   `--id 0.97`: Define a similaridade mínima em 97% para que duas sequências façam parte da mesma OTU.
*   `--relabel OTU_`: Limpa o nome original da máquina e renomeia tudo lindamente para `OTU_1`, `OTU_2`, etc.
*   `--centroids`: Salva a sequência representante de cada grupo.

#### B2. Remoção de Quimeras em OTUs
Assim como nas ASVs, as OTUs também podem conter quimeras que precisam ser removidas.

```
vsearch --uchime_denovo all.otus_com_quimeras.fasta \
    --sizein \
    --sizeout \
    --fasta_width 0 \
    --nonchimeras otus_finais.fasta
```
**O que este comando faz:**
*   Aplica o mesmo algoritmo UCHIME usado nas ASVs, gerando o arquivo limpo `otus_finais.fasta`.

*(Nota: Na abordagem de OTUs, o passo de tradução por `filtertranslate` costuma ser omitido porque o limite de 97% de semelhança naturalmente absorve e "esconde" pequenas mutações que causariam quebra de leitura).*

---

### Próximos Passos
Seja qual for o caminho que você escolher (`asvs_finais.fasta` ou `otus_finais.fasta`), o próximo passo é idêntico: pegar esse arquivo limpo e cruzar contra as nossas amostras originais para descobrir qual a abundância de cada ASV/OTU em cada local de coleta (Geração da Matriz de Abundância).
