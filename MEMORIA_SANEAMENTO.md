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