# Clover ETL Tutorial
Bill Zichos  
August 19, 2015  

Before we get started...

```r
setwd("~/GitHub/Clover-ETL")
unzip("sample-data.zip")
```

## Orders

Process the *orders* data.  This first involves loading and formatting the data.  Since we only care about orders placed in 2014, we will filter prior to merging with other data sets.


1. Load orders.txt

```r
orders <- read.fwf(paste(getwd(), "/orders.txt", sep = ""), widths = c(10, 10, 8, 8, 10),header = FALSE, skip = 1, as.is = TRUE, col.names = c("orderId", "custId", "itemId", "qty", "transDate"), colClasses = c("character", "character", "character", "numeric", "character"), strip.white = TRUE)

# format the date so we can do some processing on it later
orders$transDate <- as.Date(orders$transDate, "%d.%m.%Y")
```

2. Filter orders for 2014


```r
orders.2014 <- orders[year(orders$transDate)==2014,]
```

## Customers

1. Load customers.txt

```r
customers <- read.csv(paste(getwd(), "/customers.txt", sep =""), sep = ";", header = TRUE, col.names = c("CUST_ID", "CUST_NAME", "STREET", "CITY", "STATE", "POST_CODE"))

customers$CUST_ID <- as.character(customers$CUST_ID)
```

2. Filter C* State Customers

```r
customers.c <- customers[customers$STATE %in% c("CA", "CO", "CT"),]
```

## Items

Excel files are a bit trickier.  In reality, I would ask the source to change to CSV or manually change myself, but I want to prove that Excel can be handled in R.  This requires the "gdata" R package library and the Perl libraries. 

1. Load items.xlsx


```r
items <- read.xls(paste(getwd(), "/items.xlsx", sep = ""), sheet = 1, header = TRUE, perl = "C:\\Perl64\\bin\\perl.exe")

items$ID <- as.character(items$ID)
```

2. Exclude non-2-liter items.

```r
items.2Liter <- items[items$Unit=="2-Liter",]
```

## Combine the datasets
First merge Orders with Customers.  Then take the result and merge with Items.  The resulting dataset has 304 observations and 13 variables.

```r
df <- merge(orders.2014, customers.c, by.x = "custId", by.y = "CUST_ID")
df <- merge(df, items.2Liter, by.x = "itemId", by.y = "ID")

# calculate the sales price - Unit Price * Quantity
df$totalSales <- df$qty * df$UnitPrice
```



```r
df <- select(df, custId, custName = CUST_NAME, totalQty = qty, totalSales)

group.by.customer <- group_by(df, custId, custName)

# summarize by customer
df.agg <- arrange(summarize(group.by.customer, totalQty = sum(totalQty), totalSales = sum(totalSales)), totalQty)

# limit to just those with 30 or more orders
best.customers <- df.agg[df.agg$totalQty>30,]

# sort by order total sales
best.customers[order(best.customers$totalSales, decreasing = TRUE),]
```

```
## Source: local data frame [16 x 4]
## Groups: custId
## 
##      custId                 custName totalQty totalSales
## 1  63531739               Kecia Bile       54     110.01
## 2  47980745           Katina Ihenyen       47     106.51
## 3  45527083          Apryl Solkowitz       42     106.49
## 4  63225425            Arnulfo Dyess       39     102.04
## 5  26340216      Jimmy Phippard, Sr.       41      95.92
## 6  64728773            Carie Delphia       43      92.42
## 7  49436759 Sylva Å vestkovÃ¡, PhDr.       43      90.32
## 8  15643514          Desirae Aracena       38      89.72
## 9  43718558        JiÅÃ­ KÅemeÄek       32      83.13
## 10 45626421            Doreen Noblet       33      78.06
## 11 64818055            ZdenÄk NovÃ½       31      77.95
## 12 60240440              Vito Edgmon       38      75.35
## 13 60720182             Vella Chicon       35      74.51
## 14 10146181       Prince Badalamenti       40      74.01
## 15 49508903          Lorriane Stroup       31      73.55
## 16 19027036               Jewel Huey       33      66.51
```
