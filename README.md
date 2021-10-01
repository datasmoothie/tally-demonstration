# Using Tally, the API for market research

Following is a demonstration of how Datasmoothie's [Tally](https://tally.datasmoothie.com) and 
it's [Python client library](https://github.com/datasmoothie/tally-client) can be used for
data processing, analysis, and the generation of outputs for Excel and Powerpoint.

To dive straight in yourself, use the [demo notebook](https://github.com/datasmoothie/tally-demonstration/blob/main/Build%20Excel%20and%20Powerpoint%20from%20CSV.ipynb).

This project has also been configured to run with Gitpod, try it here:

[![Open in Gitpod](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#https://github.com/datasmoothie/tally-demonstration)

## Reading different file types
We start by reading data from SPSS on one hand and a CSV file on the other.

```
dataset = tally.DataSet("CSV data")
dataset.add_credentials(api_key=os.environ.get('tally_api_key'))
dataset.use_csv('data/Example Data (A) - CSV Excel export.csv')

dataset2 = tally.DataSet('SPSS data')
dataset2.add_credentials(api_key=os.environ.get('tally_api_key'))
dataset2.use_spss('data/Example Data (A).sav')
```

## Creating new derived variables
We want to simplify our analysis of people's locality by combining the five locality answers into two, urban and rural.

These are the current codes:
```
dataset.meta(variable='locality')
```
<table border="1" class="dataframe">  <thead>    <tr style="text-align: right;">      <th></th>      <th>codes</th>      <th>texts</th>      <th>missing</th>    </tr>  </thead>  <tbody>    <tr>      <th>1</th>      <td>1</td>      <td>CBD (central business district)</td>      <td>None</td>    </tr>    <tr>      <th>2</th>      <td>2</td>      <td>Remote</td>      <td>None</td>    </tr>    <tr>      <th>3</th>      <td>3</td>      <td>Rural</td>      <td>None</td>    </tr>    <tr>      <th>4</th>      <td>4</td>      <td>Suburban</td>      <td>None</td>    </tr>    <tr>      <th>5</th>      <td>5</td>      <td>Urban</td>      <td>None</td>    </tr>  </tbody></table>

We want to combine CBD, Urban and Suburban into one group (codes 1, 4 and 5) and Remote and
Rural into another. Our derived variable should be a single choice variable (if we were creating NETs, 
we could create a multi choice variable)
```
derive_conditions = [
    (1, "Urban", {'locality':[1,4,5]}),
    (2, "Rural", {'locality':[2,3]})
]
column = dataset.derive(name='urban', label='Urban or rural', cond_map=derive_conditions, qtype="single")
```

## Weighting the dataset

Now we want our dataset to be more representative of the population, so we use RIM weighting
with our new `urban` variable and the `gender` variable as our targets.
```
scheme={
            'urban':{1:75.0, 2:25.0},
            'gender':{1:49.0, 2:51.0}
        }
result = dataset.weight(name='Gender and urban', 
                        variable='weight_c', 
                        unique_key='resp_id', 
                        scheme=scheme)
        
```

To quality check the RIM weighting, you can check the `weighting_report` key in the result, 
but the dataset is otherwise weighted in-place.

To check the weight result, run a crosstab using the new weight variable and without it.
```
df1 = dataset.crosstab(x='urban', ci=['c%'], w='weight_c')
df2 =dataset.crosstab(x='urban', ci=['c%'])
pd.concat([df1, df2], axis=1)
```
<table border="1" class="dataframe">  <thead>    <tr>      <th></th>      <th>Question</th>      <th colspan="2" halign="left">Total</th>    </tr>    <tr>      <th></th>      <th>Values</th>      <th>Weighted</th>      <th>Unweighted</th>    </tr>    <tr>      <th>Question</th>      <th>Values</th>      <th></th>      <th></th>    </tr>  </thead>  <tbody>    <tr>      <th rowspan="3" valign="top">urban. Urban or rural</th>      <th>Base</th>      <td>8078.0</td>      <td>8078.0</td>    </tr>    <tr>      <th>Urban</th>      <td>75.0</td>      <td>80.8</td>    </tr>    <tr>      <th>Rural</th>      <td>25.0</td>      <td>19.2</td>    </tr>  </tbody></table>
