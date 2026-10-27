## 2. Remoção de Primers

Certificando-se de que você está no diretório correto onde seus arquivos demultiplexados estão localizados, vamos adaptar nosso comando de demultiplexação para cortar (trimar) os primers de nossos pares de arquivos.

As sequências dos primers que utilizamos estão listadas abaixo. Adicionaremos o parâmetro `--discard-untrimmed` ao comando. Nós também poderíamos ter adicionado esse parâmetro na etapa de demultiplexação para remover todos os arquivos "desconhecidos" (*unknown*).

* **Forward (R1):** `CCNGAYATRGCNTTYCCNCG`

* **Reverse (R2):** `TANACYTCNGGRTGNCCRAARAAYCA`

Certifique-se de analisar o output (saída) do `cutadapt` no terminal, pois ele é muito informativo. Você notará agora que alguns erros estão sendo permitidos, já que essas sequências são mais longas, e o padrão de 10% de tolerância do programa permite cerca de 2 erros nestes primers. Além disso, você notará que, diferentemente dos índices, o comprimento da sequência removida variou um pouco.

### Automatizando com um Loop em Bash

Queremos executar essa remoção em todos os nossos arquivos, idealmente sem precisar escrever o comando repetidas vezes. Podemos colocar isso em um loop usando a linguagem *bash* do terminal. Como precisamos dos nomes de todas as amostras únicas, primeiro desenhamos um comando para listá-las:

```
ls | cut -d_ -f1 | sort | uniq

```

Este comando extrai a parte de cada nome de arquivo antes do primeiro `_` (underline) e encontra os nomes únicos. Podemos então armazenar essa lista de nomes de bibliotecas em uma variável do bash e verificar seu conteúdo com o comando `echo`:

```
samples=$(ls | cut -d_ -f1 | sort | uniq)
echo $samples

```

*(Nota 1:para ver todas as variaveis que foram criadas use `vcompgen -v`)*

*(Nota 2: Certifique-se de criar o diretório de saída antes de rodar o loop, por exemplo, rodando `mkdir output`).*

Agora, podemos fazer um loop sobre o conteúdo desta variável usando o seu comando `for`:

```
for s in $samples; do
    cutadapt -g ^CCNGAYATRGCNTTYCCNCG -G ^TANACYTCNGGRTGNCCRAARAAYCA \
    -o output/${s}_R1.fastq -p output/${s}_R2.fastq --discard-untrimmed \
    ./${s}_R1.fastq ./${s}_R2.fastq ;
done

```

**Entendendo o comando:**
O loop `for` trabalha através dos itens na variável `$samples`, atribuindo-os um por um à variável `$s`. Portanto, o `$s` se refere a cada nome de amostra por vez. Note que usamos `${s}` (com chaves) porque queremos adicionar um `_` imediatamente após o nome da variável. Se usássemos apenas `$s_R1`, o bash procuraria por uma variável inexistente chamada `s_R1`.

Verifique seu diretório `output/` para garantir que você tenha todos os seus arquivos cortados e revise a saída do terminal para ter certeza de que não perdeu nenhum erro. Como sempre, revise seus números de *reads*.

### ⚠️ Aviso Importante sobre Nomes de Arquivos

Para rodar esses loops, geramos uma lista de nossos nomes de amostras únicos listando nossos arquivos e, em seguida, editando o texto para remover as extensões de arquivo e as informações de direção (R1/R2), conforme o código abaixo:

```
samples=$(ls | cut -d_ -f1 | sort | uniq)

```

Se você estiver usando seus próprios dados de laboratório, os nomes dos seus arquivos provavelmente serão diferentes e o código exato usado acima pode não funcionar! Você precisará criar a sua própria versão.
Se você tiver vários sublinhados (`_`) em seus nomes de arquivo, mas o número de sublinhados for consistente, você poderia simplesmente ajustar o número fornecido ao comando `cut -f`. Caso contrário, você pode precisar usar o `sed` para remover partes dos nomes, ou até mesmo usar o `rename` para renomear os arquivos em lote para algo que funcione.

### Avaliação de Qualidade com FastQC e MultiQC

Para visualizar o impacto da remoção dos primers e a qualidade geral das suas *reads*, podemos usar as ferramentas FastQC e MultiQC. O FastQC calcula as métricas de cada arquivo individualmente, enquanto o MultiQC vasculha os diretórios e agrega todos esses resultados em um relatório interativo e visual. 

Para termos comparações justas, vamos gerar dois relatórios separados: um para os dados antes de remover os primers e outro para os dados após a remoção.

Execute a seguinte sequência de comandos no seu terminal:

```bash
# 1. Gerar relatório para os arquivos demultiplexados (antes da remoção dos primers)
fastqc ./1_demux/*.fastq
multiqc ./1_demux --ignore output -o ./relatorio_desmultiplex

# 2. Gerar relatório para os arquivos sem primers (dentro de output)
fastqc ./1_demux/output/*.fastq
multiqc ./1_demux/output -o ./relatorio_semPrimer
```

*Nota: Utilizamos o parâmetro `--ignore output` no primeiro comando do MultiQC para que ele não inclua acidentalmente os arquivos já processados da pasta output no relatório dos dados brutos.*

Os comandos acima criarão duas pastas distintas (`relatorio_desmultiplex` e `relatorio_semPrimer`), cada uma com seu próprio arquivo `multiqc_report.html`. Você pode abrir esses arquivos HTML em qualquer navegador da web para comparar visualmente as métricas de qualidade antes e depois da ação do Cutadapt!

### Próximos Passos

Geramos com sucesso outro conjunto de arquivos, agora com os primers removidos e alguns pares de *reads* rejeitados por não apresentarem a sequência do primer.

A seguir, juntaremos esses pares complementares (*mate pairs*) em leituras únicas que representarão as sequências individuais dos nossos amplicons na etapa 

**Fusão de Pares (Pair merging)**.
