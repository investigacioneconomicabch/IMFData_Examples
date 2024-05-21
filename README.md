# IMFData_Examples

Ejemplos de uso de la librería `IMFData.jl`, a partir del [código](https://github.com/stephenbnicar/IMFData.jl) y [ejemplos](https://github.com/stephenbnicar/IMFData.jl/blob/master/examples/examples.jl) publicados. Al igual que el paquete original, los códigos se desarrollan en **Julia**, para lo cual deben instalarse las librerías detalladas a continuación:

```
#import Pkg; Pkg.add("IMFData")
using CSV, DataFrames, DataFramesMeta, IMFData

wd = @__DIR__
```

`wd`se usa para obtener el código para la ruta de origen (IMFData_Examples, en nuestro caso); en caso que se cambie el archivo de destino, debe revisarse también las rutas de lectura y escritura de los archivos detallados en el archivo `IMFData.qmd`, mismo que contiene ejemplos de utilización de las funciones disponibles en el paquete `IMFData.jl` y los códigos adicionales que permiten obtener varias series y/o indicadores; a continuación se detallan los más relevantes.

## Listar datos que pueden accederse desde la API:

El paquete solamente permite acceder a los datos de "International Financial Statistics (IFS)". El archivo `IMFDatasets.csv` contiene el ID y la descripción de las series.

```
# Get a list of datasets accessible from the API:
data = IMFData.get_imf_datasets()
names(data)
data = DataFrames.DataFrame(
    dataset_id = data.dataset_id,
    dataset_name = data.dataset_name)

df = DataFrames.DataFrame([[],[]], ["dataset_id", "dataset_name"])
for i in 1:size(data)[1]
    push!(df, (data[i,1],data[i,2]))
end
df.dataset_id = string.(df.dataset_id)
df.dataset_name = string.(df.dataset_name)
CSV.write(
    wd * "/IMFDatasets.csv",
    delim = ";",
    df)
```

Este ID se utiliza en códigos posteriores para listar los indicadores que se quieren consultar. Como solamente están disponibles los indicadores IFS, la lista que puede consultarse se guarda en el archivo "IFS_Indicators.csv":

```
ifs_structure  = get_imf_datastructure("IFS")
ifs_indicators = ifs_structure["Parameter Values"]["CL_INDICATOR_IFS"]
CSV.write(
    wd * "/IFS_Indicators.csv",
    delim = ";",
    ifs_indicators)
```

El listado de países disponibles se guarda en el archivo "IFS_Countries":

```
ifs_countries = ifs_structure["Parameter Values"]["CL_AREA_IFS"]
ifs_countries.description
CSV.write(
    wd * "/IFS_Countries.csv",
    delim = ";",
    ifs_countries)
```

La frecuencia de los datos puede generarse con el siguiente código:

```
ifs_frequency = ifs_structure["Parameter Values"]["CL_FREQ"]
```

* A = Annual
* B = Bi-annual
* Q = Quarterly (trimestral)
* M = Monthly (mensual)
* D = Daily (diaria)
* W = Weekly (semanal)

La consulta de los datos se realiza utilizando la función `get_ifs_data`, la cual solicita especificar los siguientes parámetros:

* Países (countries);
* Indicadores (indicators);
* Frecuencia de los datos (A, B, Q, M, D, W); y
* Año de inicio y año final (el resultado será con los datos que estén disponibles dentro del rango especificado).

Para consultar una lista de indicadores para un grupo de países, se construyó una función que consolida la consulta en un solo DataFrame:

```
function get_df(x)
    try
        df = countries_indicators[x].series
        df.country .= countries_indicators[x].area
        df.indicator .= countries_indicators[x].indicator
        df.frequency .= countries_indicators[x].frequency
        return df
    catch
        return DataFrames.DataFrame()
    end
end
```

Para consultar utilizando filtros por indicador ("Gross Domestic Product", "Domestic Currency" como palabras clave) puede usarse lo siguiente:

```
ifs_indicators = ifs_structure["Parameter Values"]["CL_INDICATOR_IFS"]
countries  = ["GT","HN","SV","NI","CR","PA"]
#gdp_indicators = @where(
gdp_indicators = DataFramesMeta.@subset(
	ifs_indicators,
	DataFrames.occursin.("Gross Domestic Product", :description),
	DataFrames.occursin.("Domestic Currency", :description))
indicators = gdp_indicators[!,1]
countries_indicators = get_ifs_data(countries, indicators, "Q", 1900, 2100)
ndf = size(countries_indicators)[1]
df = map(_ -> DataFrames.DataFrame(), 1:ndf)
[df[i] = get_df(i) for i in 1:ndf]
df = vcat(df...)
CSV.write(
    wd * "/IMFData_query.csv",
    delim = ";",
    df)
df
```

El resultado se guarda en el archivo "IMFData_query.csv" y se despliega en un DataFrame que contiene las siguientes columnas:

* date: fecha;
* value: valor;
* country: país;
* indicator: código del indicador; y
* frequency: frecuencia.

Para consultar una serie de indicadores para varios países, se puede guardar ambas variables (indicador, país) en un vector:

```
ifs_indicators = ifs_structure["Parameter Values"]["CL_INDICATOR_IFS"]
countries  = ["GT","HN","SV","NI","CR","PA"]
indicators = ["NGDP_SA_XDC","NI_SA_XDC"]
countries_indicators = get_ifs_data(countries, indicators, "Q", 1900, 2100)
ndf = size(countries_indicators)[1]
df = map(_ -> DataFrames.DataFrame(), 1:ndf)
[df[i] = get_df(i) for i in 1:ndf]
df = vcat(df...)
CSV.write(
    wd * "/IMFData_query.csv",
    delim = ";",
    df)
df
```
