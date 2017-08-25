---
layout: post
title: Google Summer of Code, Final Thoughts
lang: en
tags: [GSoC, ROOT]
---

Three months have already passed since the beginning of the programme, and the only task that is still in my TODO list is the writing of this post as my [*Work Product Submission*](https://developers.google.com/open-source/gsoc/help/work-product) report. Let's cross it out.

> Disclaimer: This post will be quite technical and probably really boring if you are not very interested in the ROOT development.

# TL;DR

If you just want to know what have been accomplished and the work that is left to do, you will find here a short description covering the highlights of my summer.

If you are brave enough and do not fear tricky details and documented decisions in the code design, go to the [detailed section]({{ site.baseurl }}{% post_url 2017-08-25-gsoc-final-thoughts %}#detailed-final-report) and enjoy.

## What Have I Done?

In a few words, my work in ROOT can be summarised as the generalization of the fitting algorithms interfaces in order to have parallelized and vectorized implementations that make the algorithms, and thus the computations of the final user, more efficient.

## What Code Have Been Merged?

At the end of the programme, two pull requests (PR) were merged:

* [Gradient interfaces templated.](https://github.com/root-project/root/pull/793)

The most important work is done in this PR, where the gradient interfaces are generalized and the parallelized and vectorized versions of [`Chi2`](https://github.com/root-project/root/blob/master/math/mathcore/inc/Fit/Chi2FCN.h), [`LogLikelihood`](https://github.com/root-project/root/blob/master/math/mathcore/inc/Fit/LogLikelihoodFCN.h) and [`PoissonLikelihood`](https://github.com/root-project/root/blob/master/math/mathcore/inc/Fit/PoissonLikelihoodFCN.h) gradient evaluations are implemented.

* [Fix execution policy error when IMT is OFF.](https://github.com/root-project/root/pull/871)

The commit in this PR just fixed a bug caused by the changes introduced in the first one.

## What Code Have *Not* Been Merged?

Three PRs remained unmerged at the end of the programme:

* [Adaptation of fitting interfaces to templated gradient functions](https://github.com/root-project/root/pull/890#issuecomment-324809388)

This PR finishes the work of the fitting interfaces generalization, providing the user the possibility to use gradient functions in their fittings. It also adds integration tests to check everything works as a whole.

* [Vectorization of TMath.](https://github.com/root-project/root/pull/655)

This second unmerged PR is part of an important ongoing work: the implementation of vectorized versions of the most important functions in `TMath`.

* [Test evaluation of gradient of multidimensional functions.](https://github.com/root-project/root/pull/887)

This last PR adds tests to check that the evaluations of the gradient of multidimensional functions, implemented in the first merged PR, are correct.

### What Is Left To Do?

The unmerged PRs should be finished and merged; in particular: the bug stopping the first PR to be merged should be fixed, the remaining functions in `TMath` should be vectorized and the failing test of the third PR should be inspected and the bug causing it should be fixed.

# Detailed Final Report

Ok, so now that you have an overall idea of my summer work, I will describe with details the what, the how and the why of the lines of code written through these months.

## Previous State of ROOT

Before GSoC17, ROOT had a good set of interfaces for fitting data; see, for example, the section [5. Fitting Histograms](https://root.cern.ch/root/htmldoc/guides/users-guide/ROOTUsersGuide.html#fitting-histograms) to make an idea of what ROOT can do for you. However, these interfaces were tied to a specific base type; namely, they worked only with `double`s. Furthermore, the code was first designed to serve as a set of serial algorithms, and parallelization was not taken into account.

Some work has been done in order to generalize these interfaces and add the possibility to use other types and other execution policies. See, for example, [the `struct Evaluate` in `FitUtil.h`](https://github.com/root-project/root/blob/master/math/mathcore/inc/Fit/FitUtil.h#L370), where the evaluations of `Chi2`, `LogLikelihood` and `PoissonLikelihood` functions were adapted to have vectorized and parallelized versions.

However, a lot of work in this direction needed to be done, and my summer has mainly focused in providing the possibility to evaluate the `Chi2`, `LogLikelihood` and `PoissonLikelihood` **gradient** functions.

## Using The New Code

Ok, so my code has cool tricky details and the commits I did on the PR are super cool, but before describing all the work that has been done, let's see how to use the new methods and the good results we can obtain. To illustrate all of this, we are going to inspect ---a slightly simplified version of--- the test in [`testGradient.cxx`](https://github.com/root-project/root/blob/master/math/mathcore/test/testGradient.cxx).

This test assumes that the user have some data to fit into a model defined by the following function:

{% highlight cpp %}
// Model function to test the gradient evaluation
template <class T>
static T modelFunction(const T *data, const double *params)
{
   return params[0] * exp(-(*data + (-130.)) * (*data + (-130.)) / 2) +
          params[1] * exp(-(params[2] * (*data * (0.01)) - params[3] * ((*data) * (0.01)) * ((*data) * (0.01))));
}
{% endhighlight %}

First of all, you would need to create a `TF1` to hold your function with initial parameters and a `TH1` to hold your data (here we have randomised data generated from the model function). Now you can use data types different to double for your model functions, so let's use here `Double_v`:

{% highlight cpp %}
// Create TF1 from model function and initialize the fit function
TF1 fModelFunction("TF1", modelFunction<Double_v>, 100, 200, 4);

const Double_t fParams[4] = {1, 1000, 7.5, 1.5};
fModelFunction.SetParameters(fParams);

// Create TH1 and fill it with values from model function
unsigned fNumPoints = 12801;
TH1D fHistogram("TH1", "Test random numbers", fNumPoints, 100, 200);
gRandom->SetSeed(1);
fHistogram.FillRandom("TH1", 1000000);
{% endhighlight %}

Now we can use a wrapped TF1 object to store our fitting function and populate the a BinData object with the data from the histogram.

{% highlight cpp %}
ROOT::Math::WrappedMultiTF1Templ<Double_v> fFitFunction(fModelFunction, 1);

ROOT::Fit::BinData fData;
ROOT::Fit::FillData(fData, &fHistogram, &fModelFunction);
{% endhighlight %}

And now to the fun part! We can create either a Chi2, PoissonLikelihood or LogLikelihood fitter with the vectorized fitting function and a multithread execution policy!

{% highlight cpp %}
using GradFunc = ROOT::Math::IGradientFunctionMultiDimTempl<double>;
using BaseFunc = ROOT::Math::IParamMultiFunctionTempl<Double_v>;

using ROOT::Fit;

// You can instantiate a Chi2 fitter...
Chi2FCN<GradFunc, BaseFunc> fFitter(fData, fFitFunction,
                                    ExecutionPolicy::kMultithread);

// ... or a PoissonLikelihood fitter...
PoissonLikelihoodFCN<GradFunc, BaseFunc> fFitter(fData, fFitFunction, 0, true,
                                                 ExecutionPolicy::kMultithread);

// ... or a LogLikelihood fitter.
LogLikelihoodFCN<GradFunc, BaseFunc> fFitter(fData, fFitFunction, 0, false,
                                             ExecutionPolicy::kMultithread);
{% endhighlight %}

Whatever the fitter is, what we can do now is to call its `Gradient` method:

{% highlight cpp %}
Double_t solution[4];

// This is where the magic happens <3
fFitter->Gradient(fParams, solution);
{% endhighlight %}

That this one last line works may seem trivial or unimportant, but it has been *a lot* of work, but the results are really worth it! Check the following graph, where we can see the speedups of the new (parallelized and vectorized) versions of the three gradient evaluations with this same case we have described:

![Speedups](/assets/img/speedups.png)

## Technical Details

That last graph was cool, but the work behind it was hard. Let us now describe with detail the main work of this project, implemented in [this PR](https://github.com/root-project/root/pull/793). Let's inspect every commit to see what I did and why:

* [Commit 16cbcb5 ](https://github.com/root-project/root/pull/793/commits/16cbcb5b3143f588bc01c06bc093192b236acad8): The very first commit I pushed to my ROOT fork implemented a parallelized version of the method `FitUtil::EvaluateChi2Gradient`. This was pretty direct, as I just had to add a new version of the method using the parallelization techniques provided by ROOT; in particular, `ROOT::TThreadExecutor::MapReduce` and its friends. This was implemented following the example of the method `FitUtil::EvaluateChi2Gradient`, which was already parallelized in the `Evaluate` struct. The finished implementation can be seen in the file FitUtil.cxx, method FitUtil::EvaluateChi2Gradient.
* [Commit b705cf4 ](https://github.com/root-project/root/pull/793/commits/b705cf489f85d8151dd626f7c011fdfd76ce7b54): This commit implemented a really important work, as all the remaining commits depended greatly on it: the generalization of the gradient interfaces; namely:
  * In IFunction.h, the interface IGradient{,Function}MultiDim was templated, keeping its `double` specialization in IFunctionFwd.h as a typedef that matched the old name in order to preserve backwards compatibility.
  * In ParamFunction.h, the interface IParametricGradFunctionMultiDimTempl was adapted to use the new IGradientFunctionMultiDimTempl, and the gradient-related methods were templated as well.
* [Commit f48ee1d](https://github.com/root-project/root/pull/793/commits/f48ee1d1cfc2fc13374993753828aaee9d62f0f8): While the previous commit templated the gradient interfaces, this one templated the gradient implementations; namely:
  * The method TF1::GradientPar, in TF1.h, was templated and its vectorized version was implemented.
  * The methods WrappedMultiTF1Templ::ParameterGradient and DoParameterDerivative in WrappedMultiTF1.h were templated. In this case, I needed to use some tricks to bypass cases where TFormula::EvalPar was called, as it is not vectorized yet.
* [Commit cad190c](https://github.com/root-project/root/pull/793/commits/cad190ca32b50ad9656111c226e569de2a7dba17): After the previous commit had made the vectorization scenario possible, this one tested it out by implementing a vectorized (and parallelized) version of `FitUtil::EvaluateChi2Gradient`,
* [Commit 448b894](https://github.com/root-project/root/pull/793/commits/448b8946044000ef543389aaea1cef1d175c3714): To test the previous work, a GoogleTest was added. We will come back to this later, as it is an important piece of work for the project, so let's keep the expectation until then.
* Commits [d81308c](https://github.com/root-project/root/pull/793/commits/d81308cfb7a99675fe16b1cc596c71ba51cba017), [7da041d](https://github.com/root-project/root/pull/793/commits/7da041da280e5b0ae2dd1ab8ffa520158675b552),  [66a4d9b](https://github.com/root-project/root/pull/793/commits/66a4d9bb92e6d3a86e38edaada3192db1b75ced8), [3acfc87](https://github.com/root-project/root/pull/793/commits/3acfc877dea49410bdd0637a4912c349ee8fc99c) and [2ef06c9](https://github.com/root-project/root/pull/793/commits/2ef06c9714329190c3afe0f6a261cac3405a3eab) introduced a bunch of refactors and bug fixes, but they are not that important to understand the whole of the work done.
* [Commit bd8a250](https://github.com/root-project/root/pull/793/commits/bd8a250e4c59a9c688b3a3117a5f3d087460dea5): This commit introduced an important change, as it modified a critical method, `TF1::GradientPar`, to make it thread-safe. It was necessary that this method was thread-safe, otherwise, the parallelizations that would use it would have been unsafe. Before this commit, `TF1::GradientPar` modified the private member `fParameters`. In a multi-threaded
scenario, this produced race conditions when other threads needed to access
this same parameter. Now, `GradientPar` modifies a *local* copy of `fParameters`. Others techniques to make the method thread-safe were studied, but all of them were rejected:
  * Making fParameters `thread_local`: this first option did not make sense, as `fParameters` is a non-static member and thus cannot be made `thread_local`.
  * Using a mutex to atomically call `GradientPar`: To decide between this option and the finally implemented, I did a quick benchmark, and saw that implementing the mutex made the parallelized version even slower than the original one, but always copying the vector inside GradientPar made the original version slower than before (although the parallelized version was in that case faster, as expected).
* [Commit 33229b3](https://github.com/root-project/root/pull/793/commits/33229b3e25a9e6c4fab92f90b1905773ac361413): This is a huge commit that implemented both parallelized and vectorized versions of the gradients of the two remaining fitting methods: `PoissonLikelihood` and `LogLikelihood`. I also added here tests to check that these new implementations were correct. This commit, then, can be thought as the union of the first, fourth and fifth described commits, but instead of focused on `Chi2`, it was focused on both `PoissonLikelihood` and `LogLikelihood`.
* [Commit 66db096](https://github.com/root-project/root/pull/793/commits/66db096a7f8f98befb3d3a9844aefe32b253ceb2): Final commit to adapt to a change to the codebase that was made while I was working in my local branch.