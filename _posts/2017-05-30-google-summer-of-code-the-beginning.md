---
layout: post
title: Google Summer of Code, the Beginning
lang: en
tags: [GSoC, ROOT]
---

Early this month I received an email whose subject read<!--more--> "GSoC 2017:
Congratulations, your proposal with CERN-HSF has been accepted!"



After weeks of looking for an organization, contacting with the mentors and preparing my proposal, I got finally selected in the one project I put all my efforts in. So it seems I am a GSoCer now, yay!

## The project

[ROOT][l_ROOT] is the awesome project I will spend my summer working on. But what is it exactly? According to its main site:

> ROOT is a modular scientific software framework. It provides all the
functionalities needed to deal with big data processing, statistical analysis,
visualisation and storage. It is mainly written in C++ but integrated with
other languages such as Python and R.

Ok, so basically **it is a thingy to do cool stuff with your cool data**, really nice. Let me show you an example: what if you need to plot some data of your latest particle physics discovery, say that time [when you discovered the Higgs bosson][l_higgs-plots]? Well, you obviously use ROOT for that:

![CMS 10 example](/assets/img/CMS10.png)

Indeed, ROOT is widely used in the CERN scientific community as a full-stack solution to analyse, interpret and visualise the data acquired from all their experiments.

Furthermore, ROOT comes with this shiny, breathtaking, awesome C++ interpreter. Please, take a look and enjoy the black magic behind interpreting C++ on the fly:

![ROOT interpreter](/assets/img/root_interpreter.png)

Isn't it beautiful? \<3 By the way, that code above is from the useful [ROOT Primer][l_manual], where you can learn all about its basics.

## The proposal

My proposal for the project was to improve the vectorization and parallelization of the ROOT mathematical libraries; i.e., I will try to make ROOT [faster, better, stronger][l_daft-punk].

The idea is to finish the on-going work in this direction, having a completely vectorized/parallelized ROOT maths library at the end of summer. The black magic should be transparent to the final user, who will find her functions to be magically faster than before.

I do not want to bore you with technical details now ---you will have plenty of those in the next posts---, but if you want to know anything else about the proposal, read its abstract in the [GSoC site][l_abstract]. If you *really* want to read the entire proposal, [go ahead][l_proposal].

Stay tuned for future posts, I will try to explain what I am doing, and how I do it, with all the tricky details. Until then, **happy libre hacking!**


<!--
{% highlight bash %}
cmake -Dbuiltin_veccore=ON -Dvc=ON -Dimt=ON -Droottest=ON -Dtesting=ON \
      -DCMAKE_INSTALL_PREFIX=/opt/root/math-vectorization ../root
{% endhighlight %}
-->

[l_ROOT]: http://root.cern.ch
[l_higgs-plots]: https://root.cern.ch/higgs-plots
[l_galaxies]: https://root.cern.ch/galaxy-view-0
[l_daft-punk]: https://www.youtube.com/watch?v=gAjR4_CbPpQ
[l_abstract]: https://summerofcode.withgoogle.com/projects/#5874058599071744
[l_proposal]: /assets/fls/proposal.pdf
[l_manual]: https://root.cern.ch/root/htmldoc/guides/primer/ROOTPrimer.html
