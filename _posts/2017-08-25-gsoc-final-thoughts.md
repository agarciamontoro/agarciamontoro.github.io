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

At the end of the programme, three pull requests (PR) were merged:

* [Gradient interfaces templated.](https://github.com/root-project/root/pull/793)

The most important work is done in this PR, where the gradient interfaces are generalized and the parallelized and vectorized versions of [`Chi2`](https://github.com/root-project/root/blob/master/math/mathcore/inc/Fit/Chi2FCN.h), [`LogLikelihood`](https://github.com/root-project/root/blob/master/math/mathcore/inc/Fit/LogLikelihoodFCN.h) and [`PoissonLikelihood`](https://github.com/root-project/root/blob/master/math/mathcore/inc/Fit/PoissonLikelihoodFCN.h) gradient evaluations are implemented. The whole PR was squashed into [this one commit](https://github.com/root-project/root/commit/913ac6a71bc1f0874e5ce8c49933e0ef2b5b9dd7) when it got merged into `master`.

* [Fix execution policy error when IMT is OFF.](https://github.com/root-project/root/pull/871)

After the first PR was merged, we saw that some tests failed if `IMT` was disabled. The problem was that those tests tried to use the multithread execution policy in `FitUtil` even if no parallelization was enabled. This commit, that you can see [here](https://github.com/root-project/root/commit/8a2c336b5bfe6a6667df29fda629067f25c0313e) in the `master` branch, fixed this problem forcing the execution policy to be `ROOT::Fit::ExecutionPolicy::kSerial` while warning the user that such a change has been made.

* [Add padding to all vectors in FitData and its children.](https://github.com/root-project/root/pull/896)

The classes storing the data in the fitting algorithms ---`FitData`, `BinData` and `UnBinData`--- play an important role in the vectorization. In order to ease the vectorization of the algorithms, this PR adds padding to every vector in these classes: the ones storing the coordinates, the data and the errors; assuring the length of the vectors is a multiple of the `SIMD` vector size. The commit merging this PR in the ROOT repo can be found [here](https://github.com/root-project/root/commit/a61beeca70473d10d01a53f3f051e2a3d52ec4b9).

## What Code Has *Not* Been Merged?

Three PRs remained unmerged at the end of the programme, but they contain useful work that should be used after GSoC finishes:

* [Adaptation of fitting interfaces to templated gradient functions](https://github.com/root-project/root/pull/890)

This PR completes the work of the fitting interfaces generalization, providing the user the possibility to use gradient functions in their fittings. It also adds integration tests to check everything works as a whole.

* [Vectorization of TMath.](https://github.com/root-project/root/pull/655)

This second unmerged PR is part of an important ongoing work: the implementation of vectorized versions of the most important functions in `TMath`.

* [Test evaluation of gradient of multidimensional functions.](https://github.com/root-project/root/pull/887)

This PR adds tests to check that the evaluations of the gradient of multidimensional functions, implemented in the first merged PR, are correct.

### What Is Left To Do?

After GSoC finishes, the ROOT developers may want to complete some of the work I started during the summer that did not get merged before the final evaluation. In particular, the unmerged PRs should be finished and merged: the bug stopping the first PR to be merged should be fixed, the remaining functions in `TMath` should be vectorized and the failing test of the last PR should be inspected and the bug causing it should be fixed.

# Detailed Final Report

Ok, so now that you have an overall idea of my summer work, I will describe with details and decisions behind the lines of code written through these months.

## Previous State of ROOT

Before GSoC17, ROOT had a good set of interfaces for fitting data; see, for example, the section [5. Fitting Histograms](https://root.cern.ch/root/htmldoc/guides/users-guide/ROOTUsersGuide.html#fitting-histograms) of the *User's Guide* to make an idea of what ROOT can do for you. However, these interfaces were tied to a specific base type; namely, they worked only with `double`s. Furthermore, the code was first designed to serve as a set of serial algorithms, and parallelization was not taken into account.

Considerable work has been done before the programme in order to generalize these interfaces and add the possibility to use other types and other execution policies. See, for example, the [`struct Evaluate`]((https://github.com/root-project/root/blob/master/math/mathcore/inc/Fit/FitUtil.h#L370)) in `FitUtil.h`, where the evaluations of `Chi2`, `LogLikelihood` and `PoissonLikelihood` functions were adapted to have vectorized and parallelized versions.

However, a lot more work in this direction needed to be done, and my summer has mainly focused on providing the possibility to evaluate vectorized and parallelized vrsions of `Chi2`, `LogLikelihood` and `PoissonLikelihood` **gradient** functions.

## Using The New Code

Ok, so my code has nice tricky details and the commits I did on the PR are super cool, but before describing all the work that has been done, let's see how to use the new methods and the good results we can obtain.

### Gradient and Fitting Interfaces

To illustrate the main work done with the fitting interfaces and the gradients, we are going to inspect ---a slightly simplified version of--- the tests in [`testGradient.cxx`](https://github.com/root-project/root/blob/master/math/mathcore/test/testGradient.cxx).

These tests assume that the user have some data to fit into a model defined by the following function:

{% highlight cpp %}
// Model function to test the gradient evaluation
template <class T>
static T modelFunction(const T *data, const double *params)
{
   return params[0] * exp(-(*data + (-130.)) * (*data + (-130.)) / 2) +
          params[1] * exp(-(params[2] * (*data * (0.01)) - params[3] * ((*data) * (0.01)) * ((*data) * (0.01))));
}
{% endhighlight %}

First of all, you would need to create a `TF1` to hold your function with initial parameters and a `TH1` to hold your data (here we have randomised data generated from the model function). With the work done during the summer, you can now use data types different to double for your model functions, so let's use here `ROOT::Double_v`:

{% highlight cpp %}
// Create TF1 from model function and initialize the fit function
TF1 fModelFunction("TF1", modelFunction<ROOT::Double_v>, 100, 200, 4);

const Double_t fParams[4] = {1, 1000, 7.5, 1.5};
fModelFunction.SetParameters(fParams);

// Create TH1 and fill it with values from model function
unsigned fNumPoints = 12801;
TH1D fHistogram("TH1", "Test random numbers", fNumPoints, 100, 200);
gRandom->SetSeed(1);
fHistogram.FillRandom("TH1", 1000000);
{% endhighlight %}

Now we can use a wrapped `TF1` object to store our fitting function and populate a `BinData` object with the data from the histogram.

{% highlight cpp %}
ROOT::Math::WrappedMultiTF1Templ<ROOT::Double_v> fFitFunction(fModelFunction, 1);

ROOT::Fit::BinData fData;
ROOT::Fit::FillData(fData, &fHistogram, &fModelFunction);
{% endhighlight %}

And now to the fun part! We can create either a `Chi2`, `PoissonLikelihood` or `LogLikelihood` fitter with the vectorized fitting function and a multithread execution policy!

{% highlight cpp %}
using GradFunc = ROOT::Math::IGradientFunctionMultiDimTempl<double>;
using BaseFunc = ROOT::Math::IParamMultiFunctionTempl<ROOT::Double_v>;

// You can instantiate a Chi2 fitter...
ROOT::Fit::Chi2FCN<GradFunc, BaseFunc> fFitter1(
   fData, fFitFunction,
   ROOT::Fit::ExecutionPolicy::kMultithread
);

// ... or a PoissonLikelihood fitter...
ROOT::Fit::PoissonLikelihoodFCN<GradFunc, BaseFunc> fFitter2(
   fData, fFitFunction, 0, true,
   ROOT::Fit::ExecutionPolicy::kMultithread
);

// ... or a LogLikelihood fitter.
ROOT::Fit::LogLikelihoodFCN<GradFunc, BaseFunc> fFitter3(
   fData, fFitFunction, 0, false,
   ROOT::Fit::ExecutionPolicy::kMultithread
);
{% endhighlight %}

Whatever the fitter you choose, what we can do now is to call its `Gradient` method:

{% highlight cpp %}
Double_t solution[4];

// This is where the magic happens <3
fFitter1->Gradient(fParams, solution);
{% endhighlight %}

That this one last line works may seem trivial or unimportant, but it has been *a lot* of work, but the results are really worth it! Check the following graph, where we can see the speedups of the new (parallelized and vectorized) versions of the three gradient evaluations with this same case we have described.

![Speedups](/assets/img/speedups.png)

The orange bars show the speedup when using the parallel implementation. The yellow ones show the speedup of the vectorized implementations; this benchmark is done in a machine with the instruction set [`SSE4.2`](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions). Finally, the green bars show the speedups of the parallel *and* vectorized implemetations,

## Dissecting the Commits

### Gradient and Fitting Interfaces

That last graph was cool, but the work behind it was hard. Let us now describe with detail the main work of this project, implemented in [this PR](https://github.com/root-project/root/pull/793). Let's inspect every commit to see what I did and why:

* [Commit 16cbcb5 ](https://github.com/root-project/root/pull/793/commits/16cbcb5b3143f588bc01c06bc093192b236acad8): The very first commit I pushed to my ROOT fork implemented a parallelized version of the method `FitUtil::EvaluateChi2Gradient`. This was pretty direct, as I just had to add a new version of the method using the parallelization techniques provided by ROOT; in particular, `ROOT::TThreadExecutor::MapReduce`. This was implemented following the example of the method `FitUtil::EvaluateChi2`, which was already parallelized in the `Evaluate` struct. The finished implementation can be seen in the file `FitUtil.cxx`, method `FitUtil::EvaluateChi2Gradient`.
* [Commit b705cf4 ](https://github.com/root-project/root/pull/793/commits/b705cf489f85d8151dd626f7c011fdfd76ce7b54): This commit implemented a really important work, as all the remaining commits depended greatly on it: the generalization of the gradient interfaces; namely:
  * In `IFunction.h`, the interface `IGradient{,Function}MultiDim` was templated, keeping its `double` specialization in `IFunctionFwd.h` as a typedef that matched the old name in order to preserve backwards compatibility.
  * In `ParamFunction.h`, the interface `IParametricGradFunctionMultiDimTempl` was adapted to use the new `IGradientFunctionMultiDimTempl`, and the gradient-related methods were templated as well.
* [Commit f48ee1d](https://github.com/root-project/root/pull/793/commits/f48ee1d1cfc2fc13374993753828aaee9d62f0f8): While the previous commit templated the gradient interfaces, this one templated the gradient implementations; namely:
  * The `method TF1::GradientPar`, in `TF1.h`, was templated and its vectorized version was implemented.
  * The methods `WrappedMultiTF1Templ::ParameterGradient` and `DoParameterDerivative` in `WrappedMultiTF1.h` were templated. In this case, I needed to use some tricks to bypass cases where `TFormula::EvalPar` was called, as it is not vectorized yet.
* [Commit cad190c](https://github.com/root-project/root/pull/793/commits/cad190ca32b50ad9656111c226e569de2a7dba17): After the previous commit had made the vectorization scenario possible, this one tested it out by implementing a vectorized (and parallelized) version of `FitUtil::EvaluateChi2Gradient`.
* [Commit 448b894](https://github.com/root-project/root/pull/793/commits/448b8946044000ef543389aaea1cef1d175c3714): To test the previous work, a GoogleTest was added. The previous study on the test is based on the work done here.
* Commits [d81308c](https://github.com/root-project/root/pull/793/commits/d81308cfb7a99675fe16b1cc596c71ba51cba017), [7da041d](https://github.com/root-project/root/pull/793/commits/7da041da280e5b0ae2dd1ab8ffa520158675b552),  [66a4d9b](https://github.com/root-project/root/pull/793/commits/66a4d9bb92e6d3a86e38edaada3192db1b75ced8), [3acfc87](https://github.com/root-project/root/pull/793/commits/3acfc877dea49410bdd0637a4912c349ee8fc99c) and [2ef06c9](https://github.com/root-project/root/pull/793/commits/2ef06c9714329190c3afe0f6a261cac3405a3eab) introduced a bunch of refactors and bug fixes, but they are not that important to understand the whole of the work done.
* [Commit bd8a250](https://github.com/root-project/root/pull/793/commits/bd8a250e4c59a9c688b3a3117a5f3d087460dea5): This commit introduced an important change, as it modified a critical method, `TF1::GradientPar`, to make it thread-safe. It was necessary that this method was thread-safe, otherwise, the parallelizations that would use it would have been unsafe. Before this commit, `TF1::GradientPar` modified the private member `fParameters`. In a multi-threaded
scenario, this produced race conditions when other threads needed to access
this same member. Now, `GradientPar` modifies a *local* copy of `fParameters`. Others techniques to make the method thread-safe were studied, but all of them were rejected:
  * Making fParameters `thread_local`: this first option did not make sense, as `fParameters` is a non-static member and thus cannot be made `thread_local`.
  * Using a mutex to atomically call `GradientPar`: to decide between this option and the one that was finally implemented, I did a quick benchmark, and saw that implementing the mutex made the parallelized version even slower than the original one, but always copying the vector inside `GradientPar` made the original version slower than before (although the parallelized version was in that case faster, as expected).
* [Commit 33229b3](https://github.com/root-project/root/pull/793/commits/33229b3e25a9e6c4fab92f90b1905773ac361413): This is a huge commit that implemented both parallelized and vectorized versions of the gradients of the two remaining fitting methods: `PoissonLikelihood` and `LogLikelihood`. I also added here tests to check that these new implementations were correct. This commit, then, can be thought as the union of the first, fourth and fifth described commits, but instead of focused on `Chi2`, it was focused on both `PoissonLikelihood` and `LogLikelihood`.
* [Commit 66db096](https://github.com/root-project/root/pull/793/commits/66db096a7f8f98befb3d3a9844aefe32b253ceb2): Final commit to adapt to a change to the codebase that was made while I was working in my local branch.

### Integrating Everything

After the generalization of the gradient interfaces and the implementation of the gradient evaluation of the three main functions in the fitting algorithms, we still havae one thing to do: to integrate everything so that the user do not have to worry about the details behind the calls they make. And here it comes [one of the unmerged PRs](https://github.com/root-project/root/pull/890), where `ROOT::Fit::Fitter` is adapted so that it can work with functions with templated gradients. This way, the user will be able to call the `Fit` method, so that they have nothing to change in their code but it is still way more efficient.

Let's see the two unmerged commits to know how the integration is made:

* [Commit df7bb2d](https://github.com/root-project/root/pull/890/commits/df7bb2ddbef22b943fc427d53513311303ee22f7): The main work of this PR is done here. In top of the `IGradModelFunction` used by the `Fitter` class, this commit adds a new `typedef` for the *vectorized* model functions with gradient: `IGradModelFunction_v`. The main changes can be found on `Fitter::DoLeastSquareFit`, `Fitter::DoBinnedLikelihoodFit` and `Fitter::DoUnbinnedLikelihoodFit`, that perform the fit using the now well-known classes `Chi2FCN`, `PoissonLikelihoodFCN` and `LogLikelihoodFCN`, respectively. Here, the commit adds one case uncovered until now: the case for vectorized functions with implementing the gradient interface. The new code handles the case calling the `*FCN` classes correspondingly, using the work done until now in the main PR.
* [Commit 59e3fb9](https://github.com/root-project/root/pull/890/commits/59e3fb9044e41dc96c312180040602fed69b97f7): To check that the new code works, the scalar, vectorial, unbinned and binned fitting cases are tested here. These tests are the reason why the PR is still unmerged, as they expose one bug that make the unbinned fit test fail.

Here we should also talk about one of the other unmerged PRs: [the one adding the padding to all vectors in FitData and its children](https://github.com/root-project/root/pull/896). `FitData` is basically a wrapper of an `std::vector`, and to ease the vectorization and the vectorized loops on the fitting data, this PR pushes a [single commit](https://github.com/root-project/root/pull/896/commits/d92fa845533c524e25009ec401db19efaddf4ff0) that generalizes the notion of padding to all coordinates, data and error vectors that are abstracted out by `FitData`, `BinData` and `UnBinData`. The idea behind this is to assure that the length of these vectors is always a multiple of the `SIMD` vector size, so that when we loop over them jumping as many elements as one vector can hold, we do not get into the terrifying world of uninitialised memory. This PR just adds a static function, `FitData::VectorPadding`, that computes the remaining of elements that should be artifiacilly added to a vector in order to have a length that is a multiple of the `SIMD` vector size. This function is then used throughout the initialising and resizing code of `FitData` and its children to ensure that everything is safe for the final user.

### Nice Touches

Just to finish this report, we should talk about the ongoing work of vectorizing the functions in `TMath`. This work is started by [this unmerged PR](https://github.com/root-project/root/pull/655). Here, I added vectorized implementations of six functions:

* `TMath::Min`
* `TMath::Gaus`
* `TMath::BreitWigner`
* `TMath::CauchyDist`
* `TMath::LaplaceDist`
* `TMath::LaplaceDistI`


These versions, implemented in [`TMathVectorized.cxx`](https://github.com/root-project/root/pull/655/files#diff-b2fef511d71d4d2d8920b0a596c42a4e) are good examples to know how to keep vectorizing all functions in `TMath`.

# Conclusions

This summer has been as fun as hard, I learned way more than I expected at the beginning ---and I expected to learn *a lot*--- and I am really happy to have contributed to this big complex interesting project. Furthermore, it has been awesome to work in it with my mentors Xavier and Lorenzo, to whom I have to thank all their help throughout these three months.

I have learned all kind of subtleties of `C++`, I have understood that I knew nothing about `git` until now, I have learned about processors, vectorization techniques and low-level issues I did not know they even existed, I have improved my communication skills with other developers, I have really grasped the idea of open source contribution in a big project... In conclusion, I have enjoyed the programme in so many levels that I am sure it will be one of the most useful experiences I will have ---at least in the near future---.

I hope my work is useful for the ROOT users and that my code is clear enough for the developers that will have to read it in the future!
