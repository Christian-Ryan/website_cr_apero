---
title: "3.2 - Further text cleaning"
author: "Christian Ryan"
date: '2021-11-12'
slug: []
categories: []
tags: []
---

<script src="/rmarkdown-libs/twitter-widget/widgets.js"></script>

![](/blog/2021-11-10-further-text-cleaning/featured.png)

## Getting the data

This blog post is a continuation of the previous one (3.1) examining tweets about covid we downloaded using the **rtweet** package.

If you followed along with blog post 3.1 and saved your data after we made changes, you can reload that data now, and skip to the next section. If you didn’t save it, but you have installed the **r4psych** package, to reload the data you can run `library(r4psych)` and `data(london)` (but note that I rename my dataset to `df`). If you skipped the previous post, you will need to run the following lines of code.

``` r
library(devtools)
install_github("Christian-Ryan/r4psych")
```

``` r
library(r4psych)
data(london)
df <- london
rm(london)
```

The data we used last time is a collection of 100 tweets about covid from the Greater London area. We made some changes in that post which included: converting smart quotation marks to straight ones, replaced emojis with word equivalents and expanding all of the contractions. However, I introduced the **proustr** package last time to deal with the smart quotation marks, only to find out later that our main package of interest - **textclean** - also has a function to deal with smart quotes - Doh! 😂 🤷 So, we can create a quick code chuck to make all three replacements, but this time we will use just the **textclean** functions. First we load the **tidyverse** collection and **textclean** packages.

``` r
library(tidyverse)
library(textclean)
```

Now, we can use a `mutate()` function to run each of our **textclean** functions: `replace_curly_quote()`, `replace_emoji()` and `replace_contraction()`.

``` r
df <- df %>% 
  mutate(text = replace_curly_quote(text),
         text = replace_emoji(text), 
         text = replace_contraction(text))
```

## Handle tags @

If we take a look at the tweets 3 to 5, we can see that they each contain tags that begin with an `at` symbol - “@.”

``` r
df %>% 
  slice(3:5) %>% 
  pull(text)
```

    ## [1] "@ohsouthlondon @CPFC Depends whats written in the contract and IF it is related to the pandemic. They vendor would have entered into the contract knowing the risks at the time.. To many blaming covid for poor service. If you take on a contract you deliver or face penalties"           
    ## [2] "@leoniedelt I did a covid test yesterday at home just to make sure I am healthy as I have a busy weekend. I have just got a cold. Our immune systems are not strong due to being locked in maybe that is why there are so many deaths happening now?"                                        
    ## [3] "@Jim936 For me it is been the \"post grouping\" anxiety. In the moment I am fine (alcohol probably helped that at clubs etc) but the next few days are spent convincing myself I have got covid and so I wonder if it is worth it, but it always is at the time, just not so much afterwards"

In the third tweet are the tags ‘(**ohsouthlondon?**)’ and ‘(**CPFC?**).’ In the fourth tweet is ‘(**leoniedelt?**)’ and in the fifth tweet is \`
‘(**Jim936?**).’

The ‘at’ symbol @ is used as a prefix for usernames on Twitter and we may not want to keep these in a text analysis of tweets. We can use the `replace_tags()` function from **textclean** to get rid of them. But this function has some quirky behaviour. Let’s create a short collection of made-up tags to see it in action, before we apply it to our dataset.

``` r
tags <- c("@eisler", "@lock", "@treasure", "@LeGrange", 
          "@Dare", "@Russell", "@ROBIN", "normal text")
replace_tag(tags)
```

    ## [1] ""            ""            ""            "@LeGrange"   "@Dare"      
    ## [6] "@Russell"    "@ROBIN"      "normal text"

So here we see that the first three tags are all dealt with as we might expect, by being deleted from the character vector, leaving empty entries. However, the last five are unaffected. It took me a while to realise that the function only works on tags if they are written entirely in lowercase text. There is little indication of this limitation in the help file, but there is a clue. For the `pattern` argument it shows the default value as `qdapRegex::grab("rm_tag")`. I think this means that **textclean** inherits this function from the **qdapRegex** package, and in the manual for that package, it explains that `rm_tag` default regex pattern is `(?<![@\w])@([a-z0-9_]+)\b`. This appear to be the problem as it only includes `a-z` and if we want to capture all of the tags, we need the capital letters as well `A-Z`.

Many text processing pipelines include a step to turn all capital letters into lowercase, but we have not included one in our process so far. We could do this with a function such as `str_to_lower()` from the **stringr** package, and then run the `replace_tag()` function.

``` r
replace_tag(str_to_lower(tags))
```

    ## [1] ""            ""            ""            ""            ""           
    ## [6] ""            ""            "normal text"

An alternative, if we decided not to change the case of the text, is to adapt the regex pattern that `replace_tag()` uses.

## Escape characters

Before we look at how to do this, we should briefly examine escape characters in R. While researching this part of the blogpost, I came across a paradoxical comment in **The R inferno** about escape characters - “Since backslash doesn’t mean backslash, there needs to be a way to mean backslash” (Burns, 2011). What Burns is indicating is that some characters in R are special and are treated as control characters, that do things other than simply represent a string character. This is easier to understand with the quotation mark or tab key.

If we wanted to refer to a quotation mark in a sentence in R, without the escape character (`\`), the second quotation mark would close the string and cause an error.

``` r
some_words <- "this is a string with a "quote" in it"
```

    ## Error: <text>:1:41: unexpected symbol
    ## 1: some_words <- "this is a string with a "quote
    ##                                             ^

See how the word “quote” is no longer treated as a string, and R returns an error, reporting that it is an “unexpected symbol.” We can use the escape character to let R know that each of the two quotation marks around the word “quote” are to be treated as string characters, and not as instructions to end and restart the string.

``` r
some_other_words <- "this is a string with a \"quote\" in it"
```

If we use a backslash without anything following it, we see that it does not appear as a character in our string.

``` r
st <- "test \ "
st
```

    ## [1] "test  "

We can check the number of characters in a string with the `nchar()` function.

``` r
nchar(st)
```

    ## [1] 6

Here it returns the number 6 - four letters of the word `test` and two whitespace characters either side of the backslash. However, it is not counting the backslash - so this is not treated as a printing character by R. This is indicating that it treats the backslash as a special escape character. For instance, we can use it with the letter `\t` to encode a tab. We can see this in the output using the `cat()` function.

``` r
st <- "test \t test"
cat(st)
```

    ## test      test

I think this is what Burns (2011) meant by the “backslash doesn’t mean backslash!” In our regex, we want to be able to pass the backslash or escape character as a character. To do this we need to `escape` the backslash by using the double backslash.

``` r
st <- "test \\t test"
cat(st)
```

    ## test \t test

So to return to our regex, we can see two instances of `single-backslash` that we need to fix, as the regex contains escape characters before `w` and `b`. We also need to add capital letters with `A-Z`.

``` r
replace_tag(tags, pattern = "(?<![@\\w])@([A-Za-z0-9_]+)\\b")
```

    ## [1] ""            ""            ""            ""            ""           
    ## [6] ""            ""            "normal text"

Finally, the `replace_tag()` function is performing how we want it to.

## Using replace\_tag() with our regex

Returning to our covid twitter dataset, we can use the `replace_tag()` function in a `mutate()`, and pass the new regex as the matching pattern.

``` r
df <- df %>% 
  mutate(text = replace_tag(text, pattern = "(?<![@\\w])@([A-Za-z0-9_]+)\\b"))
```

## URL

It is fairly common in tweets to see website addresses, including the specific URL - these are sometimes links to another tweet. At other times they are simply links to unrelated websites. We can see two URL examples in tweets 11 and 12.

``` r
df %>% 
  slice(11:12) %>% 
  pull(text)
```

    ## [1] "The Latest: India gives 25M vaccine doses on Modi's birthday A health worker administers the vaccine for COVID-19 during a special vaccination drive by the municipal corporation at a bus stand in Ahmedabad, India, Friday, Sept. 17, 2021. (AP Photo/Ajit Solanki)Exile Tibetan Bu... https://t.co/iHdpoiPLqu"
    ## [2] "I spent over N6m to treat COVID-19 <e2><80><93> Actor Pete Edochie's son https://t.co/tKQoOvaZEm #Nigeria #NigeriaNews https://t.co/dCVJXlwIGz"

We can try out the **textclean** function `replace_url()` on a sample string with an URL embedded in it.

``` r
urls <- "https://www.bbc.com/news & some other random text"
```

``` r
replace_url(urls)
```

    ## [1] " & some other random text"

This works well, so we can apply it with a `mutate()` function, just like we did with the tags, to our df.

``` r
df <- df %>% 
  mutate(text = replace_url(text))
```

## Hashtags - \#NHS \#covid

If we take a look at some individual tweets, we can see another kind of tagging that is common on twitter - the hashtag. This allows tweets to be linked to other tweets with the same hashtag. Let’s look at tweet 9 as an example.

``` r
df %>% 
  slice(9) %>% 
  pull(text)
```

    ## [1] "  This info above from UK gov website but cant see on the Dutch website where #NHS #covid Pass is accepted in #netherlands"

We see here a range of hashtags including \#NHS, \#covid and \#netherlands in this tweet. We might wonder whether it is wise to eliminate the hashtag from the text of the tweet, as they may contain information that could be useful in a text analysis. However, there is a good reason to remove them. Our dataframe already contains all of the hashtags neatly organised in the variable `hashtags`!

``` r
df %>% 
  slice(9) %>% 
  pull(hashtags)
```

    ## [[1]]
    ## [1] "NHS"         "covid"       "netherlands"

So we can safely delete them from the text of the tweet, knowing that any future analysis of hashtags can draw on the `hashtag` variable instead.

The **textclean** package has a `replace_hash()` function. To check how the regex works in this function, we will create a string with both lower case and upper-case hashtags.

``` r
hash_tag <- "This contains an #octothorp and an upper-case hashtag #COVID"
```

We can then pass this string to the `replace_hash()` function.

``` r
replace_hash(hash_tag)
```

    ## [1] "This contains an  and an upper-case hashtag "

We can see that this time, the replace function is working properly with both upper and lowercase text. We can apply it to our dataframe.

``` r
df <- df %>% 
  mutate(text = replace_hash(text))
```

## &amp

Another anomalous character string that appears in a few tweets is an HTML encoded ampersand (&amp). We can use the `str_detect()` function to identify which tweets contain it and a `pull()` to isolate just the text of these tweets.

``` r
df %>% 
  filter(str_detect(text, pattern = "&")) %>% 
  pull(text)
```

    ## [1] "To those calling Covid a hoax - it is not, the ITU in my hospital cannot handle the number of patients we have! To those who refuse to wear masks face with medical mask - they reduce infection &amp; protect both you &amp; others, stop being a d*ck, it is not just about you! 1/2 "                             
    ## [2] "STRICTLY IDIOTS refuse to take Covid jabs! Two pro dancers will be unjabbed! Contestants fear to dance with them! QUITE RIGHT. Sack 'em, selfish so &amp; so's.          "                                                                                                                                           
    ## [3] "     No he will not. Miliband was a disaster who only got in because of backing from the Trade Union's. Labour's vote collapsed after 2019 as trust collapsed in Corbyn &amp; Labour as an institution. Starker has clawed a lot of it back but COVID esp has made the whole mess very complex."                     
    ## [4] " Your thread yesterday was a very accessible &amp; comprehensive review of recent studies and data on covid/long covid in children. Personal attacks are just a way of saying 'I can not be bothered to engage with the substance'. Please keep going."                                                              
    ## [5] " it is been a steep learning curve! Previously fit &amp; healthy 39yo then Covid damaged my heart, lungs &amp; eyes. Not fun, especially as I am in constant pain but still need to work &amp; look after my kids, one now has asthma after Covid too. can not wait to see what more infections bring! flushed face "
    ## [6] " And in a way: good! If what goes around, comes around,  deserves nothing less than jail, for a very long time! He has willfully and wrecklessly lost tens of thousands of people's lives with  &amp; cost / lost the  SO much, all because of his fanatic  ideology"

Here we see some examples of its use. You might be wondering about the use of this string, particular as each one is followed by a semi-colon. We can investigate this a bit further by pulling out one tweet that contains it, and accessing the tweet in its original format. Our dataframe contains a link to each original tweet in a variable called `status_url`. I have already identified that the first tweet in our dataframe with the `&` encoding is tweet 17, so we can combine `slice()` with `pull()` to get just the URL as a text string - which we can save as a new variable called `tweet`. If we pass this value to the `tweet_embed()` function from the **tweetrmd** package, it will allow us to display the original tweet.

<blockquote class="twitter-tweet" data-width="550" data-lang="en" data-dnt="true" data-theme="light"><p lang="en" dir="ltr">To those calling Covid a hoax - it isn’t, the ITU in my hospital cannot handle the number of patients we have!<br>To those who refuse to wear masks 😷 - they reduce infection &amp; protect both you &amp; others, stop being a d*ck, it’s not just about you!<br>1/2 <a href="https://t.co/iPzvOa1K9u">pic.twitter.com/iPzvOa1K9u</a></p>&mdash; KayWierba (@KayWierba) <a href="https://twitter.com/KayWierba/status/1439141430795112449?ref_src=twsrc%5Etfw">September 18, 2021</a></blockquote>

It appears that the twitter api converts the original ampersand symbol `&` into the HTML encoded ampersand `&amp;`. We can use a new strategy to replace this with appropriate text. This time, rather than use white-space as the replacement, it makes more sense to convert the `&` symbol back to the English word ‘and.’ We will use a `mutate()` function, and the `mgsub()` function from the **textclean** package. This is an extension of the `sub()` and `gsub()` functions. The `sub()` function performs a text substitution, but only for the first match found. The `gsub()` function is an extension that provides “global” substitution - it replaces every match found. The `mgsub()` is an extension that can take a vector of search terms and a vector of replacements - a bit of overkill for what we are doing, but I wanted to persist with functions from the **textclean** package, so here we are! To demonstate how `mgsub()` works, let’s create a short text variable that has three different way of encoding the word “and”: `&`, `&amp` and `+`. We will create a new vector of those “ands” called `ands`.

``` r
string <- "this is a short text & it contains some nonsense &amp; it goes on + on"
ands <- c("&", "&amp;", "+")
```

Now imaging we always want to replace the various “and” encodings with the English word “and.” This is where `mgsub()` is particularly useful. We pass the vector `ands` as the pattern argument.

``` r
mgsub(string, pattern = ands, replacement = "and")
```

    ## [1] "this is a short text and it contains some nonsense and it goes on and on"

We can now use this same function on our tweets dataframe, but we will hard code the pattern: we don’t require a vector in this case.

``` r
df %>% 
  slice(17) %>% 
  mutate(text = mgsub(text, pattern = "&amp;", replacement = "and")) %>% 
  pull(text)
```

    ## [1] "To those calling Covid a hoax - it is not, the ITU in my hospital cannot handle the number of patients we have! To those who refuse to wear masks face with medical mask - they reduce infection and protect both you and others, stop being a d*ck, it is not just about you! 1/2 "

This works how we wanted, so let’s apply this to the whole dataframe.

``` r
df <- df %>% 
  mutate(text = mgsub(text, pattern = "&amp;", replacement = "and"))
```

## Leftover hexadecimal codes

So we have made good progress on cleaning our tweets. To check what is left to be tackled we can cycle through ten tweets at a time to check for anomalies. Having checked the text from tweets 1 - 10, only number 10 contains any none-text content.

``` r
df %>% 
  slice(10) %>% 
  pull(text)
```

    ## [1] "  From midnight on 22 September 2021, fully vaccinated travellers from the UK no longer have to quarantine on arrival in the Netherlands.The NHS COVID Pass is accepted as evidence of vaccination for entering the Netherlands.For further information,visit the<c2><a0>Dutch government's web"

Here we can see the hexadecimal code &lt;c2&gt;&lt;a0&gt;. This particular one stands for a non-breaking space, which is white-space that prevents a line break from occurring - definitely something we don’t need in a text analysis, particular if we are using a bag-of-words approach. We can use the `mgsub()` function that we employed earlier, to tackle this problem. We will test it on just this tweet, to make sure it works.

``` r
df %>% 
  slice(10) %>% 
  mutate(text = mgsub(text, pattern = "<c2><a0>", 
                            replacement = " ")) %>% 
  pull(text)
```

    ## [1] "  From midnight on 22 September 2021, fully vaccinated travellers from the UK no longer have to quarantine on arrival in the Netherlands.The NHS COVID Pass is accepted as evidence of vaccination for entering the Netherlands.For further information,visit the Dutch government's web"

This works fine, so we can apply it to the entire dataframe, but notice that I added a whitespace character to the `replacement` argument above, as the non-breaking space is still a space, that we wish to replace.

``` r
df <- df %>% 
  mutate(text = mgsub(text, pattern = "<c2><a0>", 
                            replacement = " "))
```

## EN dash hexadecimal code

As we continue to review the tweets, we find that tweet 12 has another hexadecimal code - &lt;e2&gt;&lt;80&gt;&lt;92&gt; which is an `EN dash` - a mark that is longer than a hyphen and is typically used to represent a range, such as a range of pages in a reference section of a journal paper.

``` r
df %>% 
  slice(12) %>% 
  pull(text)
```

    ## [1] "I spent over N6m to treat COVID-19 <e2><80><93> Actor Pete Edochie's son    "

Let’s try running the same `mgsub()` function with this new hexadecimal detritus. Note that the `EN dash` appears to have whitespace either side, so we don’t need to include any in our replacement argument.

``` r
df %>% 
  slice(12) %>% 
  mutate(text = mgsub(text, pattern = "<e2><80><93>", 
                            replacement = "")) %>% 
  pull(text)
```

    ## [1] "I spent over N6m to treat COVID-19  Actor Pete Edochie's son    "

This works as well, so lets run the change on the whole dataset.

``` r
df <- df %>% 
  mutate(text = mgsub(text, pattern = "<e2><80><93>", 
                            replacement = ""))
```

## ‘…’ - Elipsis hexadecimal code

In tweet 26, we find the hexadecimal code for an ellipsis (…).

``` r
df %>% 
  slice(26) %>% 
  pull(text)
```

    ## [1] " Wish I could<e2><80><a6> but Covid!"

Again we can use the mutate function with a `mgsub()` to replace the &lt;e2&gt;&lt;80&gt;&lt;a6&gt; from our text variable. However, even though the ellipsis may not be needed if we use a bag-of-words approach to text analysis, it is still quite meaningful, and gives more context to the sentence. Some punctuation marks are very easy to fix when conducting a tokenisation of the text using `tidytext` tools, and full-stops are included in this. I would be inclined to paste the ellipsis back in here, and deal with them at a later stage in the text analysis pipeline - thereby retaining the meaning in the raw tweet data.

``` r
df %>% 
  slice(26) %>% 
  mutate(text = mgsub(text, pattern = "<e2><80><a6>", 
                            replacement = "...")) %>% 
  pull(text)
```

    ## [1] " Wish I could... but Covid!"

This is what it will look like with our replacement. Let’s apply this to the entire dataframe.

``` r
df <- df %>% 
  mutate(text =  mgsub(text, pattern = "<e2><80><a6>",
                       replacement = "..."))
```

## Summary

In this blogpost, we re-imported our dataset, and updated the replacement of smart quotation marks, emojis and contractions. We removed handle tags and hash tags. We explored the use of escape characters when tweaking regex expressions with the `replace_tag()` function. We replaced URLs, html encoded ‘&’ symbols, and some hexadecimal codes.

In the next post, we will examine what text anomalies remain in the dataset, in preparation for the text analysis.

Finally, if we wanted to put together each of the steps we used in this blogpost into one code chuck, we could write it as follows:

``` r
df <- df %>% 
  mutate(text = replace_tag(text, pattern = "(?<![@\\w])@([A-Za-z0-9_]+)\\b"), # tags
         text = replace_url(text),                                             # urls
         text = replace_hash(text),                                            # hashtags
         text = mgsub(text, pattern = "&amp;", replacement = "and"),           # &amp;
         text = mgsub(text, pattern = "<c2><a0>", replacement = " "),          # non-breaking space
         text = mgsub(text, pattern = "<e2><80><93>", replacement = ""),       # EN dash
         text = mgsub(text, pattern = "<e2><80><a6>", replacement = "..."))    # elipsis
```

References

Burns, P. (2011). The R inferno, https://www.burns-stat.com/pages/Tutor/R\_inferno.pdf
