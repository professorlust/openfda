# Analysing adverse drug events using [R + OpenFDA](https://open.fda.gov/drug/event/)

Using a set of R packages it is easy to access and query the openFDA API endpoint for adverse drug events.

First, let's go ahead and install the required packages.

```r
# install ggplot2
if (is.null(grep('ggplot2', rownames(installed.packages())))) {
  install.packages("ggplot2")
}
# install ggpubr
if (is.null(grep('ggpubr', rownames(installed.packages())))) {
  install.packages("ggpubr")
}
# install ggsci
if (is.null(grep('ggsci', rownames(installed.packages())))) {
  install.packages("ggsci")
}
# install devtools
if (is.null(grep('devtools', rownames(installed.packages())))) {
  install.packages("devtools")
}
# install pheatmap
if (is.null(grep('pheatmap', rownames(installed.packages())))) {
  install.packages("pheatmap")
}
# install openfda
if (is.null(grep('openfda', rownames(installed.packages())))) {
  devtools::install_github("ropenhealth/openfda")
}
```

Once all required packages are installed, we can use he openfda package to interact with the API. Next - to start off with - we retrieve all reported adverse drug events per country they have been reported in.


```r
library(openfda)
countries = fda_query("/drug/event.json") %>%
  fda_count("primarysourcecountry") %>%
  fda_limit(1000) %>%
  fda_exec()
```

```
## Fetching: https://api.fda.gov/drug/event.json?search=&limit=1000&count=primarysourcecountry
```

```r
head(countries)
```

```
##   term   count
## 1   us 3658274
## 2   gb  175425
## 3   jp  146641
## 4   ca  113504
## 5   fr  109638
## 6   de   95709
```

```r
nrow(countries)
```

```
## [1] 221
```

Note that the limit had to be defined explicitly, otherwise only 100 hits would have been returned per default which is one of the main limitations the API. Also, the maximum number of results supported by the API is set to 1000 which is one of the major limitation . However, looking at the results we can see the vast majority of the reports originate from the US. Let's have a look at the top 10 countries and visualize the results in form of a barplot.

```r
library(ggplot2)
library(ggpubr)
library(ggsci)

g<-ggbarplot(data=countries[1:10,],x='term', y='count', fill='term', palette = pal_d3("category10")(10), title = 'Number of adverse drug reports by country' , xlab = 'Country', ylab = 'Number of reports')
g <- g + guides(fill = guide_legend(title='Country'))
g
```

![](unnamed-chunk-3-1.png)<!-- -->

It becomes now more obvious that the vast majority of the reports originate from the US, which is not really surprising. Let's next have a closer look if the same pattern holds up when we retrieve the adverse reactions for one given drug (Aspirin in this example).

```r
aspirin_countries<- fda_query("/drug/event.json") %>%
fda_filter("patient.drug.openfda.generic_name", "aspirin") %>%
fda_count("primarysourcecountry") %>%
fda_limit(1000) %>%
fda_exec()
```

```
## Fetching: https://api.fda.gov/drug/event.json?search=patient.drug.openfda.generic_name:aspirin&limit=1000&count=primarysourcecountry
```

```r
g<-ggbarplot(data=aspirin_countries[1:10,],x='term', y='count', fill='term', palette = pal_d3("category10")(10), title = 'Number of Aspirin-related adverse drug reports by country' , xlab = 'Country', ylab = 'Number of reports')
g <- g + guides(fill = guide_legend(title='Country'))
g
```

![](unnamed-chunk-4-1.png)<!-- -->

Overall, the dominance of reports originating from the US persists even if only one drug is considered. It is interessting to note that the ranking of the other countries slightly changes. Germany is now at third place whilst Japan drops down to 7th place which is most likely due to different market penetration for the drug in question. Next, let's retrieve the adverse reactions reported for the use of aspirin per country and scale them by the total of aspirin related reports per counbtry so the different countries can be compared more easily.


```r
for(i in seq(10)){
  tmp<- fda_query("/drug/event.json") %>%
    fda_filter("primarysourcecountry", as.character(aspirin_countries$term[i])) %>%
    fda_filter("patient.drug.openfda.generic_name", "aspirin") %>%
    fda_count("patient.reaction.reactionmeddrapt.exact") %>%
    fda_limit(1000) %>%
    fda_exec()
    tmp$count <- tmp$count / aspirin_countries$count[i]
  if(i==1){
    aspirin_countries_reactions  <- cbind(data.frame(country=rep(aspirin_countries$term[i],nrow(tmp))), tmp)
  } else {
    tmp  <- cbind(data.frame(country=rep(aspirin_countries$term[i],nrow(tmp))), tmp)
    aspirin_countries_reactions <- rbind(aspirin_countries_reactions, tmp)
  }
}
```

```
## Fetching: https://api.fda.gov/drug/event.json?search=primarysourcecountry:us+AND+patient.drug.openfda.generic_name:aspirin&limit=1000&count=patient.reaction.reactionmeddrapt.exact 
## Fetching: https://api.fda.gov/drug/event.json?search=primarysourcecountry:gb+AND+patient.drug.openfda.generic_name:aspirin&limit=1000&count=patient.reaction.reactionmeddrapt.exact 
## Fetching: https://api.fda.gov/drug/event.json?search=primarysourcecountry:de+AND+patient.drug.openfda.generic_name:aspirin&limit=1000&count=patient.reaction.reactionmeddrapt.exact 
## Fetching: https://api.fda.gov/drug/event.json?search=primarysourcecountry:ca+AND+patient.drug.openfda.generic_name:aspirin&limit=1000&count=patient.reaction.reactionmeddrapt.exact 
## Fetching: https://api.fda.gov/drug/event.json?search=primarysourcecountry:jp+AND+patient.drug.openfda.generic_name:aspirin&limit=1000&count=patient.reaction.reactionmeddrapt.exact 
## Fetching: https://api.fda.gov/drug/event.json?search=primarysourcecountry:au+AND+patient.drug.openfda.generic_name:aspirin&limit=1000&count=patient.reaction.reactionmeddrapt.exact 
## Fetching: https://api.fda.gov/drug/event.json?search=primarysourcecountry:it+AND+patient.drug.openfda.generic_name:aspirin&limit=1000&count=patient.reaction.reactionmeddrapt.exact 
## Fetching: https://api.fda.gov/drug/event.json?search=primarysourcecountry:cn+AND+patient.drug.openfda.generic_name:aspirin&limit=1000&count=patient.reaction.reactionmeddrapt.exact 
## Fetching: https://api.fda.gov/drug/event.json?search=primarysourcecountry:br+AND+patient.drug.openfda.generic_name:aspirin&limit=1000&count=patient.reaction.reactionmeddrapt.exact 
## Fetching: https://api.fda.gov/drug/event.json?search=primarysourcecountry:kr+AND+patient.drug.openfda.generic_name:aspirin&limit=1000&count=patient.reaction.reactionmeddrapt.exact
```

```r
reaction_mx <- matrix(0, nrow = 10, ncol=length(unique(aspirin_countries_reactions$term)))

allReactions <- unique(aspirin_countries_reactions$term)
allCountries <- aspirin_countries$term[1:10]

rownames(reaction_mx) <- allCountries
colnames(reaction_mx) <- allReactions

for(i in seq(allCountries)){
  for(j in seq(allReactions)){
    idx <- which(aspirin_countries_reactions[which(aspirin_countries_reactions$country==allCountries[i]),]$term==allReactions[j])
    if(length(idx)!=0){
      reaction_mx[i,j] <- aspirin_countries_reactions[which(aspirin_countries_reactions$country==allCountries[i])[idx],3]
    }  
  }
}

library(pheatmap)
pheatmap(t(reaction_mx[,which(apply(reaction_mx , 2, min)>0)]), scale = 'none', cellheight = 10, cellwidth=20, fontsize = 6)
```

![](unnamed-chunk-5-1.png)<!-- -->

If one compares adverse reactions that have been commonly reported in all 10 countries i.e. that do not contain misssing values (n=135) it becomes obvious that adverse reactions are reported differently in different countries. In the case of Canada for example the relative percentage of Fatigue or Nausae reported as adverse reactions to Aspirin are with 8% approximately 4 times higher than in Germany with 2%. 


Using the API is straightforward if the goal is to get an initial overview on basic underlying structres of the data and if there is no need for more complicated, nested queries. If one wants to dig deeper, a different technical solution is required to overcome the API's limitation. One approach would be to use Apache Spark to analyse the FDA's adverse drug events which are also publicly hosted in Amazon S3 in the form of compressed JSON files.

Regardless of the technical approach used, there are additional limiations related to the data itself that one should be aware of:

* Events can be logged by non-professionals which might casue bias and will have to corrected for.
*	There is no certainty that the reported event was actually due to the product. 
*	The FDA does not require that a causal relationship between a product and event be proven, and reports do not always contain the detail necessary to evaluate an event. Because of this, there is no way to identify the true number of events.
*	The important takeaway to all this is that the information contained in this data has not been verified to produce cause and effect relationships. 
