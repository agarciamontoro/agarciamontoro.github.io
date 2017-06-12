---
layout: post
title: Google Summer of Code, First Impressions
lang: en
tags: [GSoC, ROOT, VecCore]
---

The first two weeks of coding are already gone, so this seems like a good moment to write an update of what I have been doing lately.

SPOILER: I have been writing random code until something worked, as everyone of us developers do every day.

Before digging into the details of these two weeks work, and as my [latest post]({{ site.baseurl }}{% post_url 2017-05-30-google-summer-of-code-the-beginning %}) was more a *hello* than a technical post, let me talk first about how to have a functional development environment for ROOT.

## Environment Set Up

Remember the goal of this GSoC project: to make ROOT ---its maths library--- faster. Ok, nice, but how do we even start with this? What do we need to do just to start writing code?

First of all, we have to build the latest version of ROOT. As with every libre software project, we have to locate its [repository](https://github.com/root-project/root), clone it to our machine and take a look at the [building instructions](https://root.cern.ch/building-root).

TL;DR: open a terminal, go to any directory and type the following:

{% highlight bash %}
git clone git@github.com:root-project/root.git
mkdir root-build
cd root-build
cmake -Dveccore=ON -Dbuiltin_veccore=ON \\
      -Dvc=ON -Dbuiltin_vc=ON \\
      -Dimt=ON -Droottest=ON -Dtesting=ON \\
      ../root-fork/
make
{% endhighlight %}

Ok, first three lines are quite self-explanatory, but the fourth one requires some comments. As you can see, ROOT is configured through the obscure but wonderful `CMake` tool, and the common syntax to prepare the build of a project like this one is something like `cmake <list_of_options> <source_path>`, where each of the options follow the syntax `-D<option_key>=<option_value>`.

Let's see what each of the options does:

* `-Dveccore=ON -Dbuiltin_veccore=ON`: these two options tell ROOT to be built with support of the `VecCore` library, and to use the version shipped within ROOT, not the one we could have already installed in our system.
* `-Dvc=ON -Dbuiltin_vc=ON`: as before, these ones specify that we want support for the `Vc` library shipped within ROOT.
* `-Dimt=ON`: let ROOT enable its multithreading capabilities.
* `-Droottest=ON -Dtesting=ON`: finally, tell ROOT to build all the tests for us, we will surely need them.

After the configuration is finished ---maybe you are prompted to install some required dependencies---, we can call `make` and... *Voilà*! Project built!

Now we should have a functional development environment in our machine, with support for vectorization and parallelization. But what are those `VecCore` and `Vc` libraries exactly? Why do we need them?

### Vectorization: Our New Best Friend

When we parallelize a code, we intend to run independent chunks of code at the **same** time, maybe in different CPUs, in different processors inside a GPU or even in completely different computers.

But what does *vectorization* means? *To vectorize* a code means to refactor its logic in order to compute all the operations over vectors ---or arrays of values---, as opposed to operate only with scalars. In computer jargon, is a way of implementing the [Single Instruction, Multiple Data (SIMD)](https://en.wikipedia.org/wiki/SIMD) paradigm: we apply **one operation** over **several values**.

The hardware architecture plays an important role in all this process, and it is important to understand it well in order to take advantage of its possibilities. Modern processors implement an extension to the common instruction set `x86`: it adds registers that do not contain single values but an array of them ---the number and size of the values are determined by the exact version of the extension, see this [Wikipedia article](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) for more information---, and instructions that operate over all the values in those special registers at once.

Obviously, we won't work directly with the processor instructions, and so we need a library that helps us abstracting the hardware layer and that gives us an interface to work with the vectors. As you can imagine, [`VecCore`](https://github.com/root-project/veccore/) and [`Vc`](https://github.com/VcDevel/Vc/) are the tools we need:

* `VecCore` is the interface we will be using every day, as it provides an "architecture independent API for expressing vector operations on data". However, `VecCore` is just that, an interface, and the hard work is done by the chosen backend: in this case, a SIMD library that implements processor-level vectorization.
* `Vc` is the backend we will use to port our high-level code to the hardware layer. It will do the dirty work for us, and thanks to `VecCore`, if everything works fine, we won't even notice it exists. However, be thankful to `Vc`, our life will be way easier thanks to its existence <3


## First Steps in Hacking ROOT

Ok, so we now have everything ready: ROOT is built with support for vectorization and we just met our new favourite SIMD libraries. Let's code!

One interesting way of starting playing with the code is going to the maths files and start vectorizing everything like a mad scientist. And this is basically what I have been doing the last two weeks. Let's see an example to understand the work that has to be done.

The file we will be tweaking will be [TMath.cxx](https://github.com/root-project/root/blob/master/math/mathcore/src/TMath.cxx), where all the complex mathematical functions are implemented. Our work is to study each of these functions, consider if they should be vectorized ---there are, for example, some functions whose vectorized version is already implemented in `VecCore`--- and start working on the ones that need a refactor.

### The Boring Scalar Version

For example, let's focus our attention in `TMath::Gaus` ---see [line 447](https://github.com/root-project/root/blob/master/math/mathcore/src/TMath.cxx) of the previous file---, whose code is copied here:

{% highlight cpp %}
Double_t TMath::Gaus(Double_t x, Double_t mean, Double_t sigma, Bool_t norm)
{
   if (sigma == 0) return 1.e30;
   Double_t arg = (x-mean)/sigma;
   // for |arg| > 39  result is zero in double precision
   if (arg < -39.0 || arg > 39.0) return 0.0;
   Double_t res = TMath::Exp(-0.5*arg*arg);
   if (!norm) return res;
   return res/(2.50662827463100024*sigma); //sqrt(2*Pi)=2.50662827463100024
}
{% endhighlight %}

This function implements the usual Gaussian function,

$$Gauss(x) = \frac{1}{\sqrt{2\pi\sigma^2}} e^{-\frac{(x - \mu)^2}{2\sigma^2}},$$

which is parametrized by the mean, \\(\mu\\), and the variance, \\(\sigma^2\\). The `norm` flag just specifies whether the first factor is present in the computation.

### The Astonishing Ultra-Fast Vectorized Version

What we want now is to change this function to accept an array of values `x`, with the parameters `mean` and `sigma` still being scalars.

First thing we have to do is to change the header of the function to accept a vectorized input and return a vectorized result.

{% highlight cpp %}
Double_v TMath::Gaus(Double_v x, Double_t mean, Double_t sigma, Bool_t norm)
{% endhighlight %}

Look at the new type in the game: `Double_v`. This type, provided by `VecCore`, is defined in [`Math_vectypes.hxx`](https://github.com/root-project/root/blob/master/math/mathcore/inc/Math/Math_vectypes.hxx): it abstracts one of those special registers we talked about before, specifically, one containing a vector of doubles. In my machine, `Double_v` contains just two doubles.

Ok, now to the interesting part.

First of all, we see in the scalar version that if \\(\sigma = 0\\), we just return an *infinity* value. As `VecCore` handles very well the conversion between a scalar and a vector ---it just sets every value in the vector to the scalar value---, we can just return \\(\infty\\) as before:

{% highlight cpp %}
if (sigma == 0)
   return 1.e30;
{% endhighlight %}

In any other case, we can compute the value of the exponent \\(\frac{x - \mu}{\sigma}\\). Now, however, \\(x\\) is a vector, so the exponent will also be a vector. Again, as `VecCore` handles really well the basic operations between vector and scalars, we can just change the type of the variable `arg` and let `VecCore` do its magic:

{% highlight cpp %}
Double_v arg = (x-mean)/sigma;
{% endhighlight %}

The next thing we encounter in the original code is an `if` statement, whose condition depends on the values of the --now vector--- variable `arg`. The operations where the code branches depending on a vector value have to be studied with care: some input values can make the conditions true and others might make it false. What we need now is a way to specify which values satisfy the condition and which values don't.

Fortunately, `VecCore` provides an interesting tool to handle these issues: the masks. A mask can be seen as the vectorized version of a boolean: think of it as a vector where each position stores either a `true` or a `false`. These masks can be used to handle the code logic, using what is known as a **masked assign**: to set the values of a vector where the mask has a `true` to some result we want. Let's see some code and explain it later:

{% highlight cpp %}
// for |arg| > 39  result is zero in double precision
vecCore::Mask_v<Double_v> mask = !(arg < -39.0 || arg > 39.0);

// Initialize the result to 0.0
Double_v res(0.0);

// Compute the function only when the arg meets the criteria, using the mask computed before
vecCore::MaskedAssign<Double_v>(res, mask, vecCore::math::Exp<Double_v>(-0.5 * arg * arg));
{% endhighlight %}

First things first: we compute the mask as we could compute any boolean condition: with the usual relational operators ---now applied to vectors---. We store the mask in a new type, again provided by `VecCore`: `vecCore::Mask_v<Double_v>`, which is templated by its base scalar value ---remember that the length of the vectors differ depending on the size of the base types, so we can have, for example, vectors of four integers *and* vectors of two doubles; then, it is important for the mask to know the base type in order to know how many values to store---.

Note that we have computed the mask where \\(\vert arg \vert \leq 39\\); e.g., the mask is `true` where the final values will be different than `0.0`. So what we do next is to default all values in the result vector to zero, and compute the interesting ones using the mask.

The last line is where the magic happens: we tell `VecCore` to set `res` to the value of the exponential considering only the positions of the vector seen through the `mask`. That is one nice line of code.

The last two lines can be condensed in just one, using the function `Blend` provided by `VecCore`: `Blend` reads a mask and two inputs, and returns a vector where the values depend on the mask: the ones where the mask is `true` are set to the values of the first input, whereas the ones where the mask is `false` are set to the values of the second one. Let's see how we can do that:

{% highlight cpp %}
// Compute the two input to be blended by the mask
Double_v exponential = vecCore::math::Exp<Double_v>(-0.5 * arg * arg);
Double_v zero = 0.0;

Double_v res = vecCore::Blend(mask, exponential, zero);
{% endhighlight %}

As you can see, `Blend` is like a vectorized one-liner `if-else`. Isn't it elegant?

To finish the work, we should check the flag `norm` and act consequently, but `VecCore` and `Vc` make our life easier so we don't have to change *anything* :) To summarise, take a look at the final code:

{% highlight cpp %}
Double_v TMath::Gaus(Double_v x, Double_t mean, Double_t sigma, Bool_t norm)
{
   if (sigma == 0)
      return 1.e30;

   Double_v arg = (x-mean)/sigma;

   // for |arg| > 39  result is zero in double precision
   vecCore::Mask_v<Double_v> mask = !(arg < -39.0 || arg > 39.0);

   // Initialize the result to 0.0
   Double_v res(0.0);

   // Compute the function only when the arg meets the criteria, using the mask computed before
   vecCore::MaskedAssign<Double_v>(res, mask, vecCore::math::Exp<Double_v>(-0.5 * arg * arg));

   if (!norm)
      return res;

   return res/(2.50662827463100024*sigma); //sqrt(2*Pi)=2.50662827463100024
}
{% endhighlight %}
	
## Testing The Changes

We now have an idea of how to implement the new versions of the maths functions, but there is still one thing we have not discussed: the code tests. Testing your code is more important than writing it ---ok, that's not true, but it **is** really important---, so let's take a quick look at how we handle this. Although the testing suite in ROOT is a little bit complex, we will try to just understand how to implement and register a test using the [Google Test](https://github.com/google/googletest) framework.

ROOT uses `ctest`, the testing tool from CMake, to manage all tests in its codebase. The CMake configuration provides a series of tools to register the implemented tests, but the one where we will focus our attention is the function `ROOT_ADD_GTEST`.

Suppose we have our test implemented in the file `math/mathcore/test/vectorization/testTMathStatistic.cxx`. Then, we just need to modify the file `math/mathcore/test/CMakeLists.txt` and add these lines at the end:

{% highlight cmake %}
if(veccore)
   ROOT_ADD_GTEST(vectorizationTest vectorization/testTMathStatistic.cxx
                  LIBRARIES Core MathCore)
endif()
{% endhighlight %}

This chunk of code first checks whether ROOT has been built with `VecCore`. If this is true, then it adds a Google Test named `vectorizationTest`, which is implemented in the file with relative path `vectorization/testTMathStatistic.cxx`, and link it to the `Core` and `MathCore` libraries. To understand the details behind the `ROOT_ADD_GTEST` function, just read [its implementation](https://github.com/root-project/root/blob/master/cmake/modules/RootNewMacros.cmake#L1126).

Once we build ROOT with our changes, we will be able to execute the test with the following command:

{% highlight bash %}
ctest -R vectorizationTest
{% endhighlight %}

Hopefully, if everything is fine, we will see a beautiful succesful test output.

Ok, but what does a test look like? This is really well explained in the Google Test documentation, so I will just let the code here for testing the vectorized version of `TMath::Gaus` and  you just stay tuned for future posts, I will explain it in detail!

{% highlight cpp %}
/**
 * Common data and set up for testing vectorized functions from TMath
 */
class TVectorizedMathTest : public ::testing::Test {
protected:
   TVectorizedMathTest() {
      // Randomize input data and parameters
      TRandom1 rndmzr;
      rndmzr.RndmArray(fInputSize * vecCore::VectorSize<Double_v>(), fLinearInput);
      rndmzr.RndmArray(fInputSize, fMean);
      rndmzr.RndmArray(fInputSize, fSigma);

      // Set -100 < mean < 100 and 0 < sigma < 200
      for (size_t i = 0; i < fInputSize; i++) {
          fMean[i] = fMean[i] * 200 - 100;
          fSigma[i] *= 200;
      }

      // Copy input linear data to the vectorized array
      for (size_t caseIdx = 0; caseIdx < fInputSize; caseIdx++) {
          for (size_t vecIdx = 0; vecIdx < vecCore::VectorSize<Double_v>(); vecIdx++) {
             vecCore::Set(fVectorInput[caseIdx], vecIdx,
                          fLinearInput[vecCore::VectorSize<Double_v>() * caseIdx + vecIdx]);
          }
      }

   }
   // Data available to all tests in this suite

   static const int fInputSize = 100000;

   // Vectorized and linear input
   Double_v fVectorInput[fInputSize];
   Double_t fLinearInput[fInputSize * vecCore::VectorSize<Double_v>()];

   // Vectorized and linear output
   Double_v fVectorOutput[fInputSize];
   Double_t fLinearOutput[fInputSize * vecCore::VectorSize<Double_v>()];

   // Parameters vector
   Double_t fMean[fInputSize];
   Double_t fSigma[fInputSize];

};

/**
 * Test vectorized TMath::Gaus function
 */
TEST_F(TVectorizedMathTest, VectorizedGaus) {
   // Compute the vectorized output
   for (size_t caseIdx = 0; caseIdx < fInputSize; caseIdx++) {
      fVectorOutput[caseIdx] = TMath::Gaus(fVectorInput[caseIdx], fMean[caseIdx], fSigma[caseIdx]);
   }

   // Compute the linear output
   for (size_t caseIdx = 0; caseIdx < fInputSize; caseIdx++) {
      for (size_t vecIdx = 0; vecIdx < vecCore::VectorSize<Double_v>(); vecIdx++) {
         size_t linear_index = caseIdx * vecCore::VectorSize<Double_v>() + vecIdx;
         fLinearOutput[linear_index] = TMath::Gaus(fLinearInput[linear_index], fMean[caseIdx], fSigma[caseIdx]);
      }
   }

   // Compare linear and vectorized output
   for (size_t caseIdx = 0; caseIdx < fInputSize; caseIdx++) {
      for (size_t vecIdx = 0; vecIdx < vecCore::VectorSize<Double_v>(); vecIdx++) {
         size_t linear_index = caseIdx * vecCore::VectorSize<Double_v>() + vecIdx;
         EXPECT_NEAR(fLinearOutput[linear_index], vecCore::Get(fVectorOutput[caseIdx], vecIdx), 1e-15);
      }
   }
}

#endif // R__HAS_VECCORE
{% endhighlight %}

## Conclusions

This first two weeks have been quite fun and I learned a lot: I now understand a lot more about `VecCore` and `Vc`  than before and a bunch of functions have been vectorized. Until now, the followings functions are vectorized and tested:

* `TMath::BreitWigner`
* `TMath::CauchyDist `
* `TMath::Gaus`
* `TMath::KolmogorovProb`
* `TMath::LaplaceDist`
* `TMath::LaplaceDistI`

To vectorize these first functions took waaaaaaaay longer than expected, but I am now more confident with my code and I expect to finish this first vectorization batch at the end of the week. In fact, according to the initial schedule, my first Pull Request should be submitted at the end of this week!

I will keep you updated, so stay tuned and **happy libre hacking**!
