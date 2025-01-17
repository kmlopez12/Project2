# Project2
Online News Data

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)

library(class)
library(tree)
library(gbm)
library(caret)
library(tidyverse)
library(dplyr)
library(ggplot2)
library(gridExtra)
library(rmarkdown)
library(knitr)
library(utils)
```

## Introduction

describe data and variables, purpose of analysis

For this project, we'll be analyzing the popularity of articles published on [Mashable](mashable.com). Popularity is determined by number of shares, with 1400 or interactions considered popular. Our data is available through the [UCI Machine Learning Repository](https://archive.ics.uci.edu/ml/datasets/Online+News+Popularity), and consists of 39,644 observations of 61 variables. These variables describe numerous attributes of the articles, including the number of words in the title and content, average length of words, subjectivity and polarity of the articles. It also assigns each article to a topic or data channel and indicates on which day of the week it was published. 

```{r dataImport}
#read in with relative path
newsData <- read_csv("./Raw Data/OnlineNewsPopularity.csv")

```

```{r dayOfWeek}
#need to consolidate weekday_is_* variables into one column. 
dayOfWeek <- rep(NA, nrow(newsData))

for (i in 1:nrow(newsData)){
  for(j in 32:38){
    if(newsData[i,j]==1){dayOfWeek[i] <- dayNames[j-31]}
  }
}
newsData <- cbind(newsData, dayOfWeek)
```

```{r}
#clean up the data set a bit for regression analysis- get rid of URL and weekday_is_*
newsData <- newsData[, -c(1, 32:39)]
```


```{r mondayData}
#suggestion to look at just Monday data first
oneDayData <- newsData %>% filter(dayOfWeek==params$day)
```

```{r}
#create training data set (70%) and test set (30%)
set.seed(1)
train <- sample(1:nrow(oneDayData), size=nrow(oneDayData)*0.7)
test <- dplyr::setdiff(1:nrow(oneDayData), train)

newsDataTrain <- oneDayData[train,]
newsDataTest <- oneDayData[test,]

```


## Summarizations

First we'll examine numerical summaries of selected attributes for the articles- number of words in the title, words in the content, links, images, and vidoes.  

```{r numSums}
#numerical summaries of attributes

newsDataTrain %>% select(c(2, 3, 7, 9, 10)) %>% sapply(summary) %>% kable(digits=2, caption="Numeric Summaries of Article Attributes", col.names=c("Words in Title", "Words in Content", "Links", "Images", "Videos"))
```

Now we'll look at a histogram of the number of shares for articles. Note that this will be affected greatly by major outliers. 

```{r distribution}
ggplot(data=newsDataTrain, aes(x=shares)) +
  geom_histogram(binwidth = 2000, fill="blue") +
  labs(title="Distribution of Number of Shares", x="Number of Shares")+
  theme_minimal()
```


```{r histData, include=FALSE}
counts <- rep(NA, 6)
share.by <- rep(NA, 6)
for (i in 13:18){
 counts[i-12] <- sum(newsDataTrain[,i]==1) 
 channel <- filter(newsDataTrain, newsDataTrain[,i]==1)
 share.by[i-12] <- mean(channel$shares)
}
counts <- as.data.frame(counts)
share.by <- as.data.frame(share.by)

channel.names <-c("Lifestyle", "Entertainment", "Business", "Social Media", "Tech", "World")
counts <- cbind(counts, channel.names)
colnames(counts) <- c("article.count", "data.channel")
share.by <- cbind(share.by, channel.names)
colnames(share.by) <- c("shares.by", "data.channel")
```

Next we'll look at articles broken down by category. Here we can visualize the total number of published articles by category, as well as the mean number of shares of articles from each category.  

```{r histograms}
b1 <- ggplot(data=counts, aes(x=data.channel, y=article.count)) +
  geom_bar(stat="identity", aes(fill=as.factor(data.channel))) +
  labs(title="Articles Published", x="Data Channel", y="Number of Articles") +
  theme(legend.position = "none", axis.text.x=element_text(angle=45))

b2 <- ggplot(data=share.by, aes(x=data.channel, y=shares.by)) +
  geom_bar(stat="identity", aes(fill=as.factor(data.channel))) +
  labs(title="Mean Shares", x="Data Channel", y="Shares of Articles") +
  theme(legend.position = "none", axis.text.x=element_text(angle=45))

grid.arrange(b1, b2, nrow=1)
```

We will also look at number of shares compared to how the title ranks in both subjectivity and polarity. Articles are assigned a value for each on a scale of 0 to 1. 
```{r}
s1 <- ggplot(data=newsDataTrain, aes(y=shares)) +
  geom_point(aes(x=title_subjectivity), color="blue") +
  labs(title="Shares by Title Subjectivity", x= "Title Subjectivity", y="Shares")

s2 <- ggplot(data=newsDataTrain, aes(y=shares)) +
  geom_point(aes(x=abs_title_sentiment_polarity), color="red") +
  labs(title="Shares by Title Polarity", x= "Title Polarity", y="Shares")

grid.arrange(s1, s2, nrow=1)
  
```

Now we will assess the rates of both positive words and negative words in the content of the articles. 

```{r boxplots}
b1 <- ggplot(data=newsDataTrain, aes(y=shares, x=global_rate_positive_words)) +
  geom_point(color="blue") +
  labs(title="Shares by Positive Words", x="Rate of Positive Words", y="Shares") +
  theme(legend.position = "none")

b2 <- ggplot(data=newsDataTrain, aes(y=shares, x=global_rate_negative_words)) +
  geom_point(color="red") +
  labs(title="Shares by Negative Words", x="Rate of Negative Words", y="shares") +
  theme(legend.position = "none")

grid.arrange(b1, b2, nrow=1)
```
## Modeling

First we'll fit a regression tree model using leave one out cross validation (LOOCV). 

```{r tree, messages=FALSE, warning= FALSE}
#tree based model, chosen using leave one out cross validation

#formulate our training models
treeFit <- tree(shares~., data=newsDataTrain)
summary(treeFit)
plot(treeFit)
text(treeFit, pretty=0)
```


```{r cv, message=FALSE, warning=FALSE}
#cross validate, K=n is LOOCV
cv.treeFit <- cv.tree(treeFit)
cv.treeFit
```
Next we will construct a model using a Boosted Tree method. 

```{r boostTree, warning=FALSE, message=FALSE, include=FALSE}
#boosted tree model chosen using cross validation
#found tuning parameters using cv here, but now get an error when knitting. Commenting out to knit
#trCtrl <- trainControl(method="repeatedcv", number=10, repeats=3)
#boostTreeFit <- train(shares~.-dayOfWeek, data=newsDataTrain, method="gbm",
                #trControl=trCtrl)
#this is redundant, but train returns a list and I need a matrix for predict()
```

```{r, message=FALSE, warning=FALSE}
boostTreeFit2 <- gbm(shares~.-dayOfWeek, data=newsDataTrain, distribution="gaussian", n.trees=50, interaction.depth = 3, shrinkage = 0.1, n.minobsinnode = 10)
```

## Predictions

Now we will test our models on the reserved testing data set. 

```{r predictions, message=FALSE, warning=FALSE}
treePrediction <- predict(treeFit, newdata=newsDataTest)
treeRMSE <- sqrt(mean((treePrediction-newsDataTest$shares)^2))

boostPrediction <- predict(boostTreeFit2, newdata=newsDataTest, n.trees=50)
boostRMSE <- sqrt(mean((boostPrediction-newsDataTest$shares)^2))
```

## Final Model

For our final model, we will pick the model with lowest RMSE. 

```{r}
RMSEs <- c(treeRMSE, boostRMSE)
kable(t(RMSEs), col.names = c("Regression Tree", "Boosted Tree"), caption = "RMSEs of the Models")
```


```{r automation, include=FALSE}
#keeping this here for future reference
#day <- unique(newsData$dayOfWeek)
#output_file <- paste0(day, "Analysis.md")
#params= lapply(day, FUN=function(x){list(day=x)})
#reports <- tibble(output_file, params)

#apply(reports, MARGIN=1, FUN=function(x){render(input="./Online News Project.Rmd", output_file=x[[1]], params=x[[2]])})
```

## Packages used
```{r}
citation(package="class")
citation(package="tree")
citation(package="caret")
citation(package="tidyverse")
citation(package="dplyr")
citation(package="ggplot2")
citation(package="gridExtra")
citation(package="rmarkdown")
citation(package="knitr")
citation(package= "utils")
```

Data Source Used:

K. Fernandes, P. Vinagre and P. Cortez. A Proactive Intelligent Decision
    Support System for Predicting the Popularity of Online News. Proceedings
    of the 17th EPIA 2015 - Portuguese Conference on Artificial Intelligence,
    September, Coimbra, Portugal.
