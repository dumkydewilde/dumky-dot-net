---
title: "A Visual Leviathan: Hobbes' Schizophrenic Writing"
date: "2017-11-02"
tags:
  - "Thomas Hobbes"
  - "visualisation"
  - "data"
  - "text-analysis"
  - "tf-idf"
  - "philosophy"
description: "I have spent many hours of my life devoted to studying Thomas Hobbes' book Leviathan. It was published in 1651 and is considered a starting point of modern political philosophy. There is however a strange dichotomy between the first part of the book and the second part of the book. Read on to see how I've used data visualisation to show that difference."
---

I have spent many hours of my life devoted to studying Thomas Hobbes' book _Leviathan._ It was published in 1651 and is considered a starting point of modern political philosophy. It is a fascinating book, I even wrote a thesis on it, but one thing had always nagged me. You see, Hobbes writes very concise and structured arguments. He starts out by considering how people used to live before we had larger societies, this is where the 'solitary, poor, nasty, brutish and short' part comes in —Hobbes wasn't a fan of prehistoric human life it seem. He then goes on to consider how that 'natural' state influences and creates a framework for the justification of a centralised ruler, the _leviathan_. We subject ourselves to the leviathan —a medieval king basically— because we'll end up in a better place than the prehistoric chaos. That conclusion is drawn halfway through the book though, and after that is when things get messy. Hobbes then basically starts a new book on 'The Kingdome of God'.

I've always thought there was something odd about this change of topic. Of course, considering the place of religion in society as well as in academic work in the 17th century that is not very strange. What is strange to me is the kind of schizophrenic feeling that he creates with the divide between what seems a very modern, rational approach to politics and an esoteric investigation into the kingdom of god. I decided to try and see if I could visualise this discrepancy.

{{< rawhtml >}}
<iframe src="/hobbes-2017/number_of_words_per_chapter.html" style="border:none;margin-left:-60px;" width="1100" height="420"></iframe>
{{< /rawhtml >}}

I started out with a simple word count. Already you can see how chapter 42 'Of Power Ecclestial' stands out with about 15000 words. Chapter 31 is usually considered as the split between the two parts of the book. And you can definitely see that in just the simple word count, as that is where the longer —did I hear someone say boring?— chapters start.

{{< rawhtml >}}
<iframe src="/hobbes-2017/most_common_words_per_chapter.html" style="border:none;margin-left:-60px;" width="1100" height="420"></iframe>
{{< /rawhtml >}}

When we look at the most common words, we can see how the 15000 words of chapter 42 heavily influence our word ranking. There is definitely a difference in certain topics, as we expect, that can be seen in difference in occurrence of words like 'law', 'kingdome', or 'god'. But just counting words will not work if we really want to understand the difference in topics that occurs over the whole range of the book, especially the topics that are important, but can not necessarily be derived from how often words signifying that topic occur.

{{< rawhtml >}}
<iframe src="/hobbes-2017/punctuation.html" style="border:none;margin-left:-60px;" width="1100" height="420"></iframe>
{{< /rawhtml >}}

An approach that is sometimes used to distinguish between authors is the use of punctuation. I don't want to go into a full-blown authorship investigation, but a simple visualisation of punctuation use does point out some interesting differences. The use of punctuation is visualised relative to other punctuation marks. Most prominently we can see how in the latter chapters there is a lot more quotation from biblical works going on.

This still doesn't give us that much information about the differences in how the book is constructed. One way to better understand the importance of certain words in relation to other words is by using a model called Tf-Idf. This model consists of the term frequency (tf) in relation to how often it occurs —relatively— in a certain document (inverse document frequency, or IDF). Now a document can be anything from a tweet to an entire book, but in this case we'll take the chapters as documents, because they are units of analysis that can stand on their own and help us see how the different topics change throughout the book.

Term frequency can be a simple word count, a character (letter) count, or so-called n-grams. An n-gram is an occurrence of multiple words together. For the analysis we'll use 2-grams because of Hobbes' writing style. I'll give you an example. The 2-gram 'common wealth' is an important one for Hobbes. Using just a word count both the word 'common' and the word 'wealth' may occur often and thus show up in our analysis. But when we consider their relative importance we can see that just the word 'common' may have less significance than the 'common wealth'. The Tf-Idf model will help us distinguish between these differences in relative importance. If 'common wealth' occurs a lot in certain documents, but not in others it will score higher than just the word 'common', because the latter will occur in most of the documents. For that reason stop words like 'the', 'and' or 'for', though they occur frequently, score lower in the Tf-Idf model. It is the terms that occur a lot in certain documents, but not in others that are most often interesting to us. And that is exactly what the Tf-Idf model will help us retrieve from the _Leviathan_

{{< rawhtml >}}
<iframe src="/hobbes-2017/highest_score_by_place_of_occurence.html" style="border:none;margin-left:-60px;" width="1100" height="620"></iframe>
{{< /rawhtml >}}

Using a Tf-Idf we get a clearer picture of which words score higher (they appear darker) in the various chapters. To get an even better look at the differences in topics, I have also split up the book to calculate the most important words (highest scoring in the tf-idf model) for the two different parts. I have made the split at chapter 31 ("Of The Kingdome of God by Nature"). Words most important (top 50) in the early part of the book appear blue, words scoring high in the second part appear in orange, words in both lists appear grey.

There are of course the expected differences, 'spirit', 'church', and 'holy' occur mostly in the latter part of the book, whereas 'consequences', 'liberty' and 'warre' occur mostly in the earlier part. There are however some interesting things going on.

- Though the book's most important topics ('soveraign', 'law', 'common wealth') score highest in the first part of the book, they do seem to be more important in the second part than I remembered.
- Some words can obviously have a meaning that is both religious as well as political, like 'kingdome' or 'law'
- There are some interesting words like 'religion', 'worship', and 'private' that appear to be equally important in both parts of the book.

Now it is hard to conclude anything specific from these visualisations. And that is not what I would use them for. To me they are a way of better understanding the book itself. Where do different topics occur? How do they relate? Which topics should we investigate more closely? Does the same word really have different meanings in different places?

For me it is a reason to take a fresh look at Hobbes' work and a different way to look at such an important work of philosophical and political writing.

_\* You can find the code for the visuals on [Github](https://github.com/dumkydewilde/visual_leviathan/blob/master/Hobbes%20-%20Word%20counts%2C%20punctuation%20and%20tf-idf.ipynb). I will admit that there is also plenty to improve upon with regard to the analysis and visualisation. For example I have used the mean score in the tf-idf model across chapters as an indication of importance. That is of course not necessarily the best way, and means some words or topics will have been left out of this list. Each method has its pros and cons and this one serves my purpose for now._
