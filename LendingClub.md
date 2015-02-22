Title: Projecting Default Rates of Lending Club Notes  
Date: 2015-02-21 14:28  
Authors: Michael Toth  
Modified: 2014-11-12 20:01  
Category: R  
Tags: R, Projects  
Slug: LendingClub  
author_gplusid: 103836786232018210272  
Summary: BlahBlahBlah.  

For those unfamiliar, Lending Club is the world's largest peer-to-peer lending company, offering a platform for borrowers and lenders to work directly with one another, eliminating the need for a financial intermediary like a bank. Removing the middle-man generally allows both borrowers and lenders to receive better interest rates than they otherwise would, which makes peer-to-peer lending an attractive proposition. This post will be the first in a series of posts analyzing the probability of default and expected return of Lending Club notes. *Before investing, it is always important to fully understand the risks, and this post does not constitute investment advice in either Lending Club or in Lending Club notes.*  

## Background and Gathering Data

All types of borrowers are using peer-to-peer lending for a variety of purposes. Lending Club has an algorithm to determine the risk for any given borrower, and they set the interest rates according to the supposed risk. Some percentage of borrowers will default on their loans, so it's important to understand whether the rate 

Lending club makes all past borrower data freely available [on their website](https://www.lendingclub.com/info/download-data.action) for review, and this is the data that I will be referencing throughout this post.  

To download the 2012-2013 data from Lending Club:  

```R
# Download and extract data from Lending Club
if (!file.exists("LoanStats3b.csv")) {
    fileUrl <- "https://resources.lendingclub.com/LoanStats3b.csv.zip"
    download.file(fileUrl, destfile = "LoanStats3b.csv.zip", method="curl")
    dateDownloaded <- date()
    unzip("LoanStats3b.csv.zip")
}

# Read in Lending Club Data
if (!exists("full_dataset")) {
  full_dataset <- read.csv(file="LoanStats3b.csv", header=TRUE, skip = 1)
}
```

Next, let's extract the fields we need and format the data. I eliminate any fields that would not have been known at the time of issuance, with the exception of the loan_status field, which we will ultimately try to predict. I also eliminate a few indicative data fields that are repetitive or too granular to be analyzed, and make some formatting changes to get the data ready for analysis. Finally, I reduce the various loan status measures (current, defaulted, delinquent, etc) to binary outcomes of "Performing" and "Nonperforming" to create a more straightforward classification problem for loan outcomes:  

```R
# Select variables to keep
variables <- c("id", "loan_amnt", "term", "int_rate", "installment", "grade", 
               "sub_grade", "emp_length", "home_ownership", "annual_inc", 
               "is_inc_v", "loan_status", "desc", "purpose", 
               "addr_state", "dti", "delinq_2yrs", "earliest_cr_line", 
               "inq_last_6mths", "mths_since_last_delinq", 
               "mths_since_last_record", "open_acc", "pub_rec", "revol_bal", 
               "revol_util", "total_acc", "initial_list_status", 
               "collections_12_mths_ex_med", "mths_since_last_major_derog")

# Subset the data using only the selected variables
train <- full_dataset[variables]

# Reduce loan status to binary "Performing" and "NonPerforming" Measures:
train$new_status <- factor(ifelse(train$loan_status %in% c("Current", "Fully Paid"), 
                                  "Performing", "NonPerforming"))

# Convert interest rate numbers to numeric
train$int_rate <- as.numeric(sub("%", "", train$int_rate))
train$revol_util <- as.numeric(sub("%", "", train$revol_util))
```

To start, let's examine some of the relationships between variables and default rates to see if we can determine what the major drivers of defaults will be, and whether we can identify any relationships.

First let's look at the proportions of performing and non-performing loans by Lending Club's provided grades:

```R
by_grade <- table(train$new_status, train$grade, exclude="")
prop_grade <- prop.table(by_grade,2)
barplot(prop_grade, main = "Loan Performance by Grade", xlab = "Grade", 
        col=c("darkblue","red"), legend = rownames(prop_grade))

by_subgrade <- table(train$new_status, train$sub_grade, exclude="")
prop_subgrade <- prop.table(by_subgrade,2)
barplot(prop_subgrade, main = "Loan Performance by Sub Grade", xlab = "SubGrade",
        col=c("darkblue","red"),legend = rownames(prop_subgrade))
```

We can see from the chart below that rates of default steadily increase as the loan grades worsen from A to G, as expected.

<br>
<img src="http://www.michaeltoth.net/img/by_grade.png", alt="Performance by Grade")>  
<br> 

We see a similar pattern in the subgrades, although there is a bit of fluctuation in the rates of default for the G1-G5 subgrades.  On further investigation, I found that there are only a few hundred data points for each of these subgrades, in contrast to thousands of data points for the A-F subgrades, and these differences are not enough to be significant.

<br>
<img src="http://www.michaeltoth.net/img/by_subgrade.png", alt="Performance by SubGrade")>  
<br> 

In general, it looks like the Lending Club grading system does a pretty great job of predicting probabilities of defaults, but let's check out some of the other available data to see what else we can find.

<br>
####Home Ownership

The first variable I want to look at is home ownership. I would expect those with mortgages to default less frequently than those who rent, both because there are credit requirements to get a mortgage and because those with mortgages will in aggregate tend to be in better financial health. Let's see whether this is actually the case.

```R
ownership_status <- table(train$new_status,train$home_ownership,
                     exclude=c("OTHER","NONE",""))

prop_ownership <- round(prop.table(ownership_status, 2) * 100, 2)
```

|               | MORTGAGE | OWN   | RENT  |
|---------------|----------|-------|-------|
| NonPerforming | 9.01     | 10.71 | 12.25 |
| Performing    | 90.99    | 89.29 | 87.75 |

So those with mortgages default the least, followed by those who own their homes outright and finally those who rent.  The differences here are much smaller than when comparing different grades. Let's see whether these are significant:


```R
# Calculate the counts of mortgage, owners, and renters:
count_m <- sum(train$home_ownership == "MORTGAGE")
count_o <- sum(train$home_ownership == "OWN")
count_r <- sum(train$home_ownership == "RENT")

# Calculate the counts of default for mortgages, owners, and renters:
dflt_m <- sum(train$home_ownership == "MORTGAGE" & train$new_status == "NonPerforming")
dflt_o <- sum(train$home_ownership == "OWN" & train$new_status == "NonPerforming")
dflt_r <- sum(train$home_ownership == "RENT" & train$new_status == "NonPerforming")

# 1-sided proportion test for mortgage vs owners
prop.test(c(dflt_m,dflt_o), c(count_m,count_o), alternative = "less")

# 1-sided proportion test for owners vs renters
prop.test(c(dflt_o,dflt_r), c(count_o,count_r), alternative = "less")
```

The p-value of the first test was 6.377*10^-12 and the p-value for the second test was 3.787*10^-8, indicating that the differences in both of these proportions are very statistically significant. Although the differences in the default probabilities are only on the order of 1.5%, the number of data points is in the high tens of thousands, which contributes to the significance. Given this result, we can safely conclude that similar differences in default probabilities for other factors should also be significant, so long as a similar quantity of data points is available.


## Predictive value of variables  
For the remaining analysis, the code for analysis of each variable becomes somewhat repetitive, so I will present only the results.  If you are interested to see the code used to generate the results, you will find it in the appendix at the bottom of this post.  You can also read the [complete code on Github](https://github.com/michaeltoth/lending_club/blob/master/LendingClub.R).  

<br>
####Debt to Income Ratio
Bucketed into 5% increments, from 0% to 35%. There is a steady increase in defaults as DTI increases  

|               | 0% - 5% | 5% - 10% | 10% - 15% | 15% - 20% | 20% - 25% | 25% - 30% | 30% - 35% |
|---------------|---------|----------|-----------|-----------|-----------|-----------|-----------|
| NonPerforming | 8.08    | 8.31     | 9.23      | 10.37     | 12.06     | 12.66     | 13.75     |
| Performing    | 91.92   | 91.69    | 90.77     | 89.63     | 87.94     | 87.34     | 86.25     |


<br>
####Revolving Utilization Percent
shows a stead increase in default rates  

|               | 0% - 20% | 20% - 30% | 30% - 40% | 40% - 50% | 50% - 60% | 60% - 70% | 70% - 80% | 80% +  |
|---------------|----------|-----------|-----------|-----------|-----------|-----------|-----------|--------|
| NonPerforming | 7.32     | 8.27      | 8.93      | 9.28      | 9.68      | 10.94     | 11.57     | 12.91  |
| Performing    | 92.68    | 91.73     | 91.07     | 90.72     | 90.32     | 89.06     | 88.43     | 87.09  |


<br>
####Loan Purpose

|               | Small Biz. | Other | Moving | Medical | Wedding | House | Vacation | Consolidation | Major Purch. | Home Imp. | Car   | Credit Card |
|---------------|------------|-------|--------|---------|---------|-------|----------|---------------|--------------|-----------|-------|-------------|
| NonPerforming | 21.06      | 14.08 | 13.85  | 13.50   | 12.34   | 12.17 | 11.78    | 10.89         | 9.25         | 8.84      | 8.68  | 8.12        |
| Performing    | 78.94      | 85.92 | 86.15  | 86.50   | 87.66   | 87.83 | 88.22    | 89.11         | 90.75        | 91.16     | 91.32 | 91.88       |


<br>
####Inquiries in Past 6 Months 
Number of inquiries.  There's an increase in delinquincies as inquiries increases, until the 4+ bucket where we see a slight decrease. This may reflect that those with a very high number of inquiries are possibly more savvy borrowers.  

|               | 0     | 1     | 2     | 3     | 4     |
|---------------|-------|-------|-------|-------|-------|
| NonPerforming | 8.77  | 11.05 | 13.08 | 15.13 | 14.13 |
| Performing    | 91.23 | 88.95 | 86.92 | 84.87 | 85.87 |


<br>
####Number of Total Accounts
showed a significant decrease in delinquencies for numbers smaller than 20, but showed no real changes after that, so I focused in on the smaller ranges:  

|               | <= 7  | 8 - 12 | 13 - 17 | 18 - 22 | 23+   |
|---------------|-------|--------|---------|---------|-------|
| NonPerforming | 13.06 | 11.63  | 11.15   | 10.65   | 9.95  |
| Performing    | 86.94 | 88.37  | 88.85   | 89.35   | 90.05 |  


<br>
####Annual Income
Split into percentile buckets  

|               | 0% - 20%  | 20% - 40%       | 40% - 60%       | 60% - 80%       | 80% - 100%  |
|---------------|-----------|-----------------|-----------------|-----------------|-------------|
| Income Level  | < $42000  | $42000 - $55000 | $55000 - $70000 | $70000 - $95000 | $95000 +    |
| NonPerforming | 13.37     | 11.57           | 10.57           | 8.99            | 7.51        |
| Performing    | 86.63     | 88.43           | 89.43           | 91.01           | 92.49       |


<br>
####Loan Amount
Loan amount shows no large differences for values less than 15000. Loans from 15000 to 30000 fare slightly worse, but loans in the 30000 - 35000 range (the Lending Club maximum) perform significantly worse:  

|               | $0 - $15000 | $15000 - $30000 | $30000 - $35000 |
|---------------|-------------|-----------------|-----------------|
| NonPerforming | 10.00       | 10.95           | 12.79           |
| Performing    | 90.00       | 89.05           | 87.21           |


<br>
####Employment Length
Employment length also might have a significant impact on default probability, as I'd expect those who had been employed longer to be more stable, and thus less likely to default. Looking into the data, 3 key groups emerged: the unemployed, those employed less than 10 years, and those employed for 10+ years.  Aggregating and analyzing the differences:  

|               | None   | < 10 years | 10+ years |
|---------------|--------|------------|-----------|
| NonPerforming | 12.17  | 10.84      | 9.49      |
| Performing    | 87.83  | 89.16      | 90.51     |


<br>
####Number of Public Records
Default probability actually goes down as you move from 0 to 1 to 2, possibly indicating stricter lending standards from Lending Club on those borrowers with public records:  

|               | 0     | 1     | 2+    |
|---------------|-------|-------|-------|
| NonPerforming | 10.61 | 9.00  | 8.72  |
| Performing    | 89.39 | 91.00 | 91.28 |


<br>
####Delinquencies in Past 2 Years  
I combined all numbers 3 or larger into a single bucket. Interestingly, those with a single delinquency seem to default less frequently than those with none. In general however, the differences between 0, 1, and 2 delinquencies are relatively small, while those with greater than 3 show a significant increase in defaults.  

|               | 0     | 1     | 2     | 3+    |
|---------------|-------|-------|-------|-------|
| NonPerforming | 10.45 | 10.22 | 10.74 | 11.78 |
| Performing    | 89.55 | 89.78 | 89.26 | 88.22 |


<br>
####Number of Open Accounts
Slight decrease in delinquencies as this grows, but not a very strong indicator:  

|               | <= 5  | 6 - 10 | 11 - 15 | 16+   |
|---------------|-------|--------|---------|-------|
| NonPerforming | 10.83 | 10.37  | 10.49   | 10.40 |
| Performing    | 89.17 | 89.63  | 89.51   | 89.60 |


<br>
####Verified Income Status
Verified income shows something a bit unexpected.  

|               | Not Verified | Source Verified | Verified |
|---------------|--------------|-----------------|----------|
| NonPerforming | 9.24         | 10.49           | 11.26    |
| Performing    | 90.76        | 89.51           | 88.74    |


<br>
####Variables that were not significant:
- Months since last delinquency. Interestingly, I did not find any significant differences here.
- Months since last major derog did not show a very significant impact on default rates
- Collections previous 12 months has too few data points on which to really judge. A quick investigation of the available data shows no significant differences.



## Appendix

Below I've included the code used to generate the numbers in the tables above. You can also find the [complete code available on Github](https://github.com/michaeltoth/lending_club/blob/master/LendingClub.R)  

```R
### Explore the relationships between default rates and factor levels
### I take a few different approaches, but the key idea is the same


# Home Ownership (exclude status "OTHER" and "NONE" because of few data points)
home_ownership <- table(train$new_status,train$home_ownership,
                     exclude=c("OTHER","NONE",""))
prop_home_ownership <- round(prop.table(home_ownership, 2) * 100, 2)

# Test for significance of the difference in proportions for home ownership factors
# Calculate the counts of mortgage, owners, and renters:
count_m <- sum(train$home_ownership == "MORTGAGE")
count_o <- sum(train$home_ownership == "OWN")
count_r <- sum(train$home_ownership == "RENT")

# Calculate the counts of default for mortgages, owners, and renters:
dflt_m <- sum(train$home_ownership == "MORTGAGE" & train$new_status == "NonPerforming")
dflt_o <- sum(train$home_ownership == "OWN" & train$new_status == "NonPerforming")
dflt_r <- sum(train$home_ownership == "RENT" & train$new_status == "NonPerforming")

# 1-sided proportion test for mortgage vs owners
prop.test(c(dflt_m,dflt_o), c(count_m,count_o), alternative = "less")
# 1-sided proportion test for owners vs renters
prop.test(c(dflt_o,dflt_r), c(count_o,count_r), alternative = "less")


# Debt to Income Ratio (break into factors at 5% levels)
train$new_dti <- cut(train$dti, breaks = c(0, 5, 10, 15, 20, 25, 30, 35))
dti <- table(train$new_status, train$new_dti)
prop_dti <- round(prop.table(dti, 2) * 100, 2)


# Revolving Utilization (break into 0 - 20, then factors of 10, then 80+)
train$new_revol_util <- cut(train$revol_util, breaks = c(0, 20, 30, 40, 50, 60, 70, 80, 141))
revol_util <- table(train$new_status, train$new_revol_util)
prop_revol_util <- round(prop.table(revol_util, 2) * 100, 2)


# Loan Purpose (exclude renewable energy because so few data points)
purpose <- table(train$new_status,train$purpose, exclude = c("renewable_energy",""))
prop_purpose <- round(prop.table(purpose, 2) * 100, 2)


# Inquiries in the last 6 months (combine factor levels for any > 4)
levels(train$inq_last_6mths) <- c("0", "1", "2", "3", rep("4+", 5))
inq_last_6mths <- table(train$new_status, train$inq_last_6mths)
prop_inq_last_6mths <- round(prop.table(inq_last_6mths, 2) * 100, 2)


# Number of total accounts (combine factor levels into groups of 5, then 23+)
levels(train$total_acc) <- c(rep("<= 7", 5), rep("8 - 12", 5), 
                             rep("13 - 17", 5), rep("18 - 22", 5), 
                             rep("23+", 68))
total_acc <- table(train$new_status, train$total_acc)
prop_total_acc <- round(prop.table(total_acc, 2) * 100, 2)


# Annual Income (factor into quantiles of 20%)
train$new_annual_inc <- cut(train$annual_inc,
                            quantile(train$annual_inc, na.rm = TRUE,
                                     probs = c(0, 0.2, 0.4, 0.6, 0.8, 1)))
annual_inc <- table(train$new_status, train$new_annual_inc)
prop_annual_inc <- round(prop.table(annual_inc, 2) * 100, 2)


# Loan Amount (break into < 15k, 15k - 30k, 30k - 35k)
train$new_loan_amnt <- cut(train$loan_amnt,c(0, 15000, 30000, 35000))
loan_amnt <- table(train$new_status, train$new_loan_amnt)
prop_loan_amnt <- round(prop.table(loan_amnt, 2) * 100, 2)


# Employment Length (combine factor levels for better comparison)
levels(train$emp_length) <- c("None", "< 10 years", "< 10 years", "10+ years",
                              rep("< 10 years", 8), "None")
emp_length <- table(train$new_status, train$emp_length)
prop_emp_length <- round(prop.table(emp_length, 2) * 100, 2)


# Number of Public Records (break factor levels into 0, 1, 2+)
levels(train$pub_rec) <- c("0", "1", rep("2+", 12))
pub_rec <- table(train$new_status, train$pub_rec)
prop_pub_rec <- round(prop.table(pub_rec, 2) * 100, 2)


# Delinquencies in the past 2 Years (combine factors levels for any > 3)
levels(train$delinq_2yrs) <- c("0", "1", "2", rep("3+", 17))
delinq_2yrs <- table(train$new_status, train$delinq_2yrs)
prop_delinq_2yrs <- round(prop.table(delinq_2yrs, 2) * 100, 2)


# Number of Open Accounts (combine factor levels into groups of 5)
levels(train$open_acc) <- c(rep("<= 5", 6), rep("6 - 10", 5), 
                            rep("11 - 15", 5), rep("16+", 38))
open_acc <- table(train$new_status, train$open_acc)
prop_open_acc <- round(prop.table(open_acc, 2) * 100, 2)


# Verified income status
is_inc_v <- table(train$new_status, train$is_inc_v, exclude = "")
prop_is_inc_v <- round(prop.table(is_inc_v, 2) * 100, 2)


# Months Since Last Record (compare blank vs. non-blank)
na_last_record <- sum(is.na(train$mths_since_last_record))
not_na_last_record <- sum(!is.na(train$mths_since_last_record))
na_last_rec_dflt <- sum(is.na(train$mths_since_last_record) & train$new_status == "NonPerforming")
not_na_last_rec_dflt <- sum(!is.na(train$mths_since_last_record) & train$new_status == "NonPerforming")

not_na_last_rec_pct_dflt <- not_na_last_rec_dflt / not_na_last_record
na_last_rec_pct_dflt <- na_last_rec_dflt/na_last_record


# Months since last delinquency (break factor levels in increments of 10)
train$mths_since_last_delinq <- cut(train$mths_since_last_delinq, 
                                   breaks = c(0, 10, 20, 30, 40, 50, 60, 156))
mths_since_last_delinq <- table(train$new_status, train$mths_since_last_delinq)
prop_mths_since_last_delinq <- round(prop.table(mths_since_last_delinq, 2) * 100, 2)


# Collections last 12 months
collections <- table(train$new_status, train$collections_12_mths_ex_med)
prop_collections <- round(prop.table(collections, 2) * 100, 2)
```