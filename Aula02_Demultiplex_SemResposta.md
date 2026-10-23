## 1. Demultiplexação

### Introdução

Existem muitas maneiras pelas quais bibliotecas de metabarcoding podem ser sequenciadas. Se você estiver usando os dados de exemplo, trabalharemos aqui com um pipeline de preparação de biblioteca que envolveu a indexação de amplicons durante a PCR inicial, de modo que cada amostra tivesse um índice diferente de 6 bases dentro de uma biblioteca. Uma vez sequenciados, precisamos usar essas sequências de índice para separar as diferentes amostras.

Esse processo é chamado de **demultiplexação**. Neste caso, os amplicons foram sequenciados usando sequenciamento *paired-end* (extremidades pareadas), o que significa que as duas extremidades de cada fragmento foram lidas, trabalhando de fora para dentro. Temos oito arquivos de sequência recebidos de nossa central de sequenciamento, um para cada direção de leitura (*forward* e *reverse*) para cada uma das quatro bibliotecas.

### Eu não tenho dados multiplexados!

Se você estiver acompanhando com seus próprios dados, esta etapa se aplica apenas se os arquivos de sequência que você possui contiverem dados de múltiplas amostras misturadas. Geralmente, isso acontece se você multiplexa suas amostras juntas durante a primeira rodada de PCR, bem como a multiplexação que acontece como parte da preparação da biblioteca. O próprio sequenciador geralmente desfaz a multiplexação da preparação da biblioteca, então se você não multiplexou durante a primeira rodada de PCR, provavelmente pode pular esta etapa e ir direto para a remoção de primers.

### Dados e Software

Este tutorial trabalha com dados brutos de sequência no formato FASTQ que contêm índices no início das *reads*. Os dados de exemplo para isso podem ser encontrados no diretório `0_rawsequences`. Você deve copiar este diretório para o seu diretório de trabalho conforme fizemos na aula anterior:

```
cp -r path/to/exampledata/sectionA/0_rawsequences/ ./

```

Este tutorial usa o software **cutadapt**.

### Primers e Índices

Cada uma dessas bibliotecas contém, na verdade, três amostras diferentes de metabarcoding. Cada amostra dentro de cada biblioteca foi amplificada com um índice diferente de 6 bases. Você pode ver esses índices no arquivo `indices.txt` dentro do diretório `0_rawsequences/`. Primeiro, entre no diretório e depois visualize o arquivo:

```
cd 0_rawsequences/
cat indices.txt

```

O comando `cat` simplesmente exibe todo o conteúdo dos arquivos fornecidos a ele, um após o outro.

Podemos usar o comando `grep` para verificar a presença de índices em um arquivo. Selecione qualquer um dos arquivos FASTQ iniciais e execute o seguinte comando para procurar pelo índice `AAGAGG`:

```
head -n 24 arquivo.fastq | grep -E "AAGAGG|$"

```

Você deve ver a sequência do índice destacada no início de uma sequência. Você também pode vê-la em outras partes das sequências - é uma sequência de bases tão curta que é totalmente possível que reads reais contenham esse padrão. Se você não vir nenhum texto em negrito/destacado, tente um arquivo FASTQ diferente.

Quando fizemos a PCR, o índice fazia parte dos primers usados, de modo que a reação adicionou essa sequência aos nossos amplicons ao copiá-los. As sequências de ligação dos primers usadas para esta reação de metabarcoding foram (não rode isso!):

* **Forward (R1):** `CCNGAYATRGCNTTYCCNCG`

* **Reverse (R2):** `TANACYTCNGGRTGNCCRAARAAYCA`

Note a presença de bases não-ATCG (como N, Y, R) - estas são *ambiguidades* adicionadas aos primers para permitir que sejam menos específicos e se liguem a uma maior variedade de táxons. Os índices não são mostrados aqui: eles iriam no início da sequência de ligação do primer.

### Exercício 1

* Use o comando `grep` para procurar a sequência do primer em um arquivo. Note que as ambiguidades (qualquer base além de ATCG) devem ser substituídas por um `.` (ponto final), que é um caractere especial em Expressões Regulares que significa "qualquer caractere".

* Observe algumas bibliotecas diferentes, tanto forward quanto reverse, buscando tanto o índice quanto o primer.

```
head -n 24 input_R1.fastq | grep -E "CC.GA.AT.GC.TT.CC.CG|$"
head -n 24 input_R2.fastq | grep -E "TA.AC.TC.GG.TG.CC.AA.AA.CA|$"

```

Você provavelmente verá que há ocasiões em que nenhum índice ou primer é destacado em uma sequência. Isso significa que houve um erro de sequenciamento. Olhe de perto e você verá que uma base está faltando ou inserida, ou simplesmente errada.

### A Demultiplexação na Prática

Queremos dividir cada uma dessas bibliotecas pelo índice em um par de arquivos separado para cada uma das 12 amostras, e remover as sequências de índice no processo. Faremos isso usando a ferramenta `cutadapt`. Esta ferramenta versátil permite separar arquivos por índice e remover esses índices e primers. Ela tolera algum erro na sequência do índice e mantém os pares de *reads* em sincronia.

Instalando o cutadpt:

```
sudo apt install cutadapt

```

Como sempre ao usar uma nova ferramenta, primeiro dê uma olhada na ajuda executando:

```
cutadapt --help

```

É bastante longo, mas leia pelo menos a primeira seção para se familiarizar. É útil pensar exatamente sobre o que são nossos dados e o que queremos fazer:

* Temos dados pareados (*paired-end*).

* Nossos índices estão no início das *reads*.

* Temos vários índices (o cutadapt os chama de 'adapters').

* Queremos gerar um arquivo de saída diferente para cada índice.

O cutadapt tem configurações para todas essas situações.

* Ele permite ler dois arquivos como entrada e garantirá que os pares de reads nestes arquivos sejam mantidos em sincronia.

* Índices no início das reads são especificados usando `-g` (ou `-G` para o segundo arquivo de reads), e podemos especificá-los várias vezes, nomeando diferentes adaptadores.

* Também especificamos que os adaptadores estão bem no início, sem lacunas, usando um símbolo `^`.

* Podemos especificar que queremos arquivos de saída dependendo da combinação de adaptadores encontrados usando as opções `-o` e `-p` para o primeiro e segundo arquivos, respectivamente.

Para evitar bagunça, certifique-se de estar no diretório pai (*parent directory*) e, em seguida, crie um novo diretório:

```
cd ../        # Muda o diretório de trabalho para o nível acima (pai)
mkdir 1_demux # Cria um novo diretório chamado 1_demux

```

Consultando o arquivo `indices.txt`, agora podemos construir um comando que demultiplexa nossa Lib1.

```
cutadapt -g T4=^AAGAGG -g T9=^AATCGC -g T11=^AGCTAC \
-G T4=^AAGAGG -G T9=^AATCGC -G T11=^AGCTAC \
-o 1_demux/{name1}-{name2}_R1.fastq -p 1_demux/{name1}-{name2}_R2.fastq \
0_rawsequences/Lib1_R1.fastq 0_rawsequences/Lib1_R2.fastq

```

*(Nota: A barra invertida `\` é usada para quebrar o comando longo em várias linhas no terminal para facilitar a leitura. Você pode digitar tudo em uma linha única omitindo as barras).*


### Nota sobre erros de sequenciamento

Alguns pesquisadores usam todas essas combinações diferentes para aplicar de forma eficiente poucos índices visando identificar muitas amostras diferentes. No entanto, isso torna muito mais difícil detectar erros. Nós não temos sequências válidas que usem índices *forward* e *reverse* diferentes na mesma amostra, mas o demultiplexador encontrou várias: **são erros no sequenciamento**. O sequenciador associou erroneamente algumas *reads* como sendo do mesmo fragmento quando não são. Em alguns casos, nenhum índice pôde ser encontrado em uma ou ambas as *reads* pareadas. Elas são marcadas como *unknown* (desconhecidas). Felizmente, todos esses erros representam uma minoria, e a maioria das *reads* foi alocada corretamente.

Se você somar todas as contagens de *reads*, notará que estão faltando algumas do arquivo original: estas não tinham par correspondente e foram completamente descartadas.

### Exercício 3 e Solução Final

* Construa mais três comandos do `cutadapt` para demultiplexar as outras três bibliotecas, colocando os resultados no mesmo diretório de saída (`1_demux`).




Após rodar tudo, liste o conteúdo do diretório novamente e verifique os números de reads por arquivo para garantir que as 4 bibliotecas foram processadas com sucesso!

### Limpando os arquivos

Você agora deve ter muitos arquivos no seu diretório de saída. É uma boa prática manter um registro de como a demultiplexação ocorreu: você pode tentar escrever um comando que salve a saída do seu comando `grep` anterior em um arquivo para manter um histórico (veja a caixa de solução abaixo se não tiver certeza de como fazer isso).

**Solução:**
```bash
grep -c "^@D00" 1_demux/* > output.txt
```

Vamos nos livrar dos arquivos que não precisamos (aquelas misturas de índices que representam erros). Você dobrou a quantidade de armazenamento que está usando - aqui os arquivos não são muito grandes, mas se você estivesse fazendo isso com um conjunto de dados real, os diretórios se encheriam rapidamente. 

Navegue até a pasta `1_demux` (`cd 1_demux/`), copie **muito cuidadosamente** o comando a seguir e execute-o. Ele percorre os arquivos, extraindo o primeiro e o segundo nome da amostra (os nomes dos índices) e, em seguida, exclui o arquivo se eles não corresponderem (ex: `T4-T9`). Você não precisa digitar os comentários com `#` ou adicionar os espaços extras na hora de rodar no terminal - isso é apenas para deixar o script mais claro para o seu aprendizado.

```bash
for f in *;                    # loop através dos arquivos
do
        s1=${f%-*};
        s2=${f%_*};            # extrai os nomes das amostras
        s2=${s2#*-};
        if [ $s1 != $s2 ];      # checa se os índices são diferentes
        then
                rm $f;          # deleta se forem diferentes (erro)
        else
                mv $f ${f#*-}; # mantém se forem iguais, removendo a primeira parte do nome
        fi;                     # fim da cláusula if
done                            # fim do loop



### Dados e Software
Este tutorial trabalha com dados de sequenciamento no formato FASTQ que contêm índices no início das *reads* (leituras). Os dados de exemplo para isso podem ser encontrados no diretório `0_rawsequences` dentro do arquivo da seção A. 

Se você ainda não o fez, copie este diretório para o seu diretório de trabalho atual com o seguinte comando (ajuste o caminho conforme necessário):
```bash
cp -r path/to/exampledata/sectionA/0_rawsequences/ ./
```
*(Nota: O `./` no final do comando significa "copie para o diretório atual em que estou".)*

### Explorando arquivos FASTQ

# Módulo 1: Preparação do Ambiente e Qualidade dos Dados Brutos
Este módulo foca na configuração do ambiente de trabalho e na avaliação da qualidade das leituras (reads) de sequenciamento.

## 1.1 Configuração Inicial
Objetivo: Preparar o ambiente Linux para as análises.

Ferramentas: apt, wget, git

Comandos:


```
sudo apt update
sudo apt upgrade -y
sudo apt install -y build-essential wget git unzip screen default-jre
sudo pip install quast

```

Descrição: Atualiza o sistema, instala compiladores C/C++, `wget` para downloads, `git` para clonar repositórios, `unzip` para descompactar arquivos, `screen` para sessões persistentes no terminal e `Java` (JRE) para algumas ferramentas.

## 1.2 Download de Dados de Sequenciamento
Objetivo: Obter reads brutos de sequenciamento (FASTQ)


```



### 1.3 Avaliação da Qualidade dos Reads
Objetivo: Avaliar a qualidade dos reads brutos para identificar problemas como adaptadores de sequenciamento, bases de baixa qualidade, viés de GC e contaminação. Esta etapa é crítica para garantir que apenas dados de alta qualidade sejam usados na montagem.

Ferramentas: FastQC - https://www.bioinformatics.babraham.ac.uk/projects/fastqc/

Comandos:

```
sudo apt install -y fastqc
mkdir qc_resultados
fastqc 0_rawsequences/*.fastq -o qc_resultados/
```

Forma de Execução: Os relatórios HTML gerados em qc_resultados/ devem ser abertos em um navegador web para inspeção visual. 

### Descrição Detalhada:

- sudo apt install -y fastqc: Instala a ferramenta FastQC.

- mkdir qc_resultados: Cria um diretório dedicado para armazenar os relatórios de qualidade dos reads brutos.

- fastqc raw_data/*.fastq -o qc_raw/: Executa o FastQC em todos os arquivos .fastq dentro do diretório raw_data e direciona a saída (relatórios HTML e arquivos ZIP contendo os dados brutos do relatório) para o diretório qc_raw/.

### O que é Qualidade (Escala Phred): A qualidade de uma base em um read de sequenciamento é geralmente expressa usando a escala Phred. Um valor Phred (Q) é uma medida logarítmica da probabilidade de erro de uma base.

- `Q=−10`
- log_10(P), onde P é a probabilidade de a base estar incorreta.

- Um Q20 significa que a probabilidade de erro é de 1 em 100 (P=0.01), ou 99% de acurácia.

- Um Q30 significa que a probabilidade de erro é de 1 em 1000 (P=0.001), ou 99.9% de acurácia.

- Em geral, reads com qualidade Phred abaixo de Q20 (especialmente nas extremidades) são considerados de baixa qualidade e devem ser removidos ou trimados.

| Qualidade Phred (Q) | Probabilidade de Erro (P) | Precisão da Base (%) |
|---------------------|---------------------------|-----------------------|
| 10                  | 1 em 10                   | 90%                  |
| 20                  | 1 em 100                  | 99%                  |
| 30                  | 1 em 1.000                | 99,9%                |
| 40                  | 1 em 10.000               | 99,99%               |
| 50                  | 1 em 100.000              | 99,999%              |
| 60                  | 1 em 1.000.000            | 99,9999%             |


## Avaliação de Qualidade - FASTQC

Abaixo está uma imagem de exemplo da avaliação de qualidade (Per Base Sequence Quality) gerada pelo FastQC:

<img src="imgs/AVQ.png" alt="Avaliação de Qualidade" width="400" height="300" />


### Interpretação dos Gráficos do FastQC:

Per Base Sequence Quality:

- Mostra a qualidade média de cada posição de base ao longo dos reads.

- Padrão ideal: A caixa de bigodes deve estar acima de Q30 (linha verde) em todas as posições, com pouca variação.

- Problemas: Quedas na qualidade nas extremidades 3' dos reads são comuns e esperadas. Qualidade baixa no início do read ou flutuações significativas indicam problemas no sequenciamento.

Per Sequence Quality Scores:

- Mostra a distribuição da qualidade média para cada read.

- Padrão ideal: Um pico alto em valores de alta qualidade (Q30+).

- Problemas: Um pico em valores de baixa qualidade indica muitos reads problemáticos.

Per Base Sequence Content:

- Mostra a porcentagem de cada base (A, T, C, G) em cada posição do read.

- Padrão ideal: Linhas planas e paralelas, indicando uma distribuição aleatória de bases.

- Problemas: Desvios significativos, especialmente no início dos reads, podem indicar contaminação por adaptadores ou problemas de sequenciamento.

Per Sequence GC Content:

- Mostra a distribuição do conteúdo de GC para todos os reads.

- Padrão ideal: Uma curva gaussiana (normal) centrada no conteúdo de GC esperado para o organismo.

- Problemas: Múltiplos picos podem indicar contaminação por outras espécies ou DNA de hospedeiro.

Sequence Length Distribution:

- Mostra a distribuição dos comprimentos dos reads.

- Padrão ideal: Um único pico no comprimento esperado dos reads (e.g., 150 bp para Illumina).

- Problemas: Múltiplos picos ou uma distribuição ampla podem indicar degradação da amostra ou problemas na biblioteca.

Adapter Content:

- Detecta a presença de sequências de adaptadores de sequenciamento.

- Padrão ideal: Linhas planas na parte inferior do gráfico (sem adaptadores detectados).

- Problemas: Curvas ascendentes indicam que adaptadores estão presentes e precisam ser removidos.



