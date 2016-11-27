---
title: "Oregon State Bioacoustics Project: Identifying Birdsong in Recordings from the H. J. Andrews Research Forest"
thumb:
  filename: bioacoustics
  alt: Map showing elevation and rivers of a natural area
description: Application of machine learning to identify birdsong in recordings from H. J. Andrews Experimental Forest.
weight: 68
---

In 2010 I spent the summer working with the <a href='http://eecs.oregonstate.edu/research/bioacoustics/'>Oregon State Bioacoustics Group</a>. We parsed recordings from the H. J. Andrews Research Forest and applied machine learning techniques to identify birdsong. The ultimate goal of the project is to automatically detect the species of bird in recordings of birdsong, creating a more efficient way to collect data about **bird population distributions**.

To process the data, we split recordings into 15-second clips and performed a <a href='http://www.smbc-comics.com/?id=2874'>Fourier transform</a> to create a visual representation of the data. We worked on identifying regions of birdsong in time-frequency **spectrogram** images like the one below.

<img src='/images/spectrogram.png' title='chirp chirp' alt="White markings on a black background provide a visual representation of birdsong.">

To do that, we experimented with two techniques. First, we used **hysteresis** to identify louder regions within spectrograms. Second, we used a **Random Forest** trained on manually annotated spectrograms to assign a birdsong probability to each cell in a spectrogram.

For both methods, we identified a set of parameters that achieved a good balance between sensitivity and specificity by evaluating **ROC curves** over a range of parameter values. Using Random Forest, we achieved a per-cell true positive rate of .906 with a false positive rate of .043.

We were also able to use our results to produce clearer-sounding audio recordings by **reducing noise** outside of regions containing birdsong.
