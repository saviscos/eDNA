# Aula 5: Agrupamento Biológico (OTUs vs. ASVs)

Até agora, nós limpamos nossos dados: unimos as fitas, removemos sequências de baixa qualidade, agrupamos as idênticas (desreplicação) e excluímos os artefatos de PCR (quimeras). O nosso arquivo atual, `sequencias_sem_quimeras.fasta`, contém sequências puras.

No entanto, o sequenciador ainda pode ter cometido erros minúsculos de 1 ou 2 letras que passaram pelos filtros. Além disso, indivíduos da mesma espécie podem ter pequenas variações naturais no DNA. Como transformamos essas sequências soltas em "Espécies" para a ecologia?

Existem duas abordagens principais: **OTUs** (Clássica) e **ASVs** (Moderna).

---

## 1. Abordagem Clássica: OTUs (Unidades Taxonômicas Operacionais)

A abordagem tradicional assume que se duas sequências de DNA são **97% iguais**, elas provavelmente pertencem à mesma espécie bacteriana ou organismo. 

Neste método, o VSEARCH pega a sequência mais abundante e a define como o "Centroide". Depois, ele varre as outras sequências; se elas forem pelo menos 97% parecidas com o centroide, elas são aglomeradas dentro do mesmo grupo (OTU).

Certifique-se de estar na pasta `resultados_vsearch` e rode o comando:

```bash
# Agrupando em OTUs (97% de similaridade)
vsearch --cluster_size sequencias_sem_quimeras.fasta \
        --id 0.97 \
        --centroids OTUs_97.fasta \
        --relabel OTU_ \
        --sizein \
        --sizeout \
        --fasta_width 0
```

### Entendendo os Parâmetros:
* `--cluster_size`: Comando que agrupa as sequências, começando pelas mais abundantes.
* `--id 0.97`: Define a "régua" biológica de 97% de similaridade.
* `--centroids`: Arquivo de saída que guardará apenas uma sequência representante de cada OTU.
* `--relabel OTU_`: Um toque de mestre! Ele apaga aquele cabeçalho gigante do sequenciador e renomeia nossas sequências finais para um formato limpo: `OTU_1`, `OTU_2`, `OTU_3`, etc.
* `--sizein` e `--sizeout`: Mantêm as anotações de abundância.

---

## 2. Abordagem Moderna: ASVs (Variantes de Sequência Exatas)

O paradigma das ASVs argumenta que a biologia não é engessada em "97%". Às vezes, duas espécies diferentes variam em apenas 1 letra (99.5% iguais)! Agrupar a 97% misturaria essas duas espécies em uma só.

A abordagem ASV tenta recuperar a sequência de DNA **exata**. Em vez de usar uma porcentagem fixa de corte, ela usa o algoritmo **UNOISE**. Ele observa a abundância: se ele encontrar uma sequência com 1.000 cópias e outra, a apenas 1 letra de distância, com 2 cópias, a estatística nos diz que a de 2 cópias é apenas um erro de sequenciamento da primeira. Ele "corrige" o erro e aglomera. Se a sequência parecida tiver 800 cópias, o programa conclui que ambas são reais e mantém as duas como ASVs separadas.

Para gerar as ASVs no VSEARCH, execute:

```bash
# Agrupando/Corrigindo erros com o algoritmo UNOISE (ASVs)
vsearch --cluster_unoise sequencias_sem_quimeras.fasta \
        --centroids ASVs.fasta \
        --relabel ASV_ \
        --sizein \
        --sizeout \
        --fasta_width 0
```

### Entendendo os Parâmetros:
* `--cluster_unoise`: Aciona o algoritmo estatístico UNOISE3 (desenvolvido por Robert Edgar) para correção de erros sem o corte arbitrário de 97%.
* `--centroids ASVs.fasta`: Arquivo de saída com as sequências exatas purificadas (ASVs).
* `--relabel ASV_`: Renomeia automaticamente os cabeçalhos para `ASV_1`, `ASV_2`, etc.

**Atividade Prática:**
Conte quantas OTUs e quantas ASVs foram geradas usando o comando `grep -c "^>" OTUs_97.fasta` e depois `grep -c "^>" ASVs.fasta`. 
Geralmente, o número de ASVs é maior que o de OTUs. Discuta com os colegas: por que a abordagem de 97% "esconde" a diversidade (gera menos grupos) em comparação com a abordagem exata das ASVs?
