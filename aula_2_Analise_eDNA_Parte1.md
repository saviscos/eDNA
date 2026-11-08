# Aula 2: Primeiros Passos na Análise de eDNA (WSL)

## De Arquivos Brutos a Sequências Biológicas (VSEARCH e DADA2)

Nesta aula, vamos assumir que as nossas amostras de eDNA já saíram do sequenciador separadas por amostra (desmultiplexadas). Nossos dados estão na pasta `/eDNA/reads_fastq/` e você tem em mãos arquivos FASTQ para diversas amostras (T4, T6, T7 ... até T16).

Nosso objetivo hoje é limpar a "sujeira" gerada pelo sequenciador e encontrar as verdadeiras sequências biológicas. Faremos isso conhecendo duas abordagens principais:

1. **VSEARCH (Terminal Linux):** Focado em velocidade e regras diretas de qualidade.
2. **DADA2 (Linguagem R):** Focado em estatística e modelos matemáticos de erro.

Para entender o processo passo a passo, começaremos pelo terminal Linux e depois passaremos para o R.

---

## Parte 1: Processamento Rápido com VSEARCH (Terminal Linux)

### O que é o VSEARCH?
O VSEARCH é um software de código aberto (gratuito) que atua como um verdadeiro "canivete suíço" da bioinformática para análise de microbiomas e eDNA. Escrito na linguagem C, ele é extremamente rápido e roda diretamente no terminal. Ele é capaz de unir leituras, filtrar dados ruins, desreplicar sequências e agrupá-las. 

Antes de usá-lo, precisamos garantir que ele está instalado no nosso sistema Linux. Abra o seu terminal (Ubuntu/WSL) e digite o comando abaixo:

```bash
# Atualiza a lista de pacotes do Linux e instala o vsearch
sudo apt update
sudo apt install vsearch -y
```

### 1.1 O Passo a Passo (Exemplo com a amostra T10)

Primeiro, vamos criar uma pasta para guardar os resultados desta aula.

```bash
# Cria a pasta de resultados dentro do diretório principal
mkdir -p /eDNA/resultados_vsearch
```
> **O que faz o `-p`?** O parâmetro `-p` (de *parents*) avisa ao Linux duas coisas: 1) Crie a pasta final e qualquer outra pasta no caminho que ainda não exista; e 2) Se a pasta já existir, não mostre mensagem de erro, apenas siga em frente.

**Unindo as Leituras (Merging):**
O sequenciador lê os fragmentos pelas duas pontas (R1 e R2). Vamos pedir ao VSEARCH para uni-las, descartando as que não se sobrepõem bem ou têm muitos erros.

```bash
vsearch --fastq_mergepairs /eDNA/reads_fastq/T10_R1.fastq \
        --reverse /eDNA/reads_fastq/T10_R2.fastq \
        --fastqout /eDNA/resultados_vsearch/T10_merged.fastq \
        --fastq_minovlen 20 \
        --fastq_maxdiffs 5
```

**Desreplicação:**
O VSEARCH agrupa as sequências idênticas, guardando apenas uma cópia e anotando quantas vezes ela se repete (`size=X`), economizando muita memória.

```bash
vsearch --derep_fulllength /eDNA/resultados_vsearch/T10_merged.fastq \
        --output /eDNA/resultados_vsearch/T10_unicas.fasta \
        --sizeout \
        --minuniquesize 2
```

### 1.2 Aquecimento para Automação: Relembrando o Loop `for`

Fazer o processo acima manualmente para dezenas de amostras seria inviável. No Linux, usamos um **Loop `for`** para automatizar tarefas repetitivas. 

Vamos fazer um teste simples resgatando o comando `head` da nossa primeira aula. O bloco abaixo vai procurar todos os arquivos R1, pegar um por um, e imprimir apenas as 2 primeiras linhas na tela.

```bash
# Lendo as duas primeiras linhas de cada arquivo R1
for arquivo_r1 in /eDNA/reads_fastq/*_R1.fastq; do
    echo "Lendo o arquivo: $arquivo_r1"
    head -n 2 $arquivo_r1
    echo "-----------------------------------"
done
```
*Note como a variável `$arquivo_r1` assume o nome de um arquivo diferente a cada volta do loop!*

### 1.3 Automatizando a União (Merging)

Agora vamos usar a mesma lógica do loop `for` para rodar o VSEARCH em todas as amostras. Aqui usaremos um truque com o comando `basename` para extrair apenas o nome da amostra (ex: "T4") e usá-lo para encontrar o arquivo R2 correspondente e nomear o arquivo de saída.

```bash
# Loop para Unir as leituras (Merge) de todas as amostras
for R1 in /eDNA/reads_fastq/*_R1.fastq; do
    # Extrai o nome da amostra (ex: T10, T4, T16)
    AMOSTRA=$(basename $R1 _R1.fastq)
    R2="/eDNA/reads_fastq/${AMOSTRA}_R2.fastq"
    
    echo "Unindo leituras da amostra: $AMOSTRA ..."
    
    vsearch --fastq_mergepairs $R1 \
            --reverse $R2 \
            --fastqout /eDNA/resultados_vsearch/${AMOSTRA}_merged.fastq \
            --fastq_minovlen 20 \
            --fastq_maxdiffs 5 \
            --quiet
done
echo "União de todas as amostras concluída!"
```

### 1.4 Automatizando a Desreplicação

Agora que todos os arquivos foram unidos (gerando os arquivos `_merged.fastq`), faremos um segundo loop focado apenas em varrer a pasta de resultados para desreplicar todo mundo.

```bash
# Loop para Desreplicar todas as amostras unidas
for MERGED in /eDNA/resultados_vsearch/*_merged.fastq; do
    
    # Extrai o nome da amostra baseado no arquivo final
    AMOSTRA=$(basename $MERGED _merged.fastq)
    
    echo "Desreplicando amostra: $AMOSTRA ..."
    
    vsearch --derep_fulllength $MERGED \
            --output /eDNA/resultados_vsearch/${AMOSTRA}_unicas.fasta \
            --sizeout \
            --minuniquesize 2 \
            --quiet
done
echo "Desreplicação de todas as amostras concluída!"
```

---

## Parte 2: O Padrão Ouro Estatístico com DADA2 (Linguagem R)

### O que é o DADA2?
Enquanto ferramentas clássicas dependem de agrupamentos rígidos (OTUs) baseados em uma porcentagem de similaridade (ex: juntar tudo que for 97% igual), o **DADA2** revolucionou a área criando o conceito de **ASVs (Variantes de Sequência Exatas)**. 

O DADA2 é um pacote da linguagem R que utiliza aprendizado de máquina. Ele lê os escores de qualidade (Phred) dos seus arquivos FASTQ e cria um modelo estatístico para entender exatamente onde o seu sequenciador costuma errar. Com isso, ele consegue diferenciar o que é um erro de leitura da máquina do que é uma mutação biológica real (mesmo que seja a diferença de apenas 1 nucleotídeo!).

Digite **R** (maiúsculo) no seu terminal Linux e aperte Enter para abrir o console (`>`).

```bash
R
```

### 2.1 Mapeando os Arquivos no R

*(Todos os comandos a partir de agora devem ser digitados no `>` do R)*

Primeiro, precisamos carregar o pacote do DADA2 e dizer ao R onde nossos arquivos brutos estão guardados. A função `list.files` procura arquivos seguindo um padrão de texto. Usamos `strsplit` para fatiar o nome dos arquivos e extrair apenas o código da amostra.

```R
library(dada2)

# Definir o caminho onde os arquivos estão
caminho <- "/eDNA/reads_fastq"

# Criar listas separadas para os arquivos Forward (R1) e Reverse (R2)
fnFs <- sort(list.files(caminho, pattern="_R1.fastq", full.names = TRUE))
fnRs <- sort(list.files(caminho, pattern="_R2.fastq", full.names = TRUE))

# Extrair os nomes das amostras (ex: "T10") cortando o nome do arquivo no "_"
nomes_amostras <- sapply(strsplit(basename(fnFs), "_"), `[`, 1)

# Visualizar se o mapeamento deu certo
print(nomes_amostras)
```

### 2.2 Avaliando a Qualidade (Amostra T10)

Antes de filtrar, o bioinformata precisa inspecionar os dados. A função `plotQualityProfile` desenha um gráfico do escore Phred (qualidade). Como estamos no terminal, salvaremos isso como um PDF.

```R
# Gerar gráfico de qualidade para a T10 (Forward e Reverse)
pdf("/eDNA/T10_perfil_qualidade.pdf")
plotQualityProfile(c("/eDNA/reads_fastq/T10_R1.fastq", "/eDNA/reads_fastq/T10_R2.fastq"))
dev.off()
```
*Vá até a pasta no Windows e abra o PDF. Observe onde a linha verde (qualidade média) começa a cair.*

### 2.3 Filtragem e Corte (`filterAndTrim`)

Com base na qualidade, vamos remover ruídos. A função `filterAndTrim` vai processar todas as amostras simultaneamente. 
- **`truncLen`**: Define onde a sequência será cortada (ex: corta a ponta direita ruim).
- **`maxN`**: Exclui qualquer sequência com nucleotídeos ambíguos ("N").
- **`maxEE`**: Filtra baseando-se no "Número Máximo de Erros Esperados".

```R
# Criar uma pasta específica para guardar as leituras limpas
dir.create("/eDNA/filtradas")

# Definir como os novos arquivos vão se chamar
filtFs <- file.path("/eDNA/filtradas", paste0(nomes_amostras, "_F_filt.fastq.gz"))
filtRs <- file.path("/eDNA/filtradas", paste0(nomes_amostras, "_R_filt.fastq.gz"))
names(filtFs) <- nomes_amostras
names(filtRs) <- nomes_amostras

# Executar a filtragem em massa (Ajuste o truncLen conforme seu PDF!)
out <- filterAndTrim(fnFs, filtFs, fnRs, filtRs, 
                     truncLen=c(240,160), maxN=0, maxEE=c(2,2), truncQ=2, multithread=TRUE)

# Ver quantas sequências sobraram após o filtro
head(out)
```

### 2.4 Treinamento do Modelo de Erro (`learnErrors`)

Este é o coração do DADA2. O algoritmo examina os arquivos filtrados e calcula estatisticamente a probabilidade de um nucleotídeo 'A' ter sido lido erroneamente como 'C' pelo sequenciador, considerando sua qualidade Phred.

```R
# Aprender as taxas de erro das fitas Forward
errF <- learnErrors(filtFs, multithread=TRUE)

# Aprender as taxas de erro das fitas Reverse
errR <- learnErrors(filtRs, multithread=TRUE)
```

### 2.5 Inferência da Biologia Real (`dada`)

Finalmente, aplicamos o modelo aprendido sobre os nossos dados. A função `dada` usa a probabilidade de erro para "corrigir" sequências que são ruído gerado pela máquina, mantendo intactas as sequências que representam a diversidade biológica verdadeira (as ASVs).

```R
# Algoritmo principal de inferência de variantes de sequência (ASVs)
dadaFs <- dada(filtFs, err=errF, multithread=TRUE)
dadaRs <- dada(filtRs, err=errR, multithread=TRUE)

# Inspecionar o que ele encontrou na amostra T10
dadaFs[["T10"]]
```

### Encerrando a Aula

Para sair do console do R e voltar ao terminal Linux padrão, digite:

```R
q()
```

*(Quando o sistema perguntar `Save workspace image? [y/n/c]:`, digite `n` e aperte Enter).*
