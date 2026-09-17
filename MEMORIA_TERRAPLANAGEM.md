# MEMORIA_TERRAPLANAGEM.md

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