## 1. Demultiplexação

### Introdução

Existem muitas maneiras pelas quais bibliotecas de metabarcoding podem ser sequenciadas. Se você estiver usando os dados de exemplo, trabalharemos aqui com um pipeline de preparação de biblioteca que envolveu a indexação de amplicons durante a PCR inicial, de modo que cada amostra tivesse um índice diferente de 6 bases dentro de uma biblioteca. Uma vez sequenciados, precisamos usar essas sequências de índice para separar as diferentes amostras.

Esse processo é chamado de **demultiplexação**. Neste caso, os amplicons foram sequenciados usando sequenciamento *paired-end* (extremidades pareadas), o que significa que as duas extremidades de cada fragmento foram lidas, trabalhando de fora para dentro. Temos oito arquivos de sequência recebidos de nossa central de sequenciamento, um para cada direção de leitura (*forward* e *reverse*) para cada uma das quatro bibliotecas.

### Eu não tenho dados multiplexados!

Se você estiver acompanhando com seus próprios dados, esta etapa se aplica apenas se os arquivos de sequência que você possui contiverem dados de múltiplas amostras misturadas. Geralmente, isso acontece se você multiplexa suas amostras juntas durante a primeira rodada de PCR, bem como a multiplexação que acontece como parte da preparação da biblioteca. O próprio sequenciador geralmente desfaz a multiplexação da preparação da biblioteca, então se você não multiplexou durante a primeira rodada de PCR, provavelmente pode pular esta etapa e ir direto para a remoção de primers.


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

### Entendendo o Comando de Desmultiplexação (Cutadapt)

O comando abaixo é responsável por ler os arquivos contendo todas as sequências misturadas da nossa corrida de sequenciamento e separá-las em arquivos individuais para cada amostra, baseando-se em suas etiquetas (*barcodes* ou *tags*).

```bash
cutadapt -g T4=^AAGAGG -g T9=^AATCGC -g T11=^AGCTAC \
    -G T4=^AAGAGG -G T9=^AATCGC -G T11=^AGCTAC \
    -o 1_demux/{name1}-{name2}_R1.fastq -p 1_demux/{name1}-{name2}_R2.fastq \
    0_rawsequences/Lib1_R1.fastq 0_rawsequences/Lib1_R2.fastq
```

Vamos dissecar o que significa cada parte deste comando:

*   **`-g` (Tags na fita Forward / R1):** O parâmetro `-g` minúsculo diz ao Cutadapt para procurar uma etiqueta específica na fita *Forward*.
    *   A sintaxe `Nome=Sequencia` (ex: `T4=^AAGAGG`) diz ao programa: *"Se você encontrar a sequência AAGAGG, saiba que essa leitura pertence à amostra T4"*.
    *   **O símbolo `^` (Âncora):** É crucial! Ele obriga o programa a procurar essa etiqueta **exatamente no início** da fita. Se a sequência `AAGAGG` aparecer solta lá no meio do gene por uma coincidência biológica, o Cutadapt vai ignorar. Isso evita muitos erros de identificação cruzada.

*   **`-G` (Tags na fita Reverse / R2):** O `-G` maiúsculo faz exatamente a mesma coisa, mas instrui o programa a procurar a etiqueta correspondente na fita *Reverse* (R2) do par.

*   **`-o` e `-p` (Arquivos de Saída):** 
    *   O `-o` (*output*) define onde será salva a fita Forward desmultiplexada e com qual nome.
    *   O `-p` (*paired output*) define onde será salva a fita Reverse emparelhada. 
    *   No nosso caso, ambas estão sendo direcionadas para dentro da pasta `1_demux/`.

**O Segredo do `{name1}` e `{name2}`:**
Estas são **variáveis dinâmicas** nativas do Cutadapt. Elas funcionam como curingas (espaços em branco) que o programa preenche automaticamente com base no que ele encontrou:
*   `{name1}` será substituído pelo nome da etiqueta que ele encontrou usando o parâmetro `-g` (na fita R1).
*   `{name2}` será substituído pelo nome da etiqueta que ele encontrou usando o parâmetro `-G` (na fita R2).

*Exemplo prático de como a mágica acontece:*
Se o Cutadapt pegar um par de leituras e encontrar o barcode de `T4` na R1 e o barcode de `T4` na R2, ele vai substituir `{name1}` por `T4` e `{name2}` por `T4`. 
O arquivo final gerado será nomeado e salvo automaticamente como: `1_demux/T4-T4_R1.fastq`. 

*   **Os arquivos de entrada:** Por fim, `0_rawsequences/Lib1_R1.fastq` e `0_rawsequences/Lib1_R2.fastq` no final do comando são os nossos arquivos brutos originais que estão sendo injetados no programa para serem analisados e separados.


### Exercício 2

* Antes de verificar os arquivos de saída, pense: quantos arquivos você espera obter desse comando?

* Liste os arquivos no diretório `1_demux` e execute o comando `grep` para ver os números de *reads* por arquivo.

```
ls 1_demux/*
grep -c "^@D00" 1_demux/*

```

Há mais arquivos do que o esperado? Isso ocorre porque o comando procurou por **todas** as combinações de adaptadores. Quando temos 3 índices forward diferentes e 3 índices reverse diferentes, existem 9 combinações possíveis. Adicione a isso as 6 possibilidades onde apenas um índice forward ou reverse é usado, mais a possibilidade onde nenhum índice é usado. Logo, você deve ter 16 pares de arquivos.

### Nota sobre erros de sequenciamento

Alguns pesquisadores usam todas essas combinações diferentes para aplicar de forma eficiente poucos índices visando identificar muitas amostras diferentes. No entanto, isso torna muito mais difícil detectar erros. Nós não temos sequências válidas que usem índices *forward* e *reverse* diferentes na mesma amostra, mas o demultiplexador encontrou várias: **são erros no sequenciamento**. O sequenciador associou erroneamente algumas *reads* como sendo do mesmo fragmento quando não são. Em alguns casos, nenhum índice pôde ser encontrado em uma ou ambas as *reads* pareadas. Elas são marcadas como *unknown* (desconhecidas). Felizmente, todos esses erros representam uma minoria, e a maioria das *reads* foi alocada corretamente.

Se você somar todas as contagens de *reads*, notará que estão faltando algumas do arquivo original: estas não tinham par correspondente e foram completamente descartadas.

### Exercício 3 e Solução Final

* Construa mais três comandos do `cutadapt` para demultiplexar as outras três bibliotecas, colocando os resultados no mesmo diretório de saída (`1_demux`).

**Abaixo estão os comandos completos para processar todas as bibliotecas (Lib1 a Lib4):**

```
# Lib1
cutadapt -g T4=^AAGAGG -g T9=^AATCGC -g T11=^AGCTAC -G T4=^AAGAGG -G T9=^AATCGC -G T11=^AGCTAC -o 1_demux/{name1}-{name2}_R1.fastq -p 1_demux/{name1}-{name2}_R2.fastq 0_rawsequences/Lib1_R1.fastq 0_rawsequences/Lib1_R2.fastq

# Lib2
cutadapt -g T6=^AACACC -g T10=^AATCGC -g T16=^GAGTCA -G T6=^AACACC -G T10=^AATCGC -G T16=^GAGTCA -o 1_demux/{name1}-{name2}_R1.fastq -p 1_demux/{name1}-{name2}_R2.fastq 0_rawsequences/Lib2_R1.fastq 0_rawsequences/Lib2_R2.fastq

# Lib3
cutadapt -g T7=^AAGAGG -g T12=^AATCGC -g T14=^AGCTAC -G T7=^AAGAGG -G T12=^AATCGC -G T14=^AGCTAC -o 1_demux/{name1}-{name2}_R1.fastq -p 1_demux/{name1}-{name2}_R2.fastq 0_rawsequences/Lib3_R1.fastq 0_rawsequences/Lib3_R2.fastq

# Lib4
cutadapt -g T8=^AACACC -g T13=^AAGCCT -g T15=^AGCTAC -G T8=^AACACC -G T13=^AAGCCT -G T15=^AGCTAC -o 1_demux/{name1}-{name2}_R1.fastq -p 1_demux/{name1}-{name2}_R2.fastq 0_rawsequences/Lib4_R1.fastq 0_rawsequences/Lib4_R2.fastq

```

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
