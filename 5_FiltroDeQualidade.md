## 5. Filtragem de Qualidade

### Introdução

A filtragem de qualidade visa remover sequências que contêm erros de sequenciamento, determinados pelo método de pontuação do próprio sequenciador. Essas pontuações são armazenadas no arquivo FASTQ usando os **escores Phred** (probabilidade de a base estar incorreta).

Como estamos trabalhando com pares de amplicons que já foram fundidos (*merged*) na etapa anterior, sabemos que se as nossas leituras tiverem partes de baixa qualidade, elas estarão no **meio** da sequência (na região de sobreposição). Portanto, fazer uma "poda" (*trimming*) das extremidades não é uma estratégia viável agora. A abordagem mais conservadora e recomendada para este momento é excluir a leitura inteira com base na sua qualidade global.

### O Método Essencial: Filtragem por Taxa de Erro Esperado (VSEARCH)

Em vez de definir limites arbitrários por base, a abordagem mais moderna avalia todas as probabilidades de erro de uma leitura e calcula a probabilidade geral daquela sequência conter falhas. 

O comando abaixo exclui qualquer leitura que tenha **1 ou mais erros esperados**. A lógica aqui é simples e reflete a realidade do sequenciamento de eDNA: não importa o tamanho do nosso fragmento, nós não queremos nenhum erro nele.

Para executar este passo, certifique-se de estar no mesmo diretório do arquivo FASTQ concatenado que geramos na etapa anterior.

```bash
# Filtragem rigorosa: retém apenas sequências com menos de 1 erro esperado
vsearch --fastx_filter todas_amostras_concatenadas.fastq --fastq_maxee 1 --fastaout seqs_filtradas.fasta
```

**O que este comando faz:**
*   `--fastx_filter`: Chama a função de filtragem do VSEARCH.
*   `--fastq_maxee 1`: Define o limite máximo de Erros Esperados (*Maximum Expected Errors*) como 1. Se a soma das probabilidades de erro da sequência inteira for maior que 1, ela é descartada.
*   `--fastaout`: Salva as sequências aprovadas em um novo arquivo. **Nota:** O arquivo de saída agora é um **FASTA** (`.fasta`), pois não precisamos mais dos escores Phred (que já cumpriram seu papel na filtragem).

> **Ajustando a rigidez:** Se em análises futuras você notar que ainda há muito "ruído" nos dados, pode aumentar o rigor usando `--fastq_maxee 0.5`. Se a filtragem estiver removendo quase todos os seus dados (raro, mas possível com corridas ruins), você pode relaxar para `--fastq_maxee 1.5`, mas trate os resultados biológicos com muita cautela.

### Verificando os Resultados

Como o VSEARCH gera um arquivo FASTA (que usa o caractere `>` para iniciar o cabeçalho em vez do `@` do FASTQ), usamos uma expressão regular diferente no `grep` para contar as sequências sobreviventes:

```bash
# Conta o número de sequências aprovadas no arquivo final
grep -c "^>" seqs_filtradas.fasta
```

Compare este número com a contagem de sequências do seu arquivo concatenado original (`todas_amostras_concatenadas.fastq`) para entender exatamente quantos dados foram descartados por baixa qualidade.