---
layout: post
title: "Web-Mining: a Kaggle competition"
author: "Beydia Mohamed Lemine"
date: "31 December 2018"
categories: text_analysis
tags: text_analysis kaggle network text mining
image: text_analysis/2018/12/31/WebminingKaggle_post_files/figure-markdown_github/network.png
---

This post is about a  kaggle competition that was organised by Toulouse school of Economics and both University of Toulouse 1 & 3 (Paul Sabatier) between 3 master's programe in statisctics.

The main goal was to predict the recipients of some emails . We had to mix machine learning, network and text mining techniques to propose a final model that takes an email (its sender ,its content and some features extracted from the content) as input and output the list of its recipients.

[Officiale Kaggle page](https://www.kaggle.com/c/recipient-prediction-2018).

The remainder of this Notebook is structured as follows. I will start by setting my work Environmen , then I will download and import the data sets ; followed by a basic statistical analysis . 

Onces this done, I will go deeper by performing a graph analysis, a text mining analysis and some features extractions using these both analysis?

Finally, I will perform a machine learning modeling using python as It is much faster than R. 

Note that the first part of this Notebook, which corresponds to pre-processing and analysis, is performed with the software R.

<br>


I ENVIRONEMENT's PREPARATION 
----------------------------

First of all we will create a folder "*data*" in which we are are going to store the data then we will install all the packages that will be needed and to avoid package version issues,we will check  that these packages were not installed before.  


```r 
# I create my folder
setwd("C:/Users/mlemi/Desktop/kaggle/Webmining/notebook")
if (!file.exists("data")) {dir.create("data")}
setwd("C:/Users/mlemi/Desktop/kaggle/Webmining/notebook/data")

# I put all the packages that will be required in a vector 
packages<-c("devtools","tidyr","urltools","stringr","plyr","readr","data.table","tm","networkD3",
            "SnowballC","Matrix","lda","LDAvis","servr","RCurl","dplyr","ggplot2")


# I check if theses packages are installed yet 
diffp<-setdiff(packages, rownames(installed.packages()))
# if yes i do nothing if no i install them
if (length(diffp) > 0) {install.packages(diffp)}
```
<br>


II DATA IMPORTATION
-------------------

The data that we are going to manipulate can be found 
<https://www.kaggle.com/c/recipient-prediction-2018/data/>
We are going to download them directly from kaggle through R. T do that, we need to our login in the 
R program, note that I hided my password for safety reason .


```r
# I Set data urls 
loginurl = "https://www.kaggle.com/account/login"
fileUrl_train_info<-"https://www.kaggle.com/c/recipient-prediction/download/training_info_sid.csv"
fileUrl_train_set<-"https://www.kaggle.com/c/recipient-prediction/download/training_set_sid.csv"
fileUrl_test_info<-"https://www.kaggle.com/c/recipient-prediction/download/test_info_sid.csv"
fileUrl_test_set<-"https://www.kaggle.com/c/recipient-prediction/download/test_set_sid.csv"
all_files<-list(fileUrl_train_info,fileUrl_train_set,fileUrl_test_info,fileUrl_test_set)


library(RCurl)
#Set user account data and agent
pars=list(
  UserName="m.beydia@gmail.com",
  Password="*******"
)
agent="Mozilla/5.0" #or whatever 

#Set RCurl pars
curl = getCurlHandle()
curlSetOpt(cookiejar="cookies.txt",  useragent = agent, followlocation = TRUE, curl=curl)


#Post login form
welcome=postForm(loginurl, .params = pars, curl=curl)

bdown=function(url, file, curl){
  f = CFILE(file, mode="wb")
  curlPerform(url = url, writedata = f@ref, noprogress=FALSE, curl = curl)
  close(f)
}

ret = lapply(1:4,function(i){bdown(all_files[[i]], substring(all_files[[i]],56),curl)})

rm(curl)
gc()
```
Once the data downloaded, we load them the R console.

```r
# set my directory
setwd("C:/Users/mlemi/Desktop/kaggle/Webmining/notebook")

# Import the datasets
train_info<-read.csv("data/training_info_sid.csv",header = F)
train_set<-read.csv("data/training_set_sid.csv",header = F)

test_info<-read.csv("data/test_info_sid.csv",header = F)
test_set<-read.csv("data/test_set_sid.csv",header = F)
```
<br>


III DATA PREPARATION
--------------------

In this section, we are going to restructure our data sets in order to have one train data set and one test data set.
Each row of theses data sets will contain one mail and one sender.
```r
### TRAIN ####

# I put names to the variables
names(train_info)<-c("Mail_ID","Date","Content","Recipients")
names(train_set)<-c("Sender","Mail_ID")

# This function will allow me to reshape the train_set and test_set
# The idea is to have, for each row, one mail and one sender 
my_funct<-function(i,df){
  subdf=subset(df,rownames(df)==i)
  Sender_list=strsplit(as.character(subdf$Mail_ID)," ")
  Mail_ID=as.numeric(unlist(Sender_list))
  Sender=subdf$Sender
  mydf=cbind.data.frame(Sender,Mail_ID)
  return(mydf)
}


require(plyr)
# I apply the function to the row of the train_set
mydf = ldply(1:dim(train_set)[1], my_funct, df=train_set, .progress = 'text')

# At the end I merge the previous output with the train_info set to get a complete data set
Training<-join(mydf, train_info,type = "left")

# I rearenge order of dataset
Training<-Training[,c(2,1,3:5)]

### Test ####

# I reapeat the same preprocessing to the test dataset
names(test_info)<-c("Mail_ID","Date","Content")
names(test_set)<-c("Sender","Mail_ID")

mydftest=ldply(1:dim(test_set)[1], my_funct, df=test_set, .progress = 'text')

Test<-join(mydftest, test_info,type = "left")
# rearenge order of dataset
Test<-Test[,c(2,1,3:4)]

```
Visualize what a I have done so far :
I choose do display these row precisely just for illustration sake since the content is not to big
```r
writeLines("td, th { padding : 6px } th { background-color : brown ; color : white; border : 1px solid white; } td { color : brown ; border : 1px solid brown }", con = "mystyle.css")
knitr::kable(Training[18:19,], format = "html")

```

<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-21-1.png" style="display: block; margin: auto;" />


```r
writeLines("td, th { padding : 6px } th { background-color : brown ; color : white; border : 1px solid white; } td { color : brown ; border : 1px solid brown }", con = "mystyle.css")
knitr::kable(Test[16:18,], format = "html")

```

<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-22-1.png" style="display: block; margin: auto;" />


<br>


IV  BASIC ANALYSIS
------------------

Let's start by counting how many how many emails has been sent or received per person.

But before that we will Reshape the training data set in  order to have for each mail , one recipient

```r

# The following function will help us to have have one recipient per mail sent
my_funct2<-function(i,df){
  subdf=subset(df,Mail_ID==i)
  recipients_list=strsplit(as.character(subdf$Recipients)," ")
  recipient=(unlist(recipients_list))
  mydftest=cbind.data.frame(subdf[1:4],recipient)
  return(mydftest)
}

invisible(suppressWarnings((Trainingv2 = ldply(min(Training$Mail_ID):max(Training$Mail_ID), my_funct2,df=Training))))
    

```

display what I have done so Fare

```r
writeLines("td, th { padding : 6px } th { background-color : brown ; color : white; border : 1px solid white; } td { color : brown ; border : 1px solid brown }", con = "mystyle.css")
knitr::kable(Trainingv2[1:3,], format = "html")
```
<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-23-1.png" style="display: block; margin: auto;" />

As you can see I have one recipient per row 

Now Lets start plotting : Received Mails

```r 
library(dplyr)
# firts I summarize the Training data by grouping the emails sent by recipient
email_recevied_count= Trainingv2 %>%
  dplyr::group_by(recipient) %>%
  dplyr::summarize(count = n())
# reorder the created dataset
email_recevied_count<-as.data.frame(email_recevied_count)[order(-email_recevied_count$count),]
# make a Barplot with ggplot
library(ggplot2)

p<-ggplot(data=email_recevied_count[1:15,], aes(x=recipient, y=count)) +
  geom_bar(stat="identity")+
  scale_x_discrete(limits = email_recevied_count[1:15,]$Sender)
p + coord_flip() 
```

<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-12-1.png" style="display: block; margin: auto;" />


second plot : Mails sent
```r
# firts I summarize the Training data by grouping the emails sent by sender
email_sent_count= Trainingv2 %>%
  dplyr::group_by(Sender) %>%
  dplyr::summarize(count = n())
# order the created dataset
email_sent_count<-as.data.frame(email_sent_count)[order(-email_sent_count$count),]
#  make a Barplot
p<-ggplot(data=email_sent_count[1:15,], aes(x=Sender, y=count)) +
  geom_bar(stat="identity")+
  scale_x_discrete(limits = email_sent_count[1:15,]$Sender)
p + coord_flip() 
```

<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-13-1.png" style="display: block; margin: auto;" />


We have seen the Hillary seems the one who send and receive the most emails , which make sense :)
Now Let's make a prediction based only on frequencies to see what will it score 


```r

Top10_receivers<-list(as.character(email_recevied_count[1:10,]$recipient))

rv=character() # Initialize an empty vector
for (i in 1:10){rv=paste(rv,unlist(Top10_receivers)[i])} # in the previous vector, we put the top 10 mail_adresses for each row 

pred_freq<-as.data.frame(cbind(Test$Mail_ID,rv)) # create a final dataframe in which i merge the mail ids and the top_mail 
names(pred_freq)<-c("Id","Recipients") # rename the headers
# export the dataframe into a csv file (needed format for kaggle)
write.csv2(pred_freq,file="topfreq_submission.csv",row.names = F)
```

<br>


V NETWORK ANALYSIS
------------------

In this section we will make some network(graph) analysis 

```r
emailInteractionsLinks = Trainingv2 %>%
  dplyr::group_by(Sender, recipient) %>%
  dplyr::summarize(count = n())

# let's consider only the case we have more than one interaction
emailInteractionsLinks = emailInteractionsLinks[emailInteractionsLinks$count > 1, ]
# initialize target and source
emailInteractionsLinks$source = -1
emailInteractionsLinks$target = -1

emailInteractionsLinks$Sender<-as.character(emailInteractionsLinks$Sender)
emailInteractionsLinks$recipient<-as.character(emailInteractionsLinks$recipient)

uniqueInteractionPersons = unique(c(emailInteractionsLinks$Sender, emailInteractionsLinks$recipient))
emailInteractionsNodes <- data.frame(name = uniqueInteractionPersons, stringsAsFactors=FALSE) 
emailInteractionsNodes$group = 0
emailInteractionsNodes$size = 0
```

display what I have done so Fare 
```r 
head(emailInteractionsLinks[1:5,])
head(emailInteractionsNodes[1:5,])
```

```r


for (i in 1:nrow(emailInteractionsLinks)) {
  senderIndex = match(emailInteractionsLinks[i, "Sender"], emailInteractionsNodes$name)
  recipientIndex = match(emailInteractionsLinks[i, "recipient"], emailInteractionsNodes$name)
  
  emailInteractionsLinks[i, ]$source = senderIndex - 1
  emailInteractionsLinks[i, ]$target = recipientIndex - 1
  
  emailInteractionsNodes[senderIndex, ]$size = emailInteractionsNodes[senderIndex, ]$size + 
    emailInteractionsLinks[i, ]$count
  
  emailInteractionsNodes[recipientIndex, ]$size = emailInteractionsNodes[recipientIndex, ]$size + emailInteractionsLinks[i, ]$count
}

emailInteractionsNodes$group = round(log2(emailInteractionsNodes$size + 1))
emailInteractionsNodes$size = round(sqrt(emailInteractionsNodes$size))

# display what I have done so Fare 
head(emailInteractionsLinks[1:5,])
head(emailInteractionsNodes[1:5,])

```

Now lets visualize the network
```r
library(networkD3)

network<-forceNetwork(Links = as.data.frame(emailInteractionsLinks), Nodes = emailInteractionsNodes,
             Source = "source", Target = "target",
             Value = "count", NodeID = "name",
             Group = "group", width = 800, height = 600,
             fontSize = 10, Nodesize = "size",
             opacity = 0.9, zoom = TRUE)
network
```

**Click to open the interactive network and use the mouse to zoom and turn the plot:**

<a href="network.html"><img border="0" alt="network" src="network.png" width="500" height="500">


Now lets detect communities in our network.
To do that we need to switch to another graph package and create again the network.

```r
suppressWarnings(library(igraph))

net <- graph_from_data_frame(d=emailInteractionsLinks, vertices=emailInteractionsNodes, directed=T) 
net <- simplify(net, remove.multiple = F, remove.loops = T) 

net.sym <- as.undirected(net, mode= "collapse",edge.attr.comb=list(weight="sum", "ignore"))


# community clustering : fast greedy algorithm
cfg <- cluster_fast_greedy(as.undirected(net))
dendPlot(cfg, mode="hclust")


<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-19-1.png" style="display: block; margin: auto;" />


com_members<-membership(cfg)

community<-as.data.frame(cbind(V(net)$name,cfg$membership))
names(community)<-c("Sender","Community_grp")

```
 
The communities were created , we see on the dendograme how many communities we have .
Now lets have a look on whats is inside our communities .

```r
writeLines("td, th { padding : 6px } th { background-color : brown ; color : white; border : 1px solid white; } td { color : brown ; border : 1px solid brown }", con = "mystyle.css")
knitr::kable(head(community), format = "html")

```

what I found interesting is I get quite good groups like the group 6 in which we found 
only Hilary's family : Chelsea , bill and a common mail dress (I guess) to Hillary and bill .
This encouraged my to keep this result and to not use other community detection algorithms.

```r
writeLines("td, th { padding : 6px } th { background-color : brown ; color : white; border : 1px solid white; } td { color : brown ; border : 1px solid brown }", con = "mystyle.css")
knitr::kable(community[which(community$Community_grp %in% 6),], format = "html")
# I add the community feature to my datasets 
library(plyr)
Trainingv3<-join(Trainingv2, community,type = "left")
Testv3<-join(Test, community,type = "left")
```

<br>


VI TEXT MINING 
--------------

### Pre-processing
Let's do some  pre-processing on the mail content .
```r

#Train dataset
Trainingv3$Content <- as.character(Trainingv3$Content)
Trainingv3$Content <- gsub("'", "", Trainingv3$Content)  # remove apostrophes
Trainingv3$Content <- gsub("[[:punct:]]", " ", Trainingv3$Content)  # replace punctuation with space
Trainingv3$Content <- gsub("[[:cntrl:]]", " ", Trainingv3$Content)  # replace control characters with space
Trainingv3$Content <- gsub("^[[:space:]]+", "", Trainingv3$Content) # remove whitespace at beginning of documents
Trainingv3$Content <- gsub("[[:space:]]+$", "", Trainingv3$Content) # remove whitespace at end of documents
Trainingv3$Content <- tolower(Trainingv3$Content)  # force to lowercase
#Test dataset
Testv3$Content <- as.character(Testv3$Content)
Testv3$Content <- gsub("'", "", Testv3$Content)  # remove apostrophes
Testv3$Content <- gsub("[[:punct:]]", " ", Testv3$Content)  # replace punctuation with space
Testv3$Content <- gsub("[[:cntrl:]]", " ", Testv3$Content)  # replace control characters with space
Testv3$Content <- gsub("^[[:space:]]+", "", Testv3$Content) # remove whitespace at beginning of documents
Testv3$Content <- gsub("[[:space:]]+$", "", Testv3$Content) # remove whitespace at end of documents
Testv3$Content <- tolower(Testv3$Content)  # force to lowercase
```

### Topic extraction
Now let's extract the topics from the content of each mail.
We will use this later as a feature in our model.
```r

## Source topicmodels2LDAvis & optimal_k functions
if (!require("pacman")) install.packages("pacman")
pacman::p_load_gh("trinker/gofastr")
pacman::p_load(tm, topicmodels, dplyr, tidyr, igraph, devtools, LDAvis, ggplot2)

## Source topicmodels2LDAvis & optimal_k functions
invisible(lapply(
  file.path(
    "https://raw.githubusercontent.com/trinker/topicmodels_learning/master/functions", 
    c("topicmodels2LDAvis.R", "optimal_k.R")
  ),
  devtools::source_url
))

##########generate stopwords ##########
stops <- c(
  tm::stopwords("english"),
  tm::stopwords("SMART")
) %>%
  gofastr::prep_stopwords() 

##########Create the DocumentTermMatrix ##########

mail_body<-Corpus(VectorSource(Trainingv3$Content)) 
dtm <- DocumentTermMatrix(mail_body)


doc_term_mat <- dtm  %>%
  gofastr::remove_stopwords(stops, stem=TRUE) %>%                                                    
  gofastr::filter_tf_idf() %>%
  gofastr::filter_documents() 

############## CONTROl LIST ################

control <- list(burnin = 500, iter = 1000, keep = 100, seed = 2500)

```

```r

############## Dertmine optimal number of topics  ################

(k <- optimal_k(doc_term_mat, 50, control = control))

```

<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-25-1.png" style="display: block; margin: auto;" />

As you can see, the number of optimal topics sees to be equal fourteen.


```r

control[["seed"]] <- 100
lda_model <- topicmodels::LDA(doc_term_mat, k=40, method = "Gibbs", 
                              control = control)
```

```r

############## Run the model   ################

control[["seed"]] <- 100
lda_model <- topicmodels::LDA(doc_term_mat, k=as.numeric(k), method = "Gibbs", 
                              control = control)
```

```r 

######################### get the topic ######################### 

topics <- topicmodels::posterior(lda_model, doc_term_mat)[["topics"]]
topic_dat <- dplyr::add_rownames(as.data.frame(topics), "Mail")
colnames(topic_dat)[-1] <- apply(terms(lda_model, 10), 2, paste, collapse = ", ")


```
```r
topic_dat <- readRDS("topic_dat.rds")
topics <- readRDS("topics.rds")


```

```r

##################### Nice visualization ###########################

lda_model %>%
  topicmodels2LDAvis() %>%
  LDAvis::serVis()

```
<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-27-1.png" style="display: block; margin: auto;" />

As you can, the output is a webpage on the localhost that allows to see what are the main frequent words by topics. Note also that we have a slider that allows us to change the value of relevant metrics :


Intertopic Distance Map (via multidimensional scaling)
Overall term frequency Estimated term frequency within the selected topic
1. saliency(term w) = frequency(w) * [sum_t p(t | w) * log(p(t | w)/p(t))] for topics t; see Chuang et. al (2012)


2. relevance(term w | topic t) = ?? * p(w | t) + (1 - ??) * p(w | t)/p(w); see Sievert & Shirley (2014)



Let's have a look at the content of our topic extraction .

```r
### appending the most likly topic to my train dataset

library(data.table)
topic_dat<-as.data.table(topic_dat)


writeLines("td, th { padding : 6px } th { background-color : brown ; color : white; border : 1px solid white; } td { color : brown ; border : 1px solid brown }", con = "mystyle.css")
knitr::kable(head(topic_dat), format = "html")
```
<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-28-1.png" style="display: block; margin: auto;" />


```r

library(data.table)
topic_dat<-as.data.table(topic_dat)
names(topic_dat)[2:41]<-c(1:40)

a<-as.data.frame(t(apply(topic_dat[,-1], 1, function(x) 
  head(names(topic_dat[,-1])[order(-x)],3))), stringsAsFactors=FALSE)

topic_dat_with_top3<-cbind.data.frame(id_row=topic_dat$Mail,topic1=a$V1,topic2=a$V2,topic3=a$V3)
topic_dat_with_top3$topic1<-paste0("topic",topic_dat_with_top3$topic1)
topic_dat_with_top3$topic2<-paste0("topic",topic_dat_with_top3$topic2)
topic_dat_with_top3$topic3<-paste0("topic",topic_dat_with_top3$topic3)


library(plyr)
Trainingv3$id_row<-row.names(Trainingv3)
Trainingv4<-join(Trainingv3,topic_dat_with_top3,type="left")

##################### Apply to test dataset ###########################

test_mail_body<-Corpus(VectorSource(Testv3$Content)) 
test_dtm <- DocumentTermMatrix(test_mail_body)

## Create the DocumentTermMatrix for New Data
doc_term_mat2 <- test_dtm %>%
  gofastr::remove_stopwords(stops, stem=TRUE) %>%                                                    
  gofastr::filter_tf_idf() %>%
  gofastr::filter_documents() 


## Update Control List
control2 <- control
control2[["estimate.beta"]] <- FALSE


## Run the Model for New Data
lda_model2 <- topicmodels::LDA(doc_term_mat2, k = k, model = lda_model, 
                               control = list(seed = 100, estimate.beta = FALSE))



topics2 <- topicmodels::posterior(lda_model2, doc_term_mat2)[["topics"]]
topic_dat2 <- dplyr::add_rownames(as.data.frame(topics2), "Mails")
colnames(topic_dat2)[-1] <- apply(terms(lda_model2, 10), 2, paste, collapse = ", ")



### appending the most likly topic to my train dataset
topic_dat2<-as.data.table(topic_dat2)
names(topic_dat2)[2:41]<-c(1:40)



a2<-as.data.frame(t(apply(topic_dat2[,-1], 1, function(x) 
  head(names(topic_dat[,-1])[order(-x)],3))), stringsAsFactors=FALSE)

test_topic_dat_with_top3<-cbind.data.frame(id_row=topic_dat2$Mails,topic1=a2$V1,topic2=a2$V2,topic3=a2$V3)

test_topic_dat_with_top3$topic1<-paste0("topic",test_topic_dat_with_top3$topic1)
test_topic_dat_with_top3$topic2<-paste0("topic",test_topic_dat_with_top3$topic2)
test_topic_dat_with_top3$topic3<-paste0("topic",test_topic_dat_with_top3$topic3)



Testv3$id_row<-row.names(Testv3)
Testv4<-join(Testv3,test_topic_dat_with_top3,type="left")

```

<br>


VII FEATURE ENGINEERING 
-----------------------

In this section we will create some features from the data sets that I created so fare
the variables  that I will crate are :


**-Community_grp** : same variable created before but It will be in text here


**-weekday** : the day of the week extracted from variable date


**-weekend** : the weekend extracted from variable weekday


**-year** : the year extracted from variable date


**-week**: the week number extracted from the variable date


**-wordcount**: the number of characters of mail body


**-nb_msj_sent**: the number of msj sent by mail's sender


**-sender_username**:



We will create a function that will convert any numeric values to string value.


I will explain in the moderation section , why I did that .
```r

numbers2words <- function(x){

  helper <- function(x){
    
    digits <- rev(strsplit(as.character(x), "")[[1]])
    nDigits <- length(digits)
    if (nDigits == 1) as.vector(ones[digits])
    else if (nDigits == 2)
      if (x <= 19) as.vector(teens[digits[1]])
    else trim(paste(tens[digits[2]],
                    Recall(as.numeric(digits[1]))))
    else if (nDigits == 3) trim(paste(ones[digits[3]], "hundred and", 
                                      Recall(makeNumber(digits[2:1]))))
    else {
      nSuffix <- ((nDigits + 2) %/% 3) - 1
      if (nSuffix > length(suffixes)) stop(paste(x, "is too large!"))
      trim(paste(Recall(makeNumber(digits[
        nDigits:(3*nSuffix + 1)])),
        suffixes[nSuffix],"," ,
        Recall(makeNumber(digits[(3*nSuffix):1]))))
    }
  }
  trim <- function(text){
    #Tidy leading/trailing whitespace, space before comma
    text=gsub("^\ ", "", gsub("\ *$", "", gsub("\ ,",",",text)))
    #Clear any trailing " and"
    text=gsub(" and$","",text)
    #Clear any trailing comma
    gsub("\ *,$","",text)
  }  
  makeNumber <- function(...) as.numeric(paste(..., collapse=""))     
  #Disable scientific notation
  opts <- options(scipen=100) 
  on.exit(options(opts)) 
  ones <- c("", "one", "two", "three", "four", "five", "six", "seven",
            "eight", "nine") 
  names(ones) <- 0:9 
  teens <- c("ten", "eleven", "twelve", "thirteen", "fourteen", "fifteen",
             "sixteen", " seventeen", "eighteen", "nineteen")
  names(teens) <- 0:9 
  tens <- c("twenty", "thirty", "forty", "fifty", "sixty", "seventy", "eighty",
            "ninety") 
  names(tens) <- 2:9 
  x <- round(x)
  suffixes <- c("thousand", "million", "billion", "trillion")     
  if (length(x) > 1) return(trim(sapply(x, helper)))
  helper(x)
}
```

```r

## train dataset

#Communities
Trainingv4$Community_grp<-as.integer(Trainingv4$Community_grp)
Trainingv4$Community_grp<-paste0("community",numbers2words(Trainingv4$Community_grp))
# Weekday
Trainingv4$weekday = weekdays(as.Date(Trainingv4$Date))
# Weekend
Trainingv4$weekend = ifelse(Trainingv4$weekday %in% c('samedi','dimanche'),"wkend","nowkend") 
# Year
Trainingv4$year<-year(as.Date(Trainingv4$Date)) 
Trainingv4$year<-recode(Trainingv4$year,`2009`="dmilleneuf",`2010`="dmilledix",`2011`="dmilleonze",
                     `2012`="dmilledouze")
# Week (from 1 to 54)
Trainingv4$week<-week(as.Date(Trainingv4$Date)) 
Trainingv4$week<-paste0("wk", gsub(" ", "", numbers2words(Trainingv4$week)))
# Number of strings in the mail's body
Trainingv4$wordcount<-nchar(Trainingv4$Content)
# Number of messages sent by the sender of mail
library(sqldf)
nb_msj_sent<-sqldf("SELECT Sender,COUNT(DISTINCT(Mail_ID)) as nb_msj_sent FROM Trainingv4 GROUP BY Sender")
Trainingv4<-join(Trainingv4,nb_msj_sent,type='left')
```
Now we will transform the variables nb_msj_sent and wordcount into categorical variables by binning their numerical versions.

```r

# Binning the variables nb_msj_sent and wordcount
hist(as.integer(nb_msj_sent$nb_msj_sent),main ="Number of msj sent by mail's sender")
hist(as.numeric(Trainingv4$wordcount),main ="Number of characters per mail")
```
<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-34-1.png" style="display: block; margin: auto;" />
<img src="WebminingKaggle_post_files/figure-markdown_github/unnamed-chunk-34-2.png" style="display: block; margin: auto;" />


```r

Trainingv4$nb_msj_sent<-cut(Trainingv4$nb_msj_sent,breaks =c(0,500,1100,1500),include.lowest = T)
levels(Trainingv4$nb_msj_sent)<-c("fewsent","mediumsent","largesent")
Trainingv4$wordcount<-cut(Trainingv4$wordcount,breaks =c(0,77,227,584,7824),include.lowest = T)
levels(Trainingv4$wordcount)<-c("verysmallcnt","smallcnt","mediumcnt","hugecnt")


## test dataset

# Communities
Testv4$Community_grp<-as.integer(Testv4$Community_grp)
Testv4$Community_grp<-paste0("community",numbers2words(Testv4$Community_grp))
# Weekday
Testv4$weekday = weekdays(as.Date(Testv4$Date))
# Weekend
Testv4$weekend = ifelse(Testv4$weekday %in% c('samedi','dimanche'),"wkend","nowkend")
# Year
Testv4$year<-year(as.Date(Testv4$Date)) 
Testv4$year<-recode(Testv4$year,`2009`="dmilleneuf",`2010`="dmilledix",`2011`="dmilleonze",
                     `2012`="dmilledouze")
# Week (from 1 to 54)
Testv4$week<-week(as.Date(Testv4$Date)) 
Testv4$week<-paste0("wk", gsub(" ", "", numbers2words(Testv4$week)))
# Number of strings in the mail's body
Testv4$wordcount<-nchar(Testv4$Content)
# Number of messages sent by the sender of mail
nb_msj_sent_test<-sqldf("SELECT sender,COUNT(DISTINCT(Mail_ID)) as nb_msj_sent FROM Testv4 GROUP BY Sender")
Testv4<-join(Testv4,nb_msj_sent_test,type='left')
# Binning the variables nb_msj_sent and wordcount

Testv4$nb_msj_sent<-cut(Testv4$nb_msj_sent,breaks =c(0,500,1100,1500),include.lowest = T)
levels(Testv4$nb_msj_sent)<-c("fewsent","mediumsent","largesent")
Testv4$wordcount<-cut(Testv4$wordcount,breaks =c(0,77,227,584,7824),include.lowest = T)
levels(Testv4$wordcount)<-c("verysmallcnt","smallcnt","mediumcnt","hugecnt")

```


```r
# We clean the sender mail adress and extract his user name
# To do that we create a function 
library(exploratory)
library(stringr)
clean_funct<-function(df){
  sender_mail<-str_split(df$Sender, "@")
  df$sender_user<-gsub(",", ".", gsub("\\.", "", list_extract(sender_mail, 1)))
  return(df)
}

# Apply the function
Trainingv5<-clean_funct(Trainingv4)
Testv5<-clean_funct(Testv4)
```

Lets visualize the final train and test data set.

```r
### drop usless columns
Trainingv5<-Trainingv5[-7]
Testv5<-Testv5[-6]
## export data sets into csv
write.csv(Trainingv5,"Trainingv5.csv",row.names = F,fileEncoding = "UTF-8")
write.csv(Testv5,"Testv5.csv",row.names = F,fileEncoding = "UTF-8")


writeLines("td, th { padding : 6px } th { background-color : brown ; color : white; border : 1px solid white; } td { color : brown ; border : 1px solid brown }", con = "mystyle.css")
knitr::kable(head(Trainingv5[1:3,]), format = "html")
knitr::kable(head(Testv5[1:3,]), format = "html")
```

We have created the train and test sets that we will use for modeling.
Let's move to python to do some predictions.

------------------------------------------------------------------------

<br>

``` r
sessionInfo()
```

  ## R version 3.5.0 (2018-04-23)
  ## Platform: x86_64-w64-mingw32/x64 (64-bit)
  ## Running under: Windows >= 8 x64 (build 9200)
  ## 
  ##Matrix products: default
  ## 
  ## locale:
  ## [1] LC_COLLATE=French_France.1252  LC_CTYPE=French_France.1252    LC_MONETARY=French_France.1252 LC_NUMERIC=C                  
  ## [5] LC_TIME=French_France.1252    
  ## 
  ## attached base packages:
  ## [1] stats     graphics  grDevices utils     datasets  methods   base     
  ## 
  ## other attached packages:
  ## [1] stringr_1.3.1     data.table_1.11.2 gofastr_0.3.1     pacman_0.4.6      igraph_1.2.1      networkD3_0.4    
  ## 
  ## loaded via a namespace (and not attached):
  ## [1] Rcpp_0.12.16     knitr_1.20       bindr_0.1.1      magrittr_1.5     tidyselect_0.2.4 munsell_0.4.3    colorspace_1.3-2
  ## [8] R6_2.2.2         rlang_0.3.0.1    plyr_1.8.4       dplyr_0.7.5      tools_3.5.0      grid_3.5.0       gtable_0.2.0    
  ## [15] htmltools_0.3.6  yaml_2.1.19      lazyeval_0.2.1   assertthat_0.2.0 digest_0.6.15    tibble_1.4.2     bindrcpp_0.2.2  
  ## [22] purrr_0.2.4      ggplot2_3.1.0    htmlwidgets_1.2  glue_1.2.0       stringi_1.1.7    compiler_3.5.0   pillar_1.2.2    
  ## [29] scales_0.5.0     jsonlite_1.5     pkgconfig_2.0.1 

  