# Investment Portfolio Management

## Overview
Investing on businesses or sectors that you have knowledge about is a good way to make your hard-earned money work for you.
It may sound easy but long-term investments require regular monitoring and management because it helps us to adjust our investment strategy based on up-to-date data and facts.

As a young investor, I felt the need for a single destination where I could get the facts that I need.

This project explores the methods and techniques that help in building a portfolio dashboard with real-time data in Microsoft Power BI.

## Data Preparation and Transformation
### Investment-transactions data
All transaction-related details were saved as a Microsoft Excel worksheet. In detail, it contains the following information:
* Date & Time of the purchase or sale
* Trade type (Buy, Sell, Distribution)
* Stock information: Name of the stock and International Securities Identification Number (ISIN)
* Number of shares bought/sold
* Price details
* Dividend gains

Another worksheet consists of information about the stock like:
* Type of Stock - Individual Stock or Exchange-traded Funds
* Stock Name and Symbol

The worksheets (.xlsx) are then imported to Power BI Desktop. There are many methods to import data to Power BI Desktop based on your requirements.

* Import mode: It creates a local Power BI copy of the datasets from the source. Data Refresh can be scheduled or on-demand.
* DirectQuery mode: It creates a direct connection to the source instead of creating copies in Power BI. The data that are viewed are always up-to-date.

The best practice is to use the Import mode unless you are allowed to have copies of datasets in Power BI and frequent updating of data is not necessary.

After importing, the data are transformed in Power Query Editor. The following measures are taken:
* Rename queries
  * __investments__ (Fact table) - transaction-related information
  * __dimStockDetails__ (Dimension table) - stock information
* Promote headers
* Rename columns
* Remove unnecessary columns
* Set correct data types
* Replace null values
* Sort columns and add Index column
* Merge the two queries based on the field ISIN

The options Column Distribution, Column Profile, and Column Quality helps to examine the contents in each field.
Set the Column profiling based on entire data set, so that the entire table is considered during the transformation process.

### Real-time data
To analyze the investments, real-time data that tracks the performance of stocks is required. For this, data from yahoo finance has been linked to Power BI.
The main requirement is to make the process in a dynamic way that Power BI could fetch the necessary data automatically as per the stocks that are invested in. The following steps are taken to implement this:
* Initially get and transform data of one of the stocks: https://query1.finance.yahoo.com/v8/finance/chart/TL0.DE?range=5y&interval=1d
  * TL0.DE is the symbol of Tesla stock
  * Range: Last 5 years
* After transforming, convert the query into a function by replacing the stock symbol with a parameter:
  * ```text
    (stockSymbol as text) as table =>
    let
    Source = Json.Document(Web.Contents("https://query1.finance.yahoo.com/v8/finance/chart/"&stockSymbol&"?range=5y&interval=1d")),
    ```
* The query dimStockdetails contains stock symbol information. So, duplicate that query, rename as historicalData, and remove all columns except Symbol, Stock, and ISIN:
  * Invoke custom function by selecting the Symbol column. Choose the function query stockSymbol that was created in the previous step.
  * Expand the new column: The past 5-year performance details of all stocks which are entered in the Symbol column gets populated in the query.

After setting up the Investment-transactions data and the Real-time data, a common date table has to be created so that it can be used by the fact tables.

```text
# date table
Calendar = 
ADDCOLUMNS(
    CALENDAR(MINX(historicalData, [Date]), MAXX(historicalData, [Date])),
    "Year", YEAR([Date]),
    "Month", FORMAT([Date], "mmmm"),
    "Month Number", MONTH([Date]),
    "Quarter", "Q" & ROUNDUP(MONTH([Date]) / 3, 0) & "-" & YEAR([Date])
)
```

## Data Exploration and Modeling
The data preparation stage ends with four data tables:
* investments - share purchase-related information (fact table) 
* historicalData - historical performance of the stocks (fact table)
* dimStockDetails - general stock information (dimension table)
* Calendar - common date table (dimension table)

Now the tables should be connected using relationships. The benefits of this process are:
* Data exploration is faster
* Aggregations are simpler to build
* Reports are easier to maintain in the future

Under the Model View in Power BI, fact tables are connected to dimension tables using many-to-one (*:1) relationship with the key column selected.
In many-to-one or one-to-many type of relationship, many instances of a value in a column are related to only one unique corresponding instance in another column.
It describes the directionality between fact and dimension table.
![Relationship connections](E:\GitHub\investment-portfolio-management\report\screenshots\model_view.png)

Using DAX (Data Analysis Expressions), custom columns and measures are created:
```text
# Number of shares
NUMBER OF SHARES = 
investments[BUY_number_of_shares] + investments[SELL_number_of_shares]
---

# Cumulative total by stock
Running Total by stock = 
    CALCULATE(SUM(investments[TOTAL_BUY]),
        FILTER(investments,
        investments[ISIN_International_Securities_Identification_Number] = EARLIER(investments[ISIN_International_Securities_Identification_Number]) &&
        investments[Index] <= EARLIER(investments[Index])
    ))
---   
 
# Cumulative total by stock type
Running Total by stock-type = 
    CALCULATE(SUM(investments[TOTAL_BUY]),
        FILTER(investments,
        investments[stock_type] = EARLIER(investments[stock_type]) &&
        investments[Index] <= EARLIER(investments[Index])
    ))
---

# Gain/Loss
Capital Gain/Loss = 
(SUM(dimStockDetails[LATEST VALUE]) - SUM(investments[TOTAL_BUY])) / SUM(investments[TOTAL_BUY])
---

# Total shares
TOTAL_number_of_Shares = SUM(investments[BUY_number_of_shares]) + SUM(investments[SELL_number_of_shares])
---

# Latest total price
LATEST VALUE = dimStockDetails[TODAY'S ADJUSTED CLOSE] * [TOTAL_number_of_Shares]
---

# Latest stock price
TODAY'S ADJUSTED CLOSE = 
IF(
    ISEMPTY(
        FILTER(
            historicalData,
            historicalData[Date] = TODAY()
        )
    ),
    LOOKUPVALUE(
        historicalData[Adjusted Close],
        historicalData[Date],
        CALCULATE(
            MAX(historicalData[Date]),
            historicalData[Date] < TODAY()
        ),
        historicalData[stock],
        dimStockDetails[stock_name]
    ),
    CALCULATE(
        MAX(historicalData[Adjusted Close]),
        historicalData[Date] = TODAY()
    )
)
```

These measures and columns help us to provide more information and insights from the imported data.

## Data Visualization
The Report view in Power BI is used to build visuals. Normally, Power BI can automatically select a visual, depending upon the data type of the fields that you selected.

Our investment portfolio displays the following key performance indicators and charts:

| Visualization                                                                                     | Type of visual                 | Definition                                                                          |
|---------------------------------------------------------------------------------------------------|--------------------------------|-------------------------------------------------------------------------------------|
| Invested Amount<br/>Today's Value<br/>Gain/Loss Percentage                                        | Card                           | It displays a single value; a single data point.                                    |
| Investments over time<br/>Overview of Investments (by Stock and Stock Type)<br/>Stock Performance | Line/Area Chart                | It presents trends over time.                                                       |  
| All Stocks<br/>ETFs vs Stocks                                                                     | Pie Chart                      | It shows the relationship of parts to the whole by dividing the data into segments. |
| Stock Type<br/>All Stocks                                                                         | Table                          | It contains related data in logical series of rows and columns.                     |
| Stock Name<br/>Stock Type<br/>Date                                                                | Slicer                         | It is used to filter other visuals.                                                 |
| All Stocks<br/>ETFs vs Stocks<br/>Performance<br/>                                                | Button (Action Type: Bookmark) | It helps to navigate between different pages/bookmarks.                             |

The report consists of two pages:
* Overview
  * It displays all information regarding your stock purchases, losses/gains, portfolio value, etc.
  * The two bookmark buttons __All Stocks__ and __ETFs vs Stocks__ help to toggle between visuals that are based on individual stocks and visuals that are based on ETFs and stocks.  
* Performance
  * The Overview page contains a page navigation button called __Performance__ that helps to navigate to the __Stock Performance__ page.


![Stocks view](E:\GitHub\investment-portfolio-management\report\screenshots\stock_visual.png)
![Stock-Type view](E:\GitHub\investment-portfolio-management\report\screenshots\stock_type_visual.png)
![Stock Performance Chart](E:\GitHub\investment-portfolio-management\report\screenshots\stock_performance.png)

## Conclusion

The Refresh option in Power BI Desktop updates the data and visuals when triggered. The reports can be published to the Power BI Service and finally shared, so that the information can be accessed via Power BI Mobile app.
The portfolio visualizations provide us insights and helps us to improve our investments that leads to higher returns.
