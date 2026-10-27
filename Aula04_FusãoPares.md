## Fusão de Pares (Pair Merging)

### Introdução

Estas bibliotecas de metabarcoding foram sequenciadas de forma que 300 pb (pares de bases) do nosso fragmento fossem lidos a partir de cada extremidade (*paired-end*). Quando desenhamos nossos primers a partir de sequências completas do genoma mitocondrial, examinamos o amplicon esperado e sabemos que deve haver 418 pb de sequência entre os primers.

Dado que isso é muito mais curto do que 2 x 300 = 600 pb, deve haver sobreposição entre os fragmentos *forward* e *reverse*, e podemos usar isso para montar a sequência completa do amplicon.


### Dados e Software

Os dados de entrada para este tutorial são um diretório de pares de arquivos no formato FASTQ, um par por amostra, sem sequências de índice ou primer. Se você está acompanhando passo a passo, isso foi produzido no diretório `output` no tutorial anterior.

Este tutorial usa o software **PEAR** (Paired-End read mergeR). Caso ainda não o tenha instalado no seu ambiente, você pode instalá-lo rapidamente utilizando o gerenciador de pacotes Conda:

```bash
conda install bioconda::pear
```

### Qualidade da Sequência

Uma consideração importante ao fundir pares é que, em geral, as *reads* de sequenciamento declinam em qualidade ao longo de seu comprimento.

As extremidades distantes do primer são as partes que o algoritmo de fusão tentará combinar, e você notará que elas tendem a ser de qualidade inferior. Isso leva a duas considerações:

1. É importante estar ciente dos escores de qualidade ao fundir, pois sequências errôneas podem causar a fusão de leituras que não são do mesmo fragmento, formando uma quimera.

2. No entanto, a fusão pode, na verdade, fornecer alguma validação e melhorar os escores de qualidade para a região de sobreposição se ela combinar (match).

### Fusão de Pares com PEAR

Como sempre, dê uma olhada no arquivo de ajuda da ferramenta antes de começar:

```bash
pear --help
```

Não há muitas opções complexas. O PEAR nos dá algumas informações realmente úteis no terminal. Como você deve ter visto no arquivo de ajuda, existem muitos limites diferentes que podem ser aplicados à fusão:

* A probabilidade limite de uma sobreposição estar incorreta (`-p`).
* O escore limite para cortar extremidades de baixa qualidade (`-q`).
* A sobreposição mínima necessária (`-v`).

Geralmente, definimos o limite de qualidade (`-q`) para aparar extremidades de baixa qualidade para **26**, e a sobreposição mínima (`-v`) para um valor em torno de 20-30 bases menor do que o comprimento de sobreposição presumido.

### Entendendo a Lógica: Rodando o PEAR em uma Única Amostra

Antes de automatizarmos o processo para todas as amostras de uma vez, é fundamental entendermos como o PEAR funciona analisando uma única amostra (por exemplo, a amostra `T4`). 

Execute o comando abaixo no terminal:

```bash
# Cria a pasta para salvar os resultados (caso ainda não tenha criado)
mkdir -p 2_merged

# Executa a fusão (merging) apenas para o par de arquivos da amostra T4
pear -f T4_R1.fastq -r T4_R2.fastq -o 2_merged/T4 -q 26
```

**O que significa cada parte deste comando?**

* **`-f T4_R1.fastq`**: (*Forward*) Indica o arquivo que contém as leituras diretas da amostra.
* **`-r T4_R2.fastq`**: (*Reverse*) Indica o arquivo que contém as leituras reversas emparelhadas da mesma amostra.
* **`-o 2_merged/T4`**: (*Output*) Define o diretório de destino e o "prefixo" base do nome dos arquivos gerados. O PEAR criará automaticamente 4 arquivos de saída usando esse prefixo (ex: `T4.assembled.fastq`, `T4.discarded.fastq`, `T4.unassembled.forward.fastq` e `T4.unassembled.reverse.fastq`). O arquivo que nos interessa para continuar a análise é sempre o `.assembled.fastq`.
* **`-q 26`**: (*Quality*) Informa ao PEAR para cortar as extremidades das leituras que tiverem uma qualidade (Phred score) menor que 26 **antes** de tentar fundi-las. Isso aumenta a confiabilidade e o sucesso da sobreposição.

Ao executar o comando acima, você verá na tela do terminal um relatório detalhado e estatístico sobre quantas sequências foram fundidas com sucesso e quantas foram descartadas. 

Uma vez compreendida essa lógica básica, torna-se muito mais fácil instruir o computador a fazer isso repetidas vezes para todas as dezenas ou centenas de amostras do seu projeto através de um *loop*, como veremos a seguir.

### Rodando o PEAR em um loop

Assim como com o `cutadapt` no tutorial anterior, podemos rodar o PEAR em um loop para aplicá-lo a todas as nossas amostras. Em vez de criar uma lista complexa de nomes, vamos instruir o terminal a ler diretamente todos os arquivos *forward* (`R1`), extrair o nome base da amostra e rodar o PEAR.

Navegue até o diretório onde estão os arquivos sem primer, crie uma pasta para os arquivos mesclados e execute o loop:

```bash


# Cria a pasta para salvar os resultados mesclados
mkdir -p 2_merged

# Roda o loop lendo todos os arquivos R1
for f in *_R1.fastq; do
    # Extrai o nome da amostra removendo a extensão '_R1.fastq'
    s="${f%_R1.fastq}"
    
    # Executa o PEAR (-q 26 corta bases com qualidade menor que 26 antes de fundir)
    pear -f "${s}_R1.fastq" -r "${s}_R2.fastq" -o "2_merged/${s}" -q 26
done
```

Certifique-se de revisar essas saídas do terminal! Em seguida, liste o conteúdo do seu diretório de saída:

```bash
ls 2_merged/
```

Você verá arquivos para as *reads* montadas (`.assembled`), não montadas (`.unassembled`) e descartadas (`.discarded`). Nós não precisamos dos dois últimos, então vamos limpar nossa pasta:

```bash
cd 2_merged/

# Remove os arquivos indesejados
rm *discarded* *unassembled* 

# Renomeia os arquivos montados, tirando o ".assembled" do nome
rename "s/\.assembled//" *

cd ../
```

*(Nota: O comando `rename` usa expressões regulares para substituir a parte ".assembled" por "nada", simplificando os nomes dos seus arquivos finais).*

### Alternativas

A fusão de leituras é uma etapa muito importante e sensível no pipeline, portanto, é importante selecionar um software robusto.
Usamos o PEAR porque é uma ferramenta independente com uma abordagem probabilística para a tarefa de fusão e funciona bem para os nossos dados. No entanto, ele às vezes pode ser um pouco temperamental se suas *reads* não estiverem pareadas com precisão ou se houver *reads* extra muito curtas.

Uma alternativa comum ao PEAR é a ferramenta `fastq_mergepairs` dentro dos pacotes VSEARCH e USEARCH. Outra ferramenta frequentemente usada é o FLASH.

### Perda de Sequência e Escolha de Limites

Você deve ter notado que perdemos um grande número de sequências aqui. **Isso é completamente normal** - devemos ser muito rigorosos na etapa de fusão de pares para garantir que:

1. Fundimos corretamente um par verdadeiro para não gerar uma sequência espúria.

2. Não fundimos incorretamente um par falso causando uma sequência quimérica.

Atingimos isso usando configurações rigorosas de qualidade e sobreposição, e, portanto, a qualidade dos seus dados de sequenciamento terá um grande papel na extensão em que as sequências, e até mesmo amostras completas, podem ser perdidas nesta fase. A escolha de limites de qualidade geralmente é baseada na confiança que você deseja ter nos dados finais. A regra de ouro é: seja o mais conservador possível em suas configurações.

### Próximos Passos

Geramos *reads* completas da sequência do amplicon agora, então estamos chegando perto de concluir o processamento bruto das sequências.

No nosso exemplo, o comprimento das *reads* é longo o suficiente em comparação com o fragmento para haver sobreposição. Esse nem sempre é o caso. Se não houver sobreposição, uma opção é apenas concatenar (colar) as leituras sem mesclá-las fisicamente, ou analisar apenas uma das direções (ex: usar só as *reads Forward*).

No próximo tutorial, veremos como juntar todos os dados em **4. Concatenação de Dados (Data concatenation)**.
