```python
import pandas as pd
import seaborn as sns 
from matplotlib import pyplot as plt
import plotly
from plotly import express as px
import numpy as np
import sqlite3
from sklearn.linear_model import LinearRegression
from calendar import month_name
```

# 1. Create a database

We would like to create a database, `temps.db`, containing 3 tables: `temperatures`, `stations`, and `countries`. 



```python
data_path = "/Users/lzj/work/pic16b/blog1data/"
#open database connection
conn = sqlite3.connect(data_path + "temps.db") 

conn.execute("DROP TABLE IF EXISTS temperatures") 
#ensure we don't later append to an existing copy
```




    <sqlite3.Cursor at 0x7f7c88fa7030>



After opening the database connection, we read the temperatures data from a `.csv` file. As this file is large, we read it into the database in chunks of 100000 rows. We will use the `prepare_df` function to clean up the data chunks as we read it in.


```python
df_iter = pd.read_csv(data_path + "temps.csv", chunksize = 100000)
```


```python
# cleans temps data 
def prepare_df(df):
    df = df.set_index(keys=["ID", "Year"])
    df = df.stack()
    df = df.reset_index()
    df = df.rename(columns = {"level_2"  : "Month" , 0 : "Temp"})
    df["Month"] = df["Month"].str[5:].astype(int)
    df["Temp"]  = df["Temp"] / 100
    
    return(df)
```


```python
for df in df_iter:
    df = prepare_df(df)
    df.to_sql("temperatures", conn, if_exists = "append", index = False)
    
```

The data on stations and countries are smaller and don't have to be read in chunks.


```python
stations = pd.read_csv(data_path + "station-metadata.csv")
# create FIPS code column for merging later
stations["FIPS 10-4"] = stations["ID"].str[0:2]
stations.to_sql("stations", conn, if_exists = "replace", index = False)
```

    /Users/lzj/opt/anaconda3/envs/PIC16B/lib/python3.8/site-packages/pandas/core/generic.py:2872: UserWarning: The spaces in these column names will not be changed. In pandas versions < 0.14, spaces were converted to underscores.
      sql.to_sql(



```python
countries =  pd.read_csv(data_path + "countries.csv")
countries.to_sql("countries", conn, if_exists = "replace", index = False)
```


```python
conn.close()
```

# 2. Query the database

We can write a function that uses a SQL command to extract a Pandas dataframe from our database. We use the `LEFT JOIN` keyword to merge the three sources of data, and the `WHERE` to specify the conditions (here, a country, a range of years, and a specific month).


```python
def query_climate_database(country, year_begin, year_end, month):
    conn = sqlite3.connect(data_path +"temps.db")
    cursor = conn.cursor()
    cmd = \
    """
    SELECT S.name, S.latitude, S.longitude, C.name, T.year, T.month, T.temp
    FROM temperatures T
    LEFT JOIN stations S ON T.id = S.id
    LEFT JOIN countries C ON C.'FIPS 10-4' = S.'FIPS 10-4'
    WHERE (T.year >= """ + str(year_begin) + """ AND T.year <= """+str(year_end)+""") 
          AND T.month ="""+ str(month) +""" AND C.Name = '""" +country+"""'"""

    df = pd.read_sql_query(cmd, conn)
    return df
```


```python
india_df = query_climate_database(country = "India", 
                       year_begin = 1980, 
                       year_end = 2020,
                       month = 1)
india_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>NAME</th>
      <th>LATITUDE</th>
      <th>LONGITUDE</th>
      <th>Name</th>
      <th>Year</th>
      <th>Month</th>
      <th>Temp</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>PBO_ANANTAPUR</td>
      <td>14.583</td>
      <td>77.633</td>
      <td>India</td>
      <td>1980</td>
      <td>1</td>
      <td>23.48</td>
    </tr>
    <tr>
      <th>1</th>
      <td>PBO_ANANTAPUR</td>
      <td>14.583</td>
      <td>77.633</td>
      <td>India</td>
      <td>1981</td>
      <td>1</td>
      <td>24.57</td>
    </tr>
    <tr>
      <th>2</th>
      <td>PBO_ANANTAPUR</td>
      <td>14.583</td>
      <td>77.633</td>
      <td>India</td>
      <td>1982</td>
      <td>1</td>
      <td>24.19</td>
    </tr>
    <tr>
      <th>3</th>
      <td>PBO_ANANTAPUR</td>
      <td>14.583</td>
      <td>77.633</td>
      <td>India</td>
      <td>1983</td>
      <td>1</td>
      <td>23.51</td>
    </tr>
    <tr>
      <th>4</th>
      <td>PBO_ANANTAPUR</td>
      <td>14.583</td>
      <td>77.633</td>
      <td>India</td>
      <td>1984</td>
      <td>1</td>
      <td>24.81</td>
    </tr>
  </tbody>
</table>
</div>



# 3. Creating a geographic scatterplot


By extracting relevant subsets of our database, we can see how average yearly changes in temperature vary within a given country.

We first define a function to perform a linear regression of the temperature on the year, and return the coefficients.


```python
def coef(data_group):
    '''
    perform linear regression on a dataframe with year and temp.
    returns the coefficients
    '''
    x = data_group[["Year"]] 
    y = data_group["Temp"] 
    LR = LinearRegression()
    LR.fit(x, y)
    return LR.coef_[0]
```

Now, we can write a function to plot the temperature coefficients in a geographic scatterplot using `plotly`'s `scatter_mapbox`. The coefficients are represented as the color.


```python
def temperature_coefficient_plot(country, year_begin, year_end, month, min_obs, 
                                 mapbox_style="carto-positron",
                                 zoom = 2,
                                 color_continuous_scale=px.colors.diverging.RdGy_r, 
                                 opacity=1,
                                 **kwargs):
    '''
    plot temperature coefficients in a geographic scatterplot.
    '''
    df = query_climate_database(country, year_begin, year_end, month)
    coefs = df.groupby(["NAME"]).apply(coef)
    coefs = coefs.reset_index()
    coefs = coefs.rename(columns={0:"Estimated yearly increase (\N{DEGREE SIGN}C)"})
    min_filter = df.groupby(["NAME"]).count()["Name"]>=min_obs
    coords = df.groupby(["NAME"]).first(min_count=min_obs)[["LATITUDE","LONGITUDE"]][min_filter]
    x = pd.merge(coefs,coords,on=["NAME"])
    title = "Estimates of yearly increase in temperature in "\
        + month_name[month] + " for stations in "  + country + ", years " + str(year_begin) + "-" + str(year_end)
    
    fig = px.scatter_mapbox(x, 
                            lat = "LATITUDE",
                            lon = "LONGITUDE", 
                            hover_name = "NAME",
                            color = "Estimated yearly increase (\N{DEGREE SIGN}C)",
                            hover_data={
                                "LATITUDE":':.3f',
                                "LONGITUDE":':.3f',
                                "Estimated yearly increase (\N{DEGREE SIGN}C)":':.3f'
                            },
                            title = title,
                            zoom = zoom,
                            color_continuous_scale=color_continuous_scale,
                            mapbox_style = mapbox_style,
                            color_continuous_midpoint=0,
                            opacity=opacity
                            )

    return fig

```


```python
fig=temperature_coefficient_plot("India", 1980, 2020, 1, 10)
fig.show()
```


```python
fig = temperature_coefficient_plot("United Kingdom", 1980, 2020, 1, 10, zoom=3)
fig.show()
```

# 4a

### Plotting within-year temperature differences

Different locations exhibit different seasonal differences in temperatures. Generally, coastal places have milder winters and summers, so they tend to have a narrower within-year temperature range. We can use another geographical scatterplot to better understand how the yearly temperature range vary across a country's geography.


```python
def query_country_year(country, year):
    '''
    returns dataframe of all temperatures in a country in a single year.
    '''
    conn = sqlite3.connect(data_path +"temps.db")
    cursor = conn.cursor()
    cmd = \
    """
    SELECT S.name, S.latitude, S.longitude, C.name, T.year, T.month, T.temp
    FROM temperatures T
    LEFT JOIN stations S ON T.id = S.id
    LEFT JOIN countries C ON C.'FIPS 10-4' = S.'FIPS 10-4'
    WHERE (T.year == """ + str(year)+""" AND C.Name = '""" +country+"""')"""

    df = pd.read_sql_query(cmd, conn)
    return df
```


```python
def year_diff(df):
    '''
    computes temperature difference between hottest and coldest month
    '''
    df_group = df.groupby(["NAME"])
    obs_filter = df_group.count()["Name"]>=12
    temp_diff = (df_group.max()[['Temp']] - df_group.min()[['Temp']])[obs_filter]
    coords = df.groupby(["NAME"]).first()[["LATITUDE","LONGITUDE"]][obs_filter]
    x = pd.merge(temp_diff,coords,on='NAME')
    x = x.rename(columns={'Temp':"Temperature difference"})
    x = x.reset_index()
    return x
```

We've defined `query_country_year` to extract the relevant data, and pass the output to `year_diff` to calculate the difference between the average temperature in the hottest month and the coldest month, at the station level. The output can be used in `seasonal_difference_plot` below. 


```python
def seasonal_difference_plot(country, year, 
                             mapbox_style="carto-positron",
                             color_continuous_scale=px.colors.sequential.dense, 
                             zoom=2,
                             **kwargs):
    x = year_diff(query_country_year(country, year))
    title = "Difference in temperature between hottest and coolest month in "\
            + str(year) + " for stations in "  + country 
    fig = px.scatter_mapbox(x, 
                            lat = "LATITUDE",
                            lon = "LONGITUDE", 
                            hover_name = "NAME",
                            color = "Temperature difference",
                            hover_data={
                                "LATITUDE":':.3f',
                                "LONGITUDE":':.3f',
                                "Temperature difference":':.3f'
                            },
                            title = title,
                            zoom=zoom,
                            color_continuous_scale=color_continuous_scale,
                            mapbox_style = mapbox_style,
                            **kwargs
                            )

    return fig

```


```python
fig = seasonal_difference_plot('Russia', 2020, 
                               mapbox_style="carto-positron",
                               zoom=1)
fig.show()

```


```python
fig = seasonal_difference_plot('Australia', 1970)
fig.show()

```

In the first plot, we can see how European Russia is relatively mild compared to the extreme temperature ranges experienced in Siberia. The second plot shows the contrast between coastal and inland Australia as ocean's heat capacity buffers the seasonal variations in temperature.

# 4b

### Seasonal line graphs by decade

Having explored seasonal temperature ranges, perhaps we would like to visualize how it evolves across time. We will attempt to produce a plot that visualizes the trend across two time scales - (i) seasonality within a year, and (ii) long-term trend across decades. 


```python
def query_station(station):
    '''
    extract all data for a given station
    '''
    conn = sqlite3.connect(data_path +"temps.db")
    cursor = conn.cursor()
    cmd = \
    """
    SELECT S.name, S.latitude, S.longitude, C.name, T.year, T.month, T.temp
    FROM temperatures T
    LEFT JOIN stations S ON T.id = S.id
    LEFT JOIN countries C ON C.'FIPS 10-4' = S.'FIPS 10-4'
    WHERE S.name = '""" +station+"""'"""

    df = pd.read_sql_query(cmd, conn)
    return df
```


```python
def time_trends(df):
    '''
    identify decades and compute mean temperatures by decade and month
    '''
    df['decade'] = df[['Year']]//10 * 10
    #only keep decades with at least 80 observations
    obs = df.groupby(['decade']).transform(np.size)['Year'] >= 80
    df = df.loc[obs,]
    g = df.groupby(['Month','decade']).mean()
    g = g.reset_index()
    return g
```

Having created a column for the decade of each observation, we can now plot the mean seasonal trend for each decade in facets. Facets are subplots for different subsets of the data. Setting the option `facet_col='decade'` lets us plot each decade in separate columns, while the shared y-axis allows us to compare the trend across decades.


```python
def plot_time_trends(station, **kwargs):
    '''plot time trends in decade facets'''
    z = time_trends(query_station(station))
    fig = px.line(z, x='Month', y='Temp', facet_col='decade',
                  title = "Seasonal temperature trend by decade in station: "+station,
                 **kwargs)
    return fig
```


```python
fig = plot_time_trends('HEATHROW')

fig.show()
```

We can see that the peak monthly mean temperatures have gone up in London Heathrow in recent decades. With a slight modification, we can also combine the decades into the same graph, identifying each decade by its color. 


```python
def plot_combined_decades(station, color_discrete_sequence= px.colors.sequential.Oranges, **kwargs):
    '''plot time trends, labelling decades with colors'''
    z = time_trends(query_station(station))
    fig = px.line(z, x="Month", y="Temp", color='decade',                  
                  title = "Seasonal temperature trend by decade in station: "+station,
                  color_discrete_sequence=color_discrete_sequence,
                 **kwargs)
    return fig

```


```python
fig = plot_combined_decades('ST_PETERSBURG',
                            color_discrete_sequence= px.colors.sequential.Plotly3_r,
                            template='plotly_white')
fig.show()
```
