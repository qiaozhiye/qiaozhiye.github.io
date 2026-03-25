---
categories:
- resource
date: "2022-07-30"
slug: 10-k-public-float
tags:
- filer status
- public float
- data scraping
title: "Extract Filer Status and Public Float from 10-K"
headline: Here is my R code to extract filer status and public float of U.S. listed companies from their 10-K filings obtained from EDGAR. Note that you may need to manually verify the output and fill up the missing ones after running this code.
authors:
  - "Qiaozhi Ye"
authorURLs:
  - "https://www.yegeorge.com"

---

```r

##### Load Package #####

library(edgar)
library(stringr)
library(lubridate)
library(edgarWebR)
library(dplyr)
library(tidyr)
library(textshape)
library(readr)

##### Download 10-K from EDGAR #####

## set directory
setwd("/Users/yeqiaozhi/Dropbox/project/data/")    #enter your own directory here

get_10_k <- getFilingsHTML(cik.no = "ALL",    #enter companys' cik or use "ALL"
                           form.type = "10-K", 
                           filing.year = 2022,    #enter filing year, e.g., 2022
                           quarter = c(1, 2, 3, 4), 
                           useragent = "xxx@xxx.com"    #enter your email address
                           )

##### Extract filter status and public float #####

## create an empty dataset to save results
parse_df <- data.frame()

## extract all cik inside the folder that stores the 10-K HTML files.
cik_list <- list.files("/Users/yeqiaozhi/Dropbox/project/data/Edgar filings_HTML view/Form 10-K/")

## extract filer status and public float using iteration

for (i in 1:length(cik_list)){
  
  ## get all 10-K file name under a cik folder
  file_name <- list.files(paste0("/Users/yeqiaozhi/Dropbox/project/data/Edgar filings_HTML view/Form 10-K/",cik_list[i],"/"))
  
  for (w in 1:length(file_name)){
    
    ## get the full 10-K file path
    file <- paste0("/Users/yeqiaozhi/Dropbox/project/data/Edgar filings_HTML view/Form 10-K/",cik_list[i],"/",file_name[w])    #replace "/Users/yeqiaozhi/Dropbox/project/data/" with your own location.
    
    ## get filing date
    file_date <- as.Date(str_remove_all(str_extract(file,"10-K_.{10}"),"10-K_"),"%Y-%m-%d")
    
    ## read and get the first 10,000 characters from the 10-K file. Filer status and public float are usually shown in the front page so the first 10,000 characters should be able to capture the information.
    script <- tolower(substr(parse_filing(file, strip = TRUE, include.raw = FALSE, fix.errors = TRUE)[1],1,10000))
    
    ## remove all whitespace
    script_nospace <- gsub(x = gsub("\\s+", "", script), pattern = intToUtf8(160),replacement =   "")
    
    ## extract prior 35 characters before the symbols indicating "yes". Filer status usually presents before one of them.
    symbol_1 <- data.frame(str_extract_all(script_nospace,".{30}☒")) %>% rename(words=1)
    symbol_2 <- data.frame(str_extract_all(script_nospace,".{30}[X]")) %>% rename(words=1)
    symbol_3 <- data.frame(str_extract_all(script_nospace,".{30}[x]")) %>% rename(words=1)
    symbol_4 <- data.frame(str_extract_all(script_nospace,".{30}☑")) %>% rename(words=1)
    symbol_5 <- data.frame(str_extract_all(script_nospace,".{30}⌧")) %>% rename(words=1)    
    symbol_6 <- data.frame(str_extract_all(script_nospace,".{30}þ")) %>% rename(words=1)
    symbol_7 <- data.frame(str_extract_all(script_nospace,".{30}ý")) %>% rename(words=1)
    
    ## combine all extracted words and find the filer status
    filer_status <- bind_rows(symbol_1,symbol_2,symbol_3,symbol_4,symbol_5,symbol_6,symbol_7) %>% 
      mutate(words=str_remove_all(words," "),
             status=case_when(str_detect(words,"reporting") & str_detect(words,"company") ~"src",
                              str_detect(words,"large") & str_detect(words,"accelerated")~"laf",
                              str_detect(words,"accelerated") & !str_detect(words,"large") & !str_detect(words,"non-")~"af",
                              str_detect(words,"non-accelerated") ~"naf",
                              str_detect(words,"growth") & str_detect(words,"company")~"egc")) %>% 
      filter(!is.na(status)) %>% 
      summarise(status=paste(status,collapse = ",")) %>% 
      mutate(src=ifelse(str_detect(status,"src"),TRUE,FALSE),
             naf=ifelse(str_detect(status,"naf"),TRUE,FALSE),
             af=ifelse(str_detect(status,"af") & !str_detect(status,"laf") & !str_detect(status,"naf"),TRUE,FALSE),
             laf=ifelse(str_detect(status,"laf"),TRUE,FALSE),
             egc=ifelse(str_detect(status,"egc"),TRUE,FALSE),
             status=ifelse(status=="",NA,status))
    
    ## split the script into sentences
    sentence <- data.frame(split_sentence(script)) %>% rename(sentence=1)
    
     ## extract the sentence containing public float
    public_float <- sentence %>% 
      filter(str_detect(sentence,"affiliates"),
             str_detect(sentence,"(?i)value"),
             str_detect(sentence,"(?i)\\$")) 
    
    #if public float is non-empty
    if (nrow(public_float)>0){
    public_float <- public_float %>% 
      group_by(row_number()) %>% #iterate in each sentence
      mutate(# remove share price from the sentences
             sentence=ifelse(!is.na(str_extract(sentence,"\\$[^\\$]+par value|\\$[^\\$]+per share")) & 
                                      nchar(str_extract(sentence,"\\$[^\\$]+par value|\\$[^\\$]+per share"))<20,
                             str_remove_all(sentence,"\\$[^\\$]+par value|\\$[^\\$]+per share"),sentence),
             #  extract the float and convert it to dollar value
             publicfloat=case_when(!is.na(str_extract(sentence,"\\$[^\\$]+thousand")) & str_detect(str_extract(sentence,"\\$[^\\$]+thousand"),"\\d.") ~ 
                                     parse_number(str_extract(sentence,"\\$[^\\$]+thousand"))*1000,
                                   !is.na(str_extract(sentence,"\\$[^\\$]+million")) & str_detect(str_extract(sentence,"\\$[^\\$]+million"),"\\d.") ~ 
                                     parse_number(str_extract(sentence,"\\$[^\\$]+million"),"|")*1000000,
                                   !is.na(str_extract(sentence,"\\$[^\\$]+billion")) & str_detect(str_extract(sentence,"\\$[^\\$]+billion"),"\\d.") ~ 
                                     parse_number(str_extract(sentence,"\\$[^\\$]+billion"))*1000000000,
                                   !is.na(str_extract(sentence,"\\$[^\\$]+trillion")) & str_detect(str_extract(sentence,"\\$[^\\$]+trillion"),"\\d.") ~ 
                                     parse_number(str_extract(sentence,"\\$[^\\$]+trillion"))*1000000000000,
                                   !is.na(str_extract_all(str_remove_all(sentence," "),"\\$([0-9,.]+)")) & 
                                     !is.na(match(0,lapply(str_extract_all(str_remove_all(sentence," "),"\\$([0-9,.]+)"),parse_number)[[1]])) ~ 
                                     0,
                                   !is.na(str_extract_all(str_remove_all(sentence," "),"\\$([0-9,.]+)")) & 
                                     is.na(match(0,lapply(str_extract_all(str_remove_all(sentence," "),"\\$([0-9,.]+)"),parse_number)[[1]])) ~ 
                                     max(lapply(str_extract_all(str_remove_all(sentence," "),"\\$([0-9,.]+)"),parse_number)[[1]])),
             publicfloat=ifelse(is.infinite(publicfloat),NA,publicfloat)) %>% 
      ungroup() %>% select(-`row_number()`) %>% 
      filter(sum(!is.na(publicfloat))==0 | !is.na(publicfloat)) 
    
    ## save data into parse_df and add cik, file path, and file date
    i_parse_df <- merge(public_float, filer_status)%>% 
      mutate(cik=cik_list[i], filepath=file, filedate=file_date)
    
    parse_df <- bind_rows(parse_df,i_parse_df) 
    }
    else{#if public float is empty
      parse_df <- bind_rows(parse_df,filer_status%>% mutate(cik=cik_list[i], filepath=file, filedate=file_date)) 
    }
    
    ## print out the cik and its index in cik_list, to inform that you complete the data collection for this filer
    print(paste("cik:",cik_list[i],"; index: ",i))
  }
}

## save results
write_csv(parse_df,"filer_status_public_float.csv")
```
