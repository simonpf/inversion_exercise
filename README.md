This repository contains the second exercise for the PhD course in
inverse theory. It can be retrieved from 
[`github.com/simonpf/inverse_theory_exercise_b/`](http://github.com/simonpf/inverse_theory_exercise_b/) or from the command line:

    git clone https://github.com/simonpf/inverse_theory_exercise_b

After cloning the repository (or downloading it) run the `get_data.sh` script
to download the data for this exercise. This will create and populate the `data`
subdirectory in your exercise folder.

    chmod +x ./get_data.sh
    ./get_data.sh


# Repository Structure

-   `exercise_b.py`: The python exercise template. This should be a good
    starting point if you are planning to do the exercise in python.
-   `exercise_b.m`: The MATLAB exercise template. This should be a good
    starting point if you are planning to do the exercise in MATLAB.
-   `data/`: Subdirectory containing the data for the exercise.
    -   `/data/python/`: Contains all data required for this exercise
        in the form of `numpy` arrays.
    -   `/data/matlab/`: Contains the `exercise_b.mat` which holds all data
        required for this exercise.
    -   `/data/plots/`: Additional data required by the provided plotting
        functions.
-   `utils/`: Subdirectory containing additional code mainly for plotting
    of the results.
-   `doc/`: This exercise description.


# Background and Summary

In this exercise you will retrieve the *integrated ice column density* 
or *ice water path* (IWP) from passive microwave observations from the
*Global Preciptiation Measurement* (GPM) *Microwave Imager*. Eliasson
et al. <sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup> argue that measuring the bulk mass of ice in the atmosphere
constitutes an important gap in the current global climate observation
system, which leads to large differences in the IWP estimates of climate
models. Retrieving IWP from passive microwave sensors provides global coverage
at a much higher frequency than can be achieved with active sensors. Compared
to LIDAR observations, microwave observations also have the advantage of being
able to penetrate through thick clouds.

![img](./doc/plots/gpm.png "The Global Precipitation Measurement Microwave Imager onboard the GPM core observatory satellite.")


# Methods

The measurement of cloud properties from passive microwave observations
is based on the interaction of thermal microwave radiation with 
clouds through scattering. While possible, modeling scattering in
a radiative transfer model is computationally too costly to be performed
*during* the retrieval. In this exercise we are therefore considering
two methods that use a *precomputed* database <sup><a id="fnr.2" class="footref" href="#fn.2">2</a></sup> consisting of 
pairs \(\{(\mathbf{y}_i, x_i)\}^n_{i = 1}\) of simulated brightness
temperatures \(\mathbf{y}_i\) and corresponding IWP values \(x_i\). The 
ensemble of atmospheric states from which the database is computed,
was generated from profiles of ice water content obtained from
the DARDAR<sup><a id="fnr.3" class="footref" href="#fn.3">3</a></sup> dataset. This is to ensure that the ensemble follows a
physically meaningful a priori distribution.


## Bayesian Monte Carlo Integration

The basic idea of *Bayesian Monte Calo Integration* (BMCI) or simply
*Monte Carlo Integration* as a retrieval method is to use importance
sampling to transform samples from the a priori distribution to 
samples from the posterior distribution.

Consider the expected value \(\mathcal{E}_{x | \mathbf{y}}(f(x))\) of a function
\(f\) computed with respect to the a posteriori distribution \(p(x | \mathbf{y})\):

\begin{align}
 \int f(x') p(x' | \mathbf{y}) \: dx'
\end{align}

Using Bayes theorem, the integral can be computed as

\begin{align}
 \int f(x') p(x' | \mathbf{y}) \: dx' &=
\int f(x') \frac{p(\mathbf{y} | x')p(x')}{\int p(\mathbf{y} | x'') \: dx''} \: dx'
\end{align}

To simplify notation, we introduce the weighting function \(w(\mathbf{y}, x)\) :

\begin{align}
  w(\mathbf{y}, x) = \frac{p(\mathbf{y} | x')}{\int p(\mathbf{y} | x'') \: dx''}
\end{align}

Note that the second integral in (2) is just the expectation value 
\(\mathcal{E}_x\{f(x)w(\mathbf{y}, x)\}\) of the function \(f\) weighted with
the weighting function \(w(\mathbf{y}, x)\) but with respect to the a priori distribution. 
The integral in (1) can thus be approximated by a sum of 
\(f(x_i)w(\mathbf{y}, x_i)\) over a simulation database, which is distributed
according to the a priori distribution \(p(x)\):

\begin{align}
 \int f(x') p(x' | \mathbf{y}) \: dx' \approx \sum_{i = 1}^n f(\mathbf{x}_i) w(\mathbf{y}, x_i)
\end{align}


### The Weighting Function

Assuming that our forward model simulations \((\mathbf{y}_i, x_i)\) are exact up to
a zero-mean, Gaussian error with covariance matrix \(\mathbf{S}_e\), the weights are given
by

\begin{align}
  w(\mathbf{y}, x_i) = \frac{1}{C} \cdot \exp \left \{ 
  - \frac{(\mathbf{y} - \mathbf{y}_i)^T \mathbf{S}_e^{-1} (\mathbf{y} - \mathbf{y}_i)}
    {2} \right \}
\end{align}

for some normalization factor \(C\). The normalization factor is found to be 
\(C = \sum_{n = 1}^n w(\mathbf{y}, x_i)\) to ensure that
\(\mathcal{E}_{x|\mathbf{y}}\{1\} = 1\).


### The Retrieval

The above approach can be used to retrieve various statistics of the posterior
distribution. The most basic are the mean and the variance:

\begin{align}
 \bar{x} = \mathcal{E}_{x | \mathbf{y}} \{ x \} & \approx \sum_{i = 1}^n w(\mathbf{y}, x_i) x_i \\
\text{var}(x) = \mathcal{E}_{x | \mathbf{y}} \{ (x - \bar{x})^2 \} & \approx 
 \sum_{i = 1}^n w(\mathbf{y}, x_i) (x_i - \mathcal{E}_{x | \mathbf{y}}\{x\})^2
\end{align}

But it is even possible to approximate the cumulative distribution function of the
a posteriori distribution using:

   \begin{align}
F_{x | \mathbf{y}}(x') &=  \int_{-\infty}^{x'} p(x) \: dx \\
                       &= \mathcal{E}_{x | \mathbf{y}} \{ \mathbf{I}_{x < x'} \} \\
                       &\approx \sum_{x_i < x'} w(\mathbf{y}, x_i)
   \end{align}


## Machine Learning

Given the database, the simplest way of setting up an IWP retrieval
is probably by learning the inverse method \(x = R(\mathbf{y})\) from
the data. This can be done using machine learning methods. For regression,
machine learning methods are trained in a *supervised manner*, that is
by minimizing a given loss function over a training set.

The training set in this case will be the simulation database
\(\{(\mathbf{y}_i, x_i)\}_{i = 1}^n\). This nomenclature is a bit unfortunate
because in machine learning the input is usually denoted by \(\mathbf{x}\) and
the output to learn by \(y\).

For regression the most commonly used loss function is the mean squared error loss.
Statistically, this may be seen as training a maximum likelihood estimator
of the mean of a conditional Gaussian distribution. While this perspective would
even allow us to treat the retrieval problem in a Bayesian way, we will not pursue
this statistical interpretation here.


### Neural Network 101

Neural networks are a general computing model that compute a vector of
output activations \(\mathbf{y}\) from an input vector \(\mathbf{x}_\) by
propagating the input activations through a sequence \(i = 1, \ldots, n\) of layers with
associated learnable weight matrices \(\mathbf{W}_i\)  and bias vectors \(\mathbf{b}_i\):

\begin{align}
    \mathbf{x}_0 &= \mathbf{x}\\
    \mathbf{x}_i &= f_{i}
    \left ( \mathbf{W}_{i} \mathbf{x}_{i - 1}+ \mathbf{b}_i \right ) \\
    \mathbf{y} &= \mathbf{x}_{n}
\end{align}

The \(f_i\) s  here are the activation functions of each layer. Non-linear activation
functions allow the network to learn complex, non-linear mappings from the input
vector \(\mathbf{x}\) to the output \(\mathbf{y}\).

Neural networks are trained by finding the weight matrices \(\mathbf{W}_1\),
\ldots \mathbf{W}<sub>n</sub>$ and bias vectors \(\mathbf{b}_1, \ldots, \mathbf{b}_n\)
that minimize the mean of a given loss function
\(\mathcal{L}(\hat{\mathbf{y}}, \mathbf{y})\) over a training set. All
commonly used training algorithms are based on a technique called
*backpropagation* to compute the gradients of the training loss with respect
to the weights and biases of each layers. These gradients are then used to
update the learnable parameters. For large datasets, stochastic (batch)
gradient descent (SGD) is usually a good algorithm to start with.
In each training step, SGD computes gradients on randomized subsets of the
training set and uses them to update the weights.

While this exercise is clearly not the right place for a complete introduction to neural
networks, modern machine learning packages usually only require you to provide
training data and choose the loss function and training method, so this is hopefully
enough to get you started.


### Other Machine Learning Methods

Even though neural networks are a pretty hot topic right now, there are many other machine
learning methods that might perform just as good especially on regression tasks and
moderately sized data sets. Some examples that might be worth considering are:

-   regression trees and forests
-   boosted regression trees and boosting in general
-   support vector machines


# Exercises

The simulation database for this exercise consists of 350000 pairs \((\mathbf{y}_i, x_i)\)
of simulated brightness temperatures \(\mathbf{y}_i\) and corresponding  ice water path
values \(x_i\). Each observation vector \(\mathbf{y}_i\) consists of the brightness temperatures of
channels \(8, 9, 10, 11, 12, 13\) of the GMI radiometer. For this exercise we will assume that the
only uncertainty in our simulation database is due to thermal noise in the receiver.

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-right" />

<col  class="org-left" />

<col  class="org-left" />

<col  class="org-right" />
</colgroup>
<thead>
<tr>
<th scope="col" class="org-right">Channel</th>
<th scope="col" class="org-left">Center freq \([GHz]\)</th>
<th scope="col" class="org-left">Polarization</th>
<th scope="col" class="org-right">NEDT (K)</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-right">8</td>
<td class="org-left">\( 89\)</td>
<td class="org-left">V</td>
<td class="org-right">0.32</td>
</tr>


<tr>
<td class="org-right">9</td>
<td class="org-left">\( 89\)</td>
<td class="org-left">H</td>
<td class="org-right">0.31</td>
</tr>


<tr>
<td class="org-right">10</td>
<td class="org-left">\(166\)</td>
<td class="org-left">V</td>
<td class="org-right">0.7</td>
</tr>


<tr>
<td class="org-right">11</td>
<td class="org-left">\(166\)</td>
<td class="org-left">H</td>
<td class="org-right">0.65</td>
</tr>


<tr>
<td class="org-right">12</td>
<td class="org-left">\(183.31 \pm 3\)</td>
<td class="org-left">V</td>
<td class="org-right">0.56</td>
</tr>


<tr>
<td class="org-right">13</td>
<td class="org-left">\(183.31 \pm 7\)</td>
<td class="org-left">V</td>
<td class="org-right">0.47</td>
</tr>
</tbody>
</table>


## Data

The data required to solve this exercise is provided both as `numpy` arrays and
MATLAB arrays.

-   `y_database`: \(35000 \times 6\) array containing the simulated brightness temperatures
    for the retrieval databse.
-   `iwp_database`: \(35000 \times 1\) array containing the IWP values corresponding
    to the simulated brightness temperatures in `y_database`.
-   `y_validation`: Additional simulated brightness temperatures to test the retrieval.
-   `iwp_validation`: IWP values corresponding to the brightness temperatures in
    `y_validation`.
-   `gmi_tbs_0, gmi_tbs_1`: Data from two different GMI orbits containing observations
    of the tropical storm Saola. The observations displayed below show orbit 0, but feel
    free to use orbit 1 as well.


## BMCI

1.  The Database

    Plot the distribution of ice water path values in the database. What is the range
    of IWP values? What is the reason for the bimodal character of the distribution?

2.  Basic Implementation

    Write a function
    
        bmci(y_database, x_database, s_o, y)
    
    where
    
    -   `y_database`: Matrix containing the simulated observations along its rows.
    -   `x_database`: Vetor containing the corresponding IWP values (or any other quantity
        you may want to retrieve).
    -   `s_o`: Matrix containing the covariance matrix \(\mathbf{S}_o\) describing
        the observation uncertainty.
    -   `y`: The observations for which to retrieve the ice water path. Given either
        as a vector (for a single inversion) or as a matrix with the observations along
        its rows.
    
    The method should return two vectors containing the expected values and standard
    deviations of the posterior distributions corresponding to the observations given
    in `y`.

3.  Error Analysis

    Compute and plot the *mean absolute precentage error* (MAPE)
    
    \begin{align}
     MAPE = \frac{100\%}{n} \sum_{i = 1}^n \frac{|\hat{x}(\mathbf{y}_i) - x_i|}{x_i}
    \end{align}
    
    as a function of \(x_i > 0\) for the simulated measurements \(\mathbf{y}_i, x_i\)
    contained in the arrays `y_validation` and `iwp_validation`.
    
    Compute and plot also the mean of the  estimated standard deviation of the posterior
    distribution as a function of \(x_i\).
    
    What does this tell you about the retrieval?

4.  Retrieving the Posterior CDF

    Write a function `bmci_cdf(y_database, x_database, s_o, y)`, that
    retrieves the cumulative distribution function of the posterior for
    a single observation `y`.
    
    The CDF for the 30000th (0-based indexing!) database entry should look
    like this:
    
    ![img](./doc/plots/cdf_example.png "Posterior CDF for entry 30000 in the database.")
    
    Given the CDF of the posterior what would be your *best estimate* if you
    had to return a single IWP value as the retrieval? How does this
    compare to the expected value for the validation data set?

5.  Apply your Retrieval

    The file `data/tbs_gmi` contains the observerd calibrated brightness temperatures
    from the (extra-)tropical storm Saola as it tracked southeast of Japan 2017-10-27.
    
    ![img](./doc/plots/saola_overview.png "The tropical storm Saola seen from Modis and GMI.")
    
    Use your retrieval to retrieve the IWP path from the brightness temperatures. The
    functions `plot_modis_image` and `plot_gmi_swath` are provided to display the
    MODIS RGB and your results on a map.
    
    In addition to the expected value, plot also the median and the 
    \(10\text{th}\) percentile as a lower bound for the ice water path.


## Machine Learning Methods

In this part of the exercise you should use your
favorite machine learning regression method to build an
alternative IWP retrieval and compare it to the BMCI retrieval.

In case you are unsure what to pick, two method that should work relatively
well more or less right away are *neural networks* or *regression trees*.

If you're using python, you may have a look at the *scikit-learn*
examples for [(boosted) decision trees](http://scikit-learn.org/stable/auto_examples/ensemble/plot_adaboost_regression.html) or [neural networks](http://scikit-learn.org/stable/modules/neural_networks_supervised.html).

For MATLAB examples can be found [for neural networks](https://se.mathworks.com/help/nnet/examples/train-a-convolutional-neural-network-for-regression.html) and [regression trees](https://se.mathworks.com/help/stats/regressiontree-class.html),
as well.

1.  Comparison to BMCI

    Plot the MAPE of your machine learning retrieval and compar to the results obtained
    using BMCI.


# Footnotes

<sup><a id="fn.1" href="#fnr.1">1</a></sup> Simulations performed and kindly provided by Bengt Rydberg.

<sup><a id="fn.2" href="#fnr.2">2</a></sup> Eliasson, S., S. A. Buehler, M. Milz, P. Eriksson and V. O. John
Assessing observed and modelled spatial distributions of ice water path
using satellite data

<sup><a id="fn.3" href="#fnr.3">3</a></sup> <http://www.icare.univ-lille1.fr/projects/dardar>
