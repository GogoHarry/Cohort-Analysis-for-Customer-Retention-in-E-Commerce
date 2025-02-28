# Time-Based Cohort Analysis for Customer Retention

![image](https://github.com/user-attachments/assets/5fe93035-c4af-4419-8666-71deb6a5cf1b)

## Project Overview
This project aims to conduct a time-based cohort analysis to gain insights into customer retention patterns for an e-commerce business, "E-Shop Pro". The primary objectives are to:

1. Analyze customer retention rates across different cohorts over time.
2. Identify trends and patterns in customer behavior and churn.
3. Develop data-driven strategies to improve customer retention and lifetime value.

## Data Description
The dataset used in this analysis contains the following key features:

- `InvoiceNo`: A unique identifier for each transaction.
- `StockCode`: A code associated with each product.
- `Description`: A textual description of the product.
- `Quantity`: The number of units purchased in each transaction.
- `InvoiceDate`: The date and time of the transaction.
- `UnitPrice`: The price per unit of the product.
- `CustomerID`: A unique identifier for each customer.
- `Country`: The country where the transaction occurred.
  
```python
  # load and display the dataset
  data = pd.read_csv("cohort_data.csv", encoding='latin-1')
  data.head()
```

## Methodology
The analysis follows a time-based cohort analysis approach, which involves the following steps:

1. **Data Cleaning and Preprocessing**:
   - Handling missing values
   - Removing duplicates
   - Converting date-time columns to the appropriate format
   
```python
# Convert InvoiceDate to datetime format
data["InvoiceDate"] = pd.to_datetime(data["InvoiceDate"])

# Drop rows where CustomerID is missing (since retention analysis requires customer tracking)
data.dropna(inplace = True)

# Checking for negative values in Quantity (indicating returns)
negative_quantity = data[data["Quantity"] < 0]

# Checking for duplicate rows
duplicates = data.duplicated().sum()

# Display summary of detected issues
negative_quantity.shape[0], duplicates

# Remove duplicate rows
data_cleaned = data.drop_duplicates()

# Remove negative quantity rows (handling product returns separately if needed)
data_cleaned = data_cleaned[data_cleaned["Quantity"] >= 0]

# Verify changes
num_duplicates = data_cleaned.duplicated().sum()
negative_quantity_after = data_cleaned[data_cleaned["Quantity"] < 0].shape[0]

num_duplicates, negative_quantity_after, data_cleaned.shape
```

2. **Cohort Definition**:
   - Grouping customers based on their first purchase date to create cohorts
```python
  import datetime as dt

# Function to extract the first day of the month from InvoiceDate
def get_month(x):
    return dt.datetime(x.year, x.month, 1)

# Apply function to create Invoice Month feature
data_cleaned["InvoiceDate"] = data_cleaned["InvoiceDate"].apply(get_month)

# Function to determine the cohort date (first purchase date per CustomerID)
def get_cohort_date(data):
    return data.groupby("CustomerID")["InvoiceDate"].transform("min")

# Apply function to create Cohort Date feature
data_cleaned["CohortDate"] = get_cohort_date(data_cleaned)
```
   - Calculating the cohort index, which represents the number of months since a customer's first purchase
```python
  import datetime as dt

# Function to extract the year and month from a given date column
def get_year_and_month(data, col):
    month = data[col].dt.month
    year = data[col].dt.year
    return month, year

# Apply function to extract year and month from the cohort date column
first_month, first_year = get_year_and_month(data_cleaned, "cohort date")

# Function to calculate cohort index
def create_cohort_index(first_month, first_year, latest_month, latest_year):
    year_diff = latest_year - first_year
    month_diff = latest_month - first_month
    index = (year_diff * 12) + month_diff + 1  # +1 accounts for customers active for just 1 month
    return index

# Apply function to create cohort index feature
data_cleaned["cohort_index"] = create_cohort_index(first_month, first_year, latest_month, latest_year)
```

3. **Retention Rate Analysis**:
   - Calculating the monthly retention rate for each cohort
   - Visualizing the retention rate matrix using a heatmap
```python
  # Create Cohort Table: Count unique customers per cohort date and cohort index
cohort_info = (
    data_cleaned.groupby(["cohort date", "cohort_index"])["CustomerID"]
    .nunique()
    .reset_index()
    .rename(columns={"CustomerID": "Number of Customers"})
)

# Create a pivot table for cohort analysis
cohort_table = cohort_info.pivot(index="cohort date", columns="cohort_index", values="Number of Customers")

# Format the index to a more readable format (Month Year)
cohort_table.index = cohort_table.index.strftime('%B %Y')

# Display the cohort table
cohort_table

  # Visualize Retention Rate as a Heatmap
plt.figure(figsize=(20, 10))
sns.heatmap(
    cohort_table, 
    annot=True, 
    fmt=".2f", 
    cmap="Dark2_r"
)

# Add titles and labels
plt.title("Customer Retention Over Time by Cohort", fontsize=16)
plt.xlabel("Cohort Index (Months Since First Purchase)", fontsize=14)
plt.ylabel("Cohort Date", fontsize=14)

# Display the heatmap
plt.show()
```

4. **Churn Rate Analysis**:
   - Calculating the churn rate (1 - retention rate) for each cohort
   - Analyzing the overall churn trend over time
```python
  # Calculate Churn Rate (1 - Retention Rate)
churn_matrix = 1 - new_cohort_table  # Using the cohort table with percentage values

 # Visualize Churn Trends as a Heatmap
plt.figure(figsize=(14, 6))
sns.heatmap(
    churn_matrix, 
    annot=True, 
    fmt=".0%", 
    cmap="Reds"
)

# Add titles and labels
plt.title("Customer Churn Rate Over Time by Cohort", fontsize=16)
plt.xlabel("Cohort Index (Months Since First Purchase)", fontsize=14)
plt.ylabel("Cohort Month", fontsize=14)

# Display the heatmap
plt.show()
```

5. **Purchase Quantity Analysis**:
   - Examining the total and average quantity purchased by each cohort over time
   - Identifying patterns and anomalies in purchase behavior
```python
  # Calculate the total quantity bought per cohort
quantity_bought = (
    data_cleaned.groupby(["cohort date", "cohort_index"])["Quantity"]
    .sum()
    .reset_index()
)

# Round the quantity values
quantity_bought["Quantity"] = quantity_bought["Quantity"].round(1)

# Rename column for clarity
quantity_bought.rename(columns={"Quantity": "quantity_bought"}, inplace=True)

# Create a pivot table for visualization
quantity_table = quantity_bought.pivot(
    index="cohort date", columns="cohort_index", values="quantity_bought"
)

# Convert cohort date to a more readable format
quantity_table.index = quantity_table.index.strftime('%B %Y')

# Visualize the results using a heatmap
plt.figure(figsize=(20, 10))
sns.heatmap(quantity_table, annot=True, fmt="0", cmap="Accent_r")

# Add titles and labels
plt.title("Quantity Bought Over Time by Cohort", fontsize=16)
plt.xlabel("Cohort Index (Months Since First Purchase)", fontsize=14)
plt.ylabel("Cohort Date", fontsize=14)

# Display the heatmap
plt.show()
  
  # Calculate Average Quantity Purchased by Cohort
average_quantity = (
    data_cleaned.groupby(["cohort date", "cohort_index"])["Quantity"]
    .mean()
    .reset_index()
)

# Round the values and rename the column
average_quantity["Quantity"] = average_quantity["Quantity"].round(1)
average_quantity.rename(columns={"Quantity": "Average Quantity"}, inplace=True)

# Create a Pivot Table for Visualization
quantity_table = average_quantity.pivot(
    index="cohort date", columns="cohort_index", values="Average Quantity"
)

# Change index format to a more readable format (Month Year)
quantity_table.index = quantity_table.index.strftime('%B %Y')

# Visualize the Results Using a Heatmap
plt.figure(figsize=(20, 10))
sns.heatmap(quantity_table, annot=True, fmt="0", cmap="Accent_r")

# Add titles and labels
plt.title("Quantity Bought Over Time by Cohort", fontsize=16)
plt.xlabel("Cohort Index (Months Since First Purchase)", fontsize=14)
plt.ylabel("Cohort Date", fontsize=14)

# Display the heatmap
plt.show()
```

5. **Tools & Technologies Used**
- Python Libraries: Pandas, NumPy, Matplotlib, Seaborn
- Jupyter Notebook: For analysis and visualization
- GitHub: Version control and project documentation

## Results
The key findings from the cohort analysis are:

1. **Declining Retention Rates**: Customer retention rates consistently decline over time across all cohorts, indicating a significant churn challenge.

  ![image](https://github.com/user-attachments/assets/6c2d489d-b521-4f53-a23a-ac15c2de1f95)

3. **Cohort-Specific Patterns**: Retention rates vary considerably between different customer cohorts, with some cohorts (e.g., December 2010) maintaining higher engagement.
   
   ![image](https://github.com/user-attachments/assets/77893cba-e689-452c-8c39-3cae9d5fc246)

5. **Early Churn**: High levels of customer churn occur within the first few months after the initial purchase, with retention rates dropping sharply in the first 3-4 months.

![image](https://github.com/user-attachments/assets/de6e6549-edca-416c-aa02-c568beb9c76b)

7. **Seasonal Fluctuations**: There are indications of seasonal patterns, with slightly higher retention rates observed in the later months of the year.
   
9. **Cohort Sensitivity**: Newer customer cohorts tend to have lower overall retention rates compared to earlier cohorts, suggesting challenges in retaining newer customers.

## Conclusion
The time-based cohort analysis provided valuable insights into the customer retention challenges faced by the e-commerce business. The key findings suggest that while the business has been successful in acquiring new customers, it struggles to retain them long-term, particularly in the first few months after the initial purchase.

## Recommendations
Based on the analysis, the following recommendations are proposed:

1. **Investigate Drivers of Early Churn**: Conduct further analysis to understand the key factors contributing to high customer churn in the first few months after acquisition.
2. **Implement Targeted Retention Strategies**: Develop cohort-specific retention strategies to address the unique needs and challenges of different customer segments.
3. **Analyze Seasonal Trends**: Explore the underlying reasons for the observed seasonal fluctuations in retention rates and optimize marketing, sales, and customer success activities accordingly.
4. **Monitor Cohort Performance**: Continuously track and analyze customer retention trends across different cohorts to identify emerging issues and make data-driven decisions.

## Future Work
To further enhance the analysis and drive more impactful recommendations, the following future work can be considered:

1. **Churn Prediction Modeling**: Develop machine learning models to predict customer churn based on the cohort data and other relevant features.
2. **Cohort-Specific Segmentation**: Conduct a deeper dive into the characteristics and behaviors of specific cohorts to uncover additional insights.
3. **Integrating External Data**: Explore the possibility of incorporating external data sources to gain a more holistic understanding of the factors influencing customer retention.
4. **Experimentation and A/B Testing**: Design and implement controlled experiments or A/B tests to evaluate the effectiveness of different retention strategies and tactics.

By pursuing these future work directions, the business can further refine its understanding of customer retention dynamics and develop even more effective strategies to improve long-term customer loyalty and business growth.
