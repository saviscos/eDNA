# Aula 2: Primeiros Passos na Análise de eDNA (WSL)

## De Arquivos Brutos a Sequências Biológicas (VSEARCH)

Nesta aula, vamos assumir que as nossas amostras de eDNA já saíram do sequenciador separadas por amostra (desmultiplexadas) e que os primers já foram cortados. Nossos dados estão na pasta `reads_fastq/` e você tem em mãos arquivos para diversas amostras (T4, T6, T7 ... até T16).

Nosso objetivo hoje é unir as leituras, aplicar um controle rigoroso de qualidade para limpar a "sujeira" gerada pelo sequenciador e encontrar as verdadeiras sequências biológicas utilizando o VSEARCH.

---

## Processamento Rápido com VSEARCH (Terminal Linux)

### O que é o VSEARCH?
O VSEARCH é um software de código aberto (gratuito) que atua como um verdadeiro "canivete suíço" da bioinformática para análise de microbiomas e eDNA. Escrito na linguagem C, ele é extremamente rápido e roda diretamente no terminal. Ele é capaz de unir leituras, filtrar dados ruins, desreplicar sequências e agrupá-las. 

Antes de usá-lo, precisamos garantir que ele está instalado no nosso sistema Linux. Abra o seu terminal (Ubuntu/WSL) e digite o comando abaixo:
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
mkdir -p ./resultados_vsearch
```
> **O que faz o `-p`?** O parâmetro `-p` (de *parents*) avisa ao Linux duas coisas: 1) Crie a pasta final e qualquer outra pasta no caminho que ainda não exista; e 2) Se a pasta já existir, não mostre mensagem de erro, apenas siga em frente.

**Unindo as Leituras (Merging):**
O sequenciador lê os fragmentos pelas duas pontas (R1 e R2). Vamos pedir ao VSEARCH para uni-las, descartando as que não se sobrepõem bem ou têm muitos erros.

```bash
vsearch --fastq_mergepairs ./T10_R1.fastq \
        --reverse ./T10_R2.fastq \
        --fastqout ./resultados_vsearch/T10_merged.fastq \
        --fastq_minovlen 20 \
        --fastq_maxdiffs 5
```
**Entendendo os parâmetros da união:**
* `--fastq_mergepairs`: Indica o arquivo com as leituras Forward (R1) que inicia a sobreposição.
* `--reverse`: Indica o arquivo com as leituras Reverse (R2) correspondentes.
* `--fastqout`: Define o nome e o local onde o arquivo unido final será salvo.
* `--fastq_minovlen 20`: Exige que as leituras R1 e R2 tenham uma sobreposição de **pelo menos 20 bases**. Se sobrepuser menos, a leitura é descartada.
* `--fastq_maxdiffs 5`: Permite no máximo **5 diferenças (erros)** na região de sobreposição. Se a divergência for maior, a leitura também é descartada.

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


**2. Desreplicação:**
O VSEARCH agrupa as sequências idênticas, guardando apenas uma cópia e anotando quantas vezes ela se repete (`size=X`), economizando muita memória.

```bash
vsearch --derep_fulllength /eDNA/resultados_vsearch/T10_merged.fastq \
        --output /eDNA/resultados_vsearch/T10_unicas.fasta \
        --sizeout \
        --minuniquesize 2
```
**Entendendo os parâmetros da desreplicação:**
* `--derep_fulllength`: Comando que instrui o VSEARCH a procurar sequências 100% idênticas considerando todo o comprimento da leitura.
* `--output`: Define o nome do arquivo gerado (note que agora passamos de FASTQ para FASTA, pois a qualidade não importa mais nesta etapa).
* `--sizeout`: Adiciona a anotação da abundância (ex: `size=150`) direto no cabeçalho da sequência.
* `--minuniquesize 2`: Um excelente filtro de ruído biológico! Descarta os *singletons* (sequências que aparecem apenas **1 vez** na amostra inteira, sendo quase sempre erros aleatórios de sequenciamento).

* 
### 2.1 Automatizando a Desreplicação

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


*(Quando o sistema perguntar `Save workspace image? [y/n/c]:`, digite `n` e aperte Enter).*
