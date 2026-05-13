# Reporte de resultados — Postura pro-inmigración en Europa (ESS11)

**Autor:** Sebastián Amari · **Asignatura:** Aplicaciones Avanzadas de IA · **Fecha:** 2026-05-13

**Pregunta de investigación:** ¿En qué medida la **confianza institucional** (parlamento, sistema legal, policía, partidos políticos) predice la postura **pro-inmigración** del ciudadano europeo, frente a otros factores socio-demográficos (edad, educación, ingresos, ideología, país)?

---

## 1. Resumen ejecutivo

- **Datos:** ESS11 ronda 2023/24 (`ESS11e04_1.csv`), **47.914 personas** de **30 países europeos** + Israel, tras limpieza.
- **Variable objetivo:** `pro_imm_high` = 1 si la persona promedia **> 5** en las escalas 0-10 de `imueclt` (cultura), `imwbcnt` (calidad de vida) e `imbgeco` (economía). Distribución balanceada: **51,8 % pro / 48,2 % anti**.
- **Mejor desempeño:** Regresión Logística y Random Forest empatan virtualmente (**ROC-AUC ≈ 0,755** en test).
- **Hallazgo principal:** La **confianza institucional es el predictor más fuerte** en las tres aproximaciones (coeficientes, importancia Gini y SHAP). Un aumento de 1 desviación estándar en el índice de confianza **multiplica por 1,79** las odds de tener postura pro-inmigración.
- **Eje país:** brecha enorme entre los **nórdicos** (Islandia 83,8 %, Finlandia 78,7 %, Suecia 76,4 %) y **Europa central/oriental** (Eslovaquia 22,4 %, Hungría 24,2 %, Bulgaria 31,3 %). A nivel agregado por país, la correlación confianza↔pro-inmigración es **r = 0,63**.

---

## 2. Datos y preparación

| | Valor |
|---|---|
| Fuente | European Social Survey, Round 11 (ed. 04.1) |
| Casos originales | 50.116 |
| Casos finales | **47.914** (95,6 %) |
| Países | 30 |
| Variables predictoras finales | 34 (5 numéricas + 29 dummies país) |

**Pasos de limpieza relevantes:**

1. Recodificación de códigos especiales del ESS (77/88/99 para escalas 0-10; 999 para edad) a `NaN`.
2. Filtro `agea ≥ 18` (descarta menores y NaN de edad).
3. Filtro de filas con > 30 % de missing en variables clave.
4. Eliminación de casos con `pro_immigration` no calculable.
5. **Imputación KNN** (k = 5, ponderada por distancia) ajustada **solo en train** y aplicada a test (sin data leakage).
6. **Split estratificado 80/20** antes de imputar.

**Nota metodológica:** la versión inicial del script incluía `impcntr` en el índice de inmigración. Se corrigió porque `impcntr` está en escala 1-4 (no 0-10) y con dirección invertida; se sustituyó por `imbgeco` (también 0-10, misma dirección). Sin esa corrección, el target estaba mal definido.

---

## 3. Resultados de los modelos

Validación cruzada estratificada 5-fold sobre train (38.331 casos) + evaluación final en test (9.583 casos).

| Modelo | Accuracy CV | F1 CV | ROC-AUC CV | Accuracy Test | Precision | Recall | F1 Test | **ROC-AUC Test** |
|---|---|---|---|---|---|---|---|---|
| Regresión Logística | 0,688 ± 0,003 | 0,694 ± 0,004 | 0,754 ± 0,005 | 0,685 | 0,702 | **0,680** | **0,691** | 0,754 |
| Random Forest | 0,687 ± 0,006 | 0,679 ± 0,006 | 0,758 ± 0,004 | 0,684 | **0,724** | 0,630 | 0,674 | **0,755** |

**Lectura:**
- **Empate técnico** en AUC. El RF tiene más precisión (menos falsos positivos) pero menos recall; la LogReg detecta más casos pro-inmigración a costa de algo más de error.
- AUC ≈ 0,75 es un desempeño **moderado-bueno** para un problema actitudinal con solo 5 variables numéricas y país. Indica que las actitudes hacia la inmigración tienen un componente predecible importante, pero también mucha variabilidad individual no capturada.
- Que LogReg y RF empaten sugiere que **las relaciones son mayoritariamente lineales** — no hay interacciones no-lineales fuertes que el árbol explote.

📄 Tablas y figuras: [`tabla_resultados_modelos.csv`](tabla_resultados_modelos.csv), [`matriz_confusion.png`](matriz_confusion.png), [`curva_roc.png`](curva_roc.png).

---

## 4. ¿Qué variables pesan más?

Las tres aproximaciones (coeficientes estandarizados de LogReg, importancia Gini del RF, |SHAP| medio) **coinciden en el top 3**:

| Rank | LogReg (\|coef\|) | RF (Gini) | SHAP (\|mean\|) |
|---|---|---|---|
| 1 | **`trust_index`** (0,58) | **`trust_index`** (0,33) | **`trust_index`** (0,092) |
| 2 | `lrscale` (-0,28) | `eduyrs` (0,18) | `eduyrs` (0,058) |
| 3 | `eduyrs` (0,27) | `lrscale` (0,09) | `lrscale` (0,035) |
| 4 | dummies país | dummies país | dummies país |
| 5 | `agea`, `hinctnta` | `agea`, `hinctnta` | `agea`, `hinctnta` |

**Interpretación de los coeficientes (LogReg, sobre features escaladas → comparables entre sí):**

- **`trust_index` (+0,58, OR = 1,79)** — el predictor más fuerte y positivo. **+1 desviación estándar en confianza institucional ≈ +79 % en odds de ser pro-inmigración**.
- **`lrscale` (-0,28, OR = 0,76)** — quienes se autoubican más a la derecha tienen menos probabilidad de ser pro-inmigración. **+1 SD a la derecha ≈ -24 % en odds**.
- **`eduyrs` (+0,27, OR = 1,31)** — más años de educación → más probabilidad pro-inmigración. Coherente con la literatura.
- **`agea` y `hinctnta`** — efectos pequeños y poco estables.

📄 Coeficientes completos: [`coeficientes_logreg.csv`](coeficientes_logreg.csv). Importancias RF: [`importancias_rf.csv`](importancias_rf.csv). Gráficos: [`importancias_rf.png`](importancias_rf.png), [`shap_summary_bar.png`](shap_summary_bar.png), [`shap_summary_beeswarm.png`](shap_summary_beeswarm.png).

**Respuesta a la pregunta de investigación:** la confianza institucional **es** el factor más relevante de los considerados, por encima incluso de la ideología izquierda-derecha y de la educación.

---

## 5. Hallazgos por país

### 5.1 Ranking completo

Ordenado por porcentaje pro-inmigración descendente. *(N: casos válidos; pro_imm_mean: media de la escala 0-10; trust_mean: confianza institucional media).*

| País | N | % Pro | pro_imm (0-10) | trust (0-10) | Edad | Educ. (años) | LR (0-10) |
|---|---:|---:|---:|---:|---:|---:|---:|
| 🇮🇸 Islandia | 814 | **83,8 %** | 6,96 | 5,94 | 51,1 | 16,3 | 4,86 |
| 🇫🇮 Finlandia | 1.522 | **78,7 %** | 6,60 | 6,87 | 53,8 | 14,7 | 5,56 |
| 🇸🇪 Suecia | 1.206 | **76,4 %** | 6,46 | 6,31 | 54,7 | 14,3 | 4,88 |
| 🇳🇴 Noruega | 1.283 | **74,8 %** | 6,33 | 6,81 | 49,8 | 15,0 | 5,13 |
| 🇨🇭 Suiza | 1.342 | 73,9 % | 6,25 | 6,49 | 50,9 | 11,9 | 5,03 |
| 🇪🇸 **España** | 1.779 | **71,1 %** | 6,23 | 4,82 | 50,8 | 14,6 | 4,63 |
| 🇬🇧 Reino Unido | 1.582 | 70,4 % | 6,21 | 4,57 | 54,2 | 14,4 | 4,63 |
| 🇳🇱 Países Bajos | 1.631 | 68,6 % | 5,83 | 5,99 | 51,5 | 15,0 | 5,05 |
| 🇮🇪 Irlanda | 1.956 | 66,9 % | 6,07 | 5,37 | 54,2 | 14,8 | 4,92 |
| 🇧🇪 Bélgica | 1.516 | 64,5 % | 5,71 | 5,17 | 50,7 | 14,1 | 5,09 |
| 🇩🇪 Alemania | 2.337 | 60,8 % | 5,66 | 5,60 | 51,4 | 14,6 | 4,55 |
| 🇵🇱 Polonia | 1.340 | 58,7 % | 5,49 | 4,05 | 49,9 | 13,8 | 5,55 |
| 🇵🇹 Portugal | 1.347 | 54,5 % | 5,49 | 4,44 | 54,3 | 10,6 | 4,93 |
| 🇫🇷 Francia | 1.702 | 53,4 % | 5,33 | 4,74 | 51,4 | 13,3 | 5,03 |
| 🇺🇦 Ucrania | 2.464 | 52,4 % | 5,45 | **2,56** | 52,0 | 12,9 | 5,24 |
| 🇸🇮 Eslovenia | 1.196 | 52,3 % | 5,20 | 4,62 | 51,0 | 13,2 | 4,88 |
| 🇪🇪 Estonia | 1.229 | 50,9 % | 5,18 | 5,42 | 51,2 | 14,4 | 5,69 |
| 🇱🇹 Lituania | 1.305 | 50,0 % | 5,18 | 4,80 | 50,9 | 13,8 | 5,20 |
| 🇮🇱 Israel | 837 | 49,2 % | 4,88 | 4,13 | 44,1 | 13,8 | 5,96 |
| 🇦🇹 Austria | 2.299 | 45,0 % | 4,94 | 5,70 | 56,1 | 12,7 | 4,82 |
| 🇭🇷 Croacia | 1.491 | 43,5 % | 4,88 | 3,42 | 52,7 | 12,0 | 5,19 |
| 🇮🇹 Italia | 2.722 | 42,9 % | 4,80 | 4,92 | 53,1 | 12,0 | 5,18 |
| 🇱🇻 Letonia | 1.171 | 42,2 % | 4,71 | 3,86 | 57,4 | 13,6 | 5,96 |
| 🇷🇸 Serbia | 1.444 | 41,0 % | 4,72 | 4,18 | 53,1 | 12,3 | 4,69 |
| 🇲🇪 Montenegro | 1.536 | 36,0 % | 4,69 | 4,31 | 50,5 | 12,0 | 4,22 |
| 🇧🇬 Bulgaria | 2.107 | 31,3 % | 4,18 | 3,42 | 53,1 | 12,8 | 5,24 |
| 🇬🇷 Grecia | 2.720 | 26,4 % | 4,10 | 4,35 | 51,3 | 12,3 | 5,32 |
| 🇨🇾 Chipre | 656 | 25,2 % | 3,70 | 3,99 | 55,9 | 12,6 | 5,28 |
| 🇭🇺 Hungría | 1.981 | **24,2 %** | 3,73 | 5,04 | 51,3 | 12,1 | 5,67 |
| 🇸🇰 Eslovaquia | 1.399 | **22,4 %** | 3,52 | 4,52 | 55,0 | 13,1 | 4,52 |

📄 Datos: [`tabla_por_pais.csv`](tabla_por_pais.csv). Gráficos: [`pro_inmigracion_por_pais.png`](pro_inmigracion_por_pais.png), [`trust_vs_proimm_paises.png`](trust_vs_proimm_paises.png).

### 5.2 Patrones geográficos

**Bloque pro-inmigración (>70 %): nórdicos + occidentales de altos ingresos.**
Islandia, Finlandia, Suecia, Noruega, Suiza, España, Reino Unido. Comparten alta confianza institucional (excepto España y Reino Unido), educación alta y Estados de bienestar consolidados. Los nórdicos también muestran las medias más altas de la escala 0-10 (Islandia 6,96).

**Bloque medio (50-70 %): Europa occidental continental.**
Países Bajos, Irlanda, Bélgica, Alemania. Alemania (60,8 %) sorprende relativamente bajo dado su PIB y educación: posible efecto de la ola migratoria 2015-16 y polarización política reciente.

**Bloque dividido (40-55 %): Europa del sur y este post-soviético.**
Portugal, Francia, Italia, Polonia, países bálticos, Ucrania, Eslovenia. Francia (53,4 %) destaca por ser una economía grande con valores intermedios — la división interna del país se refleja en la cifra.

**Bloque anti-inmigración (<40 %): Europa central/oriental y mediterráneo oriental.**
Hungría, Eslovaquia, Bulgaria, Grecia, Chipre, Montenegro, Letonia. **Hungría es especialmente notable**: a pesar de tener confianza institucional media (5,04) — explicable por el control gubernamental sobre los medios — la postura anti-inmigración es la 2ª más alta. La narrativa política de Orbán sobre inmigración pesa por encima del modelo.

### 5.3 Casos atípicos que el modelo destaca

Los **dummies de país con mayor importancia** en el RF son los que el modelo necesita para *corregir* la predicción base:

- **`cntry_GR` (Grecia)** — la 4ª variable más importante en el RF y en SHAP. El modelo "aprende" que en Grecia, incluso controlando por confianza/educación/ideología, la probabilidad pro-inmigración es **mucho más baja** de lo que esas variables predirían. Posible explicación: presión migratoria por ser frontera UE-Turquía y crisis de refugiados 2015 todavía presente.
- **`cntry_HU` (Hungría)** y **`cntry_SK` (Eslovaquia)** — mismo patrón: efecto país anti-inmigración fuerte por encima de lo que predicen los controles.
- **`cntry_UA` (Ucrania)** — efecto país **positivo** importante: con `trust_index = 2,56` (el más bajo del estudio), el modelo predeciría una postura muy anti-inmigración; pero Ucrania está en 52,4 %, así que el dummy compensa. Contexto plausible: la guerra y la aspiración europea han cambiado las actitudes hacia la inmigración (los propios ucranianos refugiados, además, generan empatía con la categoría "migrante").
- **`cntry_ES` (España)** — efecto país positivo: España puntúa **alto en pro-inmigración (71,1 %) a pesar de tener confianza institucional media-baja (4,82)**. El modelo añade un "bonus España".

### 5.4 Correlación país-nivel

A nivel agregado por país, la correlación entre confianza media y % pro-inmigración es **r = 0,63** (positiva, fuerte). Esto refuerza el hallazgo individual pero también muestra que **el país agrega varianza no explicada por confianza** (de ahí r = 0,63 y no 0,9). Casos que rompen la tendencia:

- **Hungría:** confianza alta (5,04) pero solo 24,2 % pro. Cae muy por debajo de la línea de tendencia.
- **Ucrania:** confianza muy baja (2,56) pero 52,4 % pro. Está muy por encima de la tendencia.
- **España:** confianza media-baja (4,82) pero 71,1 % pro. También por encima.

Ver [`trust_vs_proimm_paises.png`](trust_vs_proimm_paises.png).

---

## 6. Explicabilidad SHAP

Aplicado al Random Forest sobre 2.000 casos de test:

- **`trust_index`** es la variable con mayor magnitud SHAP media (0,092). Su **dependence plot** (ver [`shap_dependence_trust.png`](shap_dependence_trust.png)) muestra una **relación monótona creciente**: valores bajos de confianza empujan la predicción hacia "anti-inmigración"; valores altos, hacia "pro-inmigración". El cambio es **aproximadamente lineal**, lo que explica por qué la LogReg empata con el RF.
- **`eduyrs`** tiene relación positiva clara.
- **`lrscale`** muestra relación negativa (más derecha → menos pro-inmigración).
- **`agea`** y **`hinctnta`** tienen efectos menores y más heterogéneos (la dispersión en el beeswarm es alta).
- Los **dummies país** (GR, HU, SK, BG) aparecen en SHAP como efectos negativos consistentes; ES, FI, CH como positivos.

Ver [`shap_summary_beeswarm.png`](shap_summary_beeswarm.png) y [`shap_waterfall_ejemplo.png`](shap_waterfall_ejemplo.png) para un caso individual.

📄 Tabla: [`tabla_shap_importancia.csv`](tabla_shap_importancia.csv).

---

## 7. Conclusiones

1. **La confianza institucional es el predictor individual más fuerte** de la postura pro-inmigración en Europa, por encima de ideología, educación, edad e ingresos. La asociación es **positiva, monótona y robusta** entre los tres enfoques (LogReg, RF, SHAP).
2. **Educación e ideología** son los siguientes factores en importancia, con el signo esperado por la literatura: más educación → más apertura; más derecha → menos apertura.
3. **El país añade información que las variables individuales no capturan.** Hungría, Eslovaquia, Grecia y Bulgaria son **más anti-inmigración** de lo que sus indicadores predicen; España y Ucrania son **más pro-inmigración** de lo esperado. Esto sugiere que **el contexto histórico, mediático y político nacional** modula fuertemente las actitudes — algo que un modelo con solo variables individuales no puede explicar.
4. **AUC = 0,75 sugiere techo**: con estos predictores se llega hasta aquí. Para mejorar habría que incorporar variables sobre **contacto directo con inmigrantes**, **consumo de medios**, **posición laboral expuesta a competencia migratoria** y **percepción económica subjetiva**.
5. **Implicación de política pública:** reforzar la confianza institucional (transparencia, justicia, eficacia policial) tendría como **externalidad positiva una sociedad más receptiva a la inmigración**. El vínculo no es retórico: es la asociación más fuerte de las medidas.

---

## Anexo: archivos generados

| Archivo | Descripción |
|---|---|
| [`tabla_resultados_modelos.csv`](tabla_resultados_modelos.csv) | Métricas CV + Test de ambos modelos |
| [`tabla_resultados_modelos.png`](tabla_resultados_modelos.png) | Versión visual para informe |
| [`matriz_confusion.png`](matriz_confusion.png) | Matrices de confusión en test |
| [`curva_roc.png`](curva_roc.png) | Curvas ROC en test |
| [`coeficientes_logreg.csv`](coeficientes_logreg.csv) | Coeficientes estandarizados + odds ratios |
| [`importancias_rf.csv`](importancias_rf.csv) / [`.png`](importancias_rf.png) | Importancia Gini del RF |
| [`tabla_shap_importancia.csv`](tabla_shap_importancia.csv) | \|SHAP\| medio por variable |
| [`shap_summary_bar.png`](shap_summary_bar.png) | Importancia SHAP global |
| [`shap_summary_beeswarm.png`](shap_summary_beeswarm.png) | Efecto + dirección de cada variable |
| [`shap_dependence_trust.png`](shap_dependence_trust.png) | Cómo cambia el efecto del `trust_index` |
| [`shap_waterfall_ejemplo.png`](shap_waterfall_ejemplo.png) | Explicación de un caso individual |
| [`tabla_por_pais.csv`](tabla_por_pais.csv) | Estadísticas descriptivas por país |
| [`pro_inmigracion_por_pais.png`](pro_inmigracion_por_pais.png) | Ranking visual de países |
| [`trust_vs_proimm_paises.png`](trust_vs_proimm_paises.png) | Scatter confianza vs. pro-inmigración por país |
