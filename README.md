# Dataset for: Twitter-Derived Measures of Economy and Uncertainty for the Philippines

The `Tweets-PH.csv` inside the zip file contain the data used for the paper. This file contains the actual text data, the month it was posted, and the accompanying likes and retweets data for each tweet, as of the time of data collection.

## Data collection

The data was collected using the full-archive search endpoint of the X (Twitter) v2 API under the Academic Research access, which was still available during the time of data collection and was provided for free following a formal application. This has now been moved to the Pro and Enterprise access levels. For more information, see: https://developer.x.com/en/docs/x-api/getting-started/about-x-api#v2-access-level

### Description of the data

The data is stored in CSV format, with the following headers:
- `year-month`: what year and month the post was sent out.
- `keyword`: pertains to which dataset it belongs to, either PE (Philippine Economy) or PU (Philippine Uncertainty).
- `text`: post text itself. Microsoft Excel may incorrectly render certain characters, but the file displays correctly when loaded through `pandas`.
- `likes`: number of likes of the post
- `retweets`: number of retweets of the post

## Graph Replication
1. Aggregate the data by dataset/keyword, month, and desired metric $x$ (counts, or weighted counts). Counts are defined as the number of tweet in a given month, while Weighted Counts are the sum of the total number of tweets, likes, and retweets in a given month.
```python
csv = pd.read_csv('Tweets-PH.csv', parse_dates=['year_month'])
df = (csv
		.groupby('keyword') # PE or PU+
		.resample('M', on='year_month').agg({'text':'count', 'likes':'sum', 'retweets':'sum'})
		.rename(columns={'text':'counts'}))
df = df.assign(**{'weighted_counts': lambda x: x.counts + x.likes + x.retweets}).reset_index()
```
2. For each $x_t$, calculate the exponentially weighted moving average $\mu$ and exponentially weighted moving standard deviation $\sigma$, with a predefined span of $s=12$, for the chosen metric. Shift the calculated $\mu$ and $\sigma$ forward, henceforth $\mu_{t-1}$ and $\sigma_{t-1}$.
```python
df['mu'] = df['metric'].ewm(span=span).mean()
df['std'] = df['metric'].ewm(span=span).std()
df['mu_lag'] = df['mu'].shift()
df['std_lag'] = df['std'].shift()
```
3. Identify anomalous data points by calculating rows which satisfy $x_t-\mu_{t-1}>2\sigma_{t-1}$
```python
df['is_anomaly'] = np.where(np.abs(df['metric'] - df['mu_lag']) > factor * df['std_lag'], 1, 0)
```
4. Graph the metric as a line chart and the anomalous points as points in a scatterplot
  ```python
fig, ax1 = plt.subplots(figsize=(14, 10))
ax1.plot(df.index, (df.metric))
ax1.scatter(df.index[df.is_anomaly == 1], (df.metric)[df.is_anomaly == 1], marker='o')
```

Alternatively, see the Demo.ipynb jupyter notebook.
## Acknowledgments

* [tweepy](https://www.tweepy.org/) python library for accessing the X v2 API