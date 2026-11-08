# Aula Prática 01 — Fundamentos de Linux e exploração de arquivos FASTQ

---

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

<img width="4320" height="3214" alt="image" src="https://github.com/user-attachments/assets/ef9a2bdb-3e10-480e-9289-afdceb3dcbe9" />

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


![FastQ](https://static.wixstatic.com/media/3165a3_219cd2be42a64a589a678ce3295a543e~mv2.jpg/v1/fill/w_438,h_246,al_c,q_80,usm_0.66_1.00_0.01,enc_avif,quality_auto/3165a3_219cd2be42a64a589a678ce3295a543e~mv2.jpg)

### 3.1. O que significa qualidade Phred?

O escore **Phred** expressa uma estimativa da probabilidade de erro na identificação de uma base:


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

![Sequenciamento](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTgXIq1F91DhsIDuKlvsGQ8P7PZIooRqtO2uK9Whj2rdj3_9Po868SEi5c&s=10)

**Discussão:** por que a escolha entre leituras curtas e longas deve levar em conta o marcador molecular e a pergunta ecológica?

---
## 4. Prática: obtenção e organização dos dados

### 4.1. Criar o diretório de trabalho e baixar os dados

```bash
# Cria o diretório principal e entra nele
mkdir ~/eDNA
cd ~/eDNA

# Baixa o arquivo compactado (.rar) do repositório
wget -c https://github.com/saviscos/eDNA/raw/main/reads_fastq.rar
```

### 4.2. Descompactar e acessar os arquivos

```bash
# Instalar o descompactador (caso a máquina não tenha)
sudo apt update
sudo apt install unrar -y

# Extrair os arquivos
unrar x reads_fastq.rar

# Entrar no diretório extraído e listar
cd reads_fastq/
ls -lh
```

> **Atividade Prática:** Olhando para o resultado do `ls -lh`, qual amostra gerou o arquivo R1 mais pesado? Qual o tamanho dele?

## 5. Prática: explorando os arquivos FASTQ passo a passo

Usaremos as amostras `T10`, `T12` e `T14` para as explorações a seguir.

### 5.1. Olhando o começo e o fim do arquivo

O comando `head` lê o topo do arquivo. Como sabemos que cada leitura tem 4 linhas, ler 12 linhas significa visualizar as 3 primeiras leituras completas.

```bash
head -n 12 T10_R1.fastq
```

> **Atividade Prática:** Existe um comando chamado `tail` que faz o oposto do `head`. Use o `tail` para visualizar a **última** leitura (as últimas 4 linhas) do arquivo `T14_R2.fastq`. O que você observou na qualidade dessa última leitura?

### 5.2. Contando as leituras de uma amostra

Para saber o número de linhas de um arquivo, usamos o `wc -l`.

```bash
wc -l T10_R1.fastq
```
Como sabemos que a estrutura do FASTQ ocupa 4 linhas por leitura, basta pegar o número retornado pelo comando acima e dividir por 4 em uma calculadora para descobrir quantas sequências de DNA temos nessa amostra.

> **Atividade Prática:** Use o `wc -l` no arquivo `T12_R1.fastq`. Quantas sequências biológicas totais existem dentro deste arquivo?

### 5.3. Buscando padrões com o `grep`

E se quisermos procurar algo específico dentro desses arquivos gigantes? O `grep` é um buscador. Vamos procurar quantas vezes a base `N` (quando o sequenciador falha em reconhecer o nucleotídeo) aparece isolada.

```bash
# Busca a letra A maiúscula no arquivo
grep "A" T10_R1.fastq
```

Como a tela vai ficar cheia, podemos usar `grep -c` (count) para apenas contar quantas linhas possuem esse problema:


```bash
# Busca a letra N maiúscula no arquivo
grep -c "A" T10_R1.fastq
```

O awk 'NR%4==2' extrai apenas a linha 2 (sequência) de cada bloco de 4 linhas. 

O | passa essas sequências limpas para o grep buscar o "N"

```bash

awk 'NR%4==2' T10_R1.fastq | grep "N"

```



> **Atividade Prática:** Rode a contagem de "N" na amostra `T16_R1.fastq`. Ela tem mais ou menos erros não identificados que a T10?

### 5.4. Automatizando tarefas simples (Loop `for`)

Em bioinformática, quase nunca processamos um arquivo por vez. Em vez de calcular na mão as linhas de cada arquivo, vamos usar um loop `for` simples que faz a matemática do `wc -l` dividindo por 4 automaticamente para todos os arquivos!

```bash
for arquivo in *.fastq; do
    # 1. Guarda o número de linhas na variável $linhas
    linhas=$(wc -l < "$arquivo")
    
    # 2. Faz a matemática: divide as linhas por 4
    leituras=$((linhas / 4))
    
    # 3. Imprime o resultado formatado na tela
    echo "O arquivo $arquivo tem $leituras leituras biológicas."
done
```

### 5.5. Reconhecendo pares R1 e R2

As leituras R1 (Forward) e R2 (Reverse) compõem as extremidades do mesmo fragmento de DNA. Examine o primeiro cabeçalho nos dois arquivos da amostra T10:

```bash
head -n 1 T10_R1.fastq
head -n 1 T10_R2.fastq
```
**Note a diferença no final do cabeçalho:** Geralmente você verá `1:N:0:` para Forward e `2:N:0:` para Reverse.

> **Atividade Prática:** Vamos misturar o que aprendemos no item 5.4 com o `head`! Escreva um pequeno loop `for` que imprima apenas a **primeira linha** de todos os arquivos que terminam com `_R2.fastq`.
> *(Dica: troque o `*.fastq` na primeira linha do loop por `*_R2.fastq` e coloque o comando `head` dentro)*.

## 6. Prática: Controle de Qualidade Visual (FastQC e MultiQC)

A exploração via texto é essencial para entender a estrutura dos dados, mas em bioinformática utilizamos ferramentas visuais para avaliar o escore Phred de milhões de leituras simultaneamente.

### 6.1. Instalando as ferramentas

```bash
sudo apt install fastqc multiqc -y
```

### 6.2. Rodando o FastQC

O **FastQC** vai varrer nossos arquivos e gerar relatórios em HTML repletos de gráficos.

```bash
# Criar uma pasta para guardar os relatórios
mkdir -p relatorios_fastqc

# Rodar o FastQC nas leituras da amostra T10
fastqc T10_R1.fastq T10_R2.fastq -o relatorios_fastqc/
```

### 6.3. Compilando os resultados com o MultiQC

Quando temos dezenas de arquivos, abri-los individualmente não é prático. O **MultiQC** lê todos os relatórios do FastQC e compila tudo em uma única página web interativa.

```bash
# O MultiQC vai olhar a pasta relatorios_fastqc e gerar o output na pasta relatorio_final
multiqc relatorios_fastqc/ -o relatorio_final/
```

**Como visualizar:** No Windows, abra o explorador de arquivos, navegue até a pasta da sua distribuição do WSL (ou digite `explorer.exe .` no terminal para abrir a janela mágica) e clique duas vezes no arquivo `multiqc_report.html` gerado na pasta `relatorio_final/`.

## 7. Exercícios de fixação finais

1. **Qualidade Visual:** Ao abrir o relatório gerado pelo MultiQC, como se comporta a linha de qualidade (Phred) no início da sequência em comparação ao final dela? Cai, aumenta ou se mantém estável?
2. **Exploração:** Analisando os relatórios, qual o tamanho médio (em pares de bases) das sequências (Sequence Length)?
3. **Raciocínio ecológico:** Por que a qualidade dos dados de sequenciamento pode afetar uma investigação sobre a presença de espécies raras em amostras de eDNA da coluna d'água?
---

## Leituras complementares

- **FASTQ format — Wikipédia:** https://en.wikipedia.org/wiki/FASTQ_format
- **Repositório didático da disciplina:** https://github.com/saviscos/eDNA
- **DADA2 — documentação e tutoriais:** https://benjjneb.github.io/dada2/tutorial.html

