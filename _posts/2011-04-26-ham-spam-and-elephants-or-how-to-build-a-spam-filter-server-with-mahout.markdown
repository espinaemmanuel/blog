---
author: Emmanuel Espina
comments: true
date: 2011-04-26 15:26:41+00:00
layout: post
link: https://emmaespina.wordpress.com/2011/04/26/ham-spam-and-elephants-or-how-to-build-a-spam-filter-server-with-mahout/
slug: ham-spam-and-elephants-or-how-to-build-a-spam-filter-server-with-mahout
title: Ham, spam and elephants (or how to build a spam filter server with Mahout)
wordpress_id: 51
---

# Introduction

Something quite interesting has happened with Lucene. It started as a library, then its developers began adding new projects based on it. They developed another open source project that would add crawling features (among others features) to Lucene. [Nutch](http://nutch.apache.org/) is in fact a full featured web serach engine that anyone can use or modify. Inspired in some famous papers from Google about[ Map Reduce](http://labs.google.com/papers/mapreduce.html) and the [Google Filesystem](http://labs.google.com/papers/gfs.html), new features to distribute the index where added to Nutch and, eventually, those features became a project by them own: [Hadoop](http://hadoop.apache.org/). Since then, many projects have being developed over Hadoop. We are in front of a big explosion of open source code that was ignited by the Lucene spark.

All this projects are in some way related to content processing. For all of you interested in search and information retrieval, now we are going to talk about another project for a domain that's outside the strict confines of search, but can teach you some interest things about content processing.

Recently I have been reading about this new library, Mahout, that provides all those obscure and mysterious machine learning algorithms, together in a library. A lot of modern web sites are using machine learning techniques. These algorithms are rather old and well known, but were popularized recently by its extensive use in social network sites (facebook knowing better than you who could be your best friend) or Google (reading your mind and guessing what you may have wanted to write in the search box) .

_In simple words, we can say that a computer program is said to learn from experience E with respect to some class of tasks T and performance measure P, if its performance at tasks in T, as measured by P, improves with experience E._

For example if you wanted to make a program to recognize captchas you would have:

T: recognizing captchas

P: percentage of words correctly recognized

E: a database of captchas with correct words spellings

So, it appears that all they were needing was computing power. And they need a lot of computing power. For example, in supervised machine learning (I learned that term a month ago, don't expect me to be very academic) you show the machine some examples (experience) of what you want and the machine extract some patterns from them. It is pretty the same that a person does when he is learning: from some examples shown to the person, he uses its past knowledge to infer some kind of pattern that will help him to classify future events of the same kind. Well the computers are pretty dumb in learning things, so you have to show them a lot of examples to infer the pattern.

Classification, however, is only part of the problem. There are three big areas that Mahout targets (and there will be more in the future because the project is relatively new). Classification, Clusterization, and Recommendation. Typical examples of these:

  * **Classification**: it is supervised learning, you give the machine a lot of instances of... things (documents for example) along with their category. From all of those the machine learns to classify future instances in the known categories.

  * **Clusterization**: similar to classification in the sense it makes groups of things, but this one is not supervised. You give the machine a lot of things, and the machine makes groups of similar items.

  * **Recommendation**: It is exactly what IMDb or Amazon does with the recommended movies or books at the bottom of the page. Based on other users likes (which are measured through the ranking stars that the user vote) Amazon can infer “other users that liked this book also liked these others”.

If you want a better definition of these three categories you can go to [Grant Ingersoll's blog](http://www.ibm.com/developerworks/java/library/j-mahout/).

# Offline example

Now I'll show you a simple program that I think is a pretty obvious classification example. We are going to classify mail into [spam](http://www.youtube.com/watch?v=anwy2MPT5RE) and not-spam (that is called ham by people that researches it). The steps are the following:
	
  1. Get a ham/spam corpus from the web (already classified of course).
  2. Train a classifier with 80% of the corpus, and leave 20% for testing
  3. Create a simple web service that classifies spam online. You provide it a mail, and it will say “good” or “bad” (or “ham” or “spam”)

The corpus we are going to use is the one from [Spam Assasin](http://spamassassin.apache.org/), an open source project from Apache that is an antispam filter. This is a tutorial, so we are not trying to classify very difficult mails just to show how simple it can be done with Mahout (of course, the difficult stuff was programmed by the developers of this library). This [corpus](http://spamassassin.apache.org/publiccorpus/) is a pretty easy one and the results will be very satisfying.

Mahout comes with some examples already prepared. One of those is the [20 newsgroups example](https://cwiki.apache.org/MAHOUT/twenty-newsgroups.html) that tries to classify many mails from a newsgroup into their categories. This example is found in the Mahout wiki, and luckily for us, the format of the newsgroups are pretty the same as our mails. We are going to apply the same processing chain to our mails than the 20 newsgroup. By the way we are going to use an classification algorithm called naive Bayes that uses the famous [Bayes theorem](http://en.wikipedia.org/wiki/Bayes'_theorem), and that I already mentioned in a previous post. I'm not going to explain how the algorithm works, I'll just show you that it works!

Mahout has two driver programs (they are called that way because they are also used in Hadoop to run as map reduce jobs), one for training a classifier, and the other to test it.
When you train the classifier you provide it with a file (yes, a single file) that contains one document per line, already analyzed. “Analyzed” in the same sense that Lucene analyzes documents. In fact what we are going to use is the Lucene StandardAnalizer to clean a little the documents and transform them into a stream of terms. That stream is put in a line of this training file, where the first term is the category that the item belongs. For example the training file will look like this

```
ham new mahout version released
spam buy viagra now special discount
```

A small program comes with Mahout to turn directories of documents into this format. The directory must have an internal directory for each category. In our case we are going to separate our test set into two directories, one for testing and the other for training (both in <mahout_home>/examples/bin/work/spam where <mahout home> is where you unzipped the mahout distribution).
In each of them we are going to put a spam directory and a ham directory.

```
test
  ham
  spam

train
  ham
  spam
```
We manually take about 80% of the ham and put it in train/ham, the rest int test/ham, and the same with the spam in train/spam and test/ham (it has never been easier to prepare the test set!!!)
Next, we are going to prepare the train and test files with the following commands

```
bin/mahout prepare20newsgroups -p examples/bin/work/spam/train -o examples/bin/work/spam/prepared-train -a org.apache.mahout.vectorizer.DefaultAnalyzer -c UTF-8
bin/mahout prepare20newsgroups -p examples/bin/work/spam/test -o examples/bin/work/spam/prepared-test -a org.apache.mahout.vectorizer.DefaultAnalyzer -c UTF-8
```

Default analyzer is a Lucene analyzer (actually it is wrapped within a mahout class)

We are going to train the classifier. Training the classifier implies feeding mahout with the train file and letting him build internal structures with the data (yes, as you can deduce by my use of the word “internal”, I don't have any idea how that structures work).

```
bin/mahout trainclassifier -i examples/bin/work/spam/prepared-train -o examples/bin/work/spam/bayes-model -type bayes -ng 1 -source hdfs
```

The model is created in the bayes-model directory, the algorithm is Bayes (naïve Bayes) we are using Hadoop Distributed File System (we are not but you tell that to the command when you are not using a distributed database like Hbase), and ng is the ngrams to use. The ngrams are groups of words. Giving more ngrams you add more context to each word (surrounding words). The most ngrams you use the better the results should be. We are using 1 because the better results obviously cost more processing time.

Now we run the tests with the following command
```
 bin/mahout testclassifier -m examples/bin/work/spam/bayes-model -d examples/bin/work/spam/prepared-test -type bayes -ng 1 -source hdfs -method sequential
```

And after a while we get the following results
    
    -------------------------------------------------------
     Correctly Classified Instances : 383 95,75%
     Incorrectly Classified Instances : 17 4,25%
     Total Classified Instances : 400

    =======================================================
     Confusion Matrix
     -------------------------------------------------------
     a b <--Classified as
     189 11 | 200 a = spam
     6 194 | 200 b = ham


Very good results!!!

# A server to classify spam in real time

But we haven't done anything different from the 20newsgroup example yet! Now, what can we do if we want to classify mails as they are coming. We are going to create a antispam server, where the mail server will send all the mails that it receives and our server will respond if it is ham or spam (applying this procedure)

The server will be as simple as we can (this is just a proof of concept):

```java
public class Antispam extends HttpServlet {

 private SpamClassifier sc;

 public void init() {
  try {
   sc = new SpamClassifier();
   sc.init(new File("bayes-model"));
  } catch (FileNotFoundException e) {
   e.printStackTrace();
  } catch (InvalidDatastoreException e) {
   e.printStackTrace();
  }
 }

 protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
  Reader reader = req.getReader();
  
  try {
   long t0 = System.currentTimeMillis();
   String category = sc.classify(reader);
   long t1 = System.currentTimeMillis();

   resp.getWriter().print(String.format("{\"category\":\"%s\", \"time\": %d}", category, t1 - t0));

  } catch (InvalidDatastoreException e) {
   e.printStackTrace();
  }
 }
}
```

As we are going to do a very simple example we are going to use a simple servlet. The important class is SpamClassifier

```java
public class SpamClassifier {

 private ClassifierContext context;
 private Algorithm algorithm;
 private Datastore datastore;
 private File modelDirectory;
 Analyzer analyzer;

 public SpamClassifier() {
  analyzer = new DefaultAnalyzer();
 }

 public void init(File basePath) throws FileNotFoundException, InvalidDatastoreException {

  if (!basePath.isDirectory() || !basePath.canRead()) {
   throw new FileNotFoundException(basePath.toString());
  }

  modelDirectory = basePath;

  algorithm = new BayesAlgorithm();
  BayesParameters p = new BayesParameters();
  p.set("basePath", modelDirectory.getAbsolutePath());
  p.setGramSize(1);
  datastore = new InMemoryBayesDatastore(p);
  context = new ClassifierContext(algorithm, datastore);
  context.initialize();
 }

 public String classify(Reader mail) throws IOException, InvalidDatastoreException {
  String document[] = BayesFileFormatter.readerToDocument(analyzer, mail);
  ClassifierResult result = context.classifyDocument(document, "unknown");

  return result.getLabel();
 }
}
```

You have a datastore and an algorithm. The datastore represents the model that you previously created training the classifier. We are using a InMemoryBayesDatastore (there is also HbaseBayesDatastore that uses the Hadoop database), and we are providing it the base path and the ngrams size. We are using ngrams of 1 to simplify this example. Otherwise it is necessary to postprocess the analyzed text constructing ngrams.
The algorithm is the core of the method and it is an obvious instance of the Strategy design pattern. We are using BayesAlgorithm but well we could have used CbayesAlgorithm that uses the Complementary Naive Bayes Algorithm.
ClassifierContext is the interface you'll use to classify documents.

We can test our server using curl:

```  
curl http://localhost:8080/antispam -H "Content-T-Type: text/xml" --data-binary @ham.txt
```

and we get

```    
{"category":"ham", "time": 10}
```

# Conclusions

As we have seen, the spam filtering process can be separated into two parts. An offline process where you have a lot of mails already classified by someone, and train the classifier. And an online process where you test a document to classify it using the model previously created. The model can evolve, you can add more documents with more information and after you perform the offline processing you update the online server with the new model. The model can be very big. This is where Hadoop enters the scene. The offline process can be sent to a cluster running hadoop, and using the same libraries (Mahout!) you perform what looks like the exactly same algorithms and get the results faster. Of course the algorithms is not the same because it is been executed in parallel by the thousand of computers that you surely have in your cluster (or the two or three PCs you have). Mahout was designed with this in mind. Most of its algorithms were tailored to work over Hadoop. But the interesting thing is that they can also work without it for testing purposes, or when you must incorporate the algorithms to a server without the needs of distributed computation, like how we did in this post. The combination of its possibilities to be user over a cluster and to be embedded to a application makes Mahout a powerful library for modern applications that use data of web scale.