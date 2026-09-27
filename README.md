# ConnectaTel — Análisis de comportamiento y segmentación de clientes

## Descripción

Este proyecto analiza el comportamiento de los clientes de **ConnectaTel**, una empresa de telecomunicaciones en Latinoamérica. El objetivo es explorar, limpiar y transformar información de clientes, planes y uso de servicios para construir un perfil analítico por usuario, detectar comportamientos atípicos, crear segmentos y proponer acciones de negocio.

El análisis trabaja con información registrada hasta **2024** y fue desarrollado en Python con **pandas, NumPy, Seaborn y Matplotlib**.

---

## Objetivos

- Evaluar la calidad de los datos.
- Identificar valores nulos, sentinels y fechas inválidas.
- Analizar el comportamiento de llamadas y mensajes.
- Construir métricas agregadas por cliente.
- Comparar distribuciones según el plan contratado.
- Detectar y evaluar outliers.
- Segmentar clientes por edad y nivel de uso.
- Traducir los hallazgos en recomendaciones comerciales.

---

## Datos utilizados

| Dataset | Registros | Columnas | Descripción |
|---|---:|---:|---|
| `plans.csv` | 2 | 8 | Características y precios de los planes |
| `users_latam.csv` | 4.000 | 8 | Información demográfica y contractual |
| `usage.csv` | 40.000 | 6 | Registros de llamadas y mensajes |

### Planes

| Plan | Mensajes | GB/mes | Minutos | Pago mensual USD | USD/GB extra | USD/mensaje extra | USD/minuto extra |
|---|---:|---:|---:|---:|---:|---:|---:|
| Básico | 100 | 5 | 100 | 12 | 1.20 | 0.08 | 0.10 |
| Premium | 500 | 20 | 600 | 25 | 1.00 | 0.05 | 0.07 |

La base está compuesta aproximadamente por **64,88 % de usuarios Básico** y **35,13 % Premium**.

---

## Tecnologías

- Python
- pandas
- NumPy
- Seaborn
- Matplotlib
- Jupyter Notebook

---

# Metodología

El proyecto se desarrolló en siete etapas:

1. Carga y exploración.
2. Identificación de problemas de calidad.
3. Limpieza y estandarización.
4. Agregación del uso por cliente.
5. Análisis de distribuciones y outliers.
6. Segmentación de clientes.
7. Elaboración de insights y recomendaciones.

---

# 1. Exploración inicial

Se revisaron las dimensiones, tipos de datos, valores faltantes, variables categóricas y numéricas, identificadores y columnas temporales.

Los campos `id` y `user_id` se trataron como **identificadores**, no como variables cuantitativas.

---

# 2. Calidad de datos

## Valores faltantes

### `users`

| Variable | Nulos | Proporción | Decisión |
|---|---:|---:|---|
| `city` | 469 | 11,73 % | Investigar / estandarizar ausencia |
| `churn_date` | 3.534 | 88,35 % | Conservar; puede representar cliente activo |

Además, `city` contenía **96 registros con `?`**, por lo que alrededor de **14,13 % de los clientes no tenían una ciudad válida** al considerar nulos y sentinels.

### `usage`

| Variable | Nulos | Proporción | Decisión |
|---|---:|---:|---|
| `date` | 50 | 0,125 % | Revisar/eliminar para análisis temporal |
| `duration` | 22.076 | 55,19 % | Conservar como nulo estructural |
| `length` | 17.896 | 44,74 % | Conservar como nulo estructural |

Los nulos de `duration` y `length` dependen principalmente del tipo de evento:

- una llamada utiliza `duration`;
- un mensaje utiliza `length`.

Por ello, no se imputaron con cero en la tabla transaccional.

---

## Sentinels y valores inválidos

### `age`

Se detectó el valor sentinel `-999`. El tratamiento fue:

1. convertir `-999` en `NaN`;
2. calcular la mediana de las edades válidas;
3. imputar los faltantes con la mediana.

Después de la limpieza:

- mínimo: **18 años**;
- mediana: **48 años**;
- media: **48,14 años**;
- máximo: **79 años**.

### `city`

El sentinel `?` fue reemplazado por `pd.NA`.

### `reg_date`

Se encontraron **40 registros de 2026**, equivalentes al **1 %** de los usuarios, aunque el proyecto solo contempla información hasta 2024. Estas fechas se marcaron como `NaT`.

En `usage`, los **39.950 registros con fecha válida corresponden a 2024**.

---

# 3. Construcción del perfil por cliente

`usage` registra eventos individuales. Para analizar comportamiento por cliente se crearon indicadores:

```python
usage["is_text"] = (usage["type"] == "text").astype(int)
usage["is_call"] = (usage["type"] == "call").astype(int)
```

Luego se agregó la información por `user_id` para obtener:

- `cant_mensajes`
- `cant_llamadas`
- `cant_minutos_llamada`

La tabla resultante se combinó con `users` mediante un **left join**, creando `user_profile`: una tabla con una fila por cliente.

Cuando un usuario no presentó registros agregados de uso, las métricas de consumo se llevaron a cero a nivel de `user_profile`.

---

# 4. Perfil estadístico

| Métrica | Edad | Mensajes | Llamadas | Minutos de llamada |
|---|---:|---:|---:|---:|
| Media | 48,14 | 5,52 | 4,48 | 23,31 |
| Desv. estándar | 17,69 | 2,36 | 2,15 | 18,17 |
| Mínimo | 18 | 0 | 0 | 0 |
| Q1 | 33 | 4 | 3 | 11,11 |
| Mediana | 48 | 5 | 4 | 19,78 |
| Q3 | 63 | 7 | 6 | 31,41 |
| Máximo | 79 | 17 | 15 | 155,69 |

### Hallazgos descriptivos

- La edad se distribuye de forma relativamente amplia y sin un sesgo fuerte.
- La cantidad de mensajes presenta una cola hacia consumos altos.
- Los minutos de llamada presentan un **sesgo positivo marcado**.
- Las diferencias visuales entre Básico y Premium deben interpretarse considerando que Básico tiene más clientes.

---

# 5. Visualización

Se utilizaron histogramas con `hue='plan'` para analizar:

- `age`
- `cant_mensajes`
- `cant_llamadas`
- `cant_minutos_llamada`

### Edad

La distribución es relativamente homogénea en el rango de 18 a 79 años. Se observan algunos picos locales, pero las frecuencias absolutas no permiten afirmar por sí solas que un rango etario tenga mayor preferencia por Premium.

### Mensajes

La distribución está **sesgada a la derecha**: la mayoría de clientes registra niveles bajos o medios y un grupo reducido presenta mayor actividad.

### Minutos de llamada

Existe un **sesgo fuerte hacia la derecha**. La mayoría se concentra en consumos moderados, mientras que algunos clientes alcanzan valores cercanos a 130–156 minutos.

---

# 6. Outliers

Se utilizaron boxplots y el método IQR.

| Variable | Q1 | Q3 | IQR | Límite superior | Máximo |
|---|---:|---:|---:|---:|---:|
| `cant_mensajes` | 4,00 | 7,00 | 3,00 | 11,50 | 17 |
| `cant_llamadas` | 3,00 | 6,00 | 3,00 | 10,50 | 15 |
| `cant_minutos_llamada` | 11,11 | 31,41 | 20,31 | 61,87 | 155,69 |

`age` no presentó outliers relevantes después de la limpieza.

## Decisión

Los outliers se conservaron porque:

- son estadísticamente poco frecuentes;
- no son valores imposibles;
- pueden representar clientes intensivos;
- pueden aportar información para segmentación y diseño de producto.

**Outlier estadístico ≠ dato erróneo.**

---

# 7. Segmentación

## Por nivel de uso

```text
Bajo uso
  llamadas < 5 y mensajes < 5

Uso medio
  llamadas < 10 y mensajes < 10,
  sin cumplir previamente Bajo uso

Alto uso
  resto de casos
```

Implementación:

```python
def clasificar_uso(fila):
    if fila['cant_llamadas'] < 5 and fila['cant_mensajes'] < 5:
        return 'Bajo uso'
    elif fila['cant_llamadas'] < 10 and fila['cant_mensajes'] < 10:
        return 'Uso medio'
    else:
        return 'Alto uso'
```

El grupo de **Uso medio** es el segmento visualmente predominante. Alto uso contiene menos clientes, pero es estratégico por su intensidad de consumo.

## Por edad

```text
Joven         → edad < 30
Adulto        → 30 ≤ edad < 60
Adulto mayor  → edad ≥ 60
```

El siguiente paso recomendado es cruzar `grupo_edad × grupo_uso`.

---

# Principales insights

1. Los datos requerían limpieza antes de cualquier análisis: existían sentinels, fechas futuras y ausencias con significados diferentes.
2. No todos los nulos representan errores: `duration` y `length` contienen principalmente valores estructurales.
3. El comportamiento típico es moderado: mediana de 5 mensajes, 4 llamadas y 19,78 minutos.
4. Existen clientes intensivos que deben analizarse como un segmento y no eliminarse automáticamente.
5. El plan Básico concentra la mayor parte de la base.
6. La combinación de edad, uso, plan y churn tiene mayor valor para el negocio que cualquiera de estas variables por separado.

---

# Recomendaciones

## 1. Desarrollar el segmento de Uso medio

Analizar mensualmente a los usuarios que se aproximan a Alto uso y evaluar campañas de migración a ofertas de mayor valor.

Medir:

- migraciones Básico → Premium;
- aceptación de ofertas;
- permanencia después del cambio;
- evolución mensual del consumo.

## 2. Crear una estrategia para usuarios intensivos

Construir una bandera `usuario_intensivo` y analizar:

- plan actual;
- recurrencia del consumo extremo;
- minutos promedio por llamada;
- churn;
- antigüedad.

Este segmento puede ser candidato para Premium, paquetes adicionales o beneficios de fidelización.

## 3. Relacionar Bajo uso con churn

No asumir que bajo uso significa riesgo.

```text
Bajo uso + churn alto  → posible riesgo de abandono
Bajo uso + churn bajo  → cliente estable de necesidades reducidas
```

## 4. Combinar edad, uso, plan y churn

Evitar estrategias basadas únicamente en edad.

Ejemplo:

```text
Adulto mayor + Alto uso
```

debe tratarse de forma diferente a:

```text
Adulto mayor + Bajo uso
```

## 5. Llevar el consumo a nivel mensual

Los beneficios de los planes son mensuales, mientras que `user_profile` resume consumo agregado.

Construir:

```text
user_id + mes + mensajes + llamadas + minutos + plan
```

para medir:

```text
utilización = consumo mensual / capacidad incluida
```

y clasificar:

- subutiliza;
- uso adecuado;
- cercano al límite;
- excede el plan.

## 6. Evaluar un posible plan intermedio

Existe una brecha entre Básico (USD 12) y Premium (USD 25).

Debe investigarse si existe un grupo suficientemente grande de clientes Básico que:

- tenga uso medio/alto recurrente;
- se acerque al límite mensual;
- genere excedentes;
- no migre a Premium.

Solo con esa evidencia debería evaluarse un tercer plan.

## 7. Convertir churn en una variable analítica

```python
user_profile['estado_cliente'] = np.where(
    user_profile['churn_date'].notna(),
    'Inactivo',
    'Activo'
)
```

Después analizar:

```text
grupo_uso × estado_cliente
grupo_edad × estado_cliente
plan × estado_cliente
```

## 8. Mejorar calidad desde el origen

Implementar reglas como:

```text
age       → rango válido
city      → catálogo estandarizado
reg_date  → no aceptar fechas futuras
plan      → catálogo cerrado
type      → call / text
call      → duration obligatoria
text      → length obligatoria
```

---

# Próximos pasos

1. Construir métricas mensuales.
2. Cruzar `grupo_edad` y `grupo_uso`.
3. Calcular churn por segmento.
4. Comparar consumo contra beneficios incluidos.
5. Analizar excedentes.
6. Evaluar migraciones de plan.
7. Investigar clientes intensivos.
8. Incorporar consumo de GB si está disponible.
9. Construir un dashboard.
10. Evolucionar hacia modelos de churn o propensión a upgrade.

---

# Limitaciones

- Las métricas actuales están agregadas por usuario y no necesariamente a nivel mensual.
- `usage` no incluye consumo real de GB, aunque los planes sí contemplan esta variable.
- La interpretación de `churn_date = NaN` como cliente activo debe validarse con la definición del negocio.
- Los outliers requieren análisis temporal para determinar si son comportamientos recurrentes.
- `city` mantiene información ausente que limita análisis geográficos.
- La segmentación utiliza reglas determinísticas; posteriormente puede explorarse clustering.

---

# Estructura sugerida

```text
connectatel-analysis/
│
├── README.md
├── S7 Version-Estudiante-Project-ConnectaTel.ipynb
├── data/
│   ├── plans.csv
│   ├── users_latam.csv
│   └── usage.csv
└── outputs/
    ├── figures/
    └── reports/
```

---

# Ejecución

```bash
pip install pandas numpy matplotlib seaborn jupyter
jupyter notebook
```

Abrir y ejecutar:

```text
S7 Version-Estudiante-Project-ConnectaTel.ipynb
```

Si los datasets se encuentran en otra ruta, actualizar los `pd.read_csv()`.

---

# Conclusión

El proyecto transforma datos de planes, clientes y uso en un **perfil analítico por cliente**, recorriendo un flujo completo de exploración, calidad, limpieza, agregación, visualización, detección de outliers y segmentación.

El análisis muestra que la segmentación de ConnectaTel no debería depender únicamente del plan contratado. La combinación de **edad, intensidad de uso, permanencia y plan** puede ofrecer una base más útil para decisiones de retención, upselling y diseño de producto.

La evolución natural del proyecto es llevar el consumo a **granularidad mensual**, contrastarlo con los beneficios de los planes y relacionarlo con churn para transformar el análisis descriptivo en una herramienta de decisión comercial.
