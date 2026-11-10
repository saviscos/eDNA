### 3. Remoção de Quimeras (Limpando Artefatos do Laboratório)

Agora que temos nossas sequências globais desreplicadas e com a anotação de abundância (`size=`), precisamos lidar com um dos maiores "fantasmas" do sequenciamento de DNA: as **Quimeras**.

#### O que é uma Quimera de DNA?
Durante o processo de PCR no laboratório, a enzima (Taq Polimerase) pode ser interrompida no meio da cópia de um fragmento de DNA. Na próxima rodada do PCR, esse pedaço incompleto pode grudar em um DNA de outra espécie e terminar de ser copiado. 

O resultado? Uma sequência "híbrida" ou "frankenstein" (metade da Espécie A e metade da Espécie B). Ela não existe na natureza, é apenas um erro do PCR. Se não a removermos, nosso programa achará que descobrimos uma espécie nova!

#### Como o VSEARCH identifica uma Quimera?
Nós usaremos o algoritmo `uchime_denovo`. Ele é extremamente inteligente: ele usa a anotação de abundância (`size=`) que acabamos de criar. Ele parte do princípio de que as sequências biológicas reais (os "pais") são muito abundantes na amostra, enquanto as quimeras (os "filhos") são artefatos raros. Se ele encontrar uma sequência rara que é exatamente a metade de uma sequência muito comum misturada com outra muito comum, ele a descarta como quimera.

No seu terminal, execute o comando abaixo (certifique-se de estar na pasta `resultados_vsearch`):

```bash
vsearch --uchime_denovo sequencias_unicas_globais.fasta \
        --nonchimeras sequencias_sem_quimeras.fasta \
        --sizein \
        --sizeout \
        --fasta_width 0
```

#### Entendendo os Parâmetros:
* `--uchime_denovo`: Aciona o algoritmo de busca de quimeras cruzando os dados da própria amostra (sem precisar de um banco de dados externo).
* `--nonchimeras`: Define o nome do arquivo de saída onde serão salvas **apenas as sequências reais** (que passaram no teste anti-quimera).
* `--sizein`: Avisa ao programa para ler a informação `size=X` no cabeçalho das sequências. (Sem isso, o algoritmo não consegue diferenciar quem é o "pai" e quem é o "filho").
* `--sizeout`: Garante que a anotação de abundância seja mantida no arquivo final.
* `--fasta_width 0`: Impede que o programa quebre a sequência de DNA em várias linhas no arquivo de texto, facilitando a leitura.

**Atividade Prática:**
Descubra quantas quimeras foram jogadas fora! Use o comando `grep -c "^>" sequencias_unicas_globais.fasta` para contar as sequências antes do filtro, e depois rode `grep -c "^>" sequencias_sem_quimeras.fasta` para contar as sobreviventes. A diferença entre os dois números é a quantidade de artefatos de PCR que acabamos de eliminar.