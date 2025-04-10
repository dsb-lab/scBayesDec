# scBayesDeconv

This package allows the deconvolution of two measured distributions (e.g. distributions of protein abundance using flow cytometry with a fluorescence marker) using Bayesian mixture approaches. Let

```
Z = X + Y
```

where `X` is the signal of interest (e.g. protein abundance in the case of flow cytometry measurements), `Y` is an unwanted background noise (e.g. autofluorescence in flow cytometry), and `Z` is the total measured signal (that is obscured by the noise). If we have a sample of distribution measurements of `Y` and `Z`, the package tries to deconvolve the signal to obtain a distribution of `X`. Two kinds of Bayesian mixtures are implemented:

1. Gaussian
2. Gamma

## Installation

The package can be installed from the PyPi repository with the command:

```shell
pip install scBayesDeconv
```

In case of problems, you can always compile the package from the git repository. The requirements for installation are:
 1. CMake
 2. A C++ compiler and at least c++11 standard (g++, Visual Studio, Clang...)
 3. The scikit-build library for python (if not, `pip install scikit-build`)

In the gaussian deconvolution folder, create the binary installation.

```shell
python setup.py bdist_wheel
```

This will generate a wheel in an automatically created `dist` folder. Now, we can install it.

```shell
pip install ./dist/*
```

If everything is ok, you should be happily running the code after a few seconds of compilation ;)

## Brief tutorial

The package behaves very much like the [scikit-learn](https://scikit-learn.org/) package.

Consider that we have two arrays of data, one with some noise `dataNoise` and the second with the total signal (plus noise) `dataConvolved`.
First import the package

```python
import scBayesDeconv as bd
```

The models consider by default one basis distribution for the noise and one basis distribution for the convolved data. Consider that we want to fit the noise to one and the convolved data with three basis functions.

```python
model = bd.mcmcsamplernorm(K=1, Kc=3)
```

alternativaly, you can choose another basis function and sampling method. The four implemented methods of deconvolution are:

| Basis | Sampling method | Name |
|:-:|:-:|:-:|
| Normal | MCMC | mcmcsamplernorm |
| Normal | Nested | nestedsamplernorm |
| Gamma | MCMC | mcmcsamplergamma |
| Gamma | Nested | nestedsamplergamma |

Once declared, fit the model. The model accepts as entry two `numpy` arrays with one dimension of the length of the dataset. As an example:

```python
import numpy as np
dataNoise = np.random.normal(1,1,size=1000) #The noise comes from a distribution around one with standard deviation one.
dataConvolved = np.random.normal(1,np.sqrt(2),size=1000) #The deconvolved distribution comes from a distribution around zero with standard deviation one.

#Fit the data
model.fit(dataNoise,dataConvolved)
```

With the model fit, we can sample from the model

```python
model.sample_autofluorescence(size=100)
model.sample_deconvolution(size=100)
model.sample_convolution(size=100)
```

or evaluate at certain positions. This will return the mean value, as well as any specified percentiles (by default at 0.05 and 0.95).

```python
x = np.arange(0,1,0.1)
model.score_autofluorescence(x, percentiles=[0.05,0.5,0.95])
model.score_deconvolution(x, percentiles=[0.05,0.5,0.95])
model.score_convolution(x, percentiles=[0.05,0.5,0.95])
```

In addition, for the MCMC sampler methods, it is possible to obtain some summary statistics of the sampler in order to check if the sampling process has converged to the posterior distribution.

```python
model.statistics()
```

An `rhat` close to 1 indicates that the posterior chains have mixed appropiately. `neff` is an indicator of the effective number of independent samples drawn from the model. For more information, have a look to the [Stan](https://mc-stan.org/) package and its associated Bayesian statistics book, [Chapter 11](http://www.stat.columbia.edu/~gelman/book/).

### Which model should I use?

Both models correspond to the same posterior distribution, the only difference being how samples from this distribution are drawn. 

The **MCMC** sampler is based in Gibbs and MCMC markov chain steps with help of indicator variables. This are extensively explained in the book of [Gelman](http://www.stat.columbia.edu/~gelman/book/). Such sampler have the benefit of converging *fast* to a mode of the posterior and have the nice property of concentrating around *solutions with sparse number of components*. However, the posterior distribution of the Bayesian deconvolution model is multimodal and, for large noises, can lead to high degeneracies of the system. In such cases, samplers based in Markov chains may have severe dificulties to converge in complex cases. From the above reasons, this sampler should be used mainly for exploratory purposes, in order to have a general idea of the deconvolution as well as the number of components required to describe appropiately the posterior.

The **nested** sampler is based in the ideas of nested sampling introduced by [Skilling](https://projecteuclid.org/euclid.ba/1340370944). Such sampling methods have more power at exploring complex distributions with multimodalities and complex degeneracies. The counterpart is that the sampler does not select component sparse regions of the space and the exploration becomes fast computationally expensive with the number of components. In order to speed the computation, we wrapped the well documented and recently published library for dynamic nested sampling [Dynesty](https://dynesty.readthedocs.io/en/latest/) around C++ in order to obtain reasonable sampling times for samples of data of the order of magnitude tipically encountered flow citometry datasets. The posterior distributions obtained by this sampling method capture the complexity of the gaussian deconvolution better in a reasonable amount of time, in contrast to the MCMC sampler.

Overall, one should use the MCMC sampler for exploration and number of components selection, and feed that information to a selected nested sampler model in order to obtain the most reliable results within a reasonable computational time.
