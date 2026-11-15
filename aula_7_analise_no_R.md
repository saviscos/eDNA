# Aula 7: Da Limpeza Final à Análise Ecológica no R (Phyloseq)

Após rodarmos o SINTAX, nossa tabela de taxonomia traz as classificações com porcentagens de confiança (ex: 0.99) e anotações de tamanho (`size=`). Embora isso seja ótimo para a nossa conferência visual, os softwares estatísticos (como o R) precisam de uma matriz limpa, contendo apenas o nome da ASV e a sua taxonomia final.

Nesta aula, faremos a ponte entre o terminal Linux e o ambiente R, culminando na geração dos gráficos ecológicos.

## 1. Preparando a Tabela no Terminal Linux

Antes de sair do terminal, vamos extrair apenas as informações essenciais do arquivo gerado pelo SINTAX. O comando abaixo pega a primeira coluna (Nome da ASV) e a quarta coluna (Taxonomia filtrada pelo seu *cutoff* de 0.5), e remove o texto `;size=...`.

```bash
# Limpando a anotação de 'size' e extraindo a taxonomia final
awk -F'\t' '{print $1 "\t" $4}' taxonomia_ASVs_05.txt | sed 's/;size=[0-9]*//g' > taxonomia_limpa_ASVs_05.txt
```

Inspecione o arquivo gerado com `more taxonomia_limpa_ASVs_05.txt`. 
Com este arquivo limpo e a matriz de abundância da Aula 5, estamos prontos para ir para o R!

*(Lembrete de exportação: Use `explorer.exe .` no terminal do Ubuntu (WSL) para abrir a pasta no Windows e copiar os arquivos para o seu RStudio).*

---

## 2. A "Santíssima Trindade" do Phyloseq (Ambiente R)

Abra o seu RStudio. Para realizar análises de ecologia microbiana e eDNA, o pacote `phyloseq` exige a união de três blocos de dados:
1. **Matriz de Abundância:** Quantas leituras cada ASV teve por amostra.
2. **Matriz de Taxonomia:** A identidade de cada ASV.
3. **Metadados:** Informações ambientais das amostras.

Como estamos em um ambiente didático, criaremos metadados fictícios com base exata nos nomes das amostras presentes no seu arquivo.

```r
# Carregando os pacotes necessários
library(phyloseq)
library(tidyverse)
library(ggplot2)

# ==========================================
# PASSO A: Importando a Abundância
# ==========================================
# Lendo a matriz gerada pelo VSEARCH. 
# NOTA: comment.char = "" impede que o R ignore a primeira linha por causa do "#" em "#OTU ID".
abundancia <- read.table("matriz_abundancia_ASVs.txt", header=TRUE, row.names=1, sep="\t", check.names=FALSE, comment.char="")

# ==========================================
# PASSO B: Importando e Limpando a Taxonomia
# ==========================================
tax_bruta <- read.table("taxonomia_limpa_ASVs_05.txt", header=FALSE, row.names=1, sep="\t", stringsAsFactors=FALSE)

# O SINTAX agrupa a taxonomia em uma única string separada por vírgulas.
# O código abaixo divide essa string em colunas e limpa os prefixos (k:, p:) e IDs numéricos do Midori.
tax_matriz <- tax_bruta %>%
  separate(V2, into = c("Reino", "Filo", "Classe", "Ordem", "Familia", "Genero", "Especie"), sep = ",", fill = "right") %>%
  mutate(across(everything(), ~gsub("^[kpcofgs]:", "", .x))) %>% 
  mutate(across(everything(), ~gsub("_[0-9]+", "", .x))) %>%     
  as.matrix()

rownames(tax_matriz) <- rownames(tax_bruta)

# ==========================================
# PASSO C: Criando Metadados Alinhados
# ==========================================
# Definindo as 12 amostras exatas que existem na sua matriz gerada (em ordem)
amostras_reais <- c("T10", "T11", "T12", "T13", "T14", "T15", "T16", "T4", "T6", "T7", "T8", "T9")

# Criando a tabela de metadados fictícios para a aula (divisão didática para gráficos)
metadados <- data.frame(
  Amostra = amostras_reais,
  Ambiente = c(rep("Agua_Doce", 6), rep("Estuario", 6)), 
  Impacto = rep(c("Preservado", "Poluido"), times = 6)
)

# O R exige que os nomes das linhas sejam exatamente os nomes das amostras
rownames(metadados) <- metadados$Amostra

# ==========================================
# PASSO D: Construindo o Objeto Phyloseq
# ==========================================
OTU <- otu_table(abundancia, taxa_are_rows = TRUE)
TAX <- tax_table(tax_matriz)
META <- sample_data(metadados)

# Fundindo tudo em um único objeto de estudo
ps <- phyloseq(OTU, TAX, META)

# Removendo possíveis ASVs que não tiveram classificação no Reino (ruídos)
ps <- subset_taxa(ps, !is.na(Reino))
print(ps)
```

---

## 3. Visualização Taxonômica (Gráfico de Barras)

Plotar todas as ASVs brutas gera um gráfico ilegível. A melhor prática ecológica é aglomerar os dados em um nível taxonômico mais alto (como Ordem ou Classe) e transformá-los em abundância relativa (porcentagem).

```r
# Aglomerando os dados pelo nível de Ordem
ps_ordem <- tax_glom(ps, taxrank = "Ordem", NArm = FALSE)

# Convertendo valores absolutos para abundância relativa (0 a 1)
ps_relativo <- transform_sample_counts(ps_ordem, function(x) x / sum(x))

# Gerando o gráfico com ggplot2 embutido
plot_bar(ps_relativo, x = "Amostra", fill = "Ordem") +
  geom_bar(stat = "identity", color = "black", linewidth = 0.1) +
  facet_wrap(~Ambiente, scales = "free_x") +
  theme_minimal() +
  labs(title = "Composição Taxonômica (Nível: Ordem)", y = "Abundância Relativa") +
  theme(legend.position = "right", legend.text = element_text(size = 8))
```

---

## 4. Diversidade Alfa (Riqueza Local)

A diversidade alfa avalia a biodiversidade *dentro* de uma mesma amostra ou condição. O índice *Observed* conta a riqueza bruta (quantas ASVs existem), enquanto o *Shannon* balanceia a riqueza com a equitabilidade (distribuição de abundância).

```r
# O phyloseq já calcula múltiplos índices de diversidade alfa com uma única função
plot_richness(ps, x = "Impacto", measures = c("Observed", "Shannon")) +
  geom_boxplot(aes(fill = Impacto), alpha = 0.7, outlier.shape = NA) +
  geom_jitter(width = 0.2, size = 2, alpha = 0.5) +
  theme_bw() +
  labs(title = "Diversidade Alfa: Riqueza e Índice de Shannon", x = "Condição Ambiental", y = "Valor do Índice") +
  scale_fill_manual(values = c("Preservado" = "#2ca25f", "Poluido" = "#de2d26"))
```

---

## 5. Diversidade Beta (Estrutura da Comunidade)

A diversidade beta avalia a dissimilaridade *entre* as amostras. Estamos comparando se a estrutura da comunidade de insetos/artrópodes do ambiente preservado é estatisticamente diferente da do ambiente poluído, utilizando a distância de Bray-Curtis e ordenação em PCoA.

```r
# Calculando a matriz de distância e extraindo as coordenadas (PCoA)
ord_pcoa <- ordinate(ps, method = "PCoA", distance = "bray")

# Plotando os eixos principais
plot_ordination(ps, ord_pcoa, color = "Impacto", shape = "Ambiente") +
  geom_point(size = 4, alpha = 0.8) +
  stat_ellipse(aes(group = Impacto), linetype = 2) + 
  theme_classic() +
  labs(title = "Diversidade Beta: PCoA (Distância de Bray-Curtis)", 
       color = "Status de Impacto", 
       shape = "Tipo de Ambiente") +
  scale_color_manual(values = c("Preservado" = "#2ca25f", "Poluido" = "#de2d26"))
```