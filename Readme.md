auto-phrase-tokenfilter
=======================

Lucene Auto Phrase TokenFilter implementation (forked from: https://github.com/LucidWorks/auto-phrase-tokenfilter)


Performs "auto phrasing" on a token stream. Auto phrases refer to sequences of tokens that
are meant to describe a single thing and should be searched for as such. When these phrases
are detected in the token stream, a single token representing the phrase is emitted rather than
the individual tokens that make up the phrase. The filter supports overlapping phrases.

The Autophrasing filter can be combined with a synonym filter to handle cases in which prefix or
suffix terms in a phrase are synonymous with the phrase, but where other parts of the phrase are
not. This enables searching within the phrase to occur selectively, rather than randomly.

##Overview

Search engines work by 'inverse' mapping terms or 'tokens' to the documents that contain
them. Sometimes a single token uniquely describes a real-world entity or thing but in many
other cases multiple tokens are required.  The problem that this presents is that the same
tokens may be used in multiple entity descriptions - a many-to-many problem. When users
search for a specific concept or 'thing' they are often confused by the results because of
this type of ambiguity - search engines return documents that contain the words but not
necessarily the 'things' they are looking for. Doing a better job of mapping tokens (the 'units'
of a search index) to specific things or concepts will help to address this problem.

##Algorithm

The auto phrase token filter uses a list of phrases that should be kept together as single
tokens. As tokens are received by the filter, it keeps a partial phrase that matches
the beginning of one or more phrases in this list.  It will keep appending tokens to this
‘match’ phrase as long as at least one phrase in the list continues to match the newly
appended tokens. If the match breaks before any phrase completes, the filter will replay
the now unmatched tokens that it has collected. If a phrase match completes, that phrase
will be emitted to the next filter in the chain.  If a token does not match any of the
leading terms in its phrase list, it will be passed on to the next filter unmolested.

##Example field type configuration (schema.xml)

```xml
<fieldType name="text_autophrase" class="solr.TextField" positionIncrementGap="100">
<analyzer type="index">
    <tokenizer class="solr.StandardTokenizerFactory" />
    <filter class="solr.LowerCaseFilterFactory" />
    <filter class="com.lucidworks.analysis.AutoPhrasingTokenFilterFactory"
            phrases="autophrases.txt" includeTokens="true"
            replaceWhitespaceWith="_" />
    <filter class="solr.StopFilterFactory" ignoreCase="true"
            words="stopwords.txt" enablePositionIncrements="true" />
    <filter class="solr.SynonymFilterFactory" synonyms="synonyms.txt"
            ignoreCase="true" expand="true" />
    <filter class="solr.KStemFilterFactory" />
  </analyzer>
  <analyzer type="query">
    <tokenizer class="solr.StandardTokenizerFactory" />
    <filter class="solr.LowerCaseFilterFactory" />
    <filter class="solr.StopFilterFactory" ignoreCase="true"
            words="stopwords.txt" enablePositionIncrements="true" />
    <filter class="solr.KStemFilterFactory" />
  </analyzer>
</fieldType>
```

##Input Parameters:

<table>
 <tr><td>phrases</td><td>file containing auto phrases (one per line)</td><tr>
 <tr><td>includeTokens</td><td>true|false(default) - if true adds single tokens to output. For instance, given that there is a phrase "seat belts", includeToken=true will yield three tokens: <i>seat, seat belts, belts</i>, where includeTokens=false only one: <i>seat belts</i></td></tr>
 <tr><td>replaceWhitespaceWith</td><td>single character to use to replace whitespace in phrase</td></tr>
</table>

##Query Parser Plugin

Due to an issue with Lucene/Solr query parsing, the AutoPhrasingTokenFilter is not effective at query time as=
part of a standard analyzer chain. This is due to the LUCENE-2605 issue (fixed in 6.2) in which the query parser sends each token
to the Analyzer individually and it thus cannot "see" across whitespace boundries. To redress this problem, a wrapper
QParserPlugin is incuded (AutoPhrasingQParserPlugin) that first isolates query syntax (in place), auto phrases and then
restores the query syntax (+/- operators) so that it functions as originally intended. The auto-phrased portions are
protected from the query parser by replacing whitespace within them with another character ('_'). 

To use it in a SearchHandler, add a queryParser section to solrconfig.xml:
```xml
  <queryParser name="autophrasingParser" class="com.lucidworks.analysis.AutoPhrasingQParserPlugin" >
      <str name="phrases">autophrases.txt</str>
      <str name="replaceWhitespaceWith">_</str>
  </queryParser>
```
And a new search handler that uses the query parser:
```xml
<requestHandler name="/autophrase" class="solr.SearchHandler">
   <lst name="defaults">
     <str name="echoParams">explicit</str>
     <int name="rows">10</int>
     <str name="df">text</str>
     <str name="defType">autophrasingParser</str>
   </lst>
   <lst name="invariants">
     <str name="defType">autophrasingParser</str>
   </lst>
  </requestHandler>
```
##Example of search for multi-term phrases 

To see autophrasing tokenfilter in action, we need to create two additional files: autophrases.txt(new file) and synonyms.txt (probably already existing). Autophrases.txt contains list of example phrases:
```txt
air filter
air bag
air conditioning
rear window
rear seat
rear spoiler
rear tow bar
seat belts
seat cushions
bucket seat
timing belt
fan belt
front seat
heated seat
keyless entry
keyless start
electronic ignition
```
In synonyms.txt it is now possible to add multi-term synonyms, like so:
```txt
...

# mutli-term synonyms
keyless_start,electronic_ignition
```
Important things to note are: 
* multi-term synonyms must be also defined in the autophrases.txt file
* instead of space characted, they should use *replaceWhitespaceWith* character defined in field type configuration (underscore in our case)

To do the test we will also need some example data file, testdoc.xml:
```xml
<add>
  <doc>
    <field name="id">1</field>
    <field name="name">Doc 1</field>
    <field name="text">This has a rear window defroster and really cool bucket seats.</field>
  </doc>
  <doc>
    <field name="id">2</field>
    <field name="name">Doc 2</field>
    <field name="text">This one has rear seat cushions and air conditioning – what a ride!</field>
  </doc>
  <doc>
    <field name="id">3</field>
    <field name="name">Doc 3</field>
    <field name="text">This one has gold seat belts front and rear.</field>
  </doc>
  <doc>
    <field name="id">4</field>
    <field name="name">Doc 4</field>
    <field name="text">This one has front and side air bags and a heated seat.
The fan belt never breaks.</field>
  </doc>
    <doc>
    <field name="id">5</field>
    <field name="name">Doc 5</field>
    <field name="text">This one has big rear wheels and a seat cushion.
It doesn't have a timing belt.</field>
  </doc>
 <doc>
    <field name="id">6</field>
    <field name="name">Doc 6</field>
    <field name="text">This one has electronic ignition</field>
  </doc>
</add>
```
Trying the multi-term synonym *seat cushions*, using the standard select handler:
```sh
curl "http://localhost:8983/solr/collection1/select?q=seat+cushions" | xmllint --format -
```
doesn't recognize *seat cushions* phrase and returns 5 results:
```xml
<response>
  <lst name="responseHeader">
    <int name="status">0</int>
    <int name="QTime">2</int>
    <lst name="params">
      <str name="q">seat cushions</str>
    </lst>
  </lst>
  <result name="response" numFound="5" start="0">
    <doc>
      <str name="id">2</str>
      <str name="name">Doc 2</str>
      <arr name="text">
        <str>Doc 2</str>
        <str>This one has rear seat cushions and air conditioning – what a ride!</str>
      </arr>
      <long name="_version_">1553061204035895296</long>
    </doc>
    <doc>
      <str name="id">5</str>
      <str name="name">Doc 5</str>
      <arr name="text">
        <str>Doc 5</str>
        <str>This one has big rear wheels and a seat cushion.
It doesn't have a timing belt.</str>
      </arr>
      <long name="_version_">1553061204065255424</long>
    </doc>
    <doc>
      <str name="id">1</str>
      <str name="name">Doc 1</str>
      <arr name="text">
        <str>Doc 1</str>
        <str>This has a rear window defroster and really cool bucket seats.</str>
      </arr>
      <long name="_version_">1553061203988709376</long>
    </doc>
    <doc>
      <str name="id">3</str>
      <str name="name">Doc 3</str>
      <arr name="text">
        <str>Doc 3</str>
        <str>This one has gold seat belts front and rear.</str>
      </arr>
      <long name="_version_">1553061204047429632</long>
    </doc>
    <doc>
      <str name="id">4</str>
      <str name="name">Doc 4</str>
      <arr name="text">
        <str>Doc 4</str>
        <str>This one has front and side air bags and a heated seat.
The fan belt never breaks.</str>
      </arr>
      <long name="_version_">1553061204054769664</long>
    </doc>
  </result>
</response>
```
The same query using autophrase handler:
```sh
curl "http://localhost:8983/solr/collection1/autophrase?q=seat+cushions" | xmllint --format -
```
returns only one result:
```xml
<response>
  <lst name="responseHeader">
    <int name="status">0</int>
    <int name="QTime">15</int>
    <lst name="params">
      <str name="q">seat cushions</str>
    </lst>
  </lst>
  <result name="response" numFound="1" start="0">
    <doc>
      <str name="id">2</str>
      <str name="name">Doc 2</str>
      <arr name="text">
        <str>Doc 2</str>
        <str>This one has rear seat cushions and air conditioning – what a ride!</str>
      </arr>
      <long name="_version_">1553061204035895296</long>
    </doc>
  </result>
</response>
```

##Example of search for multi-term synonyms
Previously we've defined following synonym group:
```txt
keyless_start,electronic_ignition
```
So, we should be able to search for *keyless start* and find document with phrase *electronic ignition*
```sh
curl "http://localhost:8983/solr/collection1/autophrase?q=keyless+start" | xmllint --format -
```
returns:
```xml
<response>
  <lst name="responseHeader">
    <int name="status">0</int>
    <int name="QTime">13</int>
    <lst name="params">
      <str name="q">keyless start</str>
    </lst>
  </lst>
  <result name="response" numFound="1" start="0">
    <doc>
      <str name="id">6</str>
      <str name="name">Doc 6</str>
      <arr name="text">
        <str>Doc 6</str>
        <str>This one has electronic ignition</str>
      </arr>
      <long name="_version_">1553061204075741184</long>
    </doc>
  </result>
</response>
```
The same query with /select handler will return empty set, because there is no documents with terms keyless or start.

##Deployment Procedure:

These files should be placed in SOLR_HOME/collection/conf dir, in my case it was ...solr-4.10.3-cdh5.5.4/example/solr/collection1/conf

To build the autophrasing token filter from source code you will need to install Apache Ant (http://ant.apache.org/bindownload.cgi). Install Ant and then in a linux/unix shell or Windows DOS command window, change to the auto-phrase-tokenfilter directory (i.e. where you downloaded this project to) and type: ant

Assuming that everything went well( BUILD SUCCESSFUL message from Ant), you will have a Java archive file called auto-phrase-tokenfilter-1.0.jar in the auto-phrase-tokenfilter/dist subdirectory. Copy this file to [solr-home]/lib (you may have to create the /lib folder first). In a typical Solr 4.x install, [solr-home] would be at /example/solr. Then restart Solr.

The code included in this distribution was compiled with Solr 4.10.3-cdh5.5.4
