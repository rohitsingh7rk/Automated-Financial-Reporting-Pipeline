# Financial Data Dashboard (Outlook, Google Cloud Platform, Google Drive, Power Automate, Power BI, Python)

I built this project from a case study where a team analysing US financial data had to turn around about 25 emailed files into a dashboard every day, between 3 PM and 8 PM. Downloading, merging and cleaning the files ate most of that time, caused errors, and had already led to two extra hires costing about $12,000 a month.

I did three things: set up an automatic route from Outlook into Google Drive, used Python to pull and combine the files, then built the Power BI dashboard the client asked for.

## How the files get in

The survey team's files arrive as attachments in my Outlook inbox. I set up a Power Automate flow that triggers whenever one of these emails comes in and saves the attachment into a shared Google Drive folder, so nothing has to be downloaded by hand. To let my Python script read that folder, I created a project in Google Cloud Platform, enabled the Drive API, made a service account with read-only access, and shared the Drive folder with it.

## The Python part

A script connects to a shared Google Drive folder through the Drive API (service account, read-only), downloads every CSV, Excel and Google Sheets file, and stacks them into one pandas DataFrame, `Sheet1`. That replaced the download-and-merge steps that were being done by hand.

The script is in `scripts/fetch_data.py`. You will need your own service account key and folder ID. I have not included mine.

## The dashboard

The client asked for ten things, including:

- Average income, monthly balance, payment delays and credit utilisation
- Age vs credit limit changes, and payment behaviour by credit mix
- Age distribution, and customers per age group and credit score (Teen 14-19, Young Adult 19-25, Old Adult 25-35, Old1 35-45, Old2 45+)
- Which age groups are good targets for loans (average enquiries above 7.5)
- An LTV score per age, with a promotion for each LTV band
- Average loans and credit cards held by age
- How many of each loan type have been issued

## DAX I'm most happy with

**LTV Score.** It averages income, payment delay, credit score, amount invested and monthly balance, then applies weights (0.3, 0.15, 0.4, 0.075, 0.075). Credit mix is turned into a number first: Good = 3, Above Standard = 2, Standard = 1, Bad = 0. The final `IF` maps the score to a promotion.

```dax
LTV Score =
var AvgAnnualIncome = CALCULATE(AVERAGE(Sheet1[Annual_Income]))
var AvgDelayFromDueDate = CALCULATE(AVERAGE(Sheet1[Delay_from_due_date]))
var AvgAmountInvested = CALCULATE(AVERAGE(Sheet1[Amount_invested_monthly]))
var AvgMonthlyBalance = CALCULATE(AVERAGE(Sheet1[Monthly_Balance]))
var AvgCreditScore =
    CALCULATE(
        AVERAGEX(
            VALUES(Sheet1[Credit_Mix]),
            SWITCH(
                TRUE(),
                Sheet1[Credit_Mix] = "Good", 3,
                Sheet1[Credit_Mix] = "Above Standard", 2,
                Sheet1[Credit_Mix] = "Standard", 1,
                Sheet1[Credit_Mix] = "Bad", 0,
                0
            )
        )
    )

var LTV =
    (0.3 * AvgAnnualIncome +
     0.15 * AvgDelayFromDueDate +
     0.4 * AvgCreditScore +
     0.075 * AvgAmountInvested +
     0.075 * AvgMonthlyBalance
    )

return
IF(LTV > 80000, "30% off on online purchases + home loan at 4% interest",
    IF(LTV > 60000 && LTV < 80000, "5% off on online purchases + 10000 worth gift hampers",
        IF(LTV > 50000 && LTV < 60000, "Any loan at 5% interest rate", BLANK())
    )
)
```

**LoanTypeTable.** The `Type_of_Loan` column holds several loans in one cell, so counting them directly gives wrong numbers. I replaced the commas with `|`, used `PATHLENGTH` and `PATHITEM` to split the text, and `GENERATE` to make one row per customer per loan type.

```dax
LoanTypeTable =
var LoanTypeData =
    GENERATE(
        Sheet1,
        var LoanTypeList = SUBSTITUTE(Sheet1[Type_of_Loan], ",", "|")
        return
            ADDCOLUMNS(
                GENERATESERIES(1, PATHLENGTH(LoanTypeList)),
                "LoanType", TRIM(PATHITEM(LoanTypeList, [Value], TEXT))
            )
    )

return
SELECTCOLUMNS(
    LoanTypeData,
    "CustomerID", Sheet1[Customer_ID],
    "LoanType", TRIM(SUBSTITUTE([LoanType], "and", ""))
)
```

## Screenshots

![Page 1](visuals\imag1.png)
![Page 2](visuals\imag2.png)

## Running it

1. Get the dataset from csv_dataset and put it in `data/`, or point the script at your own Drive folder.
2. `pip install pandas requests google-auth google-api-python-client`
3. Open `report.pbix` in Power BI Desktop and refresh the data source.
