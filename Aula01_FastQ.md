# Fundamentos: Navegação no Linux e Arquivos FASTQ

Este documento serve como um guia introdutório para navegar no terminal Linux e explorar arquivos de sequenciamento no formato FASTQ. 

---

## 1. O Básico: Navegando pelo Terminal Linux

Antes de manipularmos dados biológicos, é fundamental saber se mover pelo terminal. Aqui estão os comandos mais importantes:

*   **`pwd` (Print Working Directory):** Mostra o caminho completo do diretório (pasta) onde você está no momento.
    ```bash
    pwd
    ```
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

![FastQ](https://static.wixstatic.com/media/3165a3_219cd2be42a64a589a678ce3295a543e~mv2.jpg/v1/fill/w_438,h_246,al_c,q_80,usm_0.66_1.00_0.01,enc_avif,quality_auto/3165a3_219cd2be42a64a589a678ce3295a543e~mv2.jpg)

![Sequenciamento](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTgXIq1F91DhsIDuKlvsGQ8P7PZIooRqtO2uK9Whj2rdj3_9Po868SEi5c&s=10)



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
