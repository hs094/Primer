[[Dask]] [[ETL]] 
[Dask](https://docs.dask.org/en/stable/index.html?wvideo=l9sgt2saht) is a Python Library for parallel and distributed computing. 

### How to use Dask
Dask provides several APIs. Choose one that works best for you:
#### Tasks
Dask Futures parallelize arbitrary for-loop style Python code, providing:

- **Flexible** tooling allowing you to construct custom pipelines and workflows
- **Powerful** scaling techniques, processing several thousand tasks per second
- **Responsive** feedback allowing for intuitive execution, and helpful dashboards

Dask futures form the foundation for other Dask work

Learn more at [Futures Documentation](https://docs.dask.org/en/stable/futures.html) or see an example at [Futures Example](https://examples.dask.org/futures.html)

```python
from dask.distributed import LocalCluster
client = LocalCluster().get_client()
# Submit work to happen in parallel
results = []
for filename in filenames:
    data = client.submit(load, filename)
    result = client.submit(process, data)
    results.append(result)

# Gather results back to local computer
results = client.gather(results)
```
 ![[Pasted image 20250829115121.png]]

### DataFrames
Dask Dataframes parallelize the popular pandas library, providing:

- **Larger-than-memory** execution for single machines, allowing you to process data that is larger than your available RAM
- **Parallel** execution for faster processing
- **Distributed** computation for terabyte-sized datasets

Dask Dataframes are similar in this regard to Apache Spark, but use the familiar pandas API and memory model. One Dask dataframe is simply a collection of pandas dataframes on different computers.

Learn more at [DataFrame Documentation](https://docs.dask.org/en/stable/dataframe.html) or see an example at [DataFrame Example](https://examples.dask.org/dataframe.html)

```python
import dask.dataframe as dd

# Read large datasets in parallel
df = dd.read_parquet("s3://mybucket/data.*.parquet")
df = df[df.value < 0]
result = df.groupby(df.name).amount.mean()

result = result.compute()  # Compute to get pandas result
result.plot()
```


![_images/dask-dataframe.svg](https://docs.dask.org/en/stable/_images/dask-dataframe.svg)