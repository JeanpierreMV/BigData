# BigData
Este proyecto utiliza la API de OpenWeather para obtener datos meteorológicos diarios de diferentes lugares y los procesa utilizando PySpark. A continuación, se presenta una guía sobre cómo configurar y ejecutar este proyecto.

# Instalación de Dependencias
Se debe tenere estas bibliotecas instaladas antes de ejecutar el código:

--pip install pyspark py4j

--pip install geopy

# Configuración
es necesario agregar más lugares si es necesario. Puedes hacerlo editando la lista de lugares en el código.

---Lista de lugares con sus cordenadas


lugares = [
    {"nombre": "Amazonas/Peru", "lat": -4.999999999682269, "lon": -78},
    {"nombre": "Ancash/Peru", "lat": -9.499999999871093, "lon": -77.75},
    # Agrega más lugares si es necesario
]

y por supuesto tener acceso a la url de la api a la cual se le puede llamar de las diferentes formas

https://api.openweathermap.org/data/3.0/onecall/day_summary?lat=39.099724&lon=-94.578331&dt=1643803200&appid={API key}


https://api.openweathermap.org/data/3.0/onecall/timemachine?lat=39.099724&lon=-94.578331&dt=1643803200&appid={API key}


https://api.openweathermap.org/data/3.0/onecall?lat=33.44&lon=-94.04&appid={API key}


para este caso estamos utilizando 
https://api.openweathermap.org/data/3.0/onecall?lat=33.44&lon=-94.04&appid={API key}

luego utilizamo la biblioteca de panadas para crear un dataframe y luego lo comvertimos a spark


    spark_df = spark.createDataFrame(df)

 luego de aplicar todo el codigo como esta en el archivo le debe aparecer un resultado parecido a este
 
![image](https://github.com/JeanpierreMV/BigData/assets/101939039/9766b88a-51ef-4020-bb77-ec29c41c9d25)
   

# Creación de Dimensiones y Tabla de Hechos
Se han creado las siguientes dimensiones basadas en los datos meteorológicos:

1. Dimensión Tiempo (df_dim_tiempo)
Esta dimensión contiene información temporal, como la fecha (dt), el amanecer (sunrise), el atardecer (sunset), y el nombre del lugar (nombre_lugar).



df_dim_tiempo = data_dim_tiempo.select("idTiempo", "dt", "sunrise", "sunset", "nombre_lugar")


2. Dimensión Ubicación (df_dim_ubicacion)
La dimensión de ubicación incluye detalles geográficos, como la región (region), el país (country), y el nombre del lugar (nombre_lugar).


df_dim_ubicacion = data_dim_ubicacion.select("idUbicacion", "region", "country", "nombre_lugar")

3. Dimensión Clima (df_dim_clima)
La dimensión de clima presenta información resumida sobre las condiciones climáticas, como el resumen (summary).


df_dim_clima = data_dim_clima.select("idClima", "summary")


4. Dimensión Temperatura (df_dim_Temperatura)
Esta dimensión contiene datos relacionados con la temperatura, como la temperatura máxima, mínima y promedio durante el día y la noche.


df_dim_Temperatura = data_dim_Temperatura.select(
    "idTemperatura", "temp_day", "temp_min", "temp_max", "temp_night", "temp_eve", "temp_morn"
)


5. Dimensión Luna (df_dim_Luna)
La dimensión Luna proporciona información sobre el amanecer (moonrise), el atardecer (moonset), y la fase lunar (moon_phase).


df_dim_Luna = data_dim_Luna.select(
    "idLuna", "moonrise", "moonset", "moon_phase"
)


# Tabla de Hechos
La tabla de hechos (fact_table) se ha construido mediante la unión de las dimensiones y eliminando duplicados:


fact_table_temp = consolidated_df.join(df_dim_tiempo, "dt", 'inner') \
    .select(consolidated_df["*"], df_dim_tiempo["idTiempo"].alias("idTiempoTemp"))


fact_table_temp = fact_table_temp.join(df_dim_ubicacion, ["region", "country"], 'inner') \
    .select(fact_table_temp["*"], df_dim_ubicacion["idUbicacion"].alias("idUbicacionTemp"))




fact_table = fact_table_temp \
    .select('idTiempoTemp', 'idUbicacionTemp', 'idTemperatura', 'idLuna', 'idClima')


fact_table = fact_table.withColumn('id', monotonically_increasing_id())


columns = ['id'] + [col for col in fact_table.columns if col != 'id']
fact_table = fact_table.select(columns)


# Persistencia en Hive

Finalmente, las dimensiones y la tabla de hechos se han almacenado en Hive para facilitar su consulta:


openweatherHive.sql("CREATE DATABASE IF NOT EXISTS dimensiones")


df_dim_tiempo.write.saveAsTable("dimensiones.dimension_tiempo")
df_dim_ubicacion.write.saveAsTable("dimensiones.dimension_ubicacion")
df_dim_clima.write.saveAsTable("dimensiones.dimension_clima")
df_dim_Temperatura.write.saveAsTable("dimensiones.dimension_temperatura")
df_dim_Luna.write.saveAsTable("dimensiones.dimension_luna")



# Resultados
Después de ejecutar el código, se creará un DataFrame de Spark consolidado que contiene información meteorológica de todos los lugares especificados. La información consolidada se mostrará en la consola, incluyendo el esquema y los primeros registros.
