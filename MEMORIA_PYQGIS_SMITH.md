# MEMORIA_PYQGIS_SMITH.md

## 1. Escopo e Postura do Assistente IA (Smith)
*   **Responsabilidade:** O agente *Smith* atua dentro do QGIS auxiliando na operação de camadas, execução de plugins, auditorias e relatórios[cite: 5, 7]. A edição de código-fonte de plugins (ex: `modules/*.py`) é vetada neste escopo e deve ser tratada por agentes de desenvolvimento (ex: Claude Code)[cite: 5, 7].
*   **Intervenção:** Altere estruturas ou dados somente após diagnosticar a causa raiz, prever o que pode quebrar e obter autorização expressa do usuário, executando sempre a menor modificação possível sem refatorações não solicitadas[cite: 5, 7, 8].

## 2. Paradigmas e Bugs Críticos de PyQGIS
*   **Regra de Escopo do `exec()`:** A função nativa `executar_codigo_pyqgis` gerencia os dicionários globais e locais separadamente[cite: 30]. Essa arquitetura isola o escopo de `def`, *list comprehensions* e *generator expressions*, fazendo com que o `__globals__` dessas rotinas perca o acesso às variáveis/imports instanciados no nível local do script, resultando em `NameError`[cite: 30]. **Resolução:** Realize os `imports` necessários diretamente dentro das funções e evite aninhamento de comprehensions com variáveis externas[cite: 30].
*   **Ambiente Headless:** Execuções via terminal sem interface dependem das variáveis `QT_QPA_PLATFORM=offscreen`, da ativação do GDAL compatível via sistema, e do path referenciado diretamente ao QGIS (`sys.path.insert`)[cite: 8, 9].

## 3. Tratamento Geométrico DANI e Loteamentos
*   **Tratamento de Novo CAD:** Nunca processe shapes recém-convertidos sem apagar polígonos duplicados, aplicar Douglas-Peucker (1 mm) em eixos e transferir atributos[cite: 9]. Realize *costura topológica* (adicionando, mas nunca movendo, vértices) nas divisas para garantir resíduos < 1 cm[cite: 9]. Lados menores que 5 mm geram impressão de "0,00 m" e devem ser absorvidos no vizinho[cite: 8, 9].
*   **Regras DBF e Identificadores:** O formato restringe os nomes de campos a 10 caracteres (`QUADRA`, `LOTE`, `AREA`, `PERIMETRO`, `PROPRIETÁ`, `DECLIVIDAD`)[cite: 13]. Identificadores de `QUADRA`/`LOTE` duplicados em parcelas distintas causam colapso e sobrescrita nos memoriais[cite: 8, 9]. Toda parcela fundida deve ser convertida para polígono simples[cite: 8, 9].
*   **Desenvolvimento de Arcos vs Erros Virtuais:** Arcos devem ser calculados via `D = R × ângulo`, e não pela soma bruta de micro-cordas da vetorização[cite: 8, 9]. Consequentemente, o log de falso erro de fechamento geométrico (`validar_fecho()`) provocado por arredondamentos milimétricos nas curvas deve ser ignorado em produção[cite: 8, 9].
*   **Estilos no QGIS:** Propriedades visuais e fontes configuradas não sobrevivem no salvamento do shapefile cru; aplique obrigatoriamente um arquivo `.qml` associado, garantindo que "plantas de lote" utilizem formatação em *pontos* e "plantas gerais" em *unidades de mapa*[cite: 8, 9].

## 4. Contingência e Fluxo de Entrega Pragmática
*   **Pivot Tático:** Quando ferramentas nativas falharem, entrarem em loop ou não gerarem arquivos esperados devido a erro de subprocesso, a prioridade máxima é efetuar a entrega de engenharia[cite: 18]. Opere um pivot gerando a infraestrutura via scripts Python diretos e rotinas de deslocamento geométrico em Shapely, desde que as normas hidráulicas e normativas fiquem asseguradas[cite: 17, 18].
*   **Transparência:** Caso essas simulações manuais sejam necessárias, anexe um arquivo `LEIA-ME_LIMITACAO.txt` junto à pasta de saída explicando claramente o desvio de geração para a aprovação final[cite: 18, 29].

## 5. Correções e Adições (revisão do Claude Code, 19/09/2026)

Seção adicionada após revisão cruzada com o histórico real de desenvolvimento dos plugins (`CLAUDE.md` do repositório `terraplanagem-`). Não altera nada do que já estava escrito acima — só completa lacunas que já causaram erro real em produção.

*   **Ambiente Headless — faltava metade da receita.** Além de `QT_QPA_PLATFORM=offscreen`, é preciso `export XDG_RUNTIME_DIR=/tmp/runtime-root` (sem isso o Qt reclama do runtime dir). E o `sys.path.insert` tem 3 linhas específicas, nesta ordem, antes de importar qualquer coisa do QGIS:
    ```python
    sys.path.insert(0, "/usr/lib/python3/dist-packages")
    sys.path.insert(0, "/usr/share/qgis/python")
    sys.path.insert(0, "/usr/share/qgis/python/plugins")
    ```
    Em contêiner novo sem venv pronto: descubra qual `python3.X` casa com o `.so` em `/usr/lib/python3/dist-packages/osgeo/` (o nome do arquivo diz a versão), crie um `venv --system-site-packages` com esse Python e instale `pyshp shapely ezdxf matplotlib` — escrever fora do venv falha por PEP 668.

*   **Estilos/plantas no QGIS — o `.qml` sozinho não garante nada; estes detalhes já quebraram planta em produção:**
    *   `labelsEnabled` é **atributo da tag raiz `<qgis>`**, não elemento filho. Escrito solto no fim do arquivo, o QGIS ignora silenciosamente: a camada carrega a expressão de rótulo inteira, mas não rotula nada.
    *   `scaleVisibility="1"` com `scaleMin=scaleMax=0` é uma faixa de escala de ZERO a ZERO — nenhum rótulo aparece, sem erro nenhum.
    *   Propriedades definidas por dados (o botão de expressão ao lado de um campo do estilo) apontam para `auxiliary_storage_*`, que só existe dentro do **projeto** QGIS. Um shapefile carregado cru não tem esse armazenamento: a expressão dá erro e a propriedade morre calada (foi assim que sumiu a rotação de um rótulo). A expressão que gira o rótulo para dentro do lote é `90 - main_angle($geometry)` — com `main_angle - 90` o texto sai espelhado.
    *   Tamanho de fonte em **MapUnit é em METRO**: 1,5 unidade de mapa dá 1,8 mm de papel a 1:850 (planta geral), mas 15 mm a 1:100 (planta de lote). Em planta de LOTE use fonte em **ponto**, nunca MapUnit.
    *   `layer.clone()` passado direto para `setLayers()` de uma exportação é coletado pelo garbage collector antes da exportação terminar, e o mapa volta silenciosamente para a camada original do projeto (com rótulo/contorno errados). Guarde a referência clonada numa lista de módulo até a exportação acabar.
    *   Em planta de **lote**, filtre a camada de cotas por `setSubsetString` pelo lote da vez. Sem isso, cada planta desenha as cotas e os `V1..Vn` de TODOS os lotes por cima — "arco não agrupado" muitas vezes é cota de vizinho aparecendo, não erro de agrupamento.
    *   No carimbo (arquivo `.qpt`), o rótulo do formato (ex. "A3") é texto estático no XML, com UUID fixo — mas depois do `loadFromTemplate` os itens do layout ganham UUID novo a cada instância, então `itemByUuid` não acha o item do modelo. Corrigir o carimbo (formato certo por produto: A1 planta geral, A4 planta de lote) exige editar o XML por lote, nunca assumir que o texto já vem certo.

*   **`__DANI_MODO_SILENCIOSO__` grava um aviso que se autodestrói na leitura.** Ele escreve "MODO SILENCIOSO: ... VERIFICAR antes de assinar" nas observações do lote — mas `extrair_observacoes()` trata qualquer linha contendo "VERIFICAR" como cabeçalho de seção e **descarta exatamente essa linha**. Se for auditar observações geradas em modo silencioso, não confie em `extrair_observacoes()` para isso — leia o campo bruto da camada.