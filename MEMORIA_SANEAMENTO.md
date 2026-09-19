# MEMORIA_SANEAMENTO.md

## 1. Disposição Padrão e Traçado Geométrico
*   **Gabarito Precisa Agrimensura:** A galeria de drenagem deve correr no eixo da via (offset 0), o esgoto à esquerda (offset -1.5m) e a água à direita (offset +1.5m)[cite: 16, 24].
*   **Método de Traçado:** Para garantir estabilidade geométrica do paralelismo, trace utilizando `LineString.parallel_offset(dist, side, resolution=4)` da biblioteca Shapely[cite: 14, 16, 24].
*   **Faixas de Servidão:** Redes de água NUNCA entram em faixas de servidão[cite: 8, 14, 16, 24]. Servidões comportam apenas esgoto e, quando justificado pelo relevo, drenagem[cite: 9].

## 2. Parâmetros de Engenharia e Dimensionamento
*   **População:** Áreas não vendáveis (prefixos A.I. ou A.V.) não têm residências e devem ser filtradas do cálculo populacional e de contribuição de esgoto[cite: 8, 12].
*   **Recobrimentos e Vala:** 
    *   Água: 0.9m na rua[cite: 24].
    *   Esgoto: 0.9m na rua, 0.2m em servidão, vala máxima de 3.50m[cite: 8, 24].
    *   Drenagem: 0.6m na rua, 0.2m em servidão, vala máxima de 3.00m[cite: 8, 24]. Em terrenos planos, aprofunde a vala em vez de subir o diâmetro da tubulação[cite: 8].
*   **Diâmetros Mínimos e Fluxo:** O diâmetro nunca pode sofrer redução de montante para jusante[cite: 8, 24]. Mínimos exigidos: Esgoto DN 200, Drenagem DN 400, Água DN 50[cite: 8]. Velocidades acima de 5,0 m/s requerem obras de dissipação físicas, mas não são erros de software[cite: 8].
*   **Chuva de Projeto (IDF):** O IDF default do plugin superestima a precipitação em 55%; não o utilize. Extraia e interpole a intensidade exata no banco de dados do Plúvio 2.1[cite: 8, 9, 24].

## 3. Topologia e Pipeline Automatizado
*   **Nós e Bifurcações:** A tolerância de costura Voronoi (`TOL_COSTURA`) é de 3.0m para fechar bifurcações corretamente[cite: 8]. As cotas dos nós são lidas do greide da rua, adotando sempre a cota mais baixa nos encontros rua-servidão[cite: 8]. O orçamento de subida da árvore de caminhos mínimos deve ser de 2.0m[cite: 8, 12].
*   **Execução Inviolável:** O pipeline dos plugins requer a configuração obrigatória da variável de ambiente `TERRA_DADOS` e uma estrutura `pacote_<loteamento>/loteamento/`[cite: 15, 17]. A ordem de execução `rede_build` -> `rede_esgoto_build` -> `roda_*` -> `planta_rede` não pode ser saltada, sob pena de gerar falhas silenciosas nos subprocessos[cite: 9, 12, 17].
*   **Auditoria de Entrega:** Verifique se as redes invadem lotes particulares usando tolerância float de 0.999 para mitigar falsos positivos[cite: 8, 9]. Exporte camadas finalizadas no formato GeoPackage (GPKG) e colorize as plantas 1:500 (Drenagem: azul; Esgoto: marrom; Água: verde)[cite: 14, 16, 24].

## 4. Fórmulas para Contingência (Cálculo Manual)
Em caso de falha do módulo automatizado, dimensione com:
*   **Água:** Vazão = pop × 150 L/hab/dia / 86400 (DN min 50)[cite: 16, 24].
*   **Esgoto:** Vazão = pop × (80-100 L/hab/dia) / 86400 (DN min 200, y/D <= 0.75)[cite: 16, 24].
*   **Drenagem:** Método Racional: Q = (C × I × A) / 3.6, com C = 0.7 (DN min 400, y/D <= 0.80, vel min 0.6 m/s)[cite: 16, 24].

## 5. Correções e Adições (revisão do Claude Code, 19/09/2026)

Seção adicionada após conferir números direto no código (`aguas/pipeline/`, `aguas/plugins/*/core/`) e no `CLAUDE.md` do repositório `terraplanagem-`. Um item do §1 e um do §2 estavam com valor errado — os corrijo aqui em vez de reescrever lá em cima, para deixar rastro do que mudou e por quê.

*   **CORREÇÃO — o offset de esgoto/água NÃO é o número fixo "-1,5 m / +1,5 m" do §1.** Conferido em `aguas/pipeline/rede_build.py` (dict `DIRETRIZ`): os afastamentos reais usados foram esgoto **-2,0 m** e água **+4,25 m** (Santa Cruz) / **+4,75 m** (Imperial) — configurados por loteamento, não uma constante universal. O sinal segue a direção de digitalização do eixo (positivo = à esquerda dela), então o lado FÍSICO pode alternar de rua para rua — o único invariante real é: água e esgoto sempre em lados OPOSTOS, drenagem no eixo (offset 0). Antes de aplicar qualquer offset numa faixa de servidão estreita, **meça a largura real da faixa** (`2·área/perímetro`): um offset de ±0,60 m numa faixa de 1,00 m põe o tubo FORA dela, dentro do lote do vizinho — foi o que produziu 225 m de água + 248 m de esgoto em lote particular no Santa Cruz antes de alguém medir.

*   **CORREÇÃO — "Água: 0,9 m na rua" do §2 não existe como parâmetro no plugin.** Conferido em `ParametrosAgua` (`precisa_abastecimento/core/dimensionamento.py`): não há campo de recobrimento ali. A rede de água é modelada como tubo raso, de cota praticamente fixa, independente da vazão — sem PV/vala calculada como em esgoto/drenagem. O 0,9 m do §2 provavelmente veio de confundir com o recobrimento do ESGOTO (esse sim real: 0,90 m na rua).

*   **ATENÇÃO — o recobrimento da drenagem que aparece na TELA do diálogo pode não ser o padrão do escritório.** O core (`ParametrosDrenagem.recobrimento`) foi trocado de 1,00 para 0,60 m (rua) numa versão, mas o campo "Recobrimento (m)" do diálogo (`drenagem_dialog.py`) ficou hardcoded em 1,00 m e não foi atualizado junto. Como é sempre o valor do diálogo que vai para o cálculo, confira e corrija esse campo para 0,60 m antes de calcular — não assuma que o valor pré-preenchido já é o padrão certo.

*   **Cota do nó vem do GREIDE por interpolação geométrica ao longo da poligonal de estacas — nunca por amostragem de raster nem por índice de estaca.** Amostragem raster numa janela 5×5 já inventou 3 cm de barriga falsa num cruzamento; índice de estaca falha quando o eixo foi estendido pelo pipeline (Imperial: RUA B +7,20 m, RUA G +1,70 m). No encontro servidão×rua, quando as duas fontes de cota (greide da rua vs. terreno natural do fundo do lote) discordam, adote a MENOR — é a que mantém o escoamento descendo para o coletor da rua.

*   **`centerlines()` do esqueleto de servidão devolve TODOS os ramos, não só o mais longo.** Importante quando há dente transversal real (Imperial: A.I. 4 foi de 398,4 m num caminho só para 471,8 m em 6 ramos). Mas cuidado: usar todos os ramos pode duplicar corredor já atendido por outra faixa (Santa Cruz: A.I. 3 virou linha paralela à A.I. 8 a 4 m de distância) — confira sobreposição antes de aceitar todos.

*   **`offset_curve` pode devolver a linha com o sentido invertido.** Não suponha que inverte só para offset negativo — teste empiricamente qual ponta fica mais perto do início do eixo original. Já produziu 53,7 m de deslocamento de ponta quando isso foi assumido errado.

*   **O conector de uma faixa de servidão pode aterrissar em OUTRA faixa de servidão, não só em rua.** Procurar só trecho de rua já arrastou um conector 38,6 m, indo parar 16 m acima da ponta certa (A.I. 9, Santa Cruz).

*   **IDF — não perca tempo nos sites de órgãos públicos.** ANA, INMET, gov.br, SGB, UFV, UFLA, UFOP, abrhidro estão bloqueados pelo proxy deste ambiente. Use o banco do Plúvio 2.1 já extraído (`aguas/pipeline/pluvio_db.py`, 549 localidades) ou, na falta de cobertura, `aguas/pipeline/gera_idf.py` (Gumbel + desagregação de 24h — validado contra Bambuí-MG com erro de 0,016%).

*   **Reservatório de água: a cota de referência é sempre o GREIDE DA RUA em frente ao lote, nunca o terreno de dentro do lote** (o lote cai do meio-fio para dentro). A cota calculada é o N.A. MÍNIMO operacional. Volume: a NBR 12217 pede mínimo 1/3 do dia de maior consumo, mas o escritório adota 1/2 (folga para incêndio) — não "corrija" para 1/3 achando que está sendo mais fiel à norma, é decisão de projeto. Desnível de loteamento acima de 40 m não fecha numa zona só: setorize com VRP, buscando o MENOR número de válvulas por busca exaustiva nos cortes da árvore — uma heurística gulosa já pediu 3 válvulas onde 2 bastavam.

*   **Os 3 plugins de água ganharam recursos novos (a partir de set/2026) que mudam como operá-los.** A tabela de resultados agora tem colunas EDITÁVEIS (DN e, em drenagem/esgoto, também profundidade de montante) — editar uma célula recalcula em cascata os trechos a jusante automaticamente, sem precisar clicar em "Calcular" de novo (clicar em "Calcular" do zero DESCARTA esses ajustes manuais). Editar um atributo na camada de rede ou de lotes (população, área de contribuição, coeficiente C) também dispara recálculo sozinho, assim como mover/inserir vértice na rede com as ferramentas nativas do QGIS. As camadas de resultado agora têm simbologia orientada aos valores calculados (cor por velocidade/lâmina/tensão trativa fora do limite; PV/nó com X vermelho se a cota calculada fura o terreno) — não é mais cor fixa por camada. Há também "Ver Perfil (interativo)…" dentro do próprio QGIS. Detalhe completo nos manuais em `manuais/` deste mesmo repositório.

*   **Pendência aberta, ainda não resolvida:** a memória de esgoto do Santa Cruz gerada pelo diálogo tem profundidade quase constante (1,10–1,14 m) em todos os 49 trechos, mesmo com o terreno variando ~39 m — a referência já entregue varia 0,40–3,99 m, como fisicamente esperado. Suspeita: MDT ou camada de rede diferentes do que gerou a referência. Não confie na profundidade dessa rodada específica sem confirmar com o RT qual fonte de terreno foi usada.