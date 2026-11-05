# Aula Prática 01 — Fundamentos de Linux e exploração de arquivos FASTQ

**Disciplina:** DNA ambiental (eDNA) aplicado à Oceanologia  
**Público-alvo:** Pós-graduação em Oceanologia  
**Duração sugerida:** 2 horas  
**Tema:** Navegação no terminal Linux, organização de dados e leitura de arquivos FASTQ

---

## Objetivos de aprendizagem

Ao final desta aula, você deverá ser capaz de:

1. Explicar a relação entre a coleta de eDNA e a geração de arquivos de sequenciamento.
2. Navegar por diretórios e manipular arquivos básicos no terminal Linux.
3. Identificar os quatro componentes de um registro FASTQ.
4. Diferenciar sequenciamento *single-end* e *paired-end* e reconhecer arquivos R1 e R2.
5. Inspecionar arquivos FASTQ, contar leituras e explorar comprimentos de sequências.
6. Interpretar escores de qualidade Phred e reconhecer a importância da integridade dos dados.

## Pré-requisitos

- Um computador com Linux, WSL (Windows Subsystem for Linux) ou acesso a um servidor Linux.
- Conexão à internet para obter os dados.
- Utilitários `wget`, `unzip`, `awk`, `grep`, `sort`, `uniq`, `wc`, `head` e `sed` disponíveis no terminal.

> **Nota para usuários de Windows:** os comandos desta aula devem ser executados no terminal da distribuição Linux instalada pelo WSL (por exemplo, Ubuntu), e não no Prompt de Comando tradicional do Windows.

---

## 1. Do ambiente marinho ao arquivo FASTQ

O **DNA ambiental (eDNA)** é o material genético recuperado de amostras ambientais, como água do mar, sedimentos e outros substratos. Sua análise permite investigar organismos sem que seja necessário capturá-los ou observá-los diretamente. Em Oceanologia, abordagens baseadas em eDNA podem apoiar estudos de biodiversidade, detecção de espécies invasoras, monitoramento de espécies raras e caracterização de comunidades microbianas.

Uma aplicação frequente é o **metabarcoding**, no qual regiões específicas do DNA (marcadores moleculares) são amplificadas e sequenciadas para investigar a composição de comunidades biológicas. Exemplos de marcadores incluem **16S rRNA** (frequentemente utilizado para bactérias e arqueias), **18S rRNA** (eucariotos) e regiões do **12S rRNA** (frequentemente utilizadas para vertebrados, incluindo peixes).

**Fluxo simplificado:**

```text
Coleta de água/sedimento
        ↓
Extração do DNA ambiental
        ↓
Amplificação de marcadores (quando aplicável)
        ↓
Preparação da biblioteca e sequenciamento
        ↓
Arquivos FASTQ (leituras e qualidades)
        ↓
Controle de qualidade e análise bioinformática
        ↓
Interpretação ecológica
```

**Pergunta inicial:** como um conjunto de fragmentos de DNA presentes em uma amostra de água pode ser transformado em informação sobre a biodiversidade marinha?

---

## 2. Primeiros passos no terminal Linux

O **terminal** permite interagir com o sistema operacional por meio de comandos. Em bioinformática, ele é especialmente útil para organizar dados, executar programas e automatizar análises reproduzíveis.

### 2.1. Onde estou e o que existe aqui?

```bash
pwd        # Exibe o caminho completo do diretório atual
ls         # Lista arquivos e diretórios
ls -l      # Exibe detalhes, como permissões e tamanho em bytes
ls -lh     # Exibe tamanhos em unidades legíveis (K, M, G)
ls -a      # Inclui arquivos ocultos
```

### 2.2. Navegação entre diretórios

```bash
cd nome_da_pasta  # Entra em uma pasta
cd ..             # Volta ao diretório superior
cd ~              # Vai para a sua pasta pessoal (home)
cd -              # Retorna ao diretório anterior
```

> **Dica:** use a tecla `Tab` para completar caminhos e nomes de arquivos. Caminhos que começam com `/` são **absolutos**; caminhos escritos a partir da pasta atual são **relativos**.

### 2.3. Criar, copiar, mover e consultar arquivos

```bash
mkdir resultados             # Cria um diretório
mkdir -p curso_eDNA/dados    # Cria diretórios intermediários, se necessário
cp arquivo.txt copia.txt    # Copia um arquivo
cp -r pasta pasta_copia     # Copia um diretório e seu conteúdo
mv arquivo.txt resultados/  # Move um arquivo
less arquivo.txt            # Abre um arquivo para leitura (pressione q para sair)
clear                       # Limpa a tela
man ls                      # Mostra o manual do comando ls
```

**Atenção:** o comando `rm` apaga arquivos sem enviá-los para uma lixeira. Evite executá-lo sem conferir o caminho e o nome do arquivo.

**Atividade rápida:** crie uma pasta chamada `teste_linux`, entre nela, confirme sua localização com `pwd` e retorne à pasta anterior.

---

## 3. O que é um arquivo FASTQ?

O **FASTQ** é um formato amplamente utilizado para armazenar **leituras de sequenciamento** e seus **escores de qualidade por base**. Diferentemente do FASTA, que guarda principalmente identificadores e sequências, o FASTQ reúne, em um registro, a sequência e uma representação de sua qualidade.

Na representação convencional utilizada nesta aula, **cada leitura ocupa quatro linhas**:

```text
@read_001
ACGTACGT
+
IIII!!!!
```

| Linha | Componente | Significado |
|---|---|---|
| 1 | Identificador (`@`) | Identifica a leitura; campos adicionais dependem da plataforma e do processamento. |
| 2 | Sequência | Bases nucleotídicas (por exemplo, A, C, G, T e, quando necessário, N). |
| 3 | Separador (`+`) | Marca o início da informação de qualidade; pode repetir o identificador. |
| 4 | Qualidade | Um caractere por base, codificando seu escore de qualidade. |

A sequência e a linha de qualidades devem ter **o mesmo comprimento**.

### 3.1. O que significa qualidade Phred?

O escore **Phred** expressa uma estimativa da probabilidade de erro na identificação de uma base:

\[
Q = -10\log_{10}(P_{\mathrm{erro}})
\]

| Escore | Probabilidade estimada de erro por base | Probabilidade estimada de acerto |
|---|---:|---:|
| Q10 | 10% | 90% |
| Q20 | 1% | 99% |
| Q30 | 0,1% | 99,9% |
| Q40 | 0,01% | 99,99% |

No esquema comum **Phred+33**, o caractere `I` representa Q40, enquanto `!` representa Q0. No exemplo acima, as quatro primeiras bases têm qualidade estimada maior que as quatro últimas.

> Esses valores são **estimativas**, não garantias de precisão. Em aulas posteriores, veremos como as informações de qualidade orientam o processamento bioinformático.

### 3.2. *Single-end* e *paired-end*

- **Single-end:** uma leitura é obtida a partir de uma extremidade do fragmento de DNA.
- **Paired-end:** duas leituras são obtidas a partir de extremidades opostas do **mesmo fragmento**, geralmente armazenadas em arquivos correspondentes **R1** e **R2**.

Em bibliotecas *paired-end*, as leituras podem apresentar uma região de **sobreposição**. Quando ela é suficiente e a qualidade permite, programas de bioinformática podem fundir R1 e R2 em uma sequência única. Entretanto, **nem todo par de leituras pode ser fundido**: isso depende do tamanho do fragmento, do comprimento útil das leituras e da sobreposição após os cortes necessários.

O metabarcoding de eDNA utiliza com frequência dados *paired-end* de leituras curtas, mas análises de eDNA também podem empregar *single-end* e tecnologias de leituras longas.

### 3.3. Plataformas de sequenciamento

| Categoria | Exemplos | Características relevantes |
|---|---|---|
| Leituras curtas | Illumina; Ion Torrent | Alto rendimento; em muitos protocolos, leituras de algumas dezenas a centenas de bases. |
| Leituras longas | Oxford Nanopore; PacBio | Possibilitam a leitura de fragmentos longos; perfil de erros e desempenho dependem da química e do método utilizado. |

**Discussão:** por que a escolha entre leituras curtas e longas deve levar em conta o marcador molecular e a pergunta ecológica?

---

## 4. Prática: obtenção e organização dos dados

Nesta aula, utilizaremos um conjunto de dados disponibilizado no repositório didático da disciplina:

**Repositório:** https://github.com/saviscos/eDNA

### 4.1. Criar um diretório de trabalho

```bash
mkdir -p ~/curso_eDNA
cd ~/curso_eDNA
pwd
```

Manter uma pasta de trabalho exclusiva ajuda a preservar os arquivos originais e facilita a reprodução das análises.

### 4.2. Baixar e descompactar os arquivos

```bash
# Baixa o arquivo compactado; -c permite retomar downloads interrompidos
wget -c https://github.com/saviscos/eDNA/raw/main/0_rawsequences.zip

# Testa a integridade do arquivo ZIP
unzip -t 0_rawsequences.zip

# Extrai os arquivos
unzip 0_rawsequences.zip

# Lista o conteúdo
ls -lh
```

> **Importante:** apague o `.zip` somente depois de confirmar que a extração funcionou. Caso o download via GitHub falhe, informe ao docente antes de prosseguir.

Entre no diretório extraído:

```bash
cd 0_rawsequences/
ls -lh
```

**Observe:** quantos arquivos existem? Quais são seus nomes? Quais parecem formar pares R1/R2? Quais são seus tamanhos?

> Os comandos seguintes consideram arquivos descompactados com extensão `.fastq`. Caso os dados estejam em `.fastq.gz`, não use `wc`, `head` e `awk` diretamente sobre o conteúdo comprimido; será necessário descompactá-lo ou utilizar comandos como `zcat`.

---

## 5. Prática: explorando os arquivos FASTQ

### 5.1. Contar linhas

```bash
wc -l *.fastq
```

O caractere `*` é um **curinga**: `*.fastq` seleciona todos os arquivos do diretório atual cujo nome termina em `.fastq`.

Como cada leitura ocupa quatro linhas no formato utilizado, o número de leituras pode ser calculado dividindo a contagem de linhas por quatro.

### 5.2. Inspecionar as primeiras leituras

Substitua `arquivo.fastq` pelo nome real de um dos arquivos listados por `ls`:

```bash
head -n 12 arquivo.fastq   # Exibe as primeiras 3 leituras completas
```

**Identifique:** cabeçalho, sequência, separador e qualidade. Compare visualmente o comprimento da linha de bases com o da linha de qualidade.

Para exibir um trecho específico:

```bash
sed -n '5,8p' arquivo.fastq    # Mostra o segundo registro FASTQ
```

Você também pode explorar arquivos maiores com:

```bash
less arquivo.fastq
```

Pressione `q` para sair.

### 5.3. Contar leituras de forma independente do cabeçalho

```bash
awk 'END {print NR/4}' arquivo.fastq
```

Aqui, `NR` representa o número total de linhas lidas por `awk`. Essa contagem pressupõe registros convencionais de quatro linhas e um arquivo íntegro.

Para obter a contagem de **todos** os arquivos FASTQ:

```bash
for arquivo in *.fastq; do
    awk -v nome="$arquivo" 'END {printf "%s\t%d leituras\n", nome, NR/4}' "$arquivo"
done
```

> **Por que não usar apenas `grep -c '^@'`?** Embora o identificador FASTQ comece com `@`, a linha de qualidade também pode conter esse caractere. Além disso, contar cabeçalhos que começam por um prefixo específico (como `@D00`) só funciona quando esse prefixo realmente existe em todos os registros da amostra.

### 5.4. Investigar o comprimento das leituras

Em um FASTQ convencional, as linhas 2, 6, 10, 14 etc. contêm sequências. O comando seguinte mostra o comprimento das dez primeiras:

```bash
awk 'NR%4==2 {print length($0)}' arquivo.fastq | head -n 10
```

O operador `|` (**pipe**) envia a saída de um comando para a entrada do próximo.

Para contar quantas leituras apresentam cada comprimento:

```bash
awk 'NR%4==2 {print length($0)}' arquivo.fastq | sort -n | uniq -c
```

**Interpretação:** as leituras apresentam comprimentos semelhantes? O que pode explicar diferenças?

### 5.5. Reconhecer pares R1 e R2

Em dados *paired-end*, os arquivos R1 e R2 correspondentes contêm leituras originadas das duas extremidades dos mesmos fragmentos de DNA. Os identificadores devem permitir associar os pares, mas os campos que indicam R1/R2 variam de acordo com a plataforma e o formato dos cabeçalhos.

Examine os primeiros cabeçalhos dos dois arquivos correspondentes:

```bash
head -n 1 amostra_R1.fastq
head -n 1 amostra_R2.fastq
```

**Atenção:** `amostra_R1.fastq` e `amostra_R2.fastq` são nomes **ilustrativos**. Substitua-os pelos nomes reais apresentados por `ls`.

Atenção também ao conceito de **sincronia**: arquivos R1 e R2 podem ter o mesmo número de leituras e ainda assim estar fora de ordem ou conter identificadores incompatíveis. A correspondência entre os pares precisa ser preservada nas próximas etapas.

### 5.6. Verificar condições básicas de integridade

Um teste preliminar é verificar se o total de linhas é múltiplo de quatro:

```bash
awk 'END {if (NR%4==0) print "Número de linhas compatível com FASTQ de 4 linhas"; else print "ATENÇÃO: número de linhas incompatível"}' arquivo.fastq
```

Essa verificação **não substitui** uma validação completa: ela não assegura que todas as linhas estejam corretamente organizadas nem que sequência e qualidade tenham comprimentos idênticos.

---

## 6. Exercícios de fixação

Realize os exercícios a seguir utilizando os arquivos fornecidos. Registre os comandos utilizados e suas respostas.

1. **Reconhecimento dos dados:** quantos arquivos `.fastq` existem? Qual é o maior e qual é o menor em tamanho de arquivo?
2. **Quantidade de leituras:** quantas leituras existem em cada arquivo? Apresente uma tabela com nome do arquivo, número de linhas e número de leituras.
3. **Pareamento:** os arquivos R1 e R2 de cada biblioteca apresentam a mesma quantidade de leituras? Isso é suficiente para garantir que estão sincronizados? Justifique.
4. **Comprimento:** todas as leituras de um mesmo FASTQ possuem o mesmo comprimento? Mostre o comando utilizado para investigar essa questão.
5. **Estrutura:** identifique as quatro linhas de um registro FASTQ e explique a função de cada uma.
6. **Qualidade:** qual é a probabilidade estimada de erro de uma base com Q20? E de uma base com Q30?
7. **Raciocínio ecológico:** por que a qualidade dos dados de sequenciamento pode afetar uma investigação sobre a presença de espécies raras em amostras de água do mar?

### Desafio opcional — biblioteca versus amostra

Se um experimento contém vários arquivos R1/R2, isso significa necessariamente que existe uma amostra biológica para cada arquivo? Explique sua hipótese. Retomaremos essa discussão ao estudar **índices, bibliotecas e demultiplexação**.

---

## 7. Síntese e conexão com a próxima aula

Nesta aula, aprendemos a organizar dados em um ambiente Linux, reconhecer registros FASTQ, contar leituras e examinar comprimentos e escores de qualidade. Esses conhecimentos serão utilizados no restante da disciplina para interpretar os resultados de programas bioinformáticos.

**Na próxima etapa**, discutiremos como organizar e separar leituras de diferentes amostras e como o desenho experimental influencia o processamento de dados de metabarcoding.

### Checklist de aprendizagem

- [ ] Consigo navegar entre diretórios usando `pwd`, `ls` e `cd`.
- [ ] Sei criar uma pasta e identificar os arquivos armazenados nela.
- [ ] Consigo explicar as quatro linhas de um FASTQ.
- [ ] Entendo a diferença entre *single-end* e *paired-end*.
- [ ] Sei calcular quantas leituras existem em um arquivo FASTQ convencional.
- [ ] Consigo examinar o comprimento de leituras usando `awk`.
- [ ] Sei interpretar Q20 e Q30 como estimativas de probabilidade de erro.
- [ ] Entendo por que a integridade e o pareamento dos arquivos são importantes.

---

## Leituras complementares

- **FASTQ format — Wikipédia:** https://en.wikipedia.org/wiki/FASTQ_format
- **Repositório didático da disciplina:** https://github.com/saviscos/eDNA
- **DADA2 — documentação e tutoriais:** https://benjjneb.github.io/dada2/tutorial.html

> **Para o docente:** tempo sugerido: contextualização (10 min); terminal Linux (25 min); formato FASTQ/Phred (25 min); obtenção e exploração dos arquivos (40 min); exercícios e discussão (20 min). Adapte conforme o nível de familiaridade da turma com Linux.
