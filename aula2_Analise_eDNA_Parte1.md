# Aula 2: Primeiros Passos na Análise de eDNA (WSL)

## De Arquivos Brutos a Sequências Biológicas (VSEARCH)

Nesta aula, vamos assumir que as nossas amostras de eDNA já saíram do sequenciador separadas por amostra (desmultiplexadas) e que os primers já foram cortados. Nossos dados estão na pasta `reads_fastq/` e você tem em mãos arquivos para diversas amostras (T4, T6, T7 ... até T16).

Nosso objetivo hoje é unir as leituras, aplicar um controle rigoroso de qualidade para limpar a "sujeira" gerada pelo sequenciador e encontrar as verdadeiras sequências biológicas utilizando o VSEARCH.

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

# Aula 2: União das Leituras (Merging) e Controle de Qualidade

Nesta aula, daremos o primeiro passo prático na transformação dos nossos dados brutos em sequências biológicas prontas para análise.

Nossos dados de eDNA estão na pasta `reads_fastq/` e consistem em leituras "Paired-End" (R1 e R2). Como o sequenciador lê os fragmentos de DNA pelas duas pontas, precisamos "colar" (unir) essas pontas para reconstruir o fragmento inteiro. Faremos isso usando o **VSEARCH**.

---

### 1. Criando o Diretório de Trabalho

Antes de começar, precisamos de um lugar organizado para salvar os resultados desta etapa. 
No seu terminal, certifique-se de estar dentro da pasta `reads_fastq` e crie a pasta de resultados:

```bash
# Cria a pasta resultados_vsearch (se ela ainda não existir)
mkdir -p ./resultados_vsearch
```

### 2. Entendendo o Merging (Exemplo prático com a amostra T10)

Vamos testar o comando em apenas uma amostra primeiro. O VSEARCH vai pegar a leitura *Forward* (R1) e a *Reverse* (R2), procurar onde elas se sobrepõem no meio, e colá-las. 

Ao mesmo tempo, vamos pedir para ele atuar como um "segurança de porta", jogando fora qualquer leitura que tenha baixa qualidade.

Execute o comando abaixo para a amostra T10:

```bash
vsearch --fastq_mergepairs ./T10_R1.fastq \
        --reverse ./T10_R2.fastq \
        --fastqout ./resultados_vsearch/T10_merged.fastq \
        --fastq_minovlen 20 \
        --fastq_maxdiffs 5 \
        --fastq_maxee 1.0 \
        --fastq_minlen 200 \
        --fastq_maxns 0
```

### O que significa cada linha desse comando?

O VSEARCH usa parâmetros (palavras que começam com dois traços `--`) para receber nossas instruções. É essencial entender os filtros que estamos aplicando:

*   `--fastq_mergepairs ./T10_R1.fastq`: Indica o arquivo com as leituras originais da fita *Forward* (R1).
*   `--reverse ./T10_R2.fastq`: Indica o arquivo com as leituras originais da fita *Reverse* (R2).
*   `--fastqout ./resultados_vsearch/T10_merged.fastq`: Define o nome e o caminho onde o arquivo final, já unido, será salvo.
*   `--fastq_minovlen 20`: **Filtro de sobreposição.** Exige que as leituras R1 e R2 tenham uma região de contato de pelo menos 20 letras (nucleotídeos). Se sobrepuser menos que isso, a união não é confiável e a sequência é descartada.
*   `--fastq_maxdiffs 5`: **Filtro de incompatibilidade.** Permite no máximo 5 diferenças (erros) na área onde R1 e R2 se sobrepõem. Se houver muito conflito entre as fitas, a leitura é descartada.
*   `--fastq_maxee 1.0`: **Filtro estatístico de qualidade.** O VSEARCH lê o escore de qualidade (Phred) de cada base e calcula a probabilidade estatística de erro na sequência inteira. Se a soma dos erros esperados for maior que 1, a sequência inteira vai para o lixo.
*   `--fastq_maxns 0`: **Filtro de ambiguidade.** Rejeita automaticamente qualquer leitura que contenha nucleotídeos que a máquina não conseguiu identificar (marcados com a letra "N").
*   `--fastq_minlen 200: **Filtro de tamanho minimo** Se o seu amplicon alvo tem cerca de 250 pares de bases (pb), uma sequência que uniu e ficou com 100 pb é certamente lixo biológico (dímeros de primer ou amplificação inespecífica).

---

### 3. Automatizando o Merging para Todas as Amostras

Agora que sabemos que o comando funciona para a amostra T10, rodar isso manualmente para a T4, T6, T7, etc., seria exaustivo. Vamos usar o poder do **Loop `for`** do Linux para fazer isso em segundos.

Usaremos o comando `basename` para extrair apenas o nome principal da amostra a cada volta do loop.

Copie e cole o bloco inteiro no seu terminal:

```bash
# Inicia o loop procurando todos os arquivos que terminam com _R1.fastq
for R1 in ./*_R1.fastq; do
    
    # Extrai o nome da amostra (ex: tira o _R1.fastq e guarda apenas "T10")
    AMOSTRA=$(basename $R1 _R1.fastq)
    
    # Define quem é o arquivo R2 daquela amostra
    R2="./${AMOSTRA}_R2.fastq"
    
    echo "Unindo e filtrando a amostra: $AMOSTRA ..."
    
    # Roda o VSEARCH usando as variáveis
    vsearch --fastq_mergepairs $R1 \
            --reverse $R2 \
            --fastqout ./resultados_vsearch/${AMOSTRA}_merged.fastq \
            --fastq_minovlen 20 \
            --fastq_maxdiffs 5 \
            --fastq_maxee 1.0 \
            --fastq_minlen 200 \
            --fastq_maxns 0 \
            --quiet
            
done

echo "Processo de união concluído para todas as amostras!"
```
*(Nota: O parâmetro `--quiet` no final do VSEARCH serve apenas para não poluir a tela do seu terminal com milhares de textos enquanto o loop trabalha).*

**Atividade Prática:**
Após o loop terminar, digite `ls ./resultados_vsearch/` para verificar os arquivos criados. Use o comando `wc -l` em um dos arquivos gerados (como o `T10_merged.fastq`) e compare com o arquivo original (`T10_R1.fastq`). 
Lembrando de dividir o resultado por 4, quantas sequências de DNA sobreviveram aos nossos filtros de qualidade?
