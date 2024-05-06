---
title: "Information Retrieval: Web Document Indexer and Search Engine Prototype"
slug: "information-retrieval-web-document-indexer-and-search-engine-prototype"
date: 2023-05-05T16:49:27-04:00
tags: ["Python", "Information Retrieval", "Search Engine", "Inverted Index"]
categories: ["Web Development"]

draft: false
---

View the full project here: [Github Repo](https://github.com/sky-haihai/4020A3-Information-Retrieval.git)

## Introduction

In a recent academic project, I created an efficient indexer and search engine for a collection of Web documents. My goal was to implement an program that could create an inverted index for the provided Web documents(.GZ files), and search these web documents for 20 topics/queries, outputting the search results in a file.

### Input

a bunch of .GZ files, each containing multiple XML web documents. Each xml doc contains multiple articles.

```xml
.GZ file
    XML file
        Article 1
        Article 2
        Article 3
        ...
        Article n
```

Each document looks like this:

```html
<DOC>
  <DOCNO>WT01-B01-1</DOCNO>
  <DOCOLDNO>IA073-000475-B029-48</DOCOLDNO>
  <DOCHDR>
    <!-- Document Description -->
  </DOCHDR>

  <html>
    <!-- Web Content -->
  </html>
</DOC>
```

Each topic looks like this:

```xml
<top>

<num> Number: 401
<title> foreign minorities, Germany

<desc> Description:
What language and cultural differences impede the integration
of foreign minorities in Germany?

<narr> Narrative:
A relevant document will focus on the causes of the lack of
integration in a significant way; that is, the mere mention of
immigration difficulties is not relevant.  Documents that discuss
immigration problems unrelated to Germany are also not relevant.

</top>
```

### Output

Query results for 20 topics

## Challenges

1. The size of the raw token(word) collection is too large to be loaded into RAM.
2. The documents are in XML format and some documents partially miss nessessary tags like `<html>`, which is not easy to parse.
3. The documents are in different encodings, which adds difficulty to the pre-processing module.

## Approach Overview

### 1. Pre-Processing

- · Unzip GZ files and read them as XML.
- · Pre-process XML into document number dictionary { docno: docoldno } and word dictionary {docno: [word1, word2, …..]}.

### 2. Building the Inverted Index

- · Build the inverted indexer using the word dictionary.
- · Group by initial and save each file into an individual JSON file to save RAM usage when applying searching.

### 3. Searching

- · Apply the vector space model to calculate the similarity between the query and each document.
- · Sort the similarity and output the top 1000 results for each query.

## Pre-Processing

### Unzipping GZ

```python
...
for filename in files:
    if filename.endswith('.GZ') or filename.endswith('.gz'):
        gz_file = os.path.join(root, filename)
        with gzip.open(gz_file, 'rt', encoding='utf-errors='ignore') as file:
            xml_content = file.read()
            result.append(xml_content)
...
```

### Natural Language Processing

Import `nltk` and download essential data from `nltk` server:

```python
import nltk
nltk.download('stopwords')
nltk.download('punkt')
nltk.download('words')
```

Tokenize all html contents and apply stemming to keep original english word only:

```python
tokens = word_tokenize(content_str)
tokens = [word.lower() for word in tokens if word.isalpha()]
filtered_tokens = [word for word in tokens if word not in stop_words]
stemmed_tokens = [stemmer.stem(word) for word in filtered_tokens]
only_words = [word for word in stemmed_tokens if word in english_words]
```

**Note**  
Stick with one encoding format, in this case, UTF-8 to avoid decoding errors when web documents contain special characters like Hanzi(Chinese letter) and Hirakana(Japanese word).

## Building the Inverted Index

At this point, all the web documents have been extracted, tokenized, stemmed into a dictionary: `{docno: [word1, word2,...]}`.

Now to build the index, simply count the frequency of each word

```python
for docno, words in word_dict.items():
    word_counts = Counter(words)
    for word, count in word_counts.items():
        inverted_index[word][docno] += count
```

## Searching/Querying

### Pre-Processing

Tokenize the query and apply stemming:

```python
topic_tokens = preprocess.topics_to_tokens(topic_file)
```

See [preprocess.py](https://github.com/sky-haihai/4020A3-Information-Retrieval/blob/master/src/preprocess.py)

### Vector Space Model

1. Treat the query and each documents as a vector.
2. Calculate the similarity between the query vector and each document vector

```python
sim = np.dot(query_vector, doc_vector) / (np.linalg.norm(query_vector) * np.linalg.norm(doc_vector))
```

See [calculate_similarity.py](https://github.com/sky-haihai/4020A3-Information-Retrieval/blob/master/src/calculate_similarity.py)

## Optimization

1. Calculating the TF-IDF value instead of dot product to mark the importance of each token (word) during the preprocess stage.
2. Multithreading during the preprocessing stage to speed up the process of reading HTML content and during the index building stage.
3. Using the vector space function for weighting.
4. Utilizing the dot product for calculating similarity in the vector space model.

## Analysis of Results

1. A full run which involves over **48,000** XML documents(**6,000,000** lines) takes approximately 2 hours, which was a significant improvement over the initial 3-hour runtime before implementing multithreading in the preprocessing stage and indexing stage.
2. Each query takes only **250ms** to complete in average.

## Conclusion

This project taught me valuable lessons about optimizing search engine algorithms and the importance of efficient indexing for web document retrieval. By using the appropriate data structures, algorithms, and parallel processing techniques, I was able to build an efficient and effective search engine.
