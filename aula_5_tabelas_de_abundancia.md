# Aula 5: Criando as Tabelas de Abundância (OTUs e ASVs)

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
        --id 0.97 \
        --otutabout tabela_abundancia_ASVs.txt
```

**Por que usar 97% nas ASVs?**
Se a ASV é exata (100%), por que o mapeamento usa 97% (`--id 0.97`)? 
*Resposta:* O arquivo `ASVs.fasta` contém a sequência biológica verdadeira e curada. Porém, o arquivo `todas_amostras_concatenadas` ainda está cheio das leituras originais com pequenos erros de sequenciamento da máquina. Se exigirmos 100% no mapeamento, perderemos os dados reais que a máquina errou por uma letrinha. A regra é: **crie as ASVs com alta resolução, mas mapeie os dados originais de volta com 97% de tolerância** para não desperdiçar contagens de abundância.

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

**Atividade Prática para Discussão:**
Compare os números de abundância de uma OTU principal com uma ASV principal. Como as OTUs aglomeram várias sequências parecidas no mesmo grupo (a lente embaçada), a contagem de uma única OTU costuma ser a soma das contagens de várias ASVs menores juntas!