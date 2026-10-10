# Primera Competencia DMEyF 2026 — Ana Dominguez

**Entrega final:** `entrega_final/FINAL_e6019x40_N11000.csv`
(11.000 `numero_de_cliente` de 202108, sin encabezado).

## Qué es la predicción final

Es un **ensamble por promedio de 40 modelos LightGBM**. Todos tienen las mismas variables y los mismos hiperparámetros; solo cambia la semilla:

| Parte | Notebook | Qué aporta |
|---|---|---|
| 1 | `1_hiperparametros_6005/` | Bayesian Optimization que dio los hiperparámetros (`BO_log.txt`) |
| 2 | `2_modelo_6018/competencia01_gcp_6018_rank_cero_fijo.ipynb` | 20 modelos (bagging, 20 semillas) → `exp/HT6018/predicciones_futuro.csv` |
| 3 | `3_modelo_6019_y_final/competencia01_gcp_6019_6018_semillas_nuevas.ipynb` | 20 modelos más (20 semillas nuevas) + **promedio de los 40** → archivo final |

La probabilidad final de cada cliente es el promedio simple de las probabilidades de los 40 modelos (20 del 6018 + 20 del 6019). Se ordenan de mayor a menor y se envían los primeros 11.000.

### Detalle del modelo (6018 y 6019 son idénticos salvo las semillas)
- **Clase:** `clase01` = BAJA+1 + BAJA+2.
- **Meses:** entrenamiento final 202103–202106, predicción 202108 (202107 no se usa: no tiene clase).
- **Variables:**
  - aguinaldo sin datos del futuro (`mpayroll_sin_sac`, `cobra_aguinaldo`);
  - lags y deltas de orden 1 para todas las variables;
  - lag y delta de orden 2 para una lista fija de 78 variables (escrita en la notebook);
  - 62 rankings "con cero fijo" dentro de cada mes (`rk_*`): 0 → 0, positivos en (0,1], negativos en [−1,0).
- **Entrenamiento:** cada modelo usa todas las bajas + un 25% de los CONTINUA, elegido con su propia semilla (`set.seed(semilla, kind="Mersenne-Twister")`).
- **Hiperparámetros:** la mejor fila de `exp/HT6005/BO_log.txt` (num_iterations, learning_rate, feature_fraction, num_leaves, min_data_in_leaf).
- **Semillas:**
  - 6018: las 10 de `PARAM$semillas` + 10 de `set.seed(992720)`;
  - 6019: 20 nuevas de `set.seed(992720 + 6019)`, sin repetir ninguna del 6018.

### De dónde sale cada parte
Las variables del 6018 se fueron sumando en experimentos anteriores: 6006 (bagging), 6011 (orden 2 selectivo), 6016 (rankings), 6018 (rankings con cero fijo). Esas notebooks **no hacen falta para replicar**: la notebook del 6018 ya incluye todo ese feature engineering, y ninguna de ellas genera un archivo que use la entrega final. Lo único que se reutiliza de un experimento anterior son los hiperparámetros del 6005.

## Cómo replicar (en orden)

Las notebooks están en **R** y esperan la estructura del curso en Google Cloud: `BASE <- ~/buckets/b1`. Cada notebook escribe en `~/buckets/b1/exp/HTxxxx/`. El dataset se descarga solo desde la URL de la competencia.

Máquina usada: Google Cloud, e2 custom de 8 vCPU y 64 GB de RAM (template del curso `to-064ram-08vcpu`).

**Paso 1 — Hiperparámetros (6005).** La notebook `1_hiperparametros_6005/competencia01_gcp_6005.ipynb` corre una Bayesian Optimization de 30 iteraciones (entrena 202103–202105 con 25% de CONTINUA, mide la ganancia en 202106) y escribe `exp/HT6005/BO_log.txt`. La mejor fila, que es la que usan el 6018 y el 6019, es:

| num_iterations | learning_rate | feature_fraction | num_leaves | min_data_in_leaf |
|---|---|---|---|---|
| 1168 | 0.01315415 | 0.3446377 | 61 | 966 |

 Hay dos opciones:
- **A (rápida, la recomendada):** copiar `1_hiperparametros_6005/BO_log.txt` a `~/buckets/b1/exp/HT6005/BO_log.txt`. Las notebooks 6018 y 6019 solo leen la mejor fila de ese archivo.
- **B (completa):** correr la notebook de `1_hiperparametros_6005/`, que genera ese mismo `BO_log.txt`. Tarda varias horas.

**Paso 2 — 6018.** Abrir `competencia01_gcp_6018_rank_cero_fijo.ipynb` y hacer **Run All** (~1,5–2,5 h).
Genera `exp/HT6018/predicciones_futuro.csv`, con las probabilidades de los 20 modelos para 202108.

**Paso 3 — 6019 y archivo final.** Abrir `competencia01_gcp_6019_6018_semillas_nuevas.ipynb` y hacer **Run All** (~1,5–2,5 h).
Entrena los 20 modelos nuevos y, en la última celda de código, promedia con los 20 del 6018 (lee `exp/HT6018/predicciones_futuro.csv`) y escribe:

```
exp/HT6019/zulip/FINAL_e6019x40_N11000.csv   <- la entrega final
```

**Verificación:** comparar ese archivo con `entrega_final/FINAL_e6019x40_N11000.csv`.

### Notas
- Las celdas que comparan contra otros experimentos (6006, 6016) son solo informativas. Si esos archivos no existen, se saltean solas.
- LightGBM corre multihilo, así que en otra máquina puede haber diferencias ínfimas en las probabilidades y, como mucho, un puñado de clientes distintos cerca del corte 11.000.
- Si una corrida se corta, se puede volver a hacer **Run All**: las bolsas ya guardadas se saltean.
