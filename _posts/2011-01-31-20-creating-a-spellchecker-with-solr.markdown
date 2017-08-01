---
author: emmaespina
comments: true
date: 2011-01-31 14:40:15+00:00
layout: post
link: https://emmaespina.wordpress.com/2011/01/31/20/
slug: '20'
title: Creating a spellchecker with Solr
wordpress_id: 20
categories:
- Solr
tags:
- Information retieval
- spellcheckers
---

# Introduction


In a previous [post](http://emmaespina.wordpress.com/2011/01/18/solr-spellchecker-internals-now-with-tests/) I talked about how the Solr Spellchecker works and then I showed you some test results of its performance. Now we are going to see another aproach to spellchecking.

This method, as many others, use a two step procedure. A rather fast “candidate word” selection, and then a scoring of those words. We are going to select different methods from the ones that Solr uses and test its performance. Our main objective will be effectiveness in the correction, and in a second term, velocity in the results. We can tolerate a slightly slower performance considering that we are gaining in correctness of the results.

Our strategy will be to use a special Lucene index, and query it using fuzzy queries to get a candidate list. Then we are going to rank the candidates with a Python script (that can easily be transformed in a Solr spell checker subclass if we get better results).


# Candidate selection


Fuzzy queries have historically been considered a slow performance query in relation with others but , as they have been optimized in the 1.4 version, they are a good choice for the first part of our algorithm. So, the idea will be very simple: we are going to construct a Lucene index where every document will be a dictionary word. When we have to correct a misspelled word we are going to do a simple fuzzy query of that word and get a list of results. The results will be words similar to the one we provided (ie with a small edit distance). I found that with approximately 70 candidates we can get excellent results.

With fuzzy queries we are covering all the typos because, as I said in the previous post, most of the typos are of edit distance 1 with respect to the correct word. But although this is the most common error people make while typing, there are other kinds of errors.

We can find three types of misspellings [Kukich]:



	
  1. Typographic errors

	
  2. Cognitive errors

	
  3. Phonetic errors


Typographic errors are the typos, when people knows the correct spelling but makes a motor coordination slip when typing. The cognitive errors are those caused by a lack of knowledge of the person. Finally, phonetic errors are a special case of cognitive errors that are words that sound correctly but are orthographically incorrect. We already covered typographic errors with the fuzzy query, but we can also do something for the phonetic errors. Solr has a Phonetic Filter in its analysis package that, among others, has the double methaphone algorithm. In the same way we perform fuzzy query to find similar words, we can index the methaphone equivalent of the word and perform fuzzy query on it. We must manually obtain the methaphone equivalent of the word (because the Lucene query parser don't analyze fuzzy queries) and construct a fuzzy query with that word.

In few words, for the candidate selection we construct an index with the following solr schema:

```html
<fieldType name="spellcheck_text" class="solr.TextField" positionIncrementGap="100" autoGeneratePhraseQueries="true">
      <analyzer type="index">
        <tokenizer class="solr.KeywordTokenizerFactory"/>
        <filter class="solr.LowerCaseFilterFactory"/>
        <filter class="solr.PhoneticFilterFactory" encoder="DoubleMetaphone" maxCodeLength="20" inject="false"/>
     </analyzer>
    </fieldType>

   <field name="original_word" type="string" indexed="true" stored="true" multiValued="false"/>
   <field name="analyzed_word" type="spellcheck_text" indexed="true" stored="true" multiValued="false"/>
   <field name="freq" type="tfloat" stored="true" multiValued="false"/>
```

As you can see the analyzed_word field contains the “soundslike” of the word. The freq field will be used in the next phase of the algorithm. It is simply the frequency of the term in the language. How can we estimate the frequency of a word in a language? Counting the frequency of the word in a big text corpus. In this case the source of the terms is the wikipedia and we are using the TermComponents of Solr to count how many times each term appears in the wikipedia.

But the Wikipedia is written by common people that make errors! How can we trust in this as a “correct dictionary”? We make use of the “colective knowledge” of the people that writes the Wikipedia. This dictionary of terms extracted from the Wikipedia has a lot of terms! Over 1.800.00, and most of them aren't even words. It is likely that words with a high frequency are correctly spelled in the Wikipedia. This approach of building a dictionary from a big corpus of words and considering correct the most frequent ones isn't new. In [Cucerzan] they use the same concept but using query logs to build the dictionary. It apears that Google's “Did you mean” use a similar [concept](http://www.youtube.com/watch?v=syKY8CrHkck#t=22m03s).

We can add little optimizations here. I have found that we can remove some words and get good results. For example, I removed words with frequency 1, and words that begin with numbers. We can continue removing words based on other criteria, but we'll leave this like that.

So the procedure for building the index is simple, we extract all the terms from the wikipedia index via the TermsComponent of Solr along with frequencies, and then create an index in Solr, using SolrJ.


# Candidate ranking


Now the ranking of the candidates. For the second phase of the algorithm we are going to make use of information theory, in particular, the [noisy channel model](http://en.wikipedia.org/wiki/Noisy_channel_model). The noisy channel applied to this case assumes that the human knows the correct spelling of a word but some noise in the channel introduces the error and as the result we get another word, misspelled. We intuitively know that it is very unlikely that we get 'sarasa' when trying to type 'house' so the noisy channel model introduces some formality to finding how probable an error was.
For example, we have misspelled 'houze' and we want to know which one is the most likely word that we wanted to type. To accomplish that we have a big dictionary of possible words, but not all of them are equally probable. We want to obtain the word with the highest probability of having been intended to be typed. In mathematics that is called conditional probability; given that we typed 'houze' how high is the probability of each of the correct words to be the word that we intended. The notation of conditional probability is: P('house'|'houze') that stands for the probability of 'house' given 'houze'

This problem can be seen from two perspectives: we may think that the most common words are more probable, for example 'house' is more probable than 'hose' because the former is a more common word. In the other hand, we also intuitively think that 'house' is more probable than 'photosinthesis' because of the big difference in both words. Both of these aspects, are formally deduced by [Bayes theorem](http://en.wikipedia.org/wiki/Bayes'_theorem):


$latex P(house|houze) = \frac{P(houze|house) P(house)}{P(houze)}&s=2 $


We have to maximize this probability and to do that we only have one parameter: the correct candidate word ('house' in the case shown).

For that reason the probability of the misspelled word will be constant and we are not interested in it. The formula reduces to


$latex Max(P(house|houze)) = Max(P(houze|house) P(house))&s=1 $




And to add more structure to this, scientists have given named to these two factors. The P('houze'|'house') factor is the Error model (or Channel Model) and relates with how probable is that the channel introduces this particular misspell when trying to write the second word. The second term P('house') is called the Language model and gives us an idea of how common a word is in a language.


Up to this point, I only introduced the mathematical aspects of the model. Now we have to come up with a concrete model of this two probabilities. For the Language model we can use the frequency of the term in the text corpus. I have found empirically that it works much better to use the logarithm of the frequency rather than the frequency alone. Maybe this is because we want to reduce the weight of the very frequent terms more than the less frequent ones, and the logarithm does just that.

There is not only one way to construct a Channel model. Many different ideas have been proposed. We are going to use a simple one based in the Damerau-Levenshtein distance. But also I found that the fuzzy query of the first phase does a good job in finding the candidates. It gives the correct word in the first place in more than half of the test cases with some datasets. So the Channel model will be a combination of the Damerau-Levenshtein distance and the score that Lucene created for the terms of the fuzzy query.

The ranking formula will be:


$latex Score = \frac{Levenshtein}{log(freq) Fuzzy}&s=2 $


I programmed a small script (python) that does all that was previously said:

```python
from urllib import urlopen
import doubleMethaphone
import levenshtain
import json

server = "http://benchmarks:8983/solr/testSpellMeta/"

def spellWord(word, candidateNum = 70):
    #fuzzy + soundlike
    metaphone = doubleMethaphone.dm(word)
    query = "original_word:%s~ OR analyzed_word:%s~" % (word, metaphone[0])

    if metaphone[1] != None:
        query = query + " OR analyzed_word:%s~" % metaphone[1]

    doc = urlopen(server + "select?rows=%d&wt=json&fl=*,score&omitHeader=true&q=%s" % (candidateNum, query)).read( )
    response = json.loads(doc)
    suggestions = response['response']['docs']

    if len(suggestions) > 0:
        #score
        scores = [(sug['original_word'], scoreWord(sug, word)) for sug in suggestions]
        scores.sort(key=lambda candidate: candidate[1])
        return scores
    else:
        return []

def scoreWord(suggestion, misspelled):
    distance = float(levenshtain.dameraulevenshtein(suggestion['original_word'], misspelled))
    if distance == 0:
        distance = 1000
    fuzzy = suggestion['score']
    logFreq = suggestion['freq']

    return distance/(fuzzy*logFreq)
```

From the previous listing I have to make some remarks. In line 2 and 3 we use third party libraries for [Levenshtein distance](http://mwh.geek.nz/2009/04/26/python-damerau-levenshtein-distance/) and [metaphone](http://www.atomodo.com/code/double-metaphone) algorithms. In line 8 we are collecting a list of 70 candidates. This particular number was found empirically. With higher candidates the algorithm is slower and with fewer is less effective. We are also excluding the misspelled word from the candidates list in line 30. As we used the wikipedia as our source it is common that the misspelled word is found in the dictionary. So if the Leveshtain distance is 0 (same word) we add 1000 to its distance.


# Tests


I ran some tests with this algorithm. The first one will be using the dataset that [Peter Norvig](http://norvig.com/spell-correct.html) used in his article. I found the correct suggestion of the word in the first position approximately 80% of the times!!! That's is a really good result. Norvig with the same dataset (but with a different algoritm and training set) got 67%

Now let's repeat some of the test of the previous post to see the improvement. In the following table I show you the results.
<table border="1" >
<tbody >
<tr >

<td width="144" align="LEFT" height="17" >**Test set**
</td>

<td width="86" align="LEFT" >**% Solr**
</td>

<td width="86" align="LEFT" >**% new**
</td>

<td width="86" align="LEFT" >** Solr time [seconds]**
</td>

<td width="108" align="LEFT" >**New time [seconds]**
</td>

<td width="86" align="LEFT" >**Improvement**
</td>

<td width="99" align="LEFT" >**Time loss**
</td>
</tr>
<tr >

<td align="LEFT" height="17" >

    
    <em><span style="color:#00aa00;font-family:monospace;">FAWTHROP1DAT.643</span></em>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">45,61%</span>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">81,91%</span>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">31,50</span>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">74,19</span>



</td>

<td align="RIGHT" >

    
    79,58%



</td>

<td align="RIGHT" >

    
    135,55%



</td>
</tr>
<tr >

<td align="LEFT" height="17" >

    
    <em><span style="color:#00aa00;font-family:monospace;">batch0.tab</span></em>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">28,70%</span>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">56,34%</span>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">21,95</span>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">47,05</span>



</td>

<td align="RIGHT" >

    
    96,30%



</td>

<td align="RIGHT" >

    
    114,34%



</td>
</tr>
<tr >

<td align="LEFT" height="17" >

    
    <em><span style="color:#00aa00;font-family:monospace;">SHEFFIELDDAT.643</span></em>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">60,42%</span>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">86,24%</span>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">19,29</span>



</td>

<td align="RIGHT" >

    
    <span style="font-family:monospace;">35,12</span>



</td>

<td align="RIGHT" >

    
    42,75%



</td>

<td align="RIGHT" >

    
    82,06%



</td>
</tr>
</tbody>
</table>
We can see that we get very good improvements in effectiveness of the correction but it takes about twice the time.


# Future work


How can we improve this spellchecker. Well, studying the candidates list it can be found that the correct word is generally (95% of the times) contained in it. So all our efforts should be aimed to improve the scoring algorithm.

We have many ways of improving the channel model; several papers show that calculating more sophisticated distances weighting the different letter transformations according to language statistics can give us a better measure. For example we know that writing 'houpe' y less probable than writing 'houze'.

For the language model, great improvements can be obtained by adding more context to the word. For example if we misspelled 'nouse' it is very difficult to tell that the correct word is 'house' or 'mouse'. But if we add more words “paint my nouse” it is evident that the word that we were looking for was 'house' (unless you have strange habits involving rodents). These are also called ngrams (but of words in this case, instead of letters). Google has offered a big collection of ngrams that are available to download, with their frequencies.

Lastly but not least, the performance can be improved by programming the script in java. Part of the algorithm was in python.

Bye!

As an update for all of you interested, Robert Muir [told me](http://search-lucene.com/m/n0xN61iTIry/My+spellchecker+experiment&subj=My+spellchecker+experiment) in the Solr User list that there is a new spellchecker, DirectSpellChecker, that was in the trunk then and now should be part of Solr 3.1. It uses a similar technique to the one i presented in this entry without the performance loses.


### References


_[Kukich] Karen Kukich - Techniques for automatically correcting words in text - ACM Computing Surveys - Volume 24 Issue 4, Dec. 1992_

_[Cucerzan] S. Cucerzan and E. Brill Spelling correction as an iterative process that exploits the collective knowledge of web users. July 2004_

_[Peter Norvig - How to Write a Spelling Corrector](http://norvig.com/spell-correct.html)_
