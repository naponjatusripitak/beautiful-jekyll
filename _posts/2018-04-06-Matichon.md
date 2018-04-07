---
layout: post
title: จับประเด็นข่าวการเมืองจากมติชนออนไลน์
subtitle: วิเคราะห์ข้อมูลข่าวการเมืองด้วย Latent Dirichlet Allocation
gh-repo: naponjatusripitak/polisci490/Matichon
gh-badge: [star, fork, follow]
tags: [LDA]
---

# วิเคราะห์ข่าวการเมืองจากมติชนออนไลน์ด้วย Latent Dirichlet Allocation

### ก่อนอื่นต้องโหลด package ที่ต้องใช้ในการเก็บข้อมูลและการวิเคราะห์
```R
packages <- c("xml2","rvest", "dplyr", "tm", "tidytext", "ggplot2", "stopwords", "SnowballC", "tidyverse", "lubridate", "stringr", "httr", "SnowballC", "wdman", "RSelenium", "tcltk", "XML", "topicmodels", "stringi", "LDAvis", "slam", "ldatuning", "kableExtra", "widyr", "igraph", "ggraph", "fmsb", "pander")

load.packages <- function(x) {
  if (!require(x, character.only = TRUE)) {
    install.packages(x, dependencies = TRUE)
    library(x, character.only = TRUE)
  }
}

lapply(packages, load.packages)
```
### เริ่ม scrape ข้อมูลข่าวด้วย ```RSelenium```
```R
rD <- rsDriver()
remDr <- remoteDriver(remoteServerAddr = "localhost" 
                      , port = 4567L
                      , browserName = "chrome"
)

remDr$open()

matichon <- data.frame(title=NULL, text=NULL, date=NULL, link=NULL)

remDr$setTimeout(type = "page load", milliseconds = 100000)
remDr$setImplicitWaitTimeout(milliseconds = 100000)

lastpage <- 2249

for(i in 1:lastpage) {
  tryCatch({
    site <- paste0("https://www.matichon.co.th/category/politics/page/",i) # create URL for each page to scrape
    remDr$navigate(site) # navigates to webpage
    elem <- remDr$findElements(using="class", value="item-details")
    url_find <- remDr$findElements(using="xpath", "//*[@id='td-outer-wrap']/div[2]/div/div[4]/div[1]/div/div/div[2]/h3/a")
    
    url <- unlist(sapply(url_find, function(x) {x$getElementAttribute("href")}))
    title <- unlist(sapply(url_find, function(x) {x$getElementAttribute("title")}))
    meta_find <- remDr$findElements(using="xpath", "//*[@id='td-outer-wrap']/div[2]/div/div[4]/div[1]/div/div/div[2]/div[1]/span[2]/time")
    date <- unlist(sapply(meta_find, function(x) {x$getElementAttribute("datetime")}))
    
    article_body <- NULL
    for(j in 1:length(url)){
      article_html <- url[j] %>%
        read_html() 
      article_body[j] <- article_html %>%
        html_nodes(xpath="//div[@class='td-post-content']/p") %>%
        html_text() %>%
        paste(collapse = '\n\n')
    }
    
    matichon_page <- data.frame(title=title, text=article_body, date=date, link=url)
    matichon <- rbind(matichon, matichon_page)
    
  }, error=function(e){
    Sys.sleep(sample(1:10, 1))
    remDr$navigate(site) # navigates to webpage
    elem <- remDr$findElements(using="class", value="item-details")
    url_find <- remDr$findElements(using="xpath", "//*[@id='td-outer-wrap']/div[2]/div/div[4]/div[1]/div/div/div[2]/h3/a")
    
    url <- unlist(sapply(url_find, function(x) {x$getElementAttribute("href")}))
    title <- unlist(sapply(url_find, function(x) {x$getElementAttribute("title")}))
    meta_find <- remDr$findElements(using="xpath", "//*[@id='td-outer-wrap']/div[2]/div/div[4]/div[1]/div/div/div[2]/div[1]/span[2]/time")
    date <- unlist(sapply(meta_find, function(x) {x$getElementAttribute("datetime")}))
    
    article_body <- NULL
    for(j in 1:length(url)){
      article_html <- url[j] %>%
        read_html() 
      article_body[j] <- article_html %>%
        html_nodes(xpath="//div[@class='td-post-content']/p") %>%
        html_text() %>%
        paste(collapse = '\n\n')
    }
    
    matichon_page <- data.frame(title=title, text=article_body, date=date, link=url)
    matichon <- rbind(matichon, matichon_page)})
}
```

![alt text](https://naponjatusripitak.github.io/polisci490/Matichon/datastructure.png)
<p align="center">ตัวอย่าง dataframe ที่ได้จาการดึงข้อมูลข่าว</p>

### ทำการตัดคำด้วย Thai National Language Toolkit จาก https://github.com/pichaio/thainltk
```R
devtools::install_github("pichaio/thainltk")
library(thainltk)
tt <- thaiTokenizer(skipSpace = T)

matichon$text <- as.character(matichon$text)
matichon$title <- as.character(matichon$title)
matichon$link <- as.character(matichon$link)
matichon$date <- as_date(matichon$date)
matichon <- matichon %>% filter(str_length(text) > 100)

tokenize <- function(x){
  texts <- lapply(x, tt)
  tokens <- sapply(texts, function(k){
    paste(k, collapse=" ")
  })
  result <- gsub("”|“", "", tokens)
  return(result)
}

matichon$text <- tokenize(matichon$text)

# Remove duplicate rows
matichon <- distinct(matichon)

# Add doc_id
matichon <- matichon %>% mutate(doc_id = seq.int(nrow(matichon)))

# Finalize dataset
matichon <- matichon %>% select(doc_id, text, everything())

# Transform to corpus object
matichon.corpus <- VCorpus(DataframeSource(matichon))
```
![alt text](https://naponjatusripitak.github.io/polisci490/Matichon/datastructure2.png)
<p align="center">ตัวอย่าง dataframe หลังทำการตัดคำ</p>

### จัดระเบียบก่อนทำ Topic Modeling

```R
th.stop <- c(stopwords("th", source = "stopwords-iso"), c("ที่", "ให้", "ได้", "นี้", "ใช้", "ไม่", "แต่", "รู้", "ฝ่าย", "อยู่", "ผู้", "ไว้","อัน", "ใส่", "ตัว") )
th.stop <- unique(th.stop)
pattern <- paste(th.stop, collapse="|")

matichon.dtm <- matichon.corpus %>% 
  tm_map(content_transformer(function(x) gsub(x, pattern = pattern, replacement = ""))) %>%
  tm_map(removeWords, character(0)) %>%
  tm_map(removeNumbers) %>%
  tm_map(removePunctuation) %>%
  tm_map(stemDocument) %>%
  tm_map(stripWhitespace) %>%
  DocumentTermMatrix() %>%
  removeSparseTerms(sparse = 0.99)
```
### Topic Modeling ด้วย LDA (k=60)
```R
matichon.60 <- LDA(matichon.dtm, k=60, control = list(seed=6))
```

### Visualize ด้วย ```LDAvis``` (https://github.com/cpsievert/LDAvis)
```R
topicmodels_json_ldavis <- function(fitted, corpus, doc_term){
  ## Required packages
  library(topicmodels)
  library(dplyr)
  library(stringi)
  library(tm)
  library(LDAvis)
  
  ## Find required quantities
  phi <- posterior(fitted)$terms %>% as.matrix
  theta <- posterior(fitted)$topics %>% as.matrix
  vocab <- colnames(phi)
  doc_length <- vector()
  for (i in 1:length(corpus)) {
    temp <- paste(corpus[[i]]$content, collapse = ' ')
    doc_length <- c(doc_length, stri_count(temp, regex = '\\S+'))
  }
  temp_frequency <- as.matrix(doc_term[1:nrow(doc_term), 1:ncol(doc_term)])
  freq_matrix <- data.frame(ST = colnames(temp_frequency),
                            Freq = colSums(temp_frequency))
  rm(temp_frequency)
  
  ## Convert to json
  json_lda <- LDAvis::createJSON(phi = phi, theta = theta,
                                 vocab = vocab,
                                 doc.length = doc_length,
                                 term.frequency = freq_matrix$Freq)
  
  return(json_lda)
}


matichon.jason <- topicmodels_json_ldavis(matichon.60, matichon.corpus, matichon.dtm)
serVis(matichon.jason, out.dir = "~/polisci490/Matichon")
```



