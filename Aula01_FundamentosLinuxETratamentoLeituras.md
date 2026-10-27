# Fundamentos: Navegação no Linux e Arquivos FASTQ

Este documento serve como um guia introdutório para navegar no terminal Linux e explorar arquivos de sequenciamento no formato FASTQ. 

---

## 1. O Básico: Navegando pelo Terminal Linux

Antes de manipularmos dados biológicos, é fundamental saber se mover pelo terminal. Aqui estão os comandos mais importantes:

*   **`pwd` (Print Working Directory):** Mostra o caminho completo do diretório (pasta) onde você está no momento.
  *   **`ls` (List):** Lista os arquivos e pastas dentro do diretório atual.
    *   `ls` (lista simples)
    *   `ls -l` (lista com detalhes, como permissões e tamanho)
    *   `ls -lh` (lista com tamanhos de arquivo "human-readable" - ex: Kb, Mb, Gb)
*   **`cd` (Change Directory):** Usado para entrar e sair de pastas.
    *   `cd nome_da_pasta` (entra na pasta especificada)
    *   `cd ..` (volta um nível/pasta para trás)
    *   `cd ~` (volta para o seu diretório "home" ou principal)
*   **`cp` (Copy):** Copia arquivos ou diretórios. Para copiar diretórios inteiros, usamos o argumento `-r` (recursivo).

---

## 2. Fundamentos: Arquivos FASTQ

### Introdução

O formato FASTQ é o padrão absoluto para armazenar dados gerados por plataformas de sequenciamento. Diferente do formato FASTA, que guarda apenas a sequência biológica, o FASTQ armazena tanto a sequência de nucleotídeos quanto a qualidade de cada base lida pelo equipamento. Ele estrutura cada leitura em um bloco exato de quatro linhas:

1. **Identificador da Leitura (Header):** Sempre inicia com o caractere `@`. Contém informações exclusivas sobre o sequenciador, o número do fragmento e as coordenadas da leitura. Exemplo: `@SRR1234567.1 1/1`.
2. **Sequência de DNA (Bases):** A leitura literal dos nucleotídeos (A, C, T, G) obtida no sequenciamento.
3. **Divisor:** Um caractere `+` que separa a sequência de DNA das pontuações de qualidade. Opcionalmente, pode conter o identificador repetido.
4. **Qualidade Phred (Score):** Caracteres ASCII que representam a probabilidade de erro para cada base da sequência acima. Cada caractere traduz um valor de qualidade (ex: Q30, Q40) essencial para a filtragem bioinformática posterior.

![FastQ](https://static.wixstatic.com/media/3165a3_219cd2be42a64a589a678ce3295a543e~mv2.jpg/v1/fill/w_438,h_246,al_c,q_80,usm_0.66_1.00_0.01,enc_avif,quality_auto/3165a3_219cd2be42a64a589a678ce3295a543e~mv2.jpg)

### Estratégias de Leitura: Single-end vs. Paired-end

Dependendo da preparação da biblioteca e da configuração do equipamento, as sequências podem ser geradas de duas formas distintas:

* **Single-end (Leitura Simples):** O sequenciador lê o fragmento de DNA em apenas uma direção (sentido *forward*).
  * **Características:** É um processo mais rápido e barato.
  * **Aplicações:** Adequado para tarefas onde o alinhamento requer apenas fragmentos curtos, como quantificação de expressão gênica (RNA-Seq) ou varreduras rápidas. 

* **Paired-end (Leituras Pareadas):** O sequenciador lê ambos os lados do mesmo fragmento de DNA. Primeiro, ocorre a leitura a partir de uma extremidade (originando o arquivo *Forward* ou R1) e, em seguida, o fragmento é lido a partir da extremidade oposta (originando o arquivo *Reverse* ou R2).
  * **Características:** Aumenta significativamente a confiança e a resolução dos dados. As duas leituras pertencem ao mesmo fragmento e podem apresentar sobreposição na região central.
  * **Aplicações:** Fundamental para eDNA e metabarcoding. As leituras *forward* e *reverse* podem ser sobrepostas e fundidas (*merged*) no computador, corrigindo quedas de qualidade nas extremidades (típicas do sequenciamento) e permitindo cobrir sequências alvo maiores do que a capacidade isolada da plataforma (ex: sequenciar um fragmento de 400 pb utilizando ciclos de 2x250 pb).


### Plataformas e Tecnologias de Sequenciamento

As plataformas de sequenciamento variam amplamente em rendimento, tamanho das leituras e taxas de erro, sendo escolhidas com base nos objetivos do projeto:

* **Sequenciamento de Curtas Leituras (*Short-read* - ex: Illumina, Ion Torrent):** Produzem volumes massivos de dados (milhões a bilhões de leituras) com alta precisão, mas geram fragmentos curtos, variando tipicamente de 50 a 300 pares de bases (pb). São o padrão para estudos de eDNA e metabarcoding, onde marcadores curtos e alta profundidade de sequenciamento são necessários para detectar táxons raros em misturas complexas.
* **Sequenciamento de Longas Leituras (*Long-read* - ex: Oxford Nanopore, PacBio):** Capazes de gerar leituras ininterruptas de dezenas de milhares de bases. Embora historicamente tivessem taxas de erro bruto maiores, são ideais para sequenciamento de amplicons longos (ex: operon rRNA completo) ou montagem de genomas, resolvendo regiões repetitivas que as curtas leituras não alcançam.

![Sequenciamento](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTgXIq1F91DhsIDuKlvsGQ8P7PZIooRqtO2uK9Whj2rdj3_9Po868SEi5c&s=10)

### Prática - Preparação do Ambiente e Obtenção dos Dados

### Introdução
Antes de começarmos a limpar e processar as nossas sequências de eDNA, precisamos configurar o nosso diretório de trabalho e baixar os dados brutos (as leituras diretas do sequenciador) que utilizaremos ao longo de toda a disciplina.

### Baixando os Dados Brutos
Nós disponibilizamos um arquivo compactado contendo todas as sequências brutas necessárias para este curso. Vamos utilizar o comando `wget` para baixar esse arquivo diretamente do repositório no GitHub para o nosso servidor/computador.

Abra o seu terminal e execute os seguintes comandos:

```bash
# 1. Baixe o arquivo de sequências brutas (usando o link 'raw' do GitHub)
wget https://github.com/saviscos/eDNA/raw/main/0_rawsequences.zip

# 2. Descompacte o arquivo baixado
unzip 0_rawsequences.zip

# 3. (Opcional) Remova o arquivo .zip para economizar espaço
rm 0_rawsequences.zip
```

Execute o comando abaixo para entrar no diretório que contém os dados iniciais e listar seu conteúdo, mostrando os tamanhos dos arquivos:
```bash
cd 0_rawsequences/
ls -lh
```

Podemos ver quantas linhas existem em cada arquivo usando a função `wc` (word count), especificando com `-l` que queremos o número de linhas:
```bash
wc -l *.fastq
```
O `*.fastq` aqui significa que queremos aplicar o comando a **todos** os arquivos que terminam com a extensão `.fastq` no diretório. Podemos substituir isso por um nome de arquivo único se quisermos apenas contar as linhas de um arquivo específico. 

Substitua a palavra `arquivo.fastq` no comando a seguir pelo nome de um único arquivo FASTQ para inspecionar seu início:
```bash
head -n 10 arquivo.fastq
```
Você verá o formato FASTQ, que compreende: cabeçalho (header), sequência e escores de qualidade. Um ponto útil a se notar é a estrutura do cabeçalho do arquivo, especificamente que ele começa com `@D00`. Se a estrutura deste arquivo for completamente nova para você, tire alguns minutos para ler a primeira seção da [página do FASTQ na Wikipedia](https://en.wikipedia.org/wiki/FASTQ_format).

Para obter linhas específicas de um arquivo, use a função `sed`:
```bash
sed -n '4,8p' arquivo.fastq     # imprime apenas as linhas de 4 a 8
```
*Use isso para dar uma olhada em arquivos diferentes.*

**Nota sobre arquivos R1 e R2:** 
Observe que os arquivos R1 e R2 da mesma biblioteca possuem os mesmos cabeçalhos de *read*, diferindo apenas por um "1" ou "2" na segunda parte do nome. *Reads* com o mesmo cabeçalho foram lidas do mesmo local no sequenciador, portanto, assume-se que sejam a leitura *forward* (direta) e *reverse* (reversa) do mesmo fragmento de DNA - estes são chamados de **mate pairs** (pares correspondentes). 

É importante garantir que tanto a leitura *forward* quanto a *reverse* de cada fragmento estejam sempre presentes e no mesmo local relativo nos arquivos pareados ("em sincronia") para processos futuros na bioinformática.

Podemos usar a função `grep -c` para contar o número de sequências em um arquivo (novamente, substitua `arquivo.fastq` pelo nome de um dos arquivos):
```bash
grep -c "^@D00" arquivo.fastq
```
*(O `^` significa "começa com". Então estamos contando quantas linhas começam com `@D00`, o que equivale a contar o número de reads).*

Assim como a função `wc -l` acima, podemos executar o `grep` em todos os nossos arquivos de uma vez para obter o número total de *reads* para cada uma de nossas bibliotecas:
```bash
grep -c "^@D00" *.fastq
```
Podemos ver que estamos lidando com cerca de 10.000 a 12.000 *reads* por biblioteca neste exemplo.

---

### Exercício
- Todas as bibliotecas possuem os mesmos números de *reads* nos arquivos R1 e R2? 
*(Dica: Compare os resultados do comando `grep` anterior para os pares de cada biblioteca).*
