# Aula Prática 02 — Demultiplexação de Dados de eDNA com Cutadapt

**Disciplina:** DNA ambiental (eDNA) aplicado à Oceanologia  
**Público-alvo:** Pós-graduação em Oceanologia e áreas afins  
**Duração sugerida:** 2 horas  
**Pré-requisitos:** Aula 01 — navegação no Linux, estrutura FASTQ e leituras paired-end (R1/R2)  
**Ferramentas:** terminal Linux (ou WSL/Ubuntu) e [Cutadapt](https://cutadapt.readthedocs.io/en/stable/)

> **Objetivo da aula:** separar leituras de sequenciamento provenientes de amostras diferentes que foram agrupadas na mesma biblioteca, mantendo os pares R1/R2 correspondentes e registrando quantitativamente o que aconteceu durante o processamento.

---

## 1. Objetivos de aprendizagem

Ao final desta prática, você deverá conseguir:

1. Distinguir **amostra biológica**, **biblioteca de sequenciamento**, **índice (barcode/tag)** e **leitura (read)**.
2. Reconhecer índices e primers nas sequências brutas de arquivos FASTQ.
3. Utilizar o **Cutadapt** para demultiplexar bibliotecas paired-end com índices nas duas extremidades.
4. Interpretar os arquivos produzidos para combinações esperadas, inesperadas e não identificadas (`unknown`).
5. Conferir a contagem de leituras e a correspondência numérica entre os arquivos R1 e R2.
6. Preservar os dados originais e os registros da análise, sem excluir resultados de forma precipitada.

## 2. Por que demultiplexar?

Em estudos de eDNA, diversas amostras ambientais — por exemplo, água coletada em pontos costeiros diferentes — podem ser amplificadas e sequenciadas conjuntamente para reduzir custos. Cada amostra recebe uma pequena sequência de DNA de identificação, denominada **índice**, **barcode** ou **tag**.

Após o sequenciamento, as leituras de diferentes amostras podem estar misturadas nos mesmos arquivos FASTQ. A **demultiplexação** identifica os índices e separa as leituras por sua amostra de origem.

```text
Biblioteca de sequenciamento Lib1
├── Lib1_R1.fastq    (leituras forward; várias amostras misturadas)
└── Lib1_R2.fastq    (leituras reverse; várias amostras misturadas)
              |
              v  Cutadapt: identificar índice de R1 e de R2
              |
              +-- T4-T4_R1.fastq   + T4-T4_R2.fastq   → amostra T4
              +-- T9-T9_R1.fastq   + T9-T9_R2.fastq   → amostra T9
              +-- T11-T11_R1.fastq + T11-T11_R2.fastq → amostra T11
              +-- outras combinações e categorias unknown → investigar
```

**Atenção:** no nosso exemplo, as sequências de índice foram incorporadas aos amplicons durante a PCR e aparecem no começo das reads. Isso é diferente dos índices de biblioteca (como i5/i7) que, em muitos serviços de sequenciamento, já foram processados pela central antes da entrega dos FASTQ. **Se seus arquivos já foram separados por amostra, esta etapa pode não ser necessária.**

### 2.1. Entendendo o desenho experimental

Nos dados didáticos usados neste curso:

- Existem **4 bibliotecas** (`Lib1`, `Lib2`, `Lib3`, `Lib4`).
- Cada biblioteca está representada por **2 arquivos FASTQ**: um R1 e um R2.
- Portanto, recebemos **8 arquivos FASTQ brutos**.
- Cada biblioteca contém leituras provenientes de **3 amostras**, identificadas por índices de 6 nucleotídeos.
- Esperamos recuperar **12 amostras** ao final da classificação dos índices, cada uma representada por um par de arquivos FASTQ.

> **Pergunta para discussão:** oito arquivos FASTQ de entrada significam oito amostras ambientais? Por quê?

---

## 3. Preparando o ambiente de trabalho

Vamos aproveitar o diretório `~/curso_eDNA`, criado na aula anterior. Caso você tenha usado outro nome, adapte o caminho abaixo.

```bash
# cd: muda o diretório de trabalho.
# ~: representa sua pasta pessoal (HOME).
cd ~/curso_eDNA

# pwd: mostra o caminho absoluto do diretório atual.
pwd

# ls: lista arquivos e pastas.
# -l: exibe informações detalhadas.
# -h: exibe os tamanhos em unidades legíveis (KB, MB, GB).
ls -lh

# test -d: verifica se o diretório informado existe.
# &&: executa o segundo comando somente se o teste for verdadeiro.
test -d 0_rawsequences && echo 'Diretório de dados encontrado.'

# ls -lh: confere os arquivos FASTQ de entrada.
# *.fastq: seleciona todos os nomes terminados em .fastq.
ls -lh 0_rawsequences/*.fastq
```

Se os dados ainda não estiverem disponíveis, execute os comandos abaixo **uma única vez**:

```bash
# wget: baixa um arquivo de uma URL.
# -c: tenta continuar um download interrompido (quando o servidor permite).
wget -c https://github.com/saviscos/eDNA/raw/main/0_rawsequences.zip

# unzip -t: testa a integridade do ZIP, sem extrair os arquivos.
unzip -t 0_rawsequences.zip

# unzip: extrai os arquivos do ZIP para o diretório atual.
unzip 0_rawsequences.zip
```

**Não altere nem exclua os FASTQ brutos:** eles constituem a referência para repetirmos o processamento, caso seja necessário.

### 3.1. Instalando o Cutadapt (Ubuntu/WSL)

```bash
# sudo: executa o comando com privilégios administrativos.
# apt update: atualiza a lista de pacotes disponíveis; não atualiza todo o sistema.
sudo apt update

# apt install: instala o pacote informado e suas dependências.
# -y: aceita automaticamente as perguntas de confirmação do gerenciador.
sudo apt install -y cutadapt

# --version: informa a versão instalada, útil para documentar a análise.
cutadapt --version

# --help: exibe as opções e os parâmetros disponíveis no programa.
cutadapt --help
```

> A versão distribuída pelo Ubuntu pode variar. Para a análise científica, registre a versão utilizada. A instalação por `apt` foi escolhida aqui para simplificar a primeira experiência no terminal; outras formas de instalação são possíveis.

---

## 4. Conhecendo os índices moleculares

### 4.1. Inspecionando o arquivo de índices

```bash
# cat: imprime todo o conteúdo de um arquivo de texto no terminal.
# Aqui, o arquivo contém a relação entre amostras e índices do experimento.
cat 0_rawsequences/indices.txt
```

Para os comandos desta prática, usaremos os índices apresentados no material do curso:

| Biblioteca | Amostra/índice | Sequência 5′ → 3′ |
|---|---|---|
| Lib1 | T4 | AAGAGG |
| Lib1 | T9 | AATCGC |
| Lib1 | T11 | AGCTAC |
| Lib2 | T6 | AACACC |
| Lib2 | T10 | AATCGC |
| Lib2 | T16 | GAGTCA |
| Lib3 | T7 | AAGAGG |
| Lib3 | T12 | AATCGC |
| Lib3 | T14 | AGCTAC |
| Lib4 | T8 | AACACC |
| Lib4 | T13 | AAGCCT |
| Lib4 | T15 | AGCTAC |

**Importante:** uma mesma sequência de índice pode ser reutilizada em bibliotecas diferentes. Por isso, **não devemos juntar os resultados de `Lib1` a `Lib4` usando apenas o nome ou a sequência do índice**. Cada biblioteca terá sua própria pasta de saída.

### 4.2. Inspecionando as primeiras leituras

```bash
# head: mostra o início de um arquivo.
# -n 12: imprime 12 linhas, ou seja, 3 registros FASTQ completos de 4 linhas.
head -n 12 0_rawsequences/Lib1_R1.fastq

# O mesmo procedimento no arquivo pareado (R2).
head -n 12 0_rawsequences/Lib1_R2.fastq
```

**Relembrando:** no FASTQ convencional, cada sequência ocupa a **segunda linha** de um bloco de quatro linhas. Não devemos procurar índices indiscriminadamente nos cabeçalhos e nas linhas de qualidade.

### 4.3. Procurando um índice no início das sequências

```bash
# awk: processa o arquivo linha por linha.
# NR: número da linha atual; %: operador de resto da divisão.
# NR % 4 == 2: seleciona apenas as linhas de sequência do FASTQ.
# print: imprime a linha selecionada.
# |: envia a saída de um comando como entrada do próximo.
# grep '^AAGAGG': mantém somente sequências que COMEÇAM com AAGAGG.
# ^: âncora que representa o início da linha.
# head: limita a exibição às primeiras 10 sequências encontradas.
awk 'NR % 4 == 2 {print}' 0_rawsequences/Lib1_R1.fastq | grep '^AAGAGG' | head
```

Também podemos contar quantas leituras começam com o índice `AAGAGG`:

```bash
# grep -c: informa o número de linhas que correspondem ao padrão.
# Como o awk já selecionou apenas as linhas de sequência, o resultado é
# o número de reads cuja sequência começa exatamente com esse índice.
awk 'NR % 4 == 2 {print}' 0_rawsequences/Lib1_R1.fastq | grep -c '^AAGAGG'
```

Esse valor é uma **contagem exploratória**, não a contagem final de amostras. Ainda não examinamos o índice da leitura R2 correspondente nem tratamos outros tipos de correspondência.

### 4.4. Quais são os seis nucleotídeos iniciais mais frequentes?

```bash
# awk: seleciona apenas as linhas de sequência.
# substr($0, 1, 6): extrai 6 caracteres a partir da posição 1 de cada linha.
# sort: organiza as sequências iguais em grupos consecutivos.
# uniq -c: conta a frequência de cada sequência distinta.
# sort -nr: ordena os resultados numericamente (-n), do maior para o menor (-r).
# head: mostra as 10 sequências iniciais mais frequentes.
awk 'NR % 4 == 2 {print substr($0, 1, 6)}' 0_rawsequences/Lib1_R1.fastq \
  | sort \
  | uniq -c \
  | sort -nr \
  | head
```

**Interpretação:** quais sequências de seis bases predominam? Elas correspondem aos índices previstos para `Lib1`?

---

## 5. Primers degenerados e relação com os índices

Os primers empregados no material didático são:

- **Primer forward (R1):** `CCNGAYATRGCNTTYCCNCG`
- **Primer reverse (R2):** `TANACYTCNGGRTGNCCRAARAAYCA`

Esses primers contêm **bases degeneradas**, representadas por códigos IUPAC:

| Código | Possíveis nucleotídeos |
|---|---|
| N | A, C, G ou T |
| R | A ou G |
| Y | C ou T |

No desenho experimental da prática, esperamos aproximadamente a organização:

```text
Leitura R1: [ÍNDICE DE 6 BASES][PRIMER FORWARD][REGIÃO-ALVO ...]
Leitura R2: [ÍNDICE DE 6 BASES][PRIMER REVERSE][REGIÃO-ALVO ...]
```

Para uma **inspeção visual com `grep -E`**, podemos substituir cada posição degenerada por `.` (qualquer caractere). Essa substituição é apenas uma aproximação: o ponto não restringe as combinações às alternativas IUPAC válidas.

```bash
# awk: seleciona somente linhas de sequência da biblioteca R1.
# grep -E: habilita expressões regulares estendidas.
# ^AAGAGG: exige índice T4 no início da sequência.
# CC.GA.AT.GC.TT.CC.CG: padrão aproximado do primer forward,
# com cada posição degenerada substituída por um ponto.
# head: exibe até 10 sequências que combinam índice + primer.
awk 'NR % 4 == 2 {print}' 0_rawsequences/Lib1_R1.fastq \
  | grep -E '^AAGAGGCC.GA.AT.GC.TT.CC.CG' \
  | head

# Repita a verificação no arquivo reverse (R2).
# A sequência após o índice é o padrão aproximado do primer reverse.
awk 'NR % 4 == 2 {print}' 0_rawsequences/Lib1_R2.fastq \
  | grep -E '^AAGAGGTA.AC.TC.GG.TG.CC.AA.AA.CA' \
  | head
```

> **Não encontrar um índice ou primer por busca exata não comprova erro de sequenciamento.** Pode haver diferenças no desenho da biblioteca, bases com qualidade baixa, alterações na sequência, orientação inesperada ou outros fatores. Primeiro verificamos os dados e o protocolo experimental.

**Nesta aula removeremos apenas os índices encontrados na extremidade 5′. A remoção específica dos primers será tratada em outra aula**, usando os códigos degenerados IUPAC apropriados no Cutadapt.

### Exercício 1 — Investigação dos índices

1. Quais são os seis nucleotídeos iniciais mais frequentes em `Lib1_R1.fastq`?
2. Os mesmos índices podem ser encontrados em `Lib1_R2.fastq`?
3. Qual é a diferença entre identificar um índice no início de uma sequência e encontrar esse mesmo padrão no interior dela?
4. Por que o arquivo `indices.txt` deve ser considerado antes de processar os dados?

---

## 6. Demultiplexação da biblioteca Lib1

### 6.1. Criando as pastas de saída

```bash
# mkdir: cria diretórios.
# -p: cria os diretórios necessários e não gera erro se eles já existirem.
# Separamos saídas de Lib1 e relatórios para evitar misturas e sobrescritas.
mkdir -p 1_demux/Lib1
mkdir -p relatorios
```

### 6.2. Entendendo os parâmetros do Cutadapt

| Opção ou elemento | O que faz nesta prática |
|---|---|
| `cutadapt` | Inicia o software de processamento das leituras. |
| `-e 0` | Permite **zero erros** nas correspondências dos índices; exige identificação exata. |
| `--no-indels` | Desabilita inserções e deleções nas correspondências; aqui reforça nossa estratégia conservadora. |
| `-g T4=^AAGAGG` | Procura no início de **R1** (`^`) o índice `AAGAGG`, denominado `T4`. |
| `-G T4=^AAGAGG` | Faz a busca equivalente no início de **R2**. |
| `-o` | Define o modelo do nome do arquivo FASTQ de saída de R1. |
| `-p` | Define o modelo do nome do arquivo FASTQ de saída de R2. |
| `{name1}` | É substituído pelo nome do índice reconhecido em R1. |
| `{name2}` | É substituído pelo nome do índice reconhecido em R2. |
| `2>&1` | Direciona mensagens de erro para o mesmo fluxo da saída comum. |
| `| tee arquivo.log` | Mostra o relatório no terminal e simultaneamente o salva em um arquivo. |
| `\` ao final da linha | Continua o mesmo comando na linha seguinte; não coloque comentários após essa barra. |

**Por que `-e 0`?** Os índices têm apenas seis nucleotídeos. Nesta demonstração, começaremos com correspondência exata para evitar ambiguidades na atribuição. Em uma análise real, a tolerância a erros deve ser avaliada considerando o desenho dos índices e suas distâncias entre si. Não se deve aumentar a tolerância automaticamente.

### 6.3. Executando o Cutadapt

Execute o comando completo, estando na pasta `~/curso_eDNA`:

```bash
# -e 0: aceita somente índices que correspondam exatamente às sequências dadas.
# --no-indels: não permite inserções/deleções ao procurar os índices.
# -g: define índices de R1; cada nome (T4, T9, T11) corresponde a uma sequência.
# -G: define índices de R2; as maiúsculas indicam a segunda leitura do par.
# ^: obriga a correspondência a começar na posição inicial da leitura.
# -o: modelo de nomes de saída de R1 (name1 = índice de R1; name2 = índice de R2).
# -p: modelo equivalente para o arquivo correspondente de R2.
# Os dois últimos argumentos são os FASTQ brutos de entrada (R1 e R2, nessa ordem).
# 2>&1 | tee: mostra o relatório no terminal e o grava para consulta posterior.
cutadapt \
  -e 0 \
  --no-indels \
  -g 'T4=^AAGAGG' \
  -g 'T9=^AATCGC' \
  -g 'T11=^AGCTAC' \
  -G 'T4=^AAGAGG' \
  -G 'T9=^AATCGC' \
  -G 'T11=^AGCTAC' \
  -o '1_demux/Lib1/{name1}-{name2}_R1.fastq' \
  -p '1_demux/Lib1/{name1}-{name2}_R2.fastq' \
  0_rawsequences/Lib1_R1.fastq \
  0_rawsequences/Lib1_R2.fastq \
  2>&1 | tee relatorios/Lib1_cutadapt.log
```

**O que esperamos?** Quando R1 e R2 recebem `T4`, o par é escrito em `T4-T4_R1.fastq` e `T4-T4_R2.fastq`. Quando apenas uma extremidade recebe um índice, a extremidade sem correspondência é chamada `unknown`. Se R1 receber `T4` e R2 receber `T9`, os arquivos serão identificados como `T4-T9`.

O Cutadapt **remove os índices identificados** das reads, mas nesta etapa ainda **não remove os primers**.

### 6.4. Por que podem surgir mais de três pares de arquivos?

Para `Lib1`, há quatro estados possíveis para cada extremidade: `T4`, `T9`, `T11` e `unknown`. Logo, existem **até 4 × 4 = 16 combinações** e, para cada combinação observada, pode existir um arquivo R1 e um arquivo R2. **Nem todas as combinações precisam aparecer**.

Neste experimento, as combinações esperadas são `T4-T4`, `T9-T9` e `T11-T11`. As demais devem ser **quantificadas e investigadas**, e não automaticamente atribuídas a um único tipo de erro.

> **Nota técnica:** uma combinação inesperada de índices não significa, isoladamente, que o sequenciador emparelhou reads de fragmentos diferentes. Ela pode resultar de múltiplos processos, como erros de leitura, contaminação cruzada ou problemas de preparação das bibliotecas. A interpretação depende dos controles e do protocolo experimental.

---

## 7. Inspecionando e validando os resultados de Lib1

### 7.1. Listar os FASTQ gerados

```bash
# ls: lista os resultados da biblioteca Lib1.
# -lh: acrescenta detalhes e tamanhos de arquivo legíveis.
ls -lh 1_demux/Lib1/

# head -n 8: mostra as duas primeiras leituras do arquivo demultiplexado.
# Este exemplo supõe que a categoria T4-T4 esteja presente na saída.
head -n 8 1_demux/Lib1/T4-T4_R1.fastq
```

Compare as primeiras bases das reads de entrada e de saída. **O índice de seis bases foi removido? O primer continua presente?**

### 7.2. Quantidade de leituras por FASTQ

```bash
# awk: contabiliza o total de linhas de um FASTQ convencional.
# NR: total de linhas processadas até o final.
# END: executa a instrução ao terminar de ler o arquivo.
# NR / 4: converte linhas em número de reads, assumindo 4 linhas por registro.
awk 'END {print NR / 4}' 0_rawsequences/Lib1_R1.fastq

# Repita a contagem para um dos FASTQ demultiplexados.
awk 'END {print NR / 4}' 1_demux/Lib1/T4-T4_R1.fastq
```

Para obter uma tabela simples de contagens de **todos os arquivos R1** de Lib1:

```bash
# for ... in ...: percorre um arquivo por vez (todos os *_R1.fastq).
# basename: mostra apenas o nome do arquivo, sem o caminho do diretório.
# $(...): insere no comando o resultado produzido dentro dos parênteses.
# awk 'END ...': conta o número de reads presentes no arquivo atual.
# printf: imprime texto formatado; %-35s reserva espaço alinhado à esquerda,
# e %s imprime um valor como texto.
for arquivo in 1_demux/Lib1/*_R1.fastq; do
  [ -f "$arquivo" ] || continue  # Ignora o padrão se nenhum arquivo corresponder.
  n=$(awk 'END {print NR / 4}' "$arquivo")
  printf '%-35s %s pares\n' "$(basename "$arquivo")" "$n"
done
```

### 7.3. Conferindo a sincronia numérica entre R1 e R2

```bash
# Para cada arquivo R1, localiza o arquivo R2 com o mesmo nome-base.
# ${r1%_R1.fastq}: remove o sufixo '_R1.fastq' do nome.
# $(awk ...): armazena o número de reads em n1 e n2.
# -ne: testa se dois valores numéricos são diferentes.
# O laço mostra 'OK' se as contagens coincidem e 'ATENÇÃO' caso contrário.
for r1 in 1_demux/Lib1/*_R1.fastq; do
  [ -f "$r1" ] || continue
  r2="${r1%_R1.fastq}_R2.fastq"
  n1=$(awk 'END {print NR / 4}' "$r1")
  n2=$(awk 'END {print NR / 4}' "$r2")

  if [ "$n1" -ne "$n2" ]; then
    echo "ATENÇÃO: $r1 → R1=$n1 / R2=$n2"
  else
    echo "OK: $(basename "$r1") → $n1 pares"
  fi
done
```

> **Importante:** contagens iguais são uma verificação necessária, mas **não garantem** que os cabeçalhos estejam na mesma ordem ou que os pares correspondam corretamente. O Cutadapt processa reads pareadas, mas a verificação cuidadosa da integridade dos arquivos continua importante.

### 7.4. Balanço de leituras: entrada × saída

```bash
# Conta o número de pares de entrada da Lib1 (cada R1 representa um par).
awk 'END {print "Entrada Lib1:", NR / 4, "pares"}' \
  0_rawsequences/Lib1_R1.fastq

# awk pode ler vários arquivos consecutivamente; NR acumula todas as linhas.
# O padrão *_R1.fastq inclui categorias esperadas, inesperadas e unknown.
# Assim contamos TODOS os pares enviados às saídas R1 da biblioteca.
awk 'END {print "Saída Lib1:", NR / 4, "pares"}' \
  1_demux/Lib1/*_R1.fastq
```

**Resultado esperado:** sem filtros de descarte adicionais e com o processamento concluído normalmente, as duas contagens devem coincidir. Se diferirem, investigue as mensagens do Cutadapt, os caminhos dos arquivos e os parâmetros usados.

```bash
# less: abre o relatório de execução para leitura com rolagem.
# Pressione q para sair do visualizador.
less relatorios/Lib1_cutadapt.log
```

### Exercício 2 — Análise da Lib1

1. Quantos pares de leituras entraram na demultiplexação?
2. Quantos foram classificados como `T4-T4`, `T9-T9` e `T11-T11`?
3. Quantos ficaram em categorias inesperadas ou com `unknown`?
4. O total de pares de saída corresponde ao total de entrada?
5. O que aconteceria com a análise ecológica se aceitássemos qualquer combinação de índices como uma amostra válida?

---

## 8. Demultiplexando Lib2, Lib3 e Lib4

A sintaxe é a mesma de Lib1. **Mudam os nomes e as sequências dos índices, os arquivos de entrada, o diretório de saída e o arquivo de relatório.**

### 8.1. Preparar os diretórios

```bash
# mkdir -p: cria as pastas que receberão os resultados das três bibliotecas.
mkdir -p 1_demux/Lib2 1_demux/Lib3 1_demux/Lib4
```

### 8.2. Lib2

```bash
# Mesmos parâmetros da Lib1:
# -e 0 e --no-indels: correspondência exata de índices.
# -g / -G: procura índices de Lib2 em R1 / R2, respectivamente.
# -o / -p: separa as saídas pareadas em 1_demux/Lib2/.
# 2>&1 | tee: exibe o relatório e também o salva em relatorios/.
cutadapt \
  -e 0 --no-indels \
  -g 'T6=^AACACC' -g 'T10=^AATCGC' -g 'T16=^GAGTCA' \
  -G 'T6=^AACACC' -G 'T10=^AATCGC' -G 'T16=^GAGTCA' \
  -o '1_demux/Lib2/{name1}-{name2}_R1.fastq' \
  -p '1_demux/Lib2/{name1}-{name2}_R2.fastq' \
  0_rawsequences/Lib2_R1.fastq 0_rawsequences/Lib2_R2.fastq \
  2>&1 | tee relatorios/Lib2_cutadapt.log
```

### 8.3. Lib3

```bash
# -g / -G: agora utilizam os índices T7, T12 e T14 da Lib3.
# -o / -p: enviam os arquivos apenas para a pasta 1_demux/Lib3/.
# O arquivo de relatório registra exclusivamente o processamento de Lib3.
cutadapt \
  -e 0 --no-indels \
  -g 'T7=^AAGAGG' -g 'T12=^AATCGC' -g 'T14=^AGCTAC' \
  -G 'T7=^AAGAGG' -G 'T12=^AATCGC' -G 'T14=^AGCTAC' \
  -o '1_demux/Lib3/{name1}-{name2}_R1.fastq' \
  -p '1_demux/Lib3/{name1}-{name2}_R2.fastq' \
  0_rawsequences/Lib3_R1.fastq 0_rawsequences/Lib3_R2.fastq \
  2>&1 | tee relatorios/Lib3_cutadapt.log
```

### 8.4. Lib4

```bash
# -g / -G: utilizam os índices T8, T13 e T15 da Lib4.
# -o / -p: salvam os pares somente em 1_demux/Lib4/.
# O relatório de execução ficará em relatorios/Lib4_cutadapt.log.
cutadapt \
  -e 0 --no-indels \
  -g 'T8=^AACACC' -g 'T13=^AAGCCT' -g 'T15=^AGCTAC' \
  -G 'T8=^AACACC' -G 'T13=^AAGCCT' -G 'T15=^AGCTAC' \
  -o '1_demux/Lib4/{name1}-{name2}_R1.fastq' \
  -p '1_demux/Lib4/{name1}-{name2}_R2.fastq' \
  0_rawsequences/Lib4_R1.fastq 0_rawsequences/Lib4_R2.fastq \
  2>&1 | tee relatorios/Lib4_cutadapt.log
```

### 8.5. Resumo das contagens para todas as bibliotecas

```bash
# for: repete a análise para Lib1, Lib2, Lib3 e Lib4.
# echo: identifica no terminal qual biblioteca está sendo analisada.
# awk: conta todos os pares dos R1 de entrada e de saída.
# A expressão "*_R1.fastq" inclui todas as categorias demultiplexadas.
for lib in Lib1 Lib2 Lib3 Lib4; do
  echo "===== $lib ====="
  awk 'END {print "Pares de entrada:", NR / 4}' "0_rawsequences/${lib}_R1.fastq"
  awk 'END {print "Pares de saída:", NR / 4}' 1_demux/"$lib"/*_R1.fastq
done
```

### Exercício 3 — Comparando bibliotecas

Preencha a tabela a partir dos resultados reais obtidos no terminal:

| Biblioteca | Pares de entrada | Pares esperados (índices iguais) | Pares inesperados / unknown | Pares totais de saída |
|---|---:|---:|---:|---:|
| Lib1 | | | | |
| Lib2 | | | | |
| Lib3 | | | | |
| Lib4 | | | | |

**Discussão:** duas bibliotecas com o mesmo número de amostras precisam ter o mesmo número de leituras? A diferença de abundância de leituras entre amostras corresponde diretamente à diferença de abundância dos organismos no ambiente?

---

## 9. Selecionando as amostras válidas sem apagar resultados

Após inspecionar os resultados, podemos criar uma pasta com **cópias** dos pares cujos nomes de índice correspondem às combinações esperadas no desenho desta prática. **Os arquivos de categorias inesperadas e `unknown` continuarão preservados em `1_demux/`.**

O bloco a seguir é opcional e apresenta alguns recursos de programação em Bash. Ele deve ser executado **somente após conferir os índices e confirmar que o desenho experimental realmente exige índices iguais em R1 e R2**.

```bash
# for lib ...: percorre as quatro bibliotecas de sequenciamento.
for lib in Lib1 Lib2 Lib3 Lib4; do

  # mkdir -p: cria uma pasta independente para cada biblioteca.
  mkdir -p "1_demux_validas/$lib"

  # for r1 ...: percorre todos os arquivos *_R1.fastq dessa biblioteca.
  for r1 in 1_demux/"$lib"/*_R1.fastq; do

    # [ -f ... ]: testa se o arquivo existe; || continue ignora padrões vazios.
    [ -f "$r1" ] || continue

    # ${r1%_R1.fastq}: remove o sufixo e preserva o nome-base do par.
    prefixo="${r1%_R1.fastq}"

    # ${prefixo##*/}: remove tudo até a última barra do caminho.
    # Exemplo: '1_demux/Lib1/T4-T4' torna-se 'T4-T4'.
    nomes="${prefixo##*/}"

    # ${nomes%%-*}: captura o texto antes do primeiro hífen (índice de R1).
    # ${nomes#*-}: captura o texto após o primeiro hífen (índice de R2).
    indice_r1="${nomes%%-*}"
    indice_r2="${nomes#*-}"

    # [ "a" = "b" ]: testa igualdade de texto entre os dois índices.
    # &&: exige que as duas condições sejam verdadeiras.
    # !=: exclui a categoria 'unknown' da seleção.
    if [ "$indice_r1" = "$indice_r2" ] && [ "$indice_r1" != "unknown" ]; then

      # cp: COPIA R1 e R2 para a pasta das amostras esperadas.
      # A origem permanece preservada; não usamos rm nesta prática.
      cp "$r1" "1_demux_validas/$lib/"
      cp "${prefixo}_R2.fastq" "1_demux_validas/$lib/"
    fi
  done
done

# ls -R: lista arquivos recursivamente em todas as subpastas.
ls -R 1_demux_validas/
```

**Por que não usar `rm`?** A exclusão automática de combinações inesperadas pode eliminar evidências úteis para detectar contaminações, problemas de indexação ou outros eventos que precisam ser investigados.

> **Cuidado em aplicações reais:** o procedimento acima é específico deste desenho didático, em que somente pares com o mesmo identificador são previstos. Em experimentos com *dual indexing* combinatorial, índices diferentes entre R1 e R2 podem ser combinações **perfeitamente válidas**; use sempre a tabela de amostras do experimento para selecionar quais arquivos manter.

---

## 10. Desafio final — Demultiplexação e interpretação ecológica

Responda em grupos e justifique com os resultados obtidos:

1. Qual biblioteca apresentou maior proporção de pares classificados nas combinações esperadas?
2. Qual apresentou maior proporção de pares com índice `unknown` em pelo menos uma extremidade?
3. O que pode explicar um índice ser reconhecido em R1, mas não em R2?
4. Por que não é adequado atribuir todas as combinações inesperadas a um único tipo de erro?
5. O que a demultiplexação removeu das reads? O que ainda precisa ser removido antes da análise biológica?
6. Se uma amostra costeira tiver poucas reads após a demultiplexação, podemos concluir que ela possui menor biodiversidade? Que outras explicações precisam ser consideradas?
7. Quais informações devem ser registradas em um relatório para permitir que outro pesquisador reproduza esta etapa?

### Entrega sugerida

Um pequeno relatório contendo:

- versão do Cutadapt utilizada;
- tabela dos pares de leitura por biblioteca;
- número e proporção de pares nas combinações esperadas;
- número e proporção de pares inesperados/`unknown`;
- confirmação do balanço de pares de entrada e de saída;
- uma interpretação de 5 a 10 linhas relacionando os resultados à confiabilidade de estudos de eDNA.

### Checklist antes de avançar

- [ ] Sei distinguir amostra biológica, biblioteca, índice e read.
- [ ] Confirmei que a demultiplexação é necessária para estes arquivos.
- [ ] Consultei o arquivo `indices.txt` e registrei os índices de cada biblioteca.
- [ ] Demultiplexei Lib1 a Lib4 usando os índices corretos.
- [ ] Preservei os arquivos originais, os arquivos `unknown` e os relatórios.
- [ ] Comparei a quantidade de pares de entrada com a de saída.
- [ ] Conferi os pares R1/R2 e identifiquei as combinações esperadas.
- [ ] Estou preparado(a) para investigar a qualidade dos FASTQ na próxima aula.

---

## 11. Conexão com as próximas aulas

Nesta aula, a demultiplexação **separou os dados por amostra e removeu apenas os índices encontrados**. Os arquivos ainda podem conter sequências de primers, bases de qualidade insuficiente e outras características que exigem processamento.

Na sequência da disciplina, veremos como **avaliar a qualidade dos FASTQ com o FastQC**, interpretar seus módulos no contexto de metabarcoding e preparar os dados para as etapas de remoção de primers, fusão de pares e análises de diversidade.

## 12. Referências e materiais de apoio

- [Cutadapt — documentação oficial](https://cutadapt.readthedocs.io/en/stable/guide.html)
- [Cutadapt — parâmetros da linha de comando](https://cutadapt.readthedocs.io/en/stable/reference.html)
- [Repositório da disciplina eDNA (dados e roteiros)](https://github.com/saviscos/eDNA)
- [Formato FASTQ — visão geral](https://en.wikipedia.org/wiki/FASTQ_format)

> **Nota ao docente:** os nomes e índices das quatro bibliotecas seguem os roteiros de exemplo publicados no repositório da disciplina. Antes da aplicação com uma nova turma ou um novo conjunto de dados, confira os FASTQ e a tabela `indices.txt` efetivamente distribuídos. As contagens de reads foram deixadas como resultados a serem obtidos pelos próprios alunos; não foram inventados números de saída.
