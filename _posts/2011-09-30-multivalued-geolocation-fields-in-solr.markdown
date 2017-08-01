---
author: Emmanuel Espina
comments: true
date: 2011-09-30 20:36:39+00:00
layout: post
link: https://emmaespina.wordpress.com/2011/09/30/multivalued-geolocation-fields-in-solr/
slug: multivalued-geolocation-fields-in-solr
title: 'Multivalued geolocation fields in Solr '
wordpress_id: 104
---

Today we'll see a small workaround that can be used to accomplish a very common use case regarding geographic search.

The example use case is the following

_The client is located in Buenos Aires and wants a purple tie_
_He enters the query “purple tie” in the search box from or store web page_
_The system returns the ties that can be purchased in the stores near Buenos Aires, and then in our stores in Montevideo (ie ties that can be found in nearby stores)_

That is, the system does not returns ties in our stores in Spain because nobody would travel to Spain to buy a tie nor will order a tie across the Atlantic Ocean (no tie is so special). This is part of the first and probably only problem in search systems: return only relevant results. In this case relevant considering their location.
The problem in theory could be easily solved with a multivalued coordinate field, where each tie (or product in our system) will have a list of coordinates where that tie is located. We would perform a filtering of our products in a circle centered in Buenos Aires to get the nearby products.
This works fine if the product is available only in one store, but when it is in multiple locations simultaneously (ie, there is a list of coordinates in the field, not a single one) the problem arises. It happens that Solr does not allow filtering on multivalued coordinate fields.
But not everything is lost, and in this post I'll propose a workaround to solve this issue.
We are going to create another index, containing only the Stores, their id, and location. Using a C style pseudo code we can consider this as a document in our “stores index”
    
    struct {   
    	latlon location;
    	int storeID;
    } store;

Having this other index (another Solr Core for example) we are going to split our query in two:

1- For the first query you will query the “stores index”. You are going to perform a geographic filter query (using the spatial Solr functionality) and you will get a list of store ids near the central point you specified. You haven't used the query entered by the user in this phase yet, only the location of the user got by some means.    

    q=*:*&fl=storeID&fq={!geofilt pt=-34.60,-58.37 sfield=location d=5}

And you obtain a list of store ids

	id=34, id=56, id=77

2 - In the second phase you are going to perform the query (now providing the query that the user entered) in the regular way, but with the addition of a filter query (a regular filter query, not a geographic one) in the following fashion:

    q=”product brand x”&fq=(<strong>storeId:34 OR storeId:56 OR storeId:77</strong>)

where the store ids are the ones returned by the first query and, consequently, are the ones near the central point.
In this case you are restricting your results to the ones near the user. You can also boost the nearest results but also showing the ones that are far away in advanced pages of the results.
To summarize, we use two queries with regular functionality to get the advanced functionality you desire. The first query gets the stores near the zone, and the second is the actual query.
There is a patch in development to accomplish the same functionality [SOLR-2155](https://issues.apache.org/jira/browse/SOLR-2155) but is has not been commited yet. Meanwile here you have a good example of what you can do with multiple queries to Solr.
