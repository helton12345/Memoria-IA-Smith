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