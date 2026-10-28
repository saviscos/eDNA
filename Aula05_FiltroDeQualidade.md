## 5. Filtragem de Qualidade

## Preparando os Dados para o VSEARCH: Inserção de Etiquetas e Concatenação

### Introdução

Temos agora arquivos FASTQ individuais contendo as sequências de alta qualidade fundidas de cada amostra (ex: `T10.fastq`, `T11.fastq`). 

O próximo software que utilizaremos, o **VSEARCH**, é extremamente rápido, mas ele trabalha de forma otimizada quando processa todas as amostras juntas em um único arquivo gigantesco. 

Se apenas juntarmos (concatenarmos) os arquivos agora, perderemos a informação de qual sequência veio de qual amostra. Para evitar isso, precisamos adicionar uma etiqueta invisível (no formato `sample=NOME_DA_AMOSTRA;`) no cabeçalho original de cada leitura de DNA dentro do arquivo antes de misturá-las.

### Adicionando as Etiquetas e Concatenando em um Loop

Para não precisarmos abrir arquivo por arquivo, vamos usar um *loop* no terminal. Este comando usará a ferramenta `sed` (Stream Editor) para localizar o início de cada cabeçalho FASTQ (que começa com o símbolo `@`) e inserir a etiqueta correta da amostra logo após ele.

Certifique-se de estar dentro do diretório `2_merged` onde estão os seus arquivos finais do PEAR.

Execute o comando abaixo:

```bash
# 1. Garante que o arquivo final comece vazio (útil se rodar o comando mais de uma vez)
> todas_amostras_concatenadas.fastq

# 2. Inicia o loop para todos os arquivos .fastq
for f in *.fastq; do
    # Extrai o nome da amostra (ex: T10) removendo o '.fastq'
    s="${f%.fastq}"
    
    # Ignora o arquivo concatenado se ele for capturado acidentalmente pelo loop
    if [ "$s" != "todas_amostras_concatenadas" ]; then
        echo "Adicionando tags na amostra $s..."
        
        # Usa o sed para adicionar '@sample=NOME;' e anexa o resultado no arquivo final (>>)
        sed "s/^@/@sample=${s};/" "$f" >> todas_amostras_concatenadas.fastq
    fi
done

echo "Concatenação concluída com sucesso!"
```

### Verificando a Inserção das Etiquetas

Para confirmar que a "mágica" funcionou, visualize as primeiras linhas do seu novo arquivo concatenado:

```bash
head todas_amostras_concatenadas.fastq
```

Você verá que a primeira linha (o cabeçalho da leitura) agora contém algo como:
`@sample=T10;M00123:456:HV5FFBCXY...`

Pronto! Agora todas as suas milhares de sequências estão reunidas em um único arquivo de forma rastreável. Este arquivo (`todas_amostras_concatenadas.fastq`) é o ponto de partida ideal para a próxima etapa: a **Filtragem de Qualidade**.

### Filtro de qualidade

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


# (Opcional) Executando uma filtragem de qualidade rigorosa e seleção de tamanho

```bash
vsearch --fastx_filter todas_amostras_concatenadas.fastq \
        --fastq_maxee 0.5 \
        --fastq_minlen 225 \
        --fastq_maxlen 275 \
        --fastq_maxns 0 \
        --fastaout seqs_filtradas.fasta \
        --fasta_width 0
```

Entendendo os parâmetros avançados:

*  `--fastx_filter`: Chama a função de filtragem do VSEARCH.

*  `--fastq_maxee 0.5`: Define o limite de Erros Esperados (Expected Errors). Um valor de 0.5 é rigoroso e excelente para metabarcoding de alta precisão (ASVs). Ele rejeita a leitura se a soma estatística de erros ultrapassar 0.5.

*  `--fastq_minlen e` --fastq_maxlen: Seleção de tamanho. Como conhecemos o nosso primer e o gene alvo, sabemos o tamanho esperado do amplicon (ex: em torno de 250 pb). Fragmentos muito curtos ou muito longos são artefatos (dímeros de primer, amplificação inespecífica ou quimeras) e devem ser removidos.

* `--fastq_maxns 0`: Rejeita qualquer sequência que contenha bases não identificadas (representadas pela letra 'N').

* `--fastaout`: Salva as sequências aprovadas em um formato FASTA (.fasta).

* `--fasta_width 0`: Evita quebra de linhas na sequência de DNA dentro do arquivo FASTA, mantendo cada sequência em uma única linha contínua. Isso facilita muito a manipulação posterior com ferramentas de terminal (como grep ou sed).


### Verificando os Resultados

Como o VSEARCH gera um arquivo FASTA (que usa o caractere `>` para iniciar o cabeçalho em vez do `@` do FASTQ), usamos uma expressão regular diferente no `grep` para contar as sequências sobreviventes:

```bash
# Conta o número de sequências aprovadas no arquivo final
grep -c "^>" seqs_filtradas.fasta
```

Compare este número com a contagem de sequências do seu arquivo concatenado original (`todas_amostras_concatenadas.fastq`) para entender exatamente quantos dados foram descartados por baixa qualidade.
