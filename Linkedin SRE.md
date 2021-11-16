```python
import requests
import time
import pandas as pd
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
chrome_service = Service('/Users/rcsousa/Downloads/chromedriver')
import json
from bs4 import BeautifulSoup as soup
import random
import matplotlib.pyplot as plt
import requests
import json
import unidecode
import folium
from folium import plugins
import numpy as np
```

Gera lista de estados e coordenadas


```python
estados = json.loads(requests.get("https://servicodados.ibge.gov.br/api/v1/localidades/estados/").text)
UF = []
estado_nome = []
for x in estados:
    UF.append(x['sigla'])
    estado_nome.append(x['nome'])
    
coordenadas = []
for x in estados:
    coordenadas.append(json.loads(requests.get("http://servicodados.ibge.gov.br/api/v3/malhas/estados/"+x['sigla']+"/metadados").text))


latitude = []
longitude = []
for x in coordenadas:
    latitude.append(x[0]['centroide']['latitude'])
    longitude.append(x[0]['centroide']['longitude'])
    
coord = list(zip(UF, estado_nome, latitude, longitude))
df_coord = pd.DataFrame(coord, columns = ['UF', 'Location', 'latitude', 'longitude'])
coord_plot = list(zip(estado_nome, latitude, longitude))
```

Define parametros de busca


```python
url = 'https://www.linkedin.com/jobs/search?keywords=Site%20Reliability%20Engineer&location=Brasil&geoId=106057199&trk=public_jobs_jobs-search-bar_search-submit&position=1&pageNum=0'
options = Options()
options.headless = False
driver=webdriver.Chrome
s = driver(service=chrome_service, options=options)
n = random.randint(3,7)
s.get(url)
time.sleep(n)
scroll_pause_time = 1 
screen_height = s.execute_script("return window.screen.height;")
i = 1
```

Faz loop automático até último registro publico


```python
while True:
    # scroll one screen height each time
    s.execute_script("window.scrollTo(0, {screen_height}*{i});".format(screen_height=screen_height, i=i))  
    i += 1
    time.sleep(scroll_pause_time)
    # update scroll height each time after scrolled, as the scroll height can change after we scrolled the page
    scroll_height = s.execute_script("return document.body.scrollHeight;")  
    # Break the loop when the height we need to scroll to is larger than the total scroll height
    try:
        s.find_element(By.XPATH, "//button[@class='infinite-scroller__show-more-button infinite-scroller__show-more-button--visible']").click()
    except:
        pass
    if (screen_height) * i > scroll_height:
        break 
```


```python
bsobj = soup(s.page_source, 'html.parser')
```


```python
job_title = []
company = []
location = []
age = []
link = []

for item in bsobj.findAll('h3', {'class' : 'base-search-card__title'}):
    job_title.append(item.get_text().strip())
job_title
    
for item in bsobj.findAll('a', {'class' : 'hidden-nested-link'}):
    company.append(item.get_text().strip())
company
    
for item in bsobj.findAll('span', {'class' : 'job-search-card__location'}):
    location.append(item.get_text().strip())

    
for item in bsobj.findAll('time', {'class' : 'job-search-card__listdate'}):
    age.append(item.get_text().strip())
```


```python
postings = list(zip(job_title, company, location, age))
```


```python
df = pd.DataFrame(postings, columns = ['Job Opening', 'Company', 'Location', 'Age'])
```


```python
s.quit()
```

Consolida estados


```python
localidades = json.loads(requests.get("https://servicodados.ibge.gov.br/api/v1/localidades/municipios/").text)
for x in localidades:
    df.loc[df['Location'].str.contains(unidecode.unidecode(x['nome']), case=False), unidecode.unidecode('Location')] = x['microrregiao']['mesorregiao']['UF']['nome']
    df.loc[df['Location'].str.contains(x['microrregiao']['mesorregiao']['UF']['nome'], case=False), 'Location'] = x['microrregiao']['mesorregiao']['UF']['nome']
```

Tratamento de excessão


```python
df.loc[df['Location'].str.contains('Federal District', case=False), 'Location'] = 'Distrito Federal'
df.loc[df['Location'].str.contains('Ribeirão Preto', case=False), 'Location'] = 'São Paulo'
```


```python
jobs_per_company = df.groupby('Company').count()
```


```python
df = df.merge(df_coord, on='Location', how='left')
```


```python
df = df[df.latitude.notnull()]
```


```python
prep_chart_company = jobs_per_company.sort_values("Job Opening", ascending=False).head(10)
```


```python
df2 = pd.DataFrame(prep_chart_company['Job Opening'])
df2.plot(kind="barh", \
         legend=False, \
         figsize=(24, 12), \
         rot=0, \
         fontsize = 16, \
         sort_columns = False)
```




    <AxesSubplot:ylabel='Company'>




    
![png](output_20_1.png)
    


Gera Total de vagas por localidade


```python
df['Openings'] = df.groupby('Location')['Location'].transform('count')
```

Remove colunas desnecessárias


```python
for x in 'Company', 'Age', 'UF', 'Job Opening':
    df.drop(x, axis='columns', inplace=True)
```

Define HTTP Header


```python
headers = {
    'Content-Type': 'application/json;charset=UTF-8',
    'User-Agent': 'google-colab',
    'Accept': 'application/json, text/plain, */*',
    'Accept-Encoding': 'gzip, deflate, br',
    'Accept-Language': 'pt-BR,pt;q=0.9,en-US;q=0.8,en;q=0.7',
    'Connection': 'keep-alive',
}
```

Coleta Geojson do IBGE e atribui variável


```python
meshes_url = 'https://servicodados.ibge.gov.br/api/v2/malhas/?resolucao=2&formato=application/vnd.geo+json'
meshes_data = requests.get(meshes_url, headers=headers).json()
```

Coleta informacoes dos estados (com id) do IBGE e atribui variável


```python
states_url = 'https://servicodados.ibge.gov.br/api/v1/localidades/estados'
states_data = requests.get(states_url, headers=headers).json()
```


```python
# creating lists to be populated by IBGE requested data
meshes_ids = []
states_ids = []
states_names = []
states_codes = []

# populating information about meshes
for feature in meshes_data['features']:
    meshes_ids.append( str(feature['properties']['codarea']) )

meshes_ids.sort()

# populating information about Federative Units
for state in states_data:
    states_ids.append( str(state['id']) )
    states_names.append( state['nome'] )
    states_codes.append( state['sigla'] )

states_ids.sort()
```


```python
# creating a dataframe of Federative Units to be merged
states = pd.DataFrame( {'id': states_ids, 'nome': states_names, 'sigla': states_codes} )

# appending centroid coordinates columns
states['lat'] = 0
states['lng'] = 0

states.set_index('id', inplace=True)

# retrieving centroid data
for feature in meshes_data['features']:
    
    centroid = feature['properties']['centroide']
    lat = centroid[1]
    lng = centroid[0]
    
    cod = str(feature['properties']['codarea'])
    
    states.loc[cod,'lat'] = lat
    states.loc[cod,'lng'] = lng

states.reset_index(inplace=True)
```

Faz merge dos dataframes df e localidades com ids


```python
df = df.merge(states, left_on='Location', right_on='nome').drop(columns=['nome'])
```

Remove entradas duplicadas


```python
df = df.drop_duplicates()
```


```python
df
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
      <th>Location</th>
      <th>latitude</th>
      <th>longitude</th>
      <th>Openings</th>
      <th>id</th>
      <th>sigla</th>
      <th>lat</th>
      <th>lng</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Ceará</td>
      <td>-5.0932</td>
      <td>-39.6158</td>
      <td>59</td>
      <td>23</td>
      <td>CE</td>
      <td>-5.093338</td>
      <td>-39.615608</td>
    </tr>
    <tr>
      <th>59</th>
      <td>São Paulo</td>
      <td>-22.2635</td>
      <td>-48.7340</td>
      <td>464</td>
      <td>35</td>
      <td>SP</td>
      <td>-22.263541</td>
      <td>-48.733659</td>
    </tr>
    <tr>
      <th>523</th>
      <td>Rio de Janeiro</td>
      <td>-22.1885</td>
      <td>-42.6524</td>
      <td>69</td>
      <td>33</td>
      <td>RJ</td>
      <td>-22.188569</td>
      <td>-42.651667</td>
    </tr>
    <tr>
      <th>592</th>
      <td>Bahia</td>
      <td>-12.4753</td>
      <td>-41.7212</td>
      <td>8</td>
      <td>29</td>
      <td>BA</td>
      <td>-12.475101</td>
      <td>-41.720804</td>
    </tr>
    <tr>
      <th>600</th>
      <td>Pernambuco</td>
      <td>-8.3261</td>
      <td>-37.9988</td>
      <td>6</td>
      <td>26</td>
      <td>PE</td>
      <td>-8.326050</td>
      <td>-37.998299</td>
    </tr>
    <tr>
      <th>606</th>
      <td>Paraná</td>
      <td>-24.6359</td>
      <td>-51.6166</td>
      <td>30</td>
      <td>41</td>
      <td>PR</td>
      <td>-24.635840</td>
      <td>-51.616400</td>
    </tr>
    <tr>
      <th>636</th>
      <td>Minas Gerais</td>
      <td>-18.4562</td>
      <td>-44.6734</td>
      <td>41</td>
      <td>31</td>
      <td>MG</td>
      <td>-18.456155</td>
      <td>-44.673385</td>
    </tr>
    <tr>
      <th>677</th>
      <td>Distrito Federal</td>
      <td>-15.7812</td>
      <td>-47.7969</td>
      <td>13</td>
      <td>53</td>
      <td>DF</td>
      <td>-15.780746</td>
      <td>-47.797341</td>
    </tr>
    <tr>
      <th>690</th>
      <td>Amazonas</td>
      <td>-4.1541</td>
      <td>-64.6531</td>
      <td>9</td>
      <td>13</td>
      <td>AM</td>
      <td>-4.154223</td>
      <td>-64.653187</td>
    </tr>
    <tr>
      <th>699</th>
      <td>Goiás</td>
      <td>-16.0412</td>
      <td>-49.6225</td>
      <td>3</td>
      <td>52</td>
      <td>GO</td>
      <td>-16.042109</td>
      <td>-49.623608</td>
    </tr>
    <tr>
      <th>702</th>
      <td>Pará</td>
      <td>-3.9810</td>
      <td>-53.0722</td>
      <td>3</td>
      <td>15</td>
      <td>PA</td>
      <td>-3.974815</td>
      <td>-53.064197</td>
    </tr>
    <tr>
      <th>705</th>
      <td>Rio Grande do Sul</td>
      <td>-29.7046</td>
      <td>-53.3208</td>
      <td>27</td>
      <td>43</td>
      <td>RS</td>
      <td>-29.705809</td>
      <td>-53.319974</td>
    </tr>
    <tr>
      <th>732</th>
      <td>Espírito Santo</td>
      <td>-19.5749</td>
      <td>-40.6712</td>
      <td>5</td>
      <td>32</td>
      <td>ES</td>
      <td>-19.575078</td>
      <td>-40.670847</td>
    </tr>
    <tr>
      <th>737</th>
      <td>Maranhão</td>
      <td>-5.0789</td>
      <td>-45.2892</td>
      <td>1</td>
      <td>21</td>
      <td>MA</td>
      <td>-5.060433</td>
      <td>-45.279153</td>
    </tr>
  </tbody>
</table>
</div>



Gera Mapa


```python
# Coordenadas de Brasilia
federal_district = [-15.7757875,-48.0778477]

# Cria objeto Mapa
basemap = folium.Map(
    location=federal_district,
    zoom_start=4,
    tiles='openstreetmap'
)

# Permite alterar tipo de layout do mapa
tiles = ['openstreetmap', 'cartodbpositron']

# iterating over the tiles and creating the maps
for tile in tiles:
  folium.TileLayer(tile).add_to(basemap)


# Define plot
legends = 'SRE Jobs Opening in Brazil 2021'
folium.Choropleth(
    geo_data=meshes_data,
    data=df,
    name=legends,
    columns=['id','Openings'],
    key_on='feature.properties.codarea',
    fill_color='YlOrRd',
    fill_opacity=1.0,
    line_opacity=0.7,
    legend_name=legends
).add_to(basemap)

# controles do mapa
folium.LayerControl().add_to(basemap)

# Renderiza Mapa
basemap
```



```python

```
