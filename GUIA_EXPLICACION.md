# Guía de Presentación — Análisis de Canasta de Mercado

> **Objetivo:** Encontrar patrones de compra en 100 clientes usando Apriori y ECLAT, para mejorar ventas cruzadas y segmentar clientes.

---

## PREPARACIÓN DE DATOS

**Lo que se hizo en el código:**
```python
df = pd.read_excel('mercado_SB.xlsx')
cols_demo = ['edad', 'estrato_nse', 'canal_compra', 'gasto']
productos = df.drop(columns=cols_demo)
```
Se cargó el Excel con 100 clientes y 35 columnas. Se separaron las 4 columnas demográficas de las 31 columnas de productos (valores 0/1). La variable `productos` es la **matriz de transacciones** — el input de todos los algoritmos.

---

## 1. ¿Qué productos se compran más?

**Lo que se hizo:**
```python
frecuencia = productos.sum().sort_values(ascending=False)
soporte = (frecuencia / len(df) * 100)
```
Se sumaron las columnas para contar cuántos clientes compraron cada producto. El soporte es esa suma dividida por 100 (total de clientes), expresada en %.

![Frecuencia de compra](imagenes/grafica_01.png)

**Resultado obtenido:** `arroz` (71%), `café` (60%) y `azúcar` (60%) son los productos con mayor soporte. Son el núcleo del negocio — cualquier promoción que los incluya tiene el mayor alcance posible.

---

## 2. ¿Quiénes son los clientes?

**Lo que se hizo:**
```python
axes[0, 0].hist(df['edad'], bins=15)          # distribución de edades
axes[0, 1].hist(df['gasto'], bins=15)         # distribución de gasto
df['estrato_nse'].value_counts().plot(bar)    # clientes por estrato
df['canal_compra'].value_counts().plot(pie)   # canal de compra
```
Se graficaron las 4 variables demográficas con histogramas y gráficos de torta para entender el perfil del cliente.

![Variables demográficas](imagenes/grafica_02.png)

**Resultado obtenido:** Base de clientes de mediana edad, gasto concentrado en un rango específico. El canal con mayor participación define el enfoque de las estrategias de venta.

---

## 3. ¿Qué productos se compran juntos?

**Lo que se hizo:**
```python
co_matrix = productos.T.dot(productos)   # multiplicación matricial
co_arr = co_matrix.to_numpy().copy()
np.fill_diagonal(co_arr, 0)              # quitar diagonal (producto consigo mismo)
co_matrix = pd.DataFrame(co_arr, ...)
```
Se multiplicó la matriz de transacciones por su transpuesta. El resultado es una matriz donde cada celda `[i,j]` contiene cuántos clientes compraron el producto `i` y el producto `j` al mismo tiempo. Se eliminó la diagonal para no contar cada producto con sí mismo.

![Heatmap co-ocurrencia](imagenes/grafica_03.png)

**Resultado obtenido:** Vista rápida de las asociaciones más frecuentes antes de los algoritmos. Las celdas más oscuras son los pares candidatos a reglas fuertes.

---

## 4. ¿Cuánto gastan según canal y estrato?

**Lo que se hizo:**
```python
gasto_canal = df.groupby('canal_compra')['gasto'].mean()
gasto_estrato = df.groupby('estrato_nse')['gasto'].mean()
```
Se agrupó el DataFrame por canal y por estrato, calculando el gasto promedio de cada grupo con `.mean()`.

![Gasto por canal y estrato](imagenes/grafica_04.png)

**Resultado obtenido:** El canal y el estrato con barra más alta son los segmentos más rentables por cliente. Guía dónde priorizar la inversión en marketing.

---

## 5. ALGORITMO APRIORI — Proceso completo

**Lo que se hizo:**
```python
# Paso 1: encontrar itemsets frecuentes (soporte >= 20%)
frequent_itemsets = apriori(productos.astype(bool), min_support=0.2, use_colnames=True)

# Paso 2: generar reglas de asociación con lift >= 1
rules = association_rules(frequent_itemsets, metric='lift', min_threshold=1)

# Paso 3: calcular métricas adicionales
rules['affinity'] = rules['confidence'] * rules['lift']
rules['score'] = 0.4 * rules['support'] + 0.3 * rules['confidence'] + 0.3 * rules['lift']
```
Apriori trabaja en dos pasos: primero recorre las transacciones buscando productos (y combinaciones) que aparezcan en al menos el 20% de los clientes. Luego, de esos grupos frecuentes, genera reglas direccionales A→B y mide qué tan confiables son. Se añadió un `score` personalizado que combina las tres métricas para rankear reglas según su valor de negocio.

**Las 3 métricas clave:**
| Métrica | Fórmula | Valor bueno |
|---------|---------|-------------|
| **Soporte** | transacciones con A y B / total | > 20% |
| **Confianza** | transacciones con A y B / transacciones con A | > 50% |
| **Lift** | confianza / soporte(B) | > 1.0 |

![Scatter Apriori](imagenes/grafica_05.png)

**Resultado obtenido:** Se encontraron reglas de asociación. Las reglas en la esquina superior derecha (alto soporte + alta confianza) con color amarillo (lift alto) son las más accionables. Los puntos más grandes tienen mayor afinidad.

---

## 6. Top 15 mejores reglas

**Lo que se hizo:**
```python
top_rules = rules.sort_values('lift', ascending=False).head(15)
top_score = rules.sort_values('score', ascending=False).head(15)
```
Se ordenaron las reglas de mayor a menor lift (izquierda) y mayor a menor score de negocio (derecha).

![Top reglas por lift y score](imagenes/grafica_06.png)

**Resultado obtenido:** La línea punteada vertical es lift=1 (el azar). Todas las reglas listadas están por encima — **no son coincidencia**. Las reglas del score combinado son las más rentables para implementar en el negocio.

---

## 7. ¿Qué tan fuerte es cada par A → B?

**Lo que se hizo:**
```python
rules_simple = rules[(rules['antecedents'].apply(len) == 1) &
                     (rules['consequents'].apply(len) == 1)]
conf_matrix = rules_simple.pivot_table(
    index='antecedents_str', columns='consequents_str',
    values='confidence', aggfunc='max')
```
Se filtraron solo las reglas simples (1 producto → 1 producto) y se pivotearon en una matriz donde filas = antecedente, columnas = consecuente, valor = confianza máxima.

![Heatmap de confianza](imagenes/grafica_07.png)

**Resultado obtenido:** Cada celda responde: "de los que compraron el producto de la fila, X% también compró el de la columna". Celdas rojas intensas = relaciones fuertes para combos y promociones cruzadas.

---

## 8. Reglas por canal de compra

**Lo que se hizo:**
```python
for canal in df['canal_compra'].unique():
    subset = df[df['canal_compra'] == canal].drop(columns=cols_demo)
    fi = apriori(subset, min_support=0.2, use_colnames=True)
    r = association_rules(fi, metric='lift', min_threshold=1)
```
Se aplicó Apriori de forma independiente a cada canal filtrando el DataFrame. Cada subconjunto tiene sus propias reglas según el comportamiento de sus clientes.

![Reglas por canal](imagenes/grafica_08.png)

**Resultado obtenido:** Las asociaciones son distintas por canal. Clientes online tienen patrones diferentes a los de tienda física, lo que permite hacer **recomendaciones personalizadas por canal**.

---

## 9. ALGORITMO ECLAT — Proceso completo

**Lo que se hizo:**
```python
# Paso 1: construir listas de transacciones
transactions = []
for _, row in productos.iterrows():
    trans = list(productos.columns[row == 1])
    transactions.append(trans)

# Paso 2: crear TID-lists (lista de IDs de transacción por producto)
tid_lists = {}
for i, trans in enumerate(transactions):
    for item in trans:
        tid_lists.setdefault(item, set()).add(i)

# Paso 3: intersectar TID-lists para encontrar itemsets frecuentes
for combo in combinations(items, size):
    tids = tid_lists[combo[0]].copy()
    for item in combo[1:]:
        tids = tids.intersection(tid_lists[item])  # intersección de conjuntos
    support = len(tids) / n
    if support >= 0.2:
        frequent_itemsets_eclat.append({'itemset': combo, 'support': support})
```
ECLAT trabaja diferente a Apriori: en vez de recorrer las transacciones una y otra vez, construye una **lista de IDs de transacción (TID-list)** para cada producto. Para encontrar si dos productos se compran juntos, basta con **intersectar sus TID-lists** — si el tamaño de la intersección / total >= 20%, es un itemset frecuente.

![Distribución itemsets ECLAT](imagenes/grafica_09.png)

**Resultado obtenido:** Se encontraron itemsets de tamaño 1, 2 y 3. Siempre hay más grupos pequeños que grandes — es matemáticamente esperado porque exigir más productos juntos filtra más clientes.

---

## 10. Top pares y tríos de productos

**Lo que se hizo:**
```python
sub = freq_df[freq_df['size'] == 2].sort_values('support', ascending=False).head(20)
sub['label'] = sub['itemset'].apply(lambda x: ' + '.join(x))
```
Se filtraron los itemsets de tamaño 2 y 3, se ordenaron por soporte y se etiquetaron como "producto A + producto B".

![Top 2 y 3 itemsets](imagenes/grafica_10.png)

**Resultado obtenido:** Los pares más frecuentes son los mejores candidatos para **ofertas 2x1**. Los tríos con soporte > 0.40 (40% de clientes) son ideales para **combos de 3 productos o exhibición conjunta** en tienda.

---

## 11. SEGMENTACIÓN POR ESTRATO — Proceso

**Lo que se hizo:**
```python
def eclat_segmento(data, min_support=0.2):
    prods = data.drop(columns=cols_demo, errors='ignore')
    # ... misma lógica ECLAT pero aplicada al subconjunto

for estrato in df['estrato_nse'].unique():
    subset = df[df['estrato_nse'] == estrato]
    resultados_estrato[estrato] = eclat_segmento(subset)
```
Se creó una función `eclat_segmento()` que aplica el algoritmo ECLAT a cualquier subconjunto del DataFrame. Se llamó una vez por cada estrato socioeconómico, obteniendo itemsets frecuentes **específicos de cada segmento**.

![Heatmap por estrato](imagenes/grafica_11.png)

**Resultado obtenido:** Cada columna del heatmap es un estrato. Los productos con diferencias grandes entre columnas son **sensibles al nivel socioeconómico** — oportunidades de personalización por zona o segmento.

---

## 12. Radar por estrato

**Lo que se hizo:**
```python
top_prods = heat_df.mean(axis=1).nlargest(8).index.tolist()
angles = [n / float(N) * 2 * math.pi for n in range(N)]
for estrato in estratos:
    values = [data_heat[estrato].get(p, 0) for p in top_prods]
    ax.plot(angles, values, 'o-', linewidth=2, label=estrato)
    ax.fill(angles, values, alpha=0.1)
```
Se seleccionaron los 8 productos con mayor soporte promedio y se graficó el soporte de cada estrato en un gráfico polar (radar). Cada eje representa un producto.

![Radar por estrato](imagenes/grafica_12.png)

**Resultado obtenido:** Las líneas separadas indican diferencias de consumo entre estratos. El área cubierta por cada línea representa la "amplitud de consumo" — estratos con áreas más grandes tienen patrones de compra más diversos.

---

## 13. Top 5 productos por estrato

**Lo que se hizo:**
```python
singles = res[res['itemset'].apply(len) == 1].copy()
singles['label'] = singles['itemset'].apply(lambda x: x[0])
top3_estrato[estrato] = singles.sort_values('support', ascending=False).head(5)
```
De los itemsets de cada estrato, se filtraron solo los de 1 producto (soporte individual) y se tomaron los 5 con mayor soporte.

![Top 5 por estrato](imagenes/grafica_13.png)

**Resultado obtenido:** Para cada estrato, los 5 productos más importantes. Acción directa: **adaptar inventario y ofertas** según el estrato predominante en cada sucursal.

---

## 14. SEGMENTACIÓN POR EDAD — Proceso

**Lo que se hizo:**
```python
df['grupo_edad'] = pd.cut(
    df['edad'], bins=[0, 25, 35, 50, 100],
    labels=['Joven', 'Adulto joven', 'Adulto', 'Senior'])

for grupo in ['Joven', 'Adulto joven', 'Adulto', 'Senior']:
    subset = df[df['grupo_edad'] == grupo]
    resultados_edad[grupo] = eclat_segmento(subset)
```
Se crearon 4 grupos de edad con `pd.cut()` usando rangos definidos. Luego se aplicó `eclat_segmento()` a cada grupo, exactamente igual que con los estratos.

![Heatmap y barras por edad](imagenes/grafica_14.png)

**Resultado obtenido:** Los productos con más diferencia entre columnas del heatmap son los más sensibles a la edad. Diferencias generacionales claras guían el diseño de campañas por grupo etario.

---

## 15. Clientes y gasto por edad

**Lo que se hizo:**
```python
counts_edad = df['grupo_edad'].value_counts().reindex([...])
gasto_edad = df.groupby('grupo_edad', observed=True)['gasto'].mean()
```
Se contaron los clientes por grupo de edad y se calculó el gasto promedio de cada grupo.

![Clientes y gasto por edad](imagenes/grafica_15.png)

**Resultado obtenido:** Seniors son el grupo más numeroso y el que más gasta. Jóvenes son los menos y gastan menos. **Estrategia:** fidelizar seniors con programas de lealtad; captar jóvenes con canales digitales.

---

## 16. RED DE ASOCIACIONES — Proceso

**Lo que se hizo:**
```python
G = nx.DiGraph()  # grafo dirigido
for _, row in top_net.iterrows():
    ant = list(row['antecedents'])[0]
    con = list(row['consequents'])[0]
    G.add_edge(ant, con, weight=row['lift'])  # arista con peso = lift

pos = nx.spring_layout(G, seed=42, k=2)  # posición de nodos
edge_weights = [G[u][v]['weight'] for u, v in G.edges()]
node_sizes = [G.degree(n) * 300 + 500 for n in G.nodes()]  # tamaño según conexiones
```
Se tomaron las 30 mejores reglas simples y se construyó un grafo dirigido con `networkx`. Cada producto es un nodo, cada regla es una flecha. El **tamaño del nodo** es proporcional a cuántas reglas tiene (grado del nodo). El **grosor y color** de la flecha es proporcional al lift de la regla.

![Red de asociaciones](imagenes/grafica_16.png)

**Resultado obtenido:** Los nodos más grandes y conectados son los **productos ancla**. Ponerlos en puntos estratégicos de la tienda o usarlos como gancho en promociones genera más ventas cruzadas.

---

## 17. VALIDACIÓN: Apriori vs ECLAT

**Lo que se hizo:**
```python
compare = pd.merge(
    apriori_singles[['label', 'support']].rename(columns={'support': 'Apriori'}),
    eclat_singles[['label', 'support']].rename(columns={'support': 'ECLAT'}),
    on='label')
compare['diferencia'] = (compare['Apriori'] - compare['ECLAT']).abs()
```
Se extrajeron los itemsets de 1 producto de ambos algoritmos y se unieron con `merge` por nombre de producto. Se calculó la diferencia absoluta entre los soportes.

![Comparativa Apriori vs ECLAT](imagenes/grafica_17.png)

**Resultado obtenido:** La diferencia entre Apriori y ECLAT es **0.0 en todos los productos**. Esto valida que la implementación manual de ECLAT es correcta y que los resultados del análisis son confiables.

---

## CONCLUSIÓN

1. **Operativo:** colocar juntos en tienda los productos con reglas más fuertes (los nodos más conectados de la red)
2. **Segmentado:** seniors gastan más → fidelizarlos; cada estrato tiene sus productos clave → personalizar inventario
3. **Validado:** Apriori y ECLAT coinciden en todos los valores → resultados robustos y confiables

---

## Si te preguntan algo difícil

**¿Por qué el soporte mínimo es 20%?**
> Con 100 clientes, 20% = 20 personas. Es el mínimo para que una regla sea estadísticamente relevante sin perder demasiadas asociaciones.

**¿Lift de 1.3 es bueno?**
> Sí. Significa que la compra conjunta ocurre 30% más de lo esperado por azar. En supermercados con miles de productos, lift > 1.2 ya es significativo.

**¿Por qué dos algoritmos si dan los mismos soportes?**
> Apriori genera reglas con dirección (A→B). ECLAT genera itemsets sin dirección, es más eficiente en datasets grandes. Se complementan y el soporte idéntico valida que ambos están bien implementados.

**¿Para qué sirve esto en la práctica?**
> Sistemas de recomendación, diseño del layout de tienda, combos promocionales, personalización por segmento y gestión de inventario conjunto.
