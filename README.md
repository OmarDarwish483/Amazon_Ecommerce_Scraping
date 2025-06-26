# Amazon Ecommerce Analysis with Gradio & NetworkX 🛍️

![Amazon Logo](https://upload.wikimedia.org/wikipedia/commons/thumb/a/a9/Amazon_logo.svg/1000px-Amazon_logo.svg.png)

---

## ✨ Introduction

Welcome to the **Amazon Ecommerce Analysis** project! This repository contains a Jupyter Notebook designed to analyze Amazon product data, leveraging various technologies to present insightful visualizations and analyses. Our focus is on fetching product details, scraping related items, and visualizing the data through network graphs and heatmaps.

---

## 🌟 Purpose

The primary goal of this project is to demonstrate how to gather, process, and analyze Amazon ecommerce data. We use Python libraries like **pandas**, **networkx**, **matplotlib**, **seaborn**, **plotly**, and **gradio** to achieve this. We also employ **Selenium** for web scraping and **Requests** for API interactions.

---

## 🛠️ Features

- **Data Collection**: Fetch product data from Amazon using the Rainforest API.
- **Web Scraping**: Scrape frequently bought together items.
- **Data Visualization**: Create network graphs, heatmaps, and 3D scatter plots.
- **Interactive Dashboard**: Provide an interactive interface using Gradio.
- **Data Cleaning**: Handle missing data and convert prices to USD.

---

## 📦 Installation

To get started, you need to install the required packages. You can do this by running the following commands in your Jupyter Notebook or terminal:

```bash
%pip install requests selenium pandas networkx matplotlib seaborn plotly gradio webdriver_manager
!apt-get update
!apt-get install -y chromium chromium-driver libnss3 libgconf-2-4
```

---

## 🛠️ Usage

### Step 1: Import Libraries

First, import all the necessary libraries:

```python
import requests
import pandas as pd
import networkx as nx
import matplotlib.pyplot as plt
import seaborn as sns
import plotly.express as px
import gradio as gr
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from webdriver_manager.chrome import ChromeDriverManager
import time
import re
```

### Step 2: Configure Selenium

Set up Selenium for web scraping:

```python
chrome_options = Options()
chrome_options.add_argument('--headless')
chrome_options.add_argument('--no-sandbox')
chrome_options.add_argument('--disable-dev-shm-usage')
chrome_options.add_argument('--disable-gpu')
chrome_options.add_argument('window-size=1920x1080')
```

### Step 3: Fetch Product Data

Define a function to fetch product data from the Rainforest API:

```python
RAINFOREST_API_KEY = "1951CB94DC4246D38503225430F0A0D4"
COUNTRIES = {
    'us': 'amazon.com',
    'uk': 'amazon.co.uk',
    'de': 'amazon.de',
    'fr': 'amazon.fr',
    'jp': 'amazon.co.jp'
}
PRODUCT_QUERY = 'Apple iPhone 14'

def fetch_rainforest_data(query, country, domain):
    params = {
        'api_key': RAINFOREST_API_KEY,
        'type': 'search',
        'amazon_domain': domain,
        'search_term': query,
        'sort_by': 'price_low_to_high'
    }
    try:
        response = requests.get('https://api.rainforestapi.com/request', params=params)
        response.raise_for_status()
        data = response.json()
        return data.get('search_results', [])
    except Exception as e:
        print(f'Error fetching data for {country}: {e}')
        return []
```

### Step 4: Collect and Clean Data

Collect data for the specified product query and clean it:

```python
product_data = []
for country, domain in COUNTRIES.items():
    results = fetch_rainforest_data(PRODUCT_QUERY, country, domain)
    for item in results[:5]:
        product_data.append({
            'title': item.get('title', 'No title'),
            'price': item.get('price', {}).get('value', 0),
            'currency': item.get('price', {}).get('currency', 'USD'),
            'rating': item.get('rating', 0),
            'reviews': item.get('reviews', 0),
            'country': country.upper(),
            'asin': item.get('asin', 'No ASIN')
        })

product_df = pd.DataFrame(product_data)
product_df.to_csv('iphone_14_data.csv', index=False)
```

### Step 5: Scrape Related Items

Scrape frequently bought together items:

```python
def scrape_frequently_bought_together(asin, country='com'):
    url = f'https://www.amazon.{country}/dp/{asin}'
    try:
        driver.get(url)
        time.sleep(5)

        related_items = []
        elements = driver.find_elements(By.CSS_SELECTOR, 'div#desktop_buybox_btf div.a-section span')
        for elem in elements:
            text = elem.text
            if 'Frequently bought together' in text:
                related_elements = driver.find_elements(By.CSS_SELECTOR, 'div#desktop_buybox_btf div.a-section a')
                for rel_elem in related_elements:
                    title = rel_elem.text.strip()
                    if title and title not in related_items:
                        related_items.append(title)
        return related_items
    except Exception as e:
        print(f'Error scraping {url}: {e}')
        return []

sample_asin = product_df[product_df['country'] == 'US']['asin'].iloc[0] if not product_df.empty else 'B08X9Y9Z1A'
related_items = scrape_frequently_bought_together(sample_asin, 'com')
print('Frequently Bought Together Items:')
print(related_items)

driver.quit()
```

### Step 6: Data Analysis and Visualization

Perform data analysis and create visualizations:

```python
product_df['price'] = product_df['price'].fillna(0)
product_df['rating'] = product_df['rating'].fillna(0)
product_df['reviews'] = product_df['reviews'].fillna(0)

conversion_rates = {'USD': 1, 'GBP': 1.3, 'EUR': 1.1, 'JPY': 0.0067}
product_df['price_usd'] = product_df.apply(
    lambda row: row['price'] * conversion_rates.get(row['currency'], 1), axis=1
)

G = nx.Graph()
for _, row in product_df.iterrows():
    G.add_node(row['asin'], title=row['title'], country=row['country'])

edge_count = 0
for i, row1 in product_df.iterrows():
    for j, row2 in product_df.iterrows():
        if i < j:
            if row1['country'] == row2['country']:
                title1 = row1['title'].lower()
                title2 = row2['title'].lower()
                if 'iphone' in title1 and 'iphone' in title2:
                    G.add_edge(row1['asin'], row2['asin'], relationship='same_country_iphone')
                    edge_count += 1

plt.figure(figsize=(12, 8))
pos = nx.spring_layout(G)
nx.draw(G, pos, with_labels=False, node_color='lightblue', edge_color='gray', node_size=500)
labels = {node: G.nodes[node]['title'][:20] for node in G.nodes}
nx.draw_networkx_labels(G, pos, labels, font_size=8)
plt.title('Co-Purchase Network for iPhone 14')
plt.savefig('network_graph.png', dpi=300)
plt.show()

centrality = nx.degree_centrality(G)
print('Top 5 Nodes by Degree Centrality:')
for node, score in sorted(centrality.items(), key=lambda x: x[1], reverse=True)[:5]:
    print(f'{G.nodes[node]["title"][:20]}: {score:.3f}')

heatmap_data = product_df.pivot_table(
    values='price_usd',
    index='country',
    columns='title',
    aggfunc='mean'
).fillna(0)

expected_countries = ['US', 'UK', 'DE', 'FR', 'JP']
heatmap_data = heatmap_data.reindex(expected_countries, fill_value=0)
iphone_cols = [col for col in heatmap_data.columns if 'iphone' in col.lower()]
heatmap_data = heatmap_data[iphone_cols]

plt.figure(figsize=(10, 6))
sns.heatmap(
    heatmap_data,
    annot=True,
    cmap='YlGnBu',
    fmt='.2f',
    cbar_kws={'label': 'Price (USD)'},
    vmin=0,
    mask=heatmap_data == 0
)
plt.title('iPhone Price Distribution Across Countries')
plt.xlabel('Product')
plt.ylabel('Country')
plt.savefig('price_heatmap.png', dpi=300)
plt.show()

valid_df = product_df[
    (product_df['price_usd'] > 0) &
    (product_df['price_usd'] < 5000) &
    (product_df['rating'] > 0)
].copy()

valid_df['reviews_adjusted'] = valid_df['reviews'] + 1

fig = px.scatter_3d(
    valid_df,
    x='price_usd',
    y='rating',
    z='reviews_adjusted',
    color='country',
    size='price_usd',
    size_max=20,
    opacity=0.7,
    hover_data=['title'],
    title='3D Point Cloud: iPhone Price, Rating, Reviews'
)
fig.update_layout(
    scene=dict(
        xaxis_title='Price (USD)',
        yaxis_title='Rating',
        zaxis_title='Reviews (Adjusted)',
        xaxis=dict(range=[0, valid_df['price_usd'].max() * 1.1]),
        yaxis=dict(range=[0, 5.5]),
        zaxis=dict(range=[0, valid_df['reviews_adjusted'].max() * 1.1], type='log')
    ),
    margin=dict(l=0, r=0, b=0, t=50)
)
fig.write_html('point_cloud.html')
fig.show()
```

### Step 7: Interactive Dashboard with Gradio

Create an interactive dashboard using Gradio:

```python
def analyze_product(product_query):
    product_data = []
    for country, domain in COUNTRIES.items():
        results = fetch_rainforest_data(product_query, country, domain)
        for item in results[:5]:
            product_data.append({
                'title': item.get('title', 'No title'),
                'price': item.get('price', {}).get('value', 0),
                'currency': item.get('price', {}).get('currency', 'USD'),
                'rating': item.get('rating', 0),
                'reviews': item.get('reviews', 0),
                'country': country.upper(),
                'asin': item.get('asin', 'No ASIN')
            })

    product_df = pd.DataFrame(product_data)
    summary = f'Found {len(product_df)} products for "{product_query}":\n'
    summary += f'Average Price (USD): ${product_df["price"].mean():.2f}\n'
    summary += f'Average Rating: {product_df["rating"].mean():.2f}\n'
    summary += f'Total Reviews: {product_df["reviews"].sum()}\n'
    return summary

iface = gr.Interface(
    fn=analyze_product,
    inputs=gr.Textbox(label='Enter Product Query (e.g., iPhone 14)'),
    outputs=gr.Textbox(label='Analysis Results'),
    title='Amazon Product Analysis'
)

iface.launch(share=False)
```

---

## 🖥️ Technologies Used

- **Python**: The primary programming language.
- **Jupyter Notebook**: For interactive coding and visualization.
- **Pandas**: For data manipulation and analysis.
- **NetworkX**: For creating and analyzing network graphs.
- **Matplotlib & Seaborn**: For static plotting.
- **Plotly**: For interactive plotting.
- **Gradio**: For creating interactive web interfaces.
- **Selenium**: For web scraping.
- **Requests**: For API interactions.

---

## 🔍 Unique Aspects

- **Comprehensive Data Collection**: Fetches data from multiple Amazon domains.
- **Interactive Visualizations**: Uses Plotly for interactive 3D scatter plots.
- **Gradio Interface**: Provides an easy-to-use interface for non-technical users.

---

## 🚀 Possible Improvements

- **Enhanced Error Handling**: Improve error handling for API and scraping operations.
- **More Robust Scraping**: Handle more complex web scraping scenarios.
- **Advanced Analysis**: Incorporate more advanced statistical analysis and machine learning models.
- **User Authentication**: Add user authentication for the Gradio interface.

---

## 📈 Value

This project demonstrates the end-to-end process of data collection, cleaning, analysis, and visualization, making it a valuable resource for anyone interested in ecommerce data analysis. The use of Gradio adds an interactive layer, making the analysis accessible to a broader audience.

---

## 📜 License

This project is licensed under the MIT License. Feel free to use, modify, and distribute it as you see fit.

---

## 📞 Contact

For any questions or feedback, please open an issue or reach out to the project maintainers.

---

## 👨‍💻 Developer

This project is developed and maintained by **Omar Hany Darwish**. Feel free to reach out to him for any questions or contributions.

---

**Happy Analyzing!** 🎉📊🛍️

---