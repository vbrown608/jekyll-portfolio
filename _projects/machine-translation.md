---
title: Re-Ranking Text Simplification
links:
  - Senior thesis, Pomona College
  - <a href='/files/Re_Ranking_Text_Simplification_Vivian_Brown.pdf' target='_blank'>Download full paper</a>
---

For my senior thesis I worked on **machine text simplification**: translating complex sentences into simple ones.
The goal of text simplification is to preserve the meaning of a sentence while switching to a more accessible structure and vocabulary.

This project builds on Kauchak and Coster (2010), who used a **probabilistic phrase-based translation model** for text simplification.
For training and test data they used paired sentences from English Wikipedia and Simple English Wikipedia.

In this project, I researched models of linguistic complexity in order to identify **features of simple sentences**.
Coding in **Java**, I used those features to train an **SVM re-ranker** and re-rank the output of the baseline model.

Thanks to Professor Kauchak and Professor Burke, my thesis advisors!
