---
title: "2.1 - Pulling text data from the internet"
author: "Christian Ryan"
date: '2019-11-30'
output:
  html_document:
    df_print: paged
slug: pulling-text-data-from-the-internet
---



I have been working on the area of alexithymia for the last couple of years, a sub-clinical condition in which people find it difficult to identify and describe their emotions. I am currently analysing a dataset containing transcripts of interviews with people with and without alexithymia and I wanted to try out some R tools for text analysis. However, to do a blog post I needed some public data, and while mulling over which data I might use, I stumbled upon a line in **"You are a thing and I love you"** - the wonderful new book on AI by Janelle Shane. 


![](/blog/2019-11-30-pulling/2019-11-30-pulling-text-data-from-the-internet_files/you_look.png)




https://www.amazon.com/You-Look-Like-Thing-Love/dp/0316525243

She mentions training an AI on a dream dataset available at http://www.dreambank.net The website has section called "DreamBank" that allows you to search or take random samples of dreams recorded from a variety of sources. Under the Random Sample link, at:  http://www.dreambank.net/random_sample.cgi one can select a dream source. 

![](/blog/2019-11-30-pulling/2019-11-30-pulling-text-data-from-the-internet_files/st_ursula.png)

We will need a few packages for this process - **rvest** is useful for pulling data from online sources. The two text packages **stringr** and **stringi** offer a range of tools for managing text data. The **tidyverse** will simplify the management of the dataset and **knitr** is useful for managing the display of text in Rmarkdown documents.

```r
library(rvest)
library(stringr)
library(stringi)
library(tidyverse)
library(knitr)
```



Let's start by taking a look at the dreams of college women from the 1940's. We set an address for the url, then pass this as an argument to the read_html() function. 

```r
url <- "http://www.dreambank.net/random_sample.cgi?series=hall_female&min=100&max=300&n=100"
page <- read_html(url)
```

I followed the guidance in Kwartler (**'Text Mining in Practice with R'**, 2017) and checked the field with the dream text on the webpage in Chrome using the SelectorGadget plugin. This revealed that these text fields were labelled as "span". So we can include this as the type of node to select in the html_node() function from **rvest**. This allows us to pull the html text from just these fields and store them in a new variable called posts, then I will convert this html_text to raw text and store it in a variable called dream. I suppose I could have wrapped the html_nodes call within the html_text function and skipped creating an intermediate variable (posts), but I think it makes the code more readable this way. 


```r
posts <- html_nodes(page, 'span')
dream <- html_text(posts)
```

We can convert this to a dataframe - we will use the tidyverse version, a tibble, as this will avoid problems of the dreams being converted to factors. For more on why this can be problematic, read **"stringsAsFactors: An unauthorized biography"** by Roger Peng at this site:

https://simplystatistics.org/2015/07/24/stringsasfactors-an-unauthorized-biography/


```r
df <- tibble(dream = dream)
```

As a side-note, we could have done each of these steps with a more tidyverse syntax, by using the pipe, though this may have meant that each of the substeps was less transparent. We could have taken the pages object that contains
our raw data and piped it through the various functions to extract just the dreams, then converted it into a tibble. As we haven’t declared a name for the one variable in the tibble, we need to use the rename function to assign the name ‘dream’ to the column. 

```r
df <- page %>% 
  html_nodes('span') %>% 
  html_text() %>% 
  tibble() %>% 
  rename('dream' = '.')
```


Let's take a quick peak at the data. We can create a quick function to truncate the display of the dreams to 60 characters. We will call this function custom_view(). We will restrict the view to just the first 5 rows as well, using indexing.

```r
custom_view <- function(x) data.frame(lapply(x, substr, 1, 60))
custom_view(df[1:5,])
```

```
##                                                           dream
## 1 \n#0001 (Code 001, Age 24, 11/??/47)I dreamed that I was at a
## 2 \n#0008 (Code 001, Age 24, 11/??/47)I dreamed that I went to 
## 3 \n#0025 (Code 007, Age 20, 03/20/48)As I first remember the d
## 4 \n#0028 (Code 007, Age 20, 04/09/48)I was at a factory workin
## 5 \n#0036 (Code 008, Age 22, 02/25/48)I was in a house. It was
```

Currently, our dataset has just one column and we will need to fix this. Let's use substr to pull out the dream number that occurs at the beginning of the text field. The substr() function takes three arguments, the vector, character start and character stop. 

For example, this is what we get if we pull out the three numeric identifier characters (start at 4 and stop at 6). 

```r
substr(df$dream, 4, 6)
```

```
##   [1] "001" "008" "025" "028" "036" "038" "049" "050" "051" "055" "059" "065"
##  [13] "074" "081" "084" "087" "098" "101" "117" "123" "137" "156" "163" "164"
##  [25] "165" "186" "190" "221" "223" "225" "226" "230" "232" "251" "261" "271"
##  [37] "278" "290" "292" "296" "301" "302" "304" "309" "310" "313" "335" "351"
##  [49] "352" "353" "356" "360" "361" "367" "368" "371" "373" "383" "384" "396"
##  [61] "402" "404" "405" "406" "408" "415" "435" "440" "460" "468" "475" "483"
##  [73] "491" "499" "503" "506" "528" "530" "537" "540" "543" "547" "550" "556"
##  [85] "563" "582" "586" "606" "609" "616" "620" "629" "641" "652" "659" "660"
##  [97] "664" "666" "667" "681"
```

This worked fine, so let's create a new variable called code to store this data in our dataframe. This will be our id code for each dream.

```r
df$code <- substr(df$dream, 4, 6)
```

The column with the code should come first, so we will swap the order of columns with a simple index call - concatenating the order of variables, passed as the second argument. 

```r
df <- df[,c(2,1)]
```

After examining the dataframe, we can see that the pattern for ages is given by the word 'Age' with a capital 'A', followed by a space, then the actual age as two digits, like this: "Age 24". We can create a regex pattern to match this and use the stringr package to extract this string and store it in a vector called age. 

```r
age <- str_extract(df$dream, "[A][g][e][ ][0-9]{2}")
```

However, if we want to manipulate the ages as integers, we need to extract just the number and coerce it from a character vector into a numeric vector. We can do this with another regex, which just pulls out the two digits. And let's convert it into a numeric and paste the data back into the dataframe, and move it to the second column.

```r
age_refined <- str_extract(age, "([0-9]{2})")
df$age <- as.numeric(age_refined)
df <- df[,c(1,3,2)]
```

Now we want to tidy up the dream variable. At the moment we have a bunch of characters before the dream itself starts. We can experiment with the str_locate function and a regex to see if we can identify the pattern for where the dream begins. Let's try the closing brace which seems to come after the date of the dream.

```r
head(str_locate(df$dream, "[)]"))
```

```
##      start end
## [1,]    35  35
## [2,]    35  35
## [3,]    35  35
## [4,]    35  35
## [5,]    35  35
## [6,]    35  35
```

This indicates that a closing brace always occurs at the 35th character in the dream text field. We can use the Base R function substr() which takes a vector, a start and an end point. We know the start (character 36), which is the first character after the closing brace of the date, but we don't know the end, as all the dreams are different lenghts. But we can use the handy nchar() function which
counts the number of characters for us, so we treat this as a flexible endpoint. As this seems to work nicely, let's overwrite our dream variable with this new version

```r
df$dream <- substr(df$dream, 36, nchar(df$dream))
```

A quick look at the df using our custom_view() function indictes this is shaping up nicely. 

```r
custom_view(df)[1:5,]
```

```
##   code age                                                        dream
## 1  001  24 I dreamed that I was at a public affair but I don't know whi
## 2  008  24 I dreamed that I went to take an examination and I was late 
## 3  025  20 As I first remember the dream I was upstairs in a room with 
## 4  028  20 I was at a factory working. I saw a college girl-friend of m
## 5  036  22 I was in a house. It was a beautiful large home with expensi
```

But what about the end of each dream? Let's examine the first dream in detail. 

```
##  [1] "I dreamed that I was at a public affair but I don't know which affair"
##  [2] "it was although it was outdoors. There were many people around us and"
##  [3] "they were of all ages. I was at this affair with B. He is about"      
##  [4] "twenty-six years old and he is the boy-friend of one of the girls"    
##  [5] "that lives in the dormitory that I do. Whenever I felt the urge to"   
##  [6] "get away from my escort or from the people at the affair, I would"    
##  [7] "start to fly. (like superman). While up in the air I felt very uneasy"
##  [8] "and worried about how I would get back down without hurting myself. I"
##  [9] "left my escort about three times in this way. I do not remember why I"
## [10] "felt that I had to get away. Interpretation I do not know why I would"
## [11] "dream of B. I do not know him very well and I do not feel very"       
## [12] "friendly toward him when I do see him. I believe that I associated"   
## [13] "him with my studies and felt that I had to get away for a short"      
## [14] "while. When I had this dream, I hadn't been home for about eight"     
## [15] "weeks and was looking forward to going home. I felt that I wanted a"  
## [16] "short vacation from my studies and this dream was an escape mechanism"
## [17] "in the form of a fantasy to get away from my classes for a short"     
## [18] "while. Answers to questions 2. Frustrated. I felt that I had to get"  
## [19] "away.3. actual participant4. unpleasant5. Vague, but it was"          
## [20] "outdoors.6. No.7. No. (268 words)"
```

We can see that each dream includes an interpretation and I only want to analyse the dream narrative itself, not the person's reflections on the meaning of the dream. We can use the word 'Interpretation' to identify the end point of the dream narrative. We can just pull out the first 6 values by wrapping this in the head function. 

```r
head(str_locate(df$dream, "Interpretation"))
```

```
##      start end
## [1,]   643 656
## [2,]   722 735
## [3,]   642 655
## [4,]   306 319
## [5,]   415 428
## [6,]   327 340
```

We still need to do a bit of work - the str_locate() returns two values and we only want the first one. Secondly, when we trim the text, we want to start two characters to the left as we don't want the first letter of the word  "Interpretation", or the whitespace just before it. We can store the location in a new vector called loc - then we can take out the start point only, with the index [,1]. On the third line we will crop the text to start at 0 and end at 2 characters to the left (-2) of the start point. We reassign it to the same variable in our dataframe - df$dream.

```r
loc <- str_locate(df$dream, "Interpretation")
start <- loc[,1] # take out start point [,1] as a vector called start
df$dream <- substr(df$dream, 0, start-2) 
```

Finally, let's check that the changes worked by examining the final 70 characters of the first dream.

```
## [1] "is way. I do not remember why I felt that I had to get away."
```
This is just what we wanted - this is line 9 and 10 of the full dream we examined up above - finishing just before the interpretation starts.

In the next post, I will pull in the dreams from three other samples and start to look at the sentiment analysis of the dream content.

