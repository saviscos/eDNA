## 7. Refinamento de ASVs: Denoising, Tradução e Remoção de Quimeras

### Introdução
Após a desreplicação, reduzimos drasticamente o volume dos nossos dados, mas as sequências únicas (ASVs candidatas) ainda contêm erros biológicos e de sequenciamento mais complexos. Nesta etapa final de processamento de leituras, aplicaremos três filtros avançados em sequência para garantir que apenas as variantes biológicas reais permaneçam no nosso conjunto de dados.

---

### 7.1. Denoising (Algoritmo UNOISE3)
A desreplicação removeu os *singletons*, mas o que fazemos com sequências que aparecem 2, 3 ou 4 vezes e são extremamente parecidas com uma sequência que aparece 10.000 vezes? Provavelmente, as variantes raras são pequenos erros de PCR da variante super abundante.
O **Denoising** (ou redução de ruído) utiliza a frequência das leituras e a composição da sequência para inferir esses erros. Ele compara ASVs raras com abundantes e descarta as que são derivadas de erros sistemáticos.

```bash
# Executa o denoising usando o algoritmo UNOISE3 implementado no VSEARCH
vsearch --cluster_unoise all.derep.fasta \
    --minsize 4 \
    --unoise_alpha 2 \
    --centroids all.denoised.fasta
```
**Entendendo os parâmetros:**
* `--cluster_unoise`: Chama o algoritmo de denoising.
* `--minsize 4`: Descarta ASVs com abundância menor que 4 (um filtro um pouco mais conservador para nosso conjunto de dados de exemplo).
* `--unoise_alpha 2`: Controla a rigidez (dissimilaridade) na comparação entre sequências raras e abundantes.
* `--centroids`: O arquivo de saída contendo as ASVs "verdadeiras" (centróides dos clusters).

---

### 7.2. Filtragem de Erros Pontuais (Tradução)
Marcadores codificadores de proteínas (como o COI) nos dão uma vantagem incrível: eles precisam fazer sentido biológico. Mutações induzidas por PCR ou pseudogenes nucleares (NUMTs) frequentemente causam alterações de leitura (*frameshifts*) ou introduzem códons de parada (*stop codons*).
Podemos traduzir virtualmente nosso DNA em aminoácidos e descartar qualquer sequência que gere uma proteína truncada.

Vamos usar o script `filtertranslate` do pacote *metaMATE*:

```bash
# Traduz as sequências e separa as válidas das inválidas
filtertranslate -i all.denoised.fasta \
    -t 5 \
    -y separate \
    -o all.translated
```
**Entendendo os parâmetros:**
* `-t 5`: Define a tabela de código genético. O número **5** corresponde ao código mitocondrial de invertebrados (ótimo para estudos de zooplâncton/artrópodes marinhos).
* `-y separate`: Instrui o programa a criar dois arquivos distintos: um com as sequências aprovadas (`pass`) e outro com as reprovadas (`fail`).
* `-o`: Prefixo dos arquivos de saída (ex: gerará `all.translated_pass.fasta`).

---

### 7.3. Remoção de Quimeras (*Chimera Removal*)
O último grande artefato que precisamos remover são as **quimeras**. Durante a PCR, a polimerase pode interromper a cópia de uma sequência (A) e, no ciclo seguinte, esse fragmento incompleto age como *primer* em uma sequência diferente (B). O resultado é uma sequência híbrida (A+B) que não existe na natureza.
Como as quimeras são formadas por duas metades de sequências reais, elas têm qualidade perfeita e fazem sentido biológico, escapando dos filtros anteriores.

Vamos usar o algoritmo UCHIME de forma *de novo* (sem precisar de um banco de dados externo), partindo da premissa de que as sequências "pais" são mais abundantes que a quimera gerada.

```bash
# Detecta e remove quimeras das sequências que passaram na tradução
vsearch --uchime_denovo all.translated_pass.fasta \
    --sizein \
    --sizeout \
    --fasta_width 0 \
    --nonchimeras asvs_finais.fasta
```
**Entendendo os parâmetros:**
* `--uchime_denovo`: Procura por quimeras cruzando as sequências do próprio dataset.
* `--sizein / --sizeout`: Lê a abundância (`;size=N;`) gerada lá na desreplicação, pois o algoritmo UCHIME precisa saber quem é abundante (potencial 'pai') e quem é raro (potencial 'quimera').
* `--nonchimeras`: Salva apenas as sequências limpas no arquivo `asvs_finais.fasta`.

### Conclusão e Próximos Passos
Parabéns! O arquivo `asvs_finais.fasta` contém as suas **ASVs definitivas**. O lixo de sequenciamento, os artefatos de tamanho, pseudogenes, quimeras e ruídos foram eliminados. O próximo passo da nossa disciplina de bioinformática oceanográfica será cruzar as leituras iniciais contra esse arquivo para gerar a nossa **Tabela de ASVs (Matriz de Abundância)** e iniciar as análises de ecologia marinha!