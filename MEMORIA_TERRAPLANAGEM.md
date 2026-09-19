# MEMORIA_TERRAPLANAGEM.md

## 0. Instruções (siga isto — revisão do Claude Code, 19/09/2026)

1.  **Ao investigar qualquer problema de plataforma/cruzamento, meça sempre os DOIS números juntos:** pontos/zonas de cruzamento de curva E m² de caixa de rua intocada. Medir só o primeiro já mascarou a melhora (ou piora) em duas tentativas anteriores.
2.  **Não tente corrigir sozinho o item aberto "plataforma não chega no limite da caixa da rua"** sem antes trocar a associação polígono→via de "centroide mais próximo do eixo" para "o polígono que CONTÉM a estaca terminal" (`superficie_pista_poligono.py`) — é a base já mapeada para a próxima tentativa, não invente uma abordagem nova sem passar por essa mudança primeiro.
3.  **Se um combo de M2, M3, M4, M5, Platô, Locação ou Cul-de-sac vier vazio ou com a camada errada, rode o módulo anterior primeiro** — não é bug de UI, é a trava de consistência (`VerificadorConsistencia`) avisando que falta dado gravado no projeto.
4.  **Sempre exporte DXF em R2000 (AC1015), nunca em versão mais nova** — o AutoCAD do escritório recusa abrir DXF de versão superior à instalada.
5.  **Antes de mexer em `FATOR_ALARGAMENTO_MAX` ou `FOLGA_RECORTE_M`, saiba o que cada um faz:** o primeiro é o teto relativo de alargamento de plataforma em cruzamento (`modules/m3_secoes.py`, hoje 1,5); o segundo é a folga de recorte entre cruzamentos vizinhos (`engine/cruzamento.py`, hoje 0,30 m). Mudar um sem entender o outro pode reintroduzir o degrau/furo que eles existem para evitar.

## 1. Processamento de Malha (TIN) e Curvas de Nível
*   **Integridade do Terreno Natural:** O talude original fora da caixa viária deve ser rigorosamente preservado[cite: 26]. Utilize a Superfície Projetada (MTP) em alta resolução (0,5 m) e faça a integração com o terreno natural via *warp* com interpolação bilinear[cite: 26, 28]. Extraia os contornos com `gdal.ContourGenerate`[cite: 28].
*   **Buracos na Superfície:** Falhas no TIN devem ser corrigidas no editor de malha removendo vértices problemáticos um a um, pois a exclusão em bloco arruína o escore da malha[cite: 8, 9]. Ao colar o resultado, limite a um raio de ~15 m para evitar a importação de artefatos[cite: 8].
*   **Snap e Linemerge:** Cortes nas bordas viárias geram gaps de 10 a 50 cm[cite: 25]. Para continuidade, agrupe feições de mesma cota, execute um *snap* em endpoints livres a menos de 2,5 m entre o terreno natural e o projetado, e una as linhas com `shapely.ops.linemerge`[cite: 25].
*   **Triângulos-Lâmina:** Esquinas de vias ortogonais com pontos de offset a menos de 40 cm (com greides diferentes) formam arestas de 100% de declividade que distorcem as curvas em "Y"[cite: 27]. Resolva isso via Editor de Malha (excluindo vértices invasores) e dispare `_reconstruir()` diretamente do arquivo `.pkl` da sessão, nunca editando as curvas de forma vetorial e isolada[cite: 27].

## 2. Parâmetros Viários, Volumes e Relatórios
*   **Pipeline Sequencial:** A execução segue obrigatoriamente as classes do plugin: Estaqueamento (M1) -> Greide (M2) -> Seções (M3) -> Interseções -> Superfície Projetada -> Volumes (M4) -> Relatório (M5)[cite: 9, 11].
*   **Agrupamento de Nós:** Nos cruzamentos, agrupar nós sempre por DISTÂNCIA (union-find), nunca por célula de grade raster, evitando degraus métricos falsos no greide[cite: 8].
*   **Ajuste de PVI:** Ajustes de PVI são específicos para o eixo atual[cite: 9]. Se o eixo mudar de comprimento via edição CAD, a calibração antiga deve ser zerada e refeita[cite: 9].
*   **Largura da Plataforma:** Meça simultaneamente os cruzamentos de curva e os m² de caixa viária intocada, pois o modelo de seções pode gerar plataformas mais estreitas que a via projetada[cite: 8, 9].
*   **Formato de Exportação:** Todo arquivo DXF (perfis, locação, seções) deve ser gravado estritamente em R2000 (AC1015) para evitar rejeição por versões do AutoCAD[cite: 9].

## 3. Reaproveitamento de Dados e Contingências
*   **Arquivos Intermediários:** Antes de reprocessar a geometria, busque e recarregue artefatos em disco como `_resultados_m3_completo.pkl` e `_configuracao_usada.json`[cite: 29]. O `CalculadorVolumes` exige que essa lista contenha as seções completas de todas as vias ordenadas por `dist_acumulada`[cite: 29].
*   **Camadas Quebradas e Geometria Ausente:** Se uma camada zerar no projeto (`.qgz`), recarregue a fonte via `layer.setDataSource()`, sem reimportar do zero[cite: 29]. Caso falte a lista de PVIs para perfis, investigue subdiretórios vizinhos (ex: `quadros/03_greide_pvis.json`) antes de improvisar falsos PVIs a partir das estacas[cite: 29].

## 4. Correções e Adições (revisão do Claude Code, 19/09/2026)

Seção adicionada após revisão cruzada com o histórico real de desenvolvimento do plugin (`CLAUDE.md` do repositório `terraplanagem-` e relatório de código do `precisa_terraplanagem_suite/`).

*   **Constantes atuais (v2.0.9) que valem conhecer antes de mexer em cruzamento/plataforma:** `FATOR_ALARGAMENTO_MAX = 1,5` — teto RELATIVO de alargamento de plataforma quando há camada de limite de pista, para não alargar demais em cruzamento (`modules/m3_secoes.py`); `FOLGA_RECORTE_M = 0,30 m` — folga usada só nos 4 pontos de recorte de plataforma/talude entre cruzamentos vizinhos, evita furo na malha (`engine/cruzamento.py`).

*   **Pendência ABERTA (ainda não resolvida): a plataforma projetada não chega no limite da caixa da rua desenhada.** Não é falha de cul-de-sac nem é o teto de alargamento (`FATOR_ALARGAMENTO_MAX`) — medido: 330,5 m² de caixa de rua sem movimento de terra (|projeto−TN| < 5 cm), numa tira colada na linha amarela que acompanha a via inteira e engorda nas curvas. Duas tentativas de correção já foram revertidas por piorarem o resultado medido. Lição principal: **meça sempre os DOIS números juntos** — pontos/zonas de cruzamento de curva E m² de caixa intocada — medir só o primeiro (como as duas tentativas revertidas fizeram) mascara se a correção realmente ajudou. Direção já mapeada para a próxima tentativa: trocar a associação polígono→via de "centroide mais próximo do eixo" para "**o polígono que CONTÉM a estaca terminal**" — é uma fragilidade já reconhecida em `superficie_pista_poligono.py` (uma via já teve DOIS polígonos associados por causa disso). Não tente uma correção nova para este item sem essa mudança de associação como base.

*   **M2 a M5 (e Platô/Locação/Cul-de-sac) têm uma trava automática de consistência** (`modules/consistencia.py::VerificadorConsistencia`) que bloqueia com aviso claro se o módulo anterior não rodou NESTE projeto QGIS aberto — não é permissão a contornar, é dependência real de dado gravado (ex. `cota_proj` nas estacas, `secoes_json` no projeto). Desde a v1.6.1 os diálogos também pré-selecionam sozinhos, nos combos, a camada que o módulo anterior gerou (`core/encadeamento.py`) — se um combo vier vazio ou com a camada errada, é sinal de que o módulo anterior não rodou neste projeto, não bug de UI.

*   **Confirmado: DXF sempre em R2000 (AC1015), nunca R2018.** Era só a terraplenagem que gravava em versão nova demais (os plugins de água já gravavam R12) — corrigido na v2.1.0. AutoCAD recusa abrir DXF de versão mais nova que a instalada; R2000 abre em praticamente qualquer CAD (AutoCAD, BricsCAD, ZWCAD, nanoCAD, QCAD, LibreCAD) sem perder entidade.