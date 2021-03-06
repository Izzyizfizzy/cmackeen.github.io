---
layout: default
title: hEXAFS
parent: Projects
---
<button class="btn js-toggle-dark-mode">~Nightmode~</button>

<script>
const toggleDarkMode = document.querySelector('.js-toggle-dark-mode')
const cssFile = document.querySelector('[rel="stylesheet"]')
const originalCssRef = cssFile.getAttribute('href')
const darkModeCssRef = originalCssRef.replace('just-the-docs.css', 'dark-mode-preview.css')

addEvent(toggleDarkMode, 'click', function(){
  if (cssFile.getAttribute('href') === originalCssRef) {
    cssFile.setAttribute('href', darkModeCssRef)
  } else {
    cssFile.setAttribute('href', originalCssRef)
  }
})
</script>


# Historical EXAFS
{: .no_toc }

The Bridges lab in the Physics department at University California, Santa Cruz specializes in local structure characterization using X-ray absorption techniques. Bud Bridges has collected and analyzed high quality EXAFS data for over 30 years; now it is time to learn from it all! This is a step in an effort to automate certain parts of this specialized analysis and uncover correlations between features of the materials studied and their absorption spectra.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---
## Preprocessing of data for unsupervised grouping of like-spectra at a select edge
2/28/2020
{: .fw-300 }

The goal of this application and initiative is to build a foundation for grouping XAS that can scale. For this reason I am employing Spark via pyspark. The grouping will be done without specifying number of clusters, as I do not assume to know the number of unique spectra; furthermore, I would like to have the potential for clustering streaming spectra. In this post I will go through the approach to preprocessing XAS data, which was inspired by AI approaches to categorizing audio signals. 
First, let us get a look at some raw XAS data:

{%include abs_raw.html %}

We can first start with subtracting the starting energy to zero the x-axis, and also subtracting the mean of mu. We get:

{%include norm1_abs.html %}

Now we will focus in on just the post-edge region to fit our bacground splines. We will follow rough procedure from the [xray-larch](https://xraypy.github.io/xraylarch/) project. In EXAFS we want to subtract the slowly-oscillating background. Since I am going to have to also resample the spectra to match in number of data points, I will also use a continuous function interpolation of the data. Once we fit the background splines and resample we are left with the following:

{%include norm2_abs.html %}

Finally, after reading some [guides](https://medium.com/heuristics/audio-signal-feature-extraction-and-clustering-935319d2225) and [literature](https://arxiv.org/pdf/1803.08276.pdf), I found the [Mel-frequency cepstral coefficients (MFCC)](https://en.wikipedia.org/wiki/Mel-frequency_cepstrum) will be useful features for clustering spectra. The image below is an the MFCC heatmap of the above normalized and background removed data:


![](/assets/mfcc_specshow_n-233.png)

The python notebooks I used in google colab can be found here: [preproccess](/assets/hexafs_preprocess_v2.ipynb) and [dbscan](/assets/ML_dbscan_mfcc.ipynb)

The next post will go over a quantitative tuning the DBSCAN
clustering alogrithm for our application, but I also plan on optimizing the
background removal and MFCC parameters, so as to ensure robust data processing even in anomalous scenarios. 


## Unsupervised grouping of like-spectra of a select edge: Overview and application
1/28/2020
{: .fw-300 }

TMy goal in this application is to develop a scalable framework that can categorize XAS scans without labels (unsupervised), with the intent of deploying it to the beam line computers. This will serve as a part of the foundation to smart materials research at SLAC using historical data.

I have developed a pipeline for processing XAS (EXAFS) data that runs in pyspark. While I have been using a smaller XAS dataset, I wanted the ETL pipeline to be ready and equipped for streaming XAS data at SSRL (or other labs). The ETL pipeline must be refined, but it does the following:

+ Reads aggregated mu vs energy absorption spectra DB
+ Trims to post-edge region
+ Zeroes energy, subtracts average mu
+ Fits slow moving background with spline (needs work)
+ Interpolates data, and subtracts spline function
+ Resamples interpolated XAS data on 800 pt grid
+ Calculates MCFF from XAS spectogram (saves MCFF dataframe)
+ Using MCFF of each scan, Applies DBSCAN on each edge.
+ DBSCAN leaves us with autogroups of like MCFF spectra
+ Attatch grouping labels back to original EXAFS data
+ Plot EXAFS data with autogroup labels

I have only autogrouped Iron and Zinc on a subset of HEXAFS database, but the app can be seen below. Clicking it will bring you to the app's page.

[![app link](/assets/hexafs_auotg_thumbnail.jpg)](http://ec2-3-88-164-235.compute-1.amazonaws.com:8080/)
[App link](http://ec2-3-88-164-235.compute-1.amazonaws.com:8080/)

Two items that will improve upon iteration are the spline bkg fitting, and the DBSCAN parameter optimization. I am excited to delve into Spark and realize ETL benefits of parallelized computing. Much has been learned and will be posted in Lessons Learned; more work is waiting!

## Aggregating post-edge absorption in an interactive periodic table
11/22/2019
{: .fw-300 }

I have stepped away from the hEXAFS project for some months, and recently rolled up the sleeves and began digging through the attempts at naive fourier transforms I used to apply to all of the postedge x-ray absorption (XAS) data. At that point I found it was a good idea to rigorously extract, reduce, and aggregate the post-edge data  to present it in the periodic table. My strategies rely on simple linear fits of the background, and shifting the edge to be somewhat normalized around $$\mu=$$. Further, I use a linear interpolating function as I plan on applying an FFT and would appreciate normal sampling in energy. Below is a periodic table with a logarithmic heat map of how many samples were used in the displayed average XAS post-edge spectrum. 

{%include ptable_pics.html %}

When hovering over an element, your are viewing an average of the normalized/reduced post-edge spectrum for all of the data Bud collected at that edge (and reduced). There are errors, some more apparent than others, and their source varies from fluctuation of edge energy to variety in samples studied that contain that element. When I was at the beamline watching a scan *slowwwwly* scroll by, I was always interested if the spectrum was being collected properly. How could I tell?  Experience that I did not yet have. So this is a rough idea of "fingerprints" for the edges studied; some elements should always exhibit a clear white line, and others none at all.

These images and plots will improve as I learn and hone my energy-space in-house data reduction methods, but this is progress worth sharing.

I will err on the side of physics-ignorance. How would I clean and engineer this data if I did not know much about XAS? My overall goal is to use Bud's library of reduced and analyzed EXAFS data to train a model to predict reduction parameters, like the k-window, k-weighting, E-min, and fitting splines. As I learn to aggregate the data and use signal processing techniques to improve FFT analysis, I am left to wonder how much National Labs like Oak Ridge, SLAC, and BNL can benefit from these tools, accelerating user discoveries. Well, on to the next iteration!



## General characteristics of spectra
6/27/2019
{: .fw-300 }

As the data is supposed to be normalized, I thought comparing the post-edge mean and standard deviation would be valuable. 
First plot here is the distribution of extracted means of the post-edge region. Interestingly the main distribution centered near 1 seems to have a bimodal feature.

{% include post_edge_mean_mu.html %}

And here is the distribution of standard deviation in the post-edge area


{% include post_edge_std_mu.html %}

I can see several modes in the distribution of mean aborption, and I think it would be helpful to render a new interactive periodic table plot where the hover includes information of each element's mean aborption in the EXAFS region. Along with that, I will include mena standard deviation.

## A first look at historical EXAFS
5/12/2019
{: .fw-300 }

The X-ray absorption spectra Bud Bridges has collected is atom specific, so we can get an overview of what elements were looked at over 30 years. Later on we can deduce specific compounds using both metadata and statistical modeling. But for now, we get a quick look at what energies Bridges lab collected data at.

To do this, I used a couple of filters and so currently this may be pulling from roughly a quarter of all the normalized e-space EXAFS data. A chunk of data are also lost to the low sensitivity of my absorption edge identitifier. 

Either way, below you can find a colored periodic table. When you hover on an
element, you can see how may data scans have been collected. It's a start !!

{%include ptable_trends.html %}


Evidently, Bud has a lot of history with manganates. The next post will feature the effectiveness of my edge detection, and how to improve it without running through all ~50,000 absorption spectra.
