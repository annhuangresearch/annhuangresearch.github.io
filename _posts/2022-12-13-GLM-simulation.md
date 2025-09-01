---
title: "Understanding generalized linear models through data simulation (under construction)"
layout: post
date: 2022-12-13
tag:
- tutorial
- statistics
- modeling
category: blog
author: annhuang
description: Simulating data to understand the data generation process
---

## Outline
- [Summary](#summary)
- [Background](#background)
- [Preparation](#preparation)
- [Creating a dataframe](#creating-a-data-frame)
- [Compute the logits](#compute-the-logits)
- [Fit the GLM](#fit-the-glm)

## Summary

In this tutorial, I present a guide on how to simulate synthetic or fake data from a simple visual perceptual experiment, which is commonly used in psychology and neuroscience studies. The goal is to illustrate how the practice of synthetic data simulation can be useful for understanding the data generation process. This helps us better understand how statstical models work and how to interpret or explain the regression output.

---

## Background

Many questions in psychology and neuroscience are about how people respond to sensory information. For example, did the badminton shuttle land inside, outside or on the court line? Is that a cat or a tiger hiding behind the bushes? Is that my car key in the container of keys?

These types of questions deal with **detection of signals**, which also uncovers the process of how we convert everyday sensory inputs into a behavioral response like making a decision.

To answer these questions, researchers often use "psychophysics" or "perceptual" experiments to **model the relationship between physical stimuli and the subjective sensation**. Typically in these perceptual experiments, a human observer is presented with a simple visual stimulus e.g., a cloud of dots moving either predominately to the right or to the left (See Figure 1). The observer is asked to discriminate the direction of the motion by making a speeded choice response between the two alternatives, left or right. For some, the task is easy yet for others, it is not easy. Thus, each person's sensitivity to the same sensory stimulus varies.

An important side note: since I am interested in the social aspect, I conducted a **dyadic perceptual experiment** in which two people took turn to respond to the stimulus. In particular, I am interested in how the previous choice response (or choice history) influence the decision-making i.e., how does my previous response predict what I choose next when I know that my partner is observing? Specifically, on each trial, one member of a dyad is selected to respond to the stimulus. Overall, there are 10 blocks, each block has 100 trials and the entire experiment took around 3 and 1/2 hours. This dual-participant setup allowed me to better understand how we make decisions knowing that we are being observed by other people, just like in real life.

Here, we perform statistical modeling to infer the relationship between what people chose and the variables that are believed to have influenced what people chose. Note that a statistical model is simply using math to represent the relationship between two variables. Because people only made binary choices (left or right) in the experiment, this simplifies the solution to the problem a bit. Specifically, we can use a **generalized linear model (GLM)** with a logit link function to explain the likelihood that people choose left or right based on specific sets of predictors, such as the stimulus direction, what the previous choice was or what the previous stimulus direction was.

To fit the statistical model to the observed data, we can use the R programming language since it provides an extensive library for a wide-ranging statistical analyses. Fitting a statistical model is easy and straightforward because it only requires typing one or maximum two lines of code.

However, as a student starting out to learn how these statistical models work, sometimes we do not always understand exactly what the model is doing or how to really interpret the results. For example, what does a regression coefficient of 1.2 actually mean? How does effect coding versus dummy coding for the variables (−1 vs. +1 instead of 0 vs. 1) change the interpretation?

The analogy is like moving into a new apartment and using the appliances: you know how to turn on the induction stove in the kitchen; yet one day, suppose when the stove does not get turned on, why is the stove behaving this way? Which of the electrical wiring is at fault? In case like this, we may appreciate a DIY kitchen installation experience because through building the kitchen, we understand exactly the mechanism of how the stove is turn on or off.

This is how the practice of synthetic data simulation is helpful. By generating synthetic datasets where we set the **"true" parameters**, we can see how GLM estimation works in practice, check whether we can recover the parameters, and build intuition about model behavior.

In this post, I will walk through an R script that simulates a perceptual task. We will first generate some synthetic data, then fit a GLM, and assess the output of the model. Specifically, we will compare the model's estimation to the values we set at the start.

## Goal

- **Goal**: To simulate a synthetic data frame that replicates a simple perceptual decision-making experiment, define the coefficients values, and run the `glm()` to recover the logistic regression coefficients. 
- **Purpose**: To understand the data generation process, the coefficients in relation to the model, and explain the regression output. 
- **Specification of the logistic regression model**:  
  $$
  Pr(Y=1) = \text{logit}^{-1}(\beta_{0} + \beta_{1}X)
  $$
- More specifically:
  $$
  Pr(\text{Response} = 1) = \text{logit}^{-1}\big(\beta_{0} + \beta_{1}\cdot\text{Direction} + \beta_{2}\cdot\text{PrevDirection}\big)
  $$
- Coding of the observed choice response:  
  $$
  \text{Response} =
  \begin{cases}
   1 & \text{right} \\\\
   0 & \text{left}
  \end{cases}
  $$
- Everything else like the actor information, stimulus direction are effect coded (1 and -1)

## Preparation

- Load pacakges and set seed for reproducibility

- The objective of doing a data simulation exercise is to understand how different choices in the simulation design and model specification can affect the inferences that are drawn from the data. The intention is not to perfectly match the data, but to understand how different choices in the design can affect the results.

```{r}
# Remove everything
# Load necessary packages
# tidyverse for data wrangling
# performance for model evaluation (e.g., R² measures)

rm(list=ls())
library(tidyverse)
library(performance)

# set seed for reproducibility
set.seed(999)
n = 10000
```

Now according to the model specification, we need the inverse logit, which is: $$\frac{e^p}{1 + e^p}$$
This function takes the systematic component of the model and translates it into probabilities varying between 0 and 1. Then we need to define the ground truth values for the model parameters.

```{r}

# Create the inverse logit function
inv.logit <- function(p){
  return(exp(p)/(1 + exp(p)))
  }

# Define the true parameters

b0 <- 0
b1 <- 1.3  # stimulus direction, or "stimulus_n"
b2 <- 0.09 # previous direction of the stimulus
b3 <- 0.8  # previous response, "response_n_1"
b4 <- 0.8  # previous trial performer's identity, "participant_n_1"

```
## Creating a data frame

In this step, we think about what are the elements that make up the experiment? For instance, the experiment has a stimulus and the stimulus elicits the observer's responses. The experiment also consists certain number of blocks, and in each block there are certain number of trials. There are the participants who sit in different rooms, etc. The point here is that we should think about the design of the experiment, and simulate variables that code for each of these information.

```{r}
# create block, trial, room to match the real experiment data
block <- rep(1:10, size = n, each = 100)
trial <- rep(1:100, size = n, times = 10)
room <- rep(c("room_1", "room_2"), times = n/2, each = 1)

# create direction of the trial stimulus, called "stimulus_n"
stimulus_n <- sample(c(-1,1),replace = TRUE, size = n, prob = c(0.5, 0.5))
table(stimulus_n) # check; -1: 496, 1: 504. looks ok

# create previous direction of the stimulus, "stimulus_n_1"

stimulus_n_1 <- sample(c(-1,1),replace = TRUE,
                             size = n, prob = c(0.5, 0.5))

# create response variable, i.e. the actual observed left or right response
# (coded in 0s and 1s, respectively) during the task

# NOTE: this is generated independently from the "outcome",  we will need to overwrite
# this variable later with a new one that is generated from the
# "outcome" variable later, as this would ensure the purpose of the simulation study
# that the "outcome" variable is the underlying probability of a particular
# response, which is determined by the model and the parameters.

# the purpose of generating the "outcome" variable is to be able to compare
# it to the "response" variable and see how well the model is able to
# predict the observed data. therefore they need to be related as such

response <- sample(c(0, 1), size = n, replace = TRUE, prob = c(0.5, 0.5))
table(response) # check, "0":481; "1": 519. looks ok

# create the variable "participant_n_1" to indicate previous trial performer's identity
participant_n_1 <- rbinom(n = n, size = 1, prob = 0.5)
participant_n_1 <- ifelse(participant_n_1 == 1, 1, -1) # effect code: 1 is own
table(participant_n_1) # looks ok

# put into a data frame called sim_df
# I want to do this before lagging that produces previous response, "response_n_1"
# simply to match how I processed the real experiment data

sim_df <- data.frame(block, trial, chamber, stimulus_n, stimulus_n_1,
                     response, participant_n_1)

# create the response_n_1 variable in sim_df
sim_df["response_n_1"] <- lag(sim_df$response)
#response_n_1 <- sim_df["response_n_1"]

# effect code
sim_df$response_n_1 <- ifelse(sim_df$response_n_1 == 1, 1, -1) # 1 is right
# response_n_1 <- sim_df$response_n_1

# omit the NA value from lagging
sim_df <- na.omit(sim_df)
```

## Compute the logits

Now, we compute the logits to to generate a binomial distribution for the choice response by sampling from the outcome

```{r}
# compute the logits
xbs_all <- b0 + b1*stimulus_n + b3*(sim_df["response_n_1"]) + b4*participant_n_1

# generate a binomial distribution for the response by sampling from the outcome
# probability distribution
outcome <- inv.logit(xbs_all) # the simulated response to be modeled

# outcome is a data frame, but the mean function expects a numeric vector.
# convert the outcome data frame to a numeric vector, then take the mean.
outcome <- unlist(outcome)
mean(outcome) # check if bias;
              # on average, the probability of the response being 1 is about 0.511
```

This is the new "response" variable, sampled from the outcome distribution. This will overwrite the old "response" variable. The relationship between the response and outcome variable is that the response is a sample from a binomial distribution with probability equal to the outcome variable i.e, if the outcome variable is high, the response variable should have more 1's and if the outcome variable is low, the response variable should have more 0's.

If the response variable is not correctly related to the outcome variable, then the model will not be correctly estimating the relationship between the predictors and the response.

```{r}
response <- rbinom(n = n, size = 1, prob = outcome)

summary(response)
table(response)
any(is.na(response))
sum(is.na(response))
table(response, useNA = "always")
#na.omit(response)

# one issue would arise: the new response variable you generated has one more 
# observation than the original response variable in the sim_df dataframe.
# remove the last observation of the new response variable before overwriting
# the existing response variable in sim_df with the new response_new variable
# check again the dataframe to make sure the replacement was done correctly

response_new <- tail(response, -1)
sim_df$response <- response_new # overwrite the old response

# direction and response needs to be in the same coding format
# such that later when we compare to obtain the correctness/accuracy, it will work

stimulus_n_binary <- ifelse(sim_df$stimulus_n == -1, 0, 1)

# add a correct column to the dataframe
sim_df["correct"] <- sim_df$response == stimulus_n_binary

mean(sim_df$correct) # adjust b1 to check people's accuracy
#mean(sim_df$response)
```

## Fit the GLM

- Run ```glm()``` to see how well it recovers the known parameters.

```{r}
# mod_with <- glm(response ~ direction + previous_direction,
#                 data = sim_df_new, family = "binomial")
# summary(mod_with)
# r2_tjur(mod_with)
# 

# mod_without <- glm(response ~ previous_direction,
#                    data = sim_df_new, family = "binomial")
# summary(mod_without)
# r2_tjur(mod_without)
response_n_1 <- sim_df$response_n_1

mod_with_prev <- glm(response ~ stimulus_n + response_n_1, 
                data = sim_df, family = "binomial")
summary(mod_with_prev)
r2_tjur(mod_with_prev)
with(summary(mod_with_prev), 1 - deviance/null.deviance) # mcFadden's R^2

mod_with_prev$coefficients <- list("Intercept"=10, "stimulus_n"=15, "response_n_1"= 0.02)
r2_tjur(mod_with_prev)

mod_with_prev

```