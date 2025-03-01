import os
import pandas as pd
import streamlit as st
import matplotlib.pyplot as plt
import seaborn as sns
import numpy as np

# Define data paths
data_dir = r"C:\Users\usen\Mini project 2\data\output_csv"
sector_mapping_file = r"C:\Users\usen\Mini project 2\data\sector\cleaned_sector_mapping.csv"

# Load sector mapping
sector_mapping = pd.read_csv(sector_mapping_file)

# Ensure required columns exist
if "Ticker" not in sector_mapping.columns or "sector" not in sector_mapping.columns:
    st.error("Sector mapping CSV must contain 'Ticker' and 'sector' columns!")
    st.stop()

# Initialize lists for calculations
all_stock_data = []
volatility_data = []
cumulative_returns = {}
monthly_returns = []

# Process each stock CSV file
for ticker in os.listdir(data_dir):
    ticker_folder = os.path.join(data_dir, ticker)
    csv_path = os.path.join(ticker_folder, f"{ticker}.csv")
    
    if os.path.isfile(csv_path):
        df = pd.read_csv(csv_path)

        if "date" not in df.columns or "close" not in df.columns or "volume" not in df.columns:
            continue
        
        # Convert date to datetime
        df["date"] = pd.to_datetime(df["date"])
        df.sort_values("date", inplace=True)
        
        # Compute daily returns
        df["daily_return"] = df["close"].pct_change()
        
        # Compute yearly return
        df["year"] = df["date"].dt.year
        yearly_return = df.groupby("year")["daily_return"].sum().mean() * 100  # Convert to percentage
        
        # Compute volatility (std deviation of daily returns)
        volatility = df["daily_return"].std()
        
        # Compute cumulative return
        df["cumulative_return"] = (1 + df["daily_return"]).cumprod() - 1
        cumulative_returns[ticker] = df[["date", "cumulative_return"]]
        
        # Compute monthly return
        df["month"] = df["date"].dt.to_period("M")
        monthly_return = df.groupby("month")["daily_return"].sum().reset_index()
        monthly_return["Ticker"] = ticker
        monthly_returns.append(monthly_return)
        
        # Get stock sector
        sector_row = sector_mapping[sector_mapping["Ticker"] == ticker]
        sector = sector_row["sector"].values[0] if not sector_row.empty else "Unknown"
        
        all_stock_data.append({"Ticker": ticker, "Yearly Return": yearly_return, "Sector": sector, 
                               "Close Price": df["close"].iloc[-1], "Volume": df["volume"].mean()})
        
        volatility_data.append({"Ticker": ticker, "Volatility": volatility})

# Convert to DataFrames
stock_df = pd.DataFrame(all_stock_data)
volatility_df = pd.DataFrame(volatility_data)
monthly_returns_df = pd.concat(monthly_returns, ignore_index=True)

# Compute key metrics
top_gainers = stock_df.nlargest(10, "Yearly Return")
top_losers = stock_df.nsmallest(10, "Yearly Return")
green_stocks = (stock_df["Yearly Return"] > 0).sum()
red_stocks = (stock_df["Yearly Return"] < 0).sum()
avg_price = stock_df["Close Price"].mean()
avg_volume = stock_df["Volume"].mean()

# Aggregate sector-wise performance
sector_performance = stock_df.groupby("Sector")["Yearly Return"].mean().reset_index()

# Compute stock correlation matrix
all_tickers = [ticker for ticker in os.listdir(data_dir) if os.path.isfile(os.path.join(data_dir, ticker, f"{ticker}.csv"))]
price_data = {}

for ticker in all_tickers:
    csv_path = os.path.join(data_dir, ticker, f"{ticker}.csv")
    df = pd.read_csv(csv_path)
    
    if "date" in df.columns and "close" in df.columns:
        df["date"] = pd.to_datetime(df["date"])
        price_data[ticker] = df.set_index("date")["close"]

price_df = pd.DataFrame(price_data)
correlation_matrix = price_df.pct_change().corr()

# Streamlit UI
st.title("📊 Stock Market Dashboard")

## **1️⃣ Key Metrics**
st.header("📈 Key Market Metrics")
col1, col2, col3 = st.columns(3)
col1.metric("Green Stocks", green_stocks)
col2.metric("Red Stocks", red_stocks)
col3.metric("Avg Price", f"${avg_price:.2f}")

st.metric("Avg Volume", f"{avg_volume:,.0f}")

## **2️⃣ Volatility Analysis**
st.header("📊 Volatility Analysis (Top 10)")

top_volatility = volatility_df.nlargest(10, "Volatility")

fig, ax = plt.subplots(figsize=(10, 5))
sns.barplot(data=top_volatility, x="Ticker", y="Volatility", ax=ax, palette="coolwarm")
ax.set_title("Top 10 Most Volatile Stocks")
st.pyplot(fig)

## **3️⃣ Cumulative Return Over Time**
st.header("📈 Cumulative Return Over Time")

top_performers = stock_df.nlargest(5, "Yearly Return")["Ticker"]
fig, ax = plt.subplots(figsize=(12, 6))

for ticker in top_performers:
    df = cumulative_returns[ticker]
    ax.plot(df["date"], df["cumulative_return"], label=ticker)

ax.set_title("Top 5 Performing Stocks")
ax.legend()
st.pyplot(fig)

## **4️⃣ Sector-wise Performance**
st.header("🏢 Sector-wise Performance")

fig, ax = plt.subplots(figsize=(10, 5))
sns.barplot(data=sector_performance, x="Sector", y="Yearly Return", ax=ax, palette="viridis")
ax.set_xticklabels(ax.get_xticklabels(), rotation=45)
ax.set_title("Average Yearly Return by Sector")
st.pyplot(fig)

## **5️⃣ Stock Price Correlation**
st.header("📊 Stock Price Correlation")

fig, ax = plt.subplots(figsize=(12, 8))
sns.heatmap(correlation_matrix, cmap="coolwarm", annot=False)
ax.set_title("Stock Price Correlation Heatmap")
st.pyplot(fig)

## **6️⃣ Monthly Gainers & Losers**
st.header("📆 Monthly Top Gainers & Losers")

for month in monthly_returns_df["month"].unique():
    monthly_data = monthly_returns_df[monthly_returns_df["month"] == month]
    top_5 = monthly_data.nlargest(5, "daily_return")
    bottom_5 = monthly_data.nsmallest(5, "daily_return")

    fig, ax = plt.subplots(figsize=(10, 5))
    sns.barplot(data=top_5, x="Ticker", y="daily_return", ax=ax, color="green", label="Gainers")
    sns.barplot(data=bottom_5, x="Ticker", y="daily_return", ax=ax, color="red", label="Losers")
    ax.set_title(f"Top 5 Gainers & Losers - {month}")
    ax.legend()
    st.pyplot(fig)
