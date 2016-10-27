---
layout: post
title:  Querying Zenodo.org repository with R
date: "2016-10-27 15:56:29"
published: true
tags: 
   - OAI-PMH 
   - zenodo
   - R

liftr:
  from: rocker/hadleyverse
  pandoc: false
  cranpkg:
    - oai
  maintainer: Carsten Behring
  maintainer_email: carsten.behring@gmail.com
---

# Zenodo 
[Zenodo](http://zenodo.org) is a repository which allows everybody to deposit free of charge any type of research output, in all disciplines of science.

EFSA is piloting it's use for creating a knowledge base on all types of food safety related evidence(data, documents, models).

Zenodo has an API and can be queried using the standard OAI-PMH protocol, which allows to harvest the metadata and all deposits.

# 'oai' package 

R has a package available to query any OAI-PMH repository, including Zenodo.
It can be installed from CRAN like this:


{% highlight r %}
install.packages("oai")
{% endhighlight %}

The development version is available on Github at 
<https://github.com/ropensci/oai>


The libraries I use in this tutorial are: 

{% highlight r %}
library(knitr)
library(tidyverse)
library(httr)
library(oai)
library(xml2)
opts_chunk$set(echo=T)
{% endhighlight %}

# Retreive records from Zenodo

The oai package  allows to retrieve all records of a given Zenodo community, in this case the EFSA pilot community.
The following code shows all records of a community with their digital object identifier and the title. 


{% highlight r %}
record_list<- list_records("https://zenodo.org/oai2d",metadataPrefix="oai_datacite",set="user-efsa-pilot")

kable(record_list %>% select(identifier.3,title))
{% endhighlight %}



|identifier.3          |title                                                                                                                                    |
|:---------------------|:----------------------------------------------------------------------------------------------------------------------------------------|
|10.5281/zenodo.57132  |EFSA Source Attribution Model (EFSA_SAM)                                                                                                 |
|10.5281/zenodo.57017  |PRIMo rev.1 – Pesticide Residue Intake Model                                                                                             |
|10.5281/zenodo.56662  |Bee-Tool V.1                                                                                                                             |
|10.5281/zenodo.56668  |Bee-Tool V.2                                                                                                                             |
|10.5281/zenodo.154720 |Egg Pooling Module                                                                                                                       |
|10.5281/zenodo.161300 |GMOANALYSIS VERSION 2.1.0 – 10 JULY 2014                                                                                                 |
|10.5281/zenodo.159163 |Pesticide Residues Overview File: PROFile (3.0)                                                                                          |
|10.5281/zenodo.154725 |Food Additives Intake Model (FAIM) - Version 1.1 - July 2013                                                                             |
|10.5281/zenodo.163080 |Modelling continental-scale spread of Schmallenberg virus in Europe                                                                      |
|10.5281/zenodo.57079  |C-TSEMM – Cattle TSE Monitoring Model                                                                                                    |
|10.5281/zenodo.57505  |TSEi – TSE Infectivity Model                                                                                                             |
|10.5281/zenodo.159414 |Dietary Exposure Calculator Smoke Flavouring                                                                                             |
|10.5281/zenodo.159890 |CHIP: Commodity based Hazard Identification Tool                                                                                         |
|10.5281/zenodo.56287  |PRIMo rev.2 – Pesticide Residue Intake Model                                                                                             |
|10.5281/zenodo.56669  |Bee-Tool V.3                                                                                                                             |
|10.5281/zenodo.161298 |Exposure of operators, workers, residents and bystanders in risk assessment for plant protection products calculator (Version 30MAR2015) |
|10.5281/zenodo.163026 |Within farms transmission model for Schmallenberg Virus                                                                                  |
|10.5281/zenodo.154724 |User-friendly interface version of the QMRA model for Salmonella in pigs                                                                 |

Currently there are 18 records available.

## Statistics on keywords

### Query records from Zenodo

I was further on interested in the current distribution of keywords each record was tagged with. Zenodo supports two types of keywords. Simple free text keywords and 'subjects'.
Subjects need to come from  a controlled vocabulary, in which each topic has an URI.

EFSA uses the [GACS](http://browser.agrisemantics.org/gacs/en/) vocabulary, and so a certain topic 'salmonella' is represented as URI 'http://browser.agrisemantics.org/gacs/en/page/C2225'.

The API returns therefore for the subjects only the URI, which is nicely unique and clear but not user friendly as a label. On the URI of each 'subject', additional information is available.



The following code retrieves all records and extract all their subjects (which have a Xpath of //d3:subject). The current oai package has some problems with some Zenodo specific metadata,
so I parse the raw XML by hand.

The OIA-PMH standard and the oai::get_records function, allow the client to select, in which metadata format he wants to receive the metadata.
Here I have selected 'oai-datacite', because it is recommended from the 
Zenodo API [documenation](https://zenodo.org/dev#harvest-metadata) and should contain *all* metadata Zenodo supports, while other metadata formats might only support a smaller subset.


{% highlight r %}
record_data_xml <- get_records(record_list$identifier,url="https://zenodo.org/oai2d",prefix="oai_datacite",as="raw")  
keyword_counts <- record_data_xml %>%
    map(read_xml) %>%
    map(xml_find_all,"//d3:subject") %>%
    map(xml_text) %>%
    reduce(c) %>%
    table() %>%
    tbl_df()
kable(keyword_counts %>% filter(grepl(".*C22.*|^food",`.`)))
{% endhighlight %}



|.                                       |  n|
|:---------------------------------------|--:|
|food additives                          |  1|
|food additives intake model             |  1|
|food composition difference testing     |  1|
|http://id.agrisemantics.org/gacs/C22070 |  2|
|http://id.agrisemantics.org/gacs/C22092 |  1|
|http://id.agrisemantics.org/gacs/C2225  |  3|

I use the 'map' function from the 'purrr' package to apply to every vector in the result (which is first an xml string) a number of transformations:

1.  read_xml() - to convert from string to class xml_document
2.  xml_find_all() - to find all xml nodes given by xpath expression
3.  xml_text() - get the text from the xml node
 
 Then I combine all this via c() and the reduce() function to obtain a single list of all subjects.
 
 The API returns both types of subjects, the generic keywords and the terms referring to a controlled vocabulary.
 
 The table() command produces then a frequency table for them, of which I show here a subset.
 We have in this table entries with an English label, and some with the GACS URI.
 
### Add human readable label to GACS topics


To add a human readable label to each GACS URI, I use the GACS API which allows to query information on each topic.
So I call the API for each URI and make a table where each row contains a list of (URI,label). This gets the converted into a table with bind_rows()

I use again the 'map' function with an anonymous function, which does the call to the GACS API. GACS uses the (Skomsos)[https://github.com/NatLibFi/Skosmos] software, so has an (API)[http://api.finto.fi/doc/] to query the vocabulary.
                                        

{% highlight r %}
gacs <- keyword_counts %>% filter(grepl("*gacs*",.))

gacs_label_en <- map(gacs$`.`,function(uri) {

    r=GET("http://browser.agrisemantics.org/rest/v1/gacs/label",query=list(uri=uri,lang="en"))
    list(uri=uri,label=content(r)$prefLabel)
    
}) %>%
    bind_rows()
kable(gacs_label_en[1:5,])
{% endhighlight %}



|uri                                     |label                        |
|:---------------------------------------|:----------------------------|
|http://id.agrisemantics.org/gacs/C10152 |Bayesian theory              |
|http://id.agrisemantics.org/gacs/C10826 |commodities                  |
|http://id.agrisemantics.org/gacs/C12237 |flavourings                  |
|http://id.agrisemantics.org/gacs/C1263  |screening                    |
|http://id.agrisemantics.org/gacs/C14046 |emerging infectious diseases |



## Distributions of labels in efsa-pilot community

To get the final table, I join the label-GACS pairs with the former table and do some clean-up with the 
functions from tidyr package.

The table is then sorted by frequency and shown on the screen.

As we can see, the most frequent words are 'risk assessment' and 'exposure assessment', which is no surprise as these is the core of EFSA's
scientific work.


{% highlight r %}
table <- left_join(keyword_counts,gacs_label_en,by=c("."="uri")) %>% 
  replace_na(list(label="")) %>%
  unite("label",c(label,`.`),sep=" - ") %>%
  mutate(label = gsub("^ - ","",label)) %>%
  rename(count=n) %>%
  arrange(-count)

write.csv(table,"keywords.csv",row.names = F)
knitr::kable(table %>% slice(1:20))
{% endhighlight %}



|label                                                                      | count|
|:--------------------------------------------------------------------------|-----:|
|risk assessment - http://id.agrisemantics.org/gacs/C1470                   |     8|
|quantitative analysis - http://id.agrisemantics.org/gacs/C603              |     7|
|exposure assessment - http://id.agrisemantics.org/gacs/C29232              |     6|
|population - http://id.agrisemantics.org/gacs/C2955                        |     5|
|prion diseases - http://id.agrisemantics.org/gacs/C18728                   |     4|
|pesticides - http://id.agrisemantics.org/gacs/C284                         |     4|
|Apoidea - http://id.agrisemantics.org/gacs/C1932                           |     3|
|Salmonella - http://id.agrisemantics.org/gacs/C2225                        |     3|
|pesticide residues - http://id.agrisemantics.org/gacs/C3009                |     3|
|linear models - http://id.agrisemantics.org/gacs/C3504                     |     3|
|model validation - http://id.agrisemantics.org/gacs/C4332                  |     3|
|time - http://id.agrisemantics.org/gacs/C4525                              |     3|
|pollinators - http://id.agrisemantics.org/gacs/C5325                       |     3|
|decision support systems - http://id.agrisemantics.org/gacs/C8154          |     3|
|acute risk assesment                                                       |     2|
|chronic risk assesment                                                     |     2|
|Epidemiology                                                               |     2|
|exposure assessment                                                        |     2|
|bovine spongiform encephalopathy - http://id.agrisemantics.org/gacs/C14182 |     2|
|calculation - http://id.agrisemantics.org/gacs/C15337                      |     2|

To monitor regularly this distribution can help in keeping the list of all keywords clean and eventually propose additional subjects to the GACS vocabulary.


# Session info

{% highlight r %}
sessionInfo()
{% endhighlight %}



{% highlight text %}
## R version 3.2.2 (2015-08-14)
## Platform: x86_64-pc-linux-gnu (64-bit)
## Running under: Ubuntu 15.10
## 
## locale:
##  [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
##  [3] LC_TIME=en_US.UTF-8        LC_COLLATE=en_US.UTF-8    
##  [5] LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8   
##  [7] LC_PAPER=en_US.UTF-8       LC_NAME=C                 
##  [9] LC_ADDRESS=C               LC_TELEPHONE=C            
## [11] LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       
## 
## attached base packages:
## [1] stats     graphics  grDevices utils     datasets  base     
## 
## other attached packages:
##  [1] xml2_1.0.0      oai_0.2.1.9100  httr_1.2.1      dplyr_0.5.0    
##  [5] purrr_0.2.2     readr_1.0.0     tidyr_0.6.0     tibble_1.2     
##  [9] ggplot2_2.1.0   tidyverse_1.0.0 knitr_1.14     
## 
## loaded via a namespace (and not attached):
##  [1] Rcpp_0.12.7      servr_0.4        magrittr_1.5    
##  [4] munsell_0.4.3    colorspace_1.2-7 R6_2.2.0        
##  [7] highr_0.6        stringr_1.1.0    plyr_1.8.4      
## [10] tools_3.2.2      grid_3.2.2       gtable_0.2.0    
## [13] DBI_0.5-1        lazyeval_0.2.0   digest_0.6.10   
## [16] assertthat_0.1   formatR_1.4      evaluate_0.10   
## [19] stringi_1.1.2    methods_3.2.2    scales_0.4.0    
## [22] httpuv_1.3.3
{% endhighlight %}


