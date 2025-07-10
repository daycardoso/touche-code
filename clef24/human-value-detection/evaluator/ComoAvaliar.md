# Como Utilizar o Avaliador de Detecção de Valores Humanos (CLEF 2024)

Este documento detalha o funcionamento e o uso do script de avaliação para a tarefa de Detecção de Valores Humanos do CLEF 2024.

## 1. Visão Geral

O avaliador foi projetado para medir o desempenho de modelos na identificação de valores humanos em textos. Ele compara os resultados gerados por um modelo (o "run") com um gabarito ("ground truth") e calcula métricas de performance para duas subtarefas:

*   **Subtarefa 1:** Classificação binária para cada valor, indicando se o valor está presente na sentença.
*   **Subtarefa 2:** Classificação para sentenças onde um valor foi identificado, determinando se o valor é "alcançado" (attained) ou "restringido" (constrained).

## 2. Estrutura de Diretórios e Arquivos

Para utilizar o avaliador, seus dados devem seguir uma estrutura de diretórios específica. O avaliador espera dois diretórios principais como entrada: um contendo o dataset (gabarito) e outro contendo a run (predições do modelo).

```
/seu/diretorio/
├───dataset/
│   ├───labels.tsv
│   └───sentences.tsv
└───run/
    └───run.tsv
```

-   **Diretório do Dataset (`--inputDataset`):** Deve conter os arquivos de gabarito.
    -   `labels.tsv`: Contém os rótulos de verdade para cada sentença.
    -   `sentences.tsv`: Contém o texto das sentenças.
-   **Diretório da Run (`--inputRun`):** Deve conter o arquivo de predição do seu modelo.
    -   `run.tsv`: Arquivo com as predições do seu modelo no mesmo formato do `labels.tsv`.

## 3. Formato dos Arquivos

Todos os arquivos de dados devem ser no formato TSV (valores separados por tabulação) e codificados em UTF-8.

### 3.1. `sentences.tsv`

Este arquivo contém os textos e seus identificadores.

| Text-ID | Sentence-ID | Text |
| :--- | :--- | :--- |
| text_01 | sent_01 | Exemplo de uma sentença sobre valores. |
| text_01 | sent_02 | Outra sentença do mesmo texto. |

### 3.2. `labels.tsv` (Gabarito) e `run.tsv` (Predição)

Estes arquivos contêm os valores de confiança para cada um dos 19 valores humanos, tanto para a dimensão "attained" quanto "constrained".

As colunas de identificação são `Text-ID` e `Sentence-ID`. As demais colunas representam os valores humanos e suas dimensões. O formato do nome da coluna é `NomeDoValor <dimensão>`, por exemplo, `Self-direction: thought attained`.

**Exemplo de colunas:**

| Text-ID | Sentence-ID | Self-direction: thought attained | Self-direction: thought constrained | ... | Universalism: tolerance attained | Universalism: tolerance constrained |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| text_01 | sent_01 | 0.8 | 0.1 | ... | 0.0 | 0.0 |
| text_01 | sent_02 | 0.0 | 0.0 | ... | 0.9 | 0.7 |

-   Os valores de confiança devem estar no intervalo de `[0.0, 1.0]`.
-   O script `evaluator.py` processa esses valores para calcular as métricas. Para a **Subtarefa 1**, a confiança de um valor é a soma das confianças `attained` e `constrained`. Um rótulo binário é atribuído (1 se a soma for >= 0.5, caso contrário 0).
-   Para a **Subtarefa 2**, a avaliação só considera as sentenças onde a Subtarefa 1 foi positiva (soma >= 0.5). A confiança para `attained` e `constrained` é normalizada.

## 4. Executando o Avaliador

A maneira recomendada para executar o avaliador é via Docker, o que garante que todas as dependências estejam corretamente instaladas.

**Comando de Execução:**
```bash
# build
docker build -f Dockerfile_predict -t valueeval24-arthur-schopenhauer-ensemble:1.0.0 .

# run
docker run --rm -v //c/Users/dayan/OneDrive/Documents/PLN/touche-code/valueeval24/test:/dataset -v //c/Users/dayan/OneDrive/Documents/PLN/touche-code/valueeval24/output:/output valueeval24-arthur-schopenhauer-ensemble:1.0.0

# view results
cat output/run.tsv
```
```bash
docker run --rm -v //c/Users/dayan/OneDrive/Documents/PLN/touche-code/valueeval24/test:/labels -v //c/Users/dayan/OneDrive/Documents/PLN/touche-code/valueeval24/run:/run -v //c/Users/dayan/OneDrive/Documents/PLN/touche-code/valueeval24/output:/output webis/touche-human-value-detection-evaluator:1.0.2 --inputDataset /labels --inputRun /run --outputDataset /output
```

```bash
docker run --rm -v //c/Users/dayan/OneDrive/Documents/PLN/touche-code/valueeval24/test-english:/labels -v //c/Users/dayan/OneDrive/Documents/PLN/touche-code/valueeval24/run:/run -v //c/Users/dayan/OneDrive/Documents/PLN/touche-code/valueeval24/output:/output webis/touche-human-value-detection-evaluator:1.0.2 --inputDataset /labels --inputRun /run --outputDataset /output
```

```bash 
bash valueeval24/output/prototext-conversion/proto2tsv.sh dayana valueeval24/test-english/labels.tsv HoA 1 valueeval24/output/evaluation.prototext > output.tsv

```


-   **`/caminho/para/seu/dataset`**: O caminho absoluto para o diretório que contém `labels.tsv` e `sentences.tsv`.
-   **`/caminho/para/sua/run`**: O caminho absoluto para o diretório que contém seu arquivo `run.tsv`.
-   **`/caminho/para/saida`**: O caminho absoluto para um diretório onde os resultados da avaliação serão salvos.

## 5. Métricas de Avaliação

O script calcula as seguintes métricas, baseadas em `scikit-learn`:

-   **Precision, Recall e F1-Score.**

As métricas são agregadas de maneiras diferentes para cada subtarefa:

-   **Subtarefa 1:** As métricas são calculadas para cada um dos 19 valores e então agregadas usando **Macro Average**. Isso significa que cada valor tem o mesmo peso no resultado final, independentemente da sua frequência no dataset.
-   **Subtarefa 2 (attained/constrained):** As métricas também são calculadas para cada valor e agregadas com **Macro Average**.
-   **Subtarefa 2 (overall):** As métricas são calculadas usando **Micro Average** sobre as predições de `attained` e `constrained`. Isso é útil para uma visão geral do desempenho na subtarefa 2, considerando todas as decisões.

## 6. Saída da Avaliação

O avaliador gera dois arquivos no diretório de saída (`/output`):

1.  **`evaluation.prototext`**: Um arquivo de texto com as métricas detalhadas para cada valor e as médias gerais. Este formato é usado pela plataforma TIRA.
    -   **Exemplo de conteúdo:**
        ```
        measure {
          key: "F1 (Subtask 1)"
          value: "0.85"
        }
        measure {
          key: "Precision Self-direction: thought"
          value: "0.92"
        }
        ```

2.  **`index.html`**: Uma página web interativa com a visualização dos resultados. Ela inclui:
    -   Uma tabela de resumo com as principais métricas.
    -   Curvas ROC para cada valor e subtarefa.
    -   Gráficos de linha comparando F1, Precision e Recall entre os valores.
    -   Uma tabela detalhada para análise de erros, mostrando a diferença (delta) entre a predição e o gabarito para cada sentença.

```