---
author: emmaespina
comments: true
date: 2011-01-18 13:59:40+00:00
layout: post
link: https://emmaespina.wordpress.com/2011/01/18/solr-spellchecker-internals-now-with-tests/
slug: solr-spellchecker-internals-now-with-tests
title: Solr Spellchecker internals (now with tests!)
wordpress_id: 8
categories:
- Solr
tags:
- Information retieval
- spellcheckers
---

Let’s talk about spellcheckers. A spellchecker, as you may know, is that device that tells you whether you misspelled or not a word, and makes you some suggestions. One of the first spellcheckers that i remember seeing is the MS Word spellchecker. One could say that MS Word defined the “standard interface” of word processors spellcheckers: you misspelled a word, it was underlined with a zig zag style line in red, and if you right clicked on it, a list of suggested words appeared. I have seen this interface in many different software, for example, google docs.




Another more modern example is the famous “Did you mean” from Google. You type some words like “britny spears” and google would suggest “Britney Spears”. It appears that a lot of people have issues spelling [Britney Spears](http://www.google.com/jobs/britney.html). But Google is different. As usual, they use artificial intelligence algorithms to suggest misspelled words. Google algorithms are the closest you'll get to magic in computer engineering.
But today I’m going to talk about Solr SpellChecker. In contrast with from google, Solr spellcheker isn't much more than a pattern similarity algorithm. You give it a word and it will find similar words. But what is interpreted as “similar” by Solr? The words are interpreted just as an array of characters, so, two words are similar if they have many coincidences in their character sequences. That may sound obvious, but in natural languages the bytes (letters) have little meaning. It is the entire word that has a meaning. So, Solr algorithms won't even know that you are giving them words. Those byte sequences could be sequences of numbers, or sequences of colors. Solr will find the sequences of numbers that have small differences with the input, or the sequences of colors, etc. By the way, this is not the approach that Google follows. Google knows the frequent words, the frequent misspelled words, and the frequent way humans make mistakes. It is my intention to talk about these interesting topics in a next post, but now let's study how solr spellchecker works in detail, and then make some tests.




Solr spellchecker follows the same strategy as many other spellcheckers. It has a dictionary of correct spelled terms (correct by definition, if there is a misspelled word in the dictionary that will pass as a correct word). When somebody asks for suggestions to a word, Solr Spellchecker first obtains a list of candidate words and then ranks those candidates according to some criteria. The first step is accomplished by ngrams. A ngram is a substring in a string. For example if you have the word 'house', some ngrams would be 'hou', 'ous', 'se' (there are many other ngrams of differents lenghts, i’ve shown you only three of them). Two similar words will have many matching ngrams: 'mouse' also has 'ous' and 'se' but not 'hou'. What Solr does is create a Lucene index of the words in the dictionary and filter them with ngram filters. So when you ask for suggestions for “house”, Solr searches 'ho' OR 'ou' OR 'us' OR 'se' OR 'hou' OR 'ous' OR 'use' OR 'hous' OR 'ouse' and, because Solr ranks boolean querys by the document with more coincidences, what you'll get is a list of some similar words from our dictionary.




How does Solr rank the words afterwards? There is something called edit distance that tells us how many operations we have to perform in order to transform a word into another. By operations we understand insertions, deletions, or modifications of single characters. There are many algorithms to find the edit distance, one is Levenshtein (that is the default algorithm used in Solr). These algorithms are computationally complex, and that's the reason why Solr doesn’t use them as the first choice in selecting the suggestions from among all the words in the dictionary. The dictionary is reduced and then this “difficult” ranking process is performed.
Perhaps, now you understand what I meant by “solr spellchecker only find similar byte arrays”. You never introduce information about our natural language into the algorithm and the only thing you provide is “a set of byte secuences” (ie a dictionary)




So far, so well. Does this approach work? Yes. Could it work better? Of course! And we have a lot of things that we can do to improve the algorithm. But first, let’s try to make this look scientific (if you remember, that was the idea of internet in the first place...) We need tests to see where we are standing. Something that I find boring is moving from the theoretical side to the experimental side. But that is a must in this that we call research. So, next I present a series of different tests that I performed on a Solr instance (that we have for experimental purposes) of the wikipedia (I recommend reading this [post](http://juanggrande.wordpress.com/2010/12/20/solr-index-size-analysis/) about how we indexed wikipedia, for all of you trying to index huge amounts of text)
I created a dictionary using the words from wikipedia and then tested a lot of different misspelled words taken from different origins.








For each test case, i created a small Python script that simply queries every single misspelled word against Solr, and counts in which place the correct spelled word returns. The test case includes the correct expected word. You can download the source from [here](http://dl.dropbox.com/u/18984879/spell.zip).




The first set is a synthetic misspelled word list, that I created from a dictionary taken from Internet (it is a dictionary from Ispell) and applying modifications of “edit distance 1” to the words. I used part of the algorithm by Peter Norvig, from his excelent [article](http://norvig.com/spell-correct.html) on spellcheckers




Total Word processed: 19708
Found in the first 1 positions: 53%
Found in the first 2 positions: 58%
Found in the first 3 positions: 60%
Found in the first 10 positions: 61%




That means that 53% of the words were properly corrected in the first suggestion, 58% in the first two, and so on. Pretty awful results even with an easy dataset. But let's try something more difficult.




Aspell is the GNU spellchecker library from GNU. They provide a [dataset](http://aspell.net/test/cur/) that they use to test their library. They have very good results, but they use a different method.
I tried that library against our test environment and this is the result




Total Word processed: 547
Found in the first 1 positions: 27%
Found in the first 2 positions: 33%
Found in the first 3 positions: 37%
Found in the first 10 positions: 45%




Even worse. They do not specify the origin of these words. A good test would be using real mistakes from real humans. These were taken from a freely available [file](http://ota.oucs.ox.ac.uk/headers/0643.xml). I tested a list of common misspelled words from collage students, a list of typos, and a list of known spelling errors in the English language (all that can be seen in the Readme file that comes with the download, i don’t want to extend in this). The format of some of these files needed to be converted. In the previous code download I included the scripts for this.




**MASTERS** _Misspellings of about 260 words made in  spelling  tests  by  600 students  in  Iowa  in  the  1920's - 200 8th graders, 200 high-school seniors and 200 college seniors_




Total Word processed: 13020
Found in the first 1 positions: 27 %
Found in the first 2 positions: 35 %
Found in the first 3 positions: 38 %
Found in the first 10 positions: 44 %




**SHEFFIELD** _A list of about 380 misspellings,  mostly  keying  errors,  taken from  typewritten or computer-terminal input, collected from staff and students  in  the  Department  of  Information  Studies  of  Sheffield University  by  Angell,  Freund  and  Willett  as  part  of a piece of_
_ research into spelling correction._




Total Word processed: 384
Found in the first 1 positions: 57 %
Found in the first 2 positions: 62 %
Found in the first 3 positions: 65 %
Found in the first 10 positions: 70 %




**FAWTHROP** _A compilation of four  collections  of  American  spelling  errors, already in published form._




Total Word processed: 809
Found in the first 1 positions: 44 %
Found in the first 2 positions: 50 %
Found in the first 3 positions: 52 %
Found in the first 9 positions: 55 %




The best result and very similar to my first test is the one of typos. That's because typos are generally words with edit distance 1 to real words (you don't usually make two typos in the same word) The others are pretty bad. This scenario is the same that anyone indexing a big document corpus (as wikipedia) and creating an index with it would do, and that is the effectiveness that he'll get in the spellchecker.




What could be the reason of these results? Probably the results are bad because Solr spellchecker doesn’'t know anything about the natural languange that we call English.
How can you improve these results? With a different approach to spellchecking. There are algorithms that find words that sound similar to making suggestions. How a word sounds adds a lot of information about the natural language (and the psychology of making mistakes) to the system! (this is the aproach followed by GNU Aspell). Other algorithms use the information theory that Shannon created and consider that we humans are noisy channels that introduce noise (spelling mistakes) to words. If you introduce natural languages information via statistics, you can get better results (like in the [article](http://norvig.com/spell-correct.html) by Peter Norvig)
In future posts we'll implement some of these algorithms and (following the scientific tradition) we'll test them.
I'll see you in the future!!!




