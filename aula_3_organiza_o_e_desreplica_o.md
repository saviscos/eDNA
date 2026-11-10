# Aula 3: Organização Global e Desreplicação (VSEARCH)

Na aula anterior, nós unimos as fitas de DNA e aplicamos filtros rigorosos de qualidade e tamanho. Agora, dentro da nossa pasta `./resultados_vsearch`, temos vários arquivos limpos, um para cada amostra (ex: `T10_merged.fastq`, `T11_merged.fastq`).

O objetivo de hoje é preparar esses arquivos para a etapa de agrupamento biológico. O VSEARCH trabalha de forma muito mais inteligente e rápida se ele puder ver todas as sequências do seu projeto de uma só vez. Mas temos um problema: se apenas misturarmos os arquivos agora, como saberemos de qual amostra veio cada sequência no final?

A solução é colocar uma **etiqueta (tag)** no cabeçalho de cada leitura de DNA antes de misturá-las.

---

### 1. Etiquetando e Concatenando (A Mágica do `awk`)

Vamos usar um *loop* combinado com o comando `awk` (uma ferramenta nativa do Linux incrível para editar textos). O `awk` vai abrir cada arquivo, procurar apenas a linha do cabeçalho (que no formato FASTQ é sempre a 1ª, 5ª, 9ª linha, etc.) e inserir a etiqueta `sample=NOME_DA_AMOSTRA;`. Ao mesmo tempo, ele vai jogar tudo dentro de um arquivo único.

No seu terminal, vá para a pasta onde estão os arquivos unidos e rode o loop:

```bash
# 1. Entrar na pasta de resultados
cd ./resultados_vsearch/

# 2. Iniciar o loop para todos os arquivos _merged.fastq
for ARQUIVO in *_merged.fastq; do
    
    # Extrai apenas o nome da amostra (ex: T10)
    AMOSTRA=$(basename $ARQUIVO _merged.fastq)
    
    echo "Adicionando etiquetas na amostra $AMOSTRA..."
    
    # O awk verifica: se a linha for a número 1, 5, 9 (NR%4==1)
    # ele insere a tag logo após o @ do cabeçalho.
    # O símbolo >> vai empilhando (concatenando) tudo em um arquivo só.
    awk -v amos="$AMOSTRA" 'NR%4==1 {sub(/^@/, "@sample=" amos ";")} {print}' $ARQUIVO >> todas_amostras_concatenadas.fastq

done

echo "Concatenação segura concluída com sucesso!"
```

### O que aconteceu aqui? 
- O `NR%4==1` é pura matemática de arquivos FASTQ. Como cada leitura ocupa exatamente 4 linhas, a linha 1, 5 e 9 sempre serão os cabeçalhos (onde o sequenciador coloca o ID).
- A instrução `sub(/^@/, "@sample=" amos ";")` substitui o `@` inicial por `@sample=T10;`. O VSEARCH foi programado para reconhecer exatamente esse formato de etiqueta!

### 💡 Entendendo a Mágica: O que significa `NR%4==1`?

Para entender esse comando, primeiro precisamos lembrar da "anatomia" de um arquivo FASTQ. Nele, **cada leitura de DNA ocupa exatamente 4 linhas**, sempre nesta mesma ordem:

1. **Cabeçalho:** O nome da sequência (começa com `@`)
2. **Sequência:** As letras do DNA (A, T, C, G)
3. **Separador:** Um sinal de `+`
4. **Qualidade:** As letrinhas do escore Phred

Se nós usássemos a substituição em todo o arquivo, o Linux tentaria colocar a etiqueta `sample=T10;` no meio da sequência de DNA ou na linha de qualidade, corrompendo nossos dados! Nós queremos etiquetar **apenas as linhas de cabeçalho**.

É aí que entra o `NR%4==1`. Ele é uma instrução matemática para o comando `awk`:

* **`NR` (Number of Record):** Significa "Número da Linha atual".
* **`% 4` (Módulo de 4):** É uma operação matemática que pega o número da linha e divide por 4, mas **guarda apenas o resto da divisão**.
* **`== 1`**: Queremos que a ação ocorra apenas quando esse resto for igual a 1.

**Veja como funciona na prática:**
- **Linha 1:** 1 dividido por 4. Resto = **1**. *(Bingo! É o cabeçalho. Insere a etiqueta aqui)*
- **Linha 2:** 2 dividido por 4. Resto = 2. *(Ignora)*
- **Linha 3:** 3 dividido por 4. Resto = 3. *(Ignora)*
- **Linha 4:** 4 dividido por 4. Resto = 0. *(Ignora)*
- **Linha 5:** 5 dividido por 4. Resto = **1**. *(Bingo! É o próximo cabeçalho)*

Basicamente, o `NR%4==1` diz ao computador: *"Conte de 4 em 4 linhas, começando pela linha 1, e faça a alteração apenas nessas linhas"*. Isso garante que a nossa etiqueta vá parar apenas no lugar certo!



**Atividade Prática:**
Vamos verificar se a mágica funcionou. Digite o comando abaixo para ver as primeiras linhas do arquivo gigante que criamos:
```bash
head -n 4 todas_amostras_concatenadas.fastq
```
Você deverá ver a sua nova etiqueta (ex: `sample=T10;`) estampada logo no começo do cabeçalho!

---

### 2. Desreplicação Global (Removendo a Redundância)

Neste momento, o arquivo `todas_amostras_concatenadas.fastq` está gigantesco e cheio de sequências repetidas (afinal, milhares de bactérias iguais deixaram o mesmo DNA na água). 

Processar dados repetidos gasta memória do computador à toa. A **Desreplicação** serve para colapsar todas as sequências 100% idênticas em uma única cópia representativa. O programa vai anotar no cabeçalho quantas vezes aquela sequência apareceu no projeto inteiro (`size=X`).

Ainda dentro da pasta `resultados_vsearch`, execute:

```bash
vsearch --fastx_uniques todas_amostras_concatenadas.fastq \
        --output sequencias_unicas_globais.fasta \
        --sizeout \
        --minuniquesize 2
```

**Entendendo os parâmetros:**
- `--fastx_uniques`: Lê o nosso grande arquivo FASTQ, agrupa as leituras idênticas e joga fora a informação de qualidade (Phred), pois não precisamos mais dela.
- `--output`: O arquivo final agora é salvo no formato FASTA (apenas cabeçalho e DNA).
- `--sizeout`: Adiciona a contagem de abundância total no cabeçalho (ex: `size=3450;`).
- `--minuniquesize 2`: Descarta os *singletons*. Se, entre **todas** as amostras misturadas, uma sequência apareceu apenas 1 vez, é quase certeza que ela é um erro aleatório do sequenciador (ou um artefato) e não um organismo real.

**Atividade Prática:**
Compare o tamanho e o número de linhas do arquivo antes e depois da desreplicação usando os comandos `ls -lh` e `wc -l`. 

```bash
# Contando as linhas do arquivo original (lembre de dividir por 4!)
wc -l todas_amostras_concatenadas.fastq

# Contando as sequências do arquivo desreplicado (como é FASTA, contamos o >)
grep -c "^>" sequencias_unicas_globais.fasta
```
Você notará que a quantidade de sequências no arquivo `sequencias_unicas_globais.fasta` é drasticamente menor, o que tornará a próxima etapa de análises muito mais rápida!
