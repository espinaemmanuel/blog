---
author: Emmanuel Espina
comments: true
date: 2011-08-23 16:49:32+00:00
layout: post
link: https://emmaespina.wordpress.com/2011/08/23/enhancing-search-results-using-machine-learning/
slug: enhancing-search-results-using-machine-learning
title: Enhancing search results using machine learning
wordpress_id: 77
categories:
- mahout
- Solr
tags:
- machine learning
- naive bayes
---
<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

Today’s blog will be about enhancing search results by the use of machine learning. The principle is simple: if you have a very popular site you can make use of the behavior of your users to improve the search quality of new users.
Well I don’t have a very popular site :( So this post will be about a theoretical idea that still have to be tested. If you want to perform tests about the solution I’m about to give, please contact me (or leave a comment). That is what I am missing to be able to write a [serious](http://seriouscat.com/) paper (or a serious post at least).

# Introduction

To introduce you in the topic let’s think about how the users are used to work with “information retrieval platforms” (I mean, search engines). The user enters your site, sees a little rectangular box with a button that reads “search” besides it, and figures out that he must think about some keywords to describe what he wants, write them in the search box and hit search. Despite we are all very used to this, a deeper analysis of the workings of this procedure leads to the conclusion that it is a quite unintuitive procedure. Before search engines, the action of “mentally extracting keywords” from concepts was not a so common activity.
It is something natural to categorize things, to classify the ideas or concepts, but extracting keywords is a different intellectual activity. While searching, the user must think like the search engine! The user must think “well, this machine will give me documents with the words I am going to enter, so which are the words that have the best chance to give me what I want”
Would’t be great to search by concept, for example, the user wants documents from the World War II about the Normandy Landings. So he enter “Normandy Landings” and the search engine figures out that that guy is looking for documents about the WWII more precisely about the Normandy Landings, not just documents with the words “Normandy” and “Landings”

# The problem

Well readers, the problem that I just described already exists, and its called [Web Query Classification](http://en.wikipedia.org/wiki/Web_query_classification). You can read the full idea there, in the wikipedia.
What I propose here is to use a simple yet powerful machine learning technique (that is [Naive Bayes](http://en.wikipedia.org/wiki/Naive_Bayes_classifier)) to extract the categories from the user entered query to enhance the results quality, based on query logs to train the model.
But what do I mean with categories? Intentionally ambiguous, that term applies to a broad range of problems. In particular I came up with this idea from a client’s problem. This client wanted to improve the results extracting brand names and product types from the query string. So if the person entered “cherry coke” they wanted to show the results of “Coca Cola Cherry”. Having extracted the brand names (if any) from the query, they wanted (in some way) to improve (boost or filter) the results. So for “cheery coke” they would extract “Coca Cola” as the brand, and from among all the Coca Cola products they will search for “cherry” or “coke” (it is likely to get the Coca Cola Cherry with a high ranking in the results)

# The possible solution

The web server logs are a good source to train the Naive Bayes algorithm. This idea is similar to the one found in this paper: Probabilistic Query Expansion Using Query Logs - Hang Cui. They are storing the query that the user entered and the document that the user clicked afterwards.

```
session := <query text> [clicked document]*
```

The difference that I’m proposing here is not to store the clicked document but it’s associated categories (or “ideas”, “concepts”, etc). The query will be composed of queries and the categories of the documents that the user clicked after the search. For example, some entries in the logs could be:

```
“world war” => [World War II]
“world war II” => [World War II]
“Normandy landings” => [World War II]
“Germany 1945” => [World War II]
“Germany 1945” => [World War II]
“germany 1945” => [German Cinema]
```

[World War II] is just a tag name. It wont be tokenized, or anything as it is just a representation of an idea or concept. “Normandy landings” refers to the concept of [World War II], and “Germany 1945” also refers to the same concept.
Note the last entry in the example. In this case the user clicked an article about Wim Wenders, a german film director that was born in 1945. This is an example of a non representative result. If you say “Germany 1945”, most people will think about the war. However, somebody looking for a german film director that was born in 1945 whose name he doesn’t recall, could enter “Germany 1945” to get to the page of Wim Wenders. Looking for “germany 1945 movies” could improve his results to find Wenders. Our system should return two “ideas” from this text [World War II] and [German Cinema], and the results should contain mainly German propaganda films from that period, because that is what a human would most likely be looking for with that text.
To summarize, we are trying to get the categories with the higher chance.
How did we created this log? Easy, each time the user clicked on a document we check if he had performed a search. If he has, then we extract the categories from the document and store a log entry in the previously described format. But a question that will naturally arise is how to extract the categories from a document. That is beyond the scope of this post, and I assume that somebody has read all the documents that we have and have manually assigned categories to them. The category (or idea, or brand, or concept) is just another field in our documents database, available from the very beginning of our experiment.

## Naive Bayes
To get the categories with the greater chance, we are going to have to deal with probabilities. I’ll give you a super fast description of the Naive Bayes algorithm. I used it in previous [posts](http://emmaespina.wordpress.com/2011/01/31/20/) but now I’ll give you a brief introduction to what it does.

Suppose you have a text and you want to extract which category it belongs to. Since you don’t know the “true” category of the text you would like to get the category with the greater chance from among all the possible categories (or the three most probable categories, for instance). This, written in math language, would be the following formula

$$
max P(\text{<cat>} \mid \text{germany 1945})
$$

That is, given that the user entered “germany 1945”, we get the probabilities of each of the categories of being associated with that text. We take the category that gave the higher probability and return that category.

The problem is solved better by flipping the condition by means of the Bayes theorem

$$
P(\text{<cat>} \mid \text{germany 1945}) = \frac{P( \text{germany 1945} \mid \text{<cat>}) \times P(\text{<cat>})}{P(\text{germany 1945})}
$$

To maximize this the parameter that we are allowed to modify is the category (the text is given and we can not modify that). So we can omit the \\( P(\text{germany 1945})\\) to simplify the problem

$$
max P( \text{germany 1945} | \text{<cat>}) \times P(\text{<cat>})
$$

There are two parts here. Some categories are more probable than others. For example think about the categories \<Quantum Electrodynamics\> and \<Sex\> and you’ll immediately recognize a tendency in the popularity of those terms. Those probabilities can be also estimated through the log.

For the first part we need to consider the individual terms. Not “germany 1945” but “germany” and “1945”. To calculate the probability of \\(P( \text{germany} \cap \text{1945} \mid \text{<cat>})\\) we need to make the “naive assumption” of Naive Bayes: all the terms in the query string are conditionally independent. That means that “germany” is completely independent from “1945”. We know that it is not true, but with this assumption the experience has proven to give very good results. From this assumption we can change the formula to the following:

$$
P( \text{germany} | \text{<cat>}) \times P( 1945 | \text{<cat>}) \times P(\text{<cat>})
$$

I don’t extend much further, but there are some issues if a word have never appeared associated with a category. For example searching “World War II Bananas” the probability of \\( P( \text{bananas} \mid \text{[WWII]} ) \\) would be 0 and the entire calculation will be 0. But we know that the entered text, despite being odd, is related to [WWII]. A solution to that is adding 1 to each probability. The final calculation that we must perform is:

$$
[P(\text{germany} \mid \text{<cat>}) + 1] \times [P(1945 \mid \text{<cat>}) + 1] \times [P(\text{<cat>}) + 1]
$$

This is called Laplace smoothing or [additive smoothing](http://en.wikipedia.org/wiki/Additive_smoothing)


# Implementation

We are not going to implement all that! We are going to use Mahout (again). We can recycle some of the code from the previous post about classifying mail for this.

We need some script to pre-process the query log to take it to the format that the Naive Bayes algorithm accepts.

That is to get something like:

```
World_War_II normandy landings
World_War_II germany 1945
World_War_II germany 1945
German_Cinema germany 1945
```

This is very similar to the text input from the previous post (the category must be expressed as a single word). After training the model we can use exactly the same program to create an online classifier of queries.

Then and assuming that we are using Solr as the search engine we can add boost queries to the regular query. This process would be:

The user enters “Germany 1945 movies”. The system queries our “online query classifier” and gets two possible categories: [WWII] and [German films]. Now we submit a query to Solr in the following fashion:

```
http://solrserver/solr/dismax?q=Germany+1945+movies&bq=category:WWII&bq=category:german_films
```

The category classifier can give us a score (extracted from the Naive Bayes calculation). That, with some application logic, can be used to boost the categories. For example if we got [WWII] with a high ranking we can give a higher boost to the query:

```
q=Germany+1945+movies&bq=category:WWII^3
```

#  Conclusions

There is little to say here since I don’t have a data set to test this blackboard experiment. So I can not say much more than that it is a very elegant idea. However, and as I previously said, this is not a very original idea. It has been studied before and a search in Internet will give you plenty of results. The interesting thing here is that all that we need to apply this in our homes is already available through open source software. Both Solr and Mahout are open source project. With them you can build a learning search engine in a couple of minutes.

Returning to the dataset, I would be glad that somebody shares some query logs. This things work better with a lot of queries and the best example I could find is [http://www.sigkdd.org/kdd2005/kddcup.html ](http://www.sigkdd.org/kdd2005/kddcup.html)that is a Data mining competition where they have 800 classified queries. This is not enough for our purposes.


