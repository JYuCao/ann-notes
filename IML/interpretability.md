## 1 Interpretability

* Definition: **Interpretability** is the degree to which a human can understand the cause of a decision.

* Interpretable method: A method is **interpretable** if a user can correctly and efficiently predict the method's results.

* Interpretability vs Explainability

    * **Interpretability** is about mapping an abstract concept from the models into an understandable form.

    * **Explainability** is a stronger term requiring interpretability and additional context.


## 2 Importance of Interpretability*

* human curiosity and learning
* find meaning in the world
* the goal of science
* safety measures
* detecting bias
* increase social acceptance
* manage social interactions
* debug and audit

## 3 situations where interpretability is not important

* the model has no significant impact
* the problem is well studied
* system might be manipulated through interpretability

## 4 Human cognition and good interpretability🌟🌟🌟

> As an explanation for an event, humans prefer short explanations (only 1 or 2 causes) that contrast the current situation with a situation in which the event would not have occurred. Especially abnormal causes provide good explanations. Explanations are social interactions between the explainer and the explainee (recipient of the explanation), and therefore the social context has a great influence on the actual content of the explanation.

> This section is mostly summarized from the paper **"Explanation in AI: Insights from the Social Sciences" by Miller (2017)**

* Explanations are contrastive
    * Humans tend to use **contrastive** explanations, which explain why an event occurred instead of another event. 

* Explanations are selected
    * Selecting one or two causes from a variety of possible causes as THE explanation, instead of providing a complete list of all causes.

* Explanations are social
    * Explanations are the interaction or conversation between **the explainer and the explainee**(the recevier of the explanation).
    * The social context determines the content and nature of the explanations.

* Explanations focus on the abnormal
    * Humans tend to focus on the **abnormal or unusual** causes when receiving explanations, as they are more likely to be seen as significant or relevant to the event being explained.

* Explanations are truthful
    * Good explanations prove to be true in reality (i.e., in other situations).
    * This is not the most important factor for a "good" explanation. (e.g., Selectiveness seems to be more important than truthfulness)
    * **Fidelity**: the degree to which an explanation accurately reflects the true underlying causes of an event *for machine learning models*.

* Good explanations are consistent with prior beliefs of the explainee
    * **Confirmation bias**: A effect that people tend to favor information that confirms their existing beliefs and ignore or discount information that contradicts those beliefs.
    * If this attribute is integrated into machine learning models, it would probably drastically compromise the performance.

* Good explanations are general and probable
    * Abnormal causes beat general causes. Only in the absence of an abnormal event, a general explanation is considered a good explanation.
    * General explanations can be applied to a wide range of situations.
    * Generality can easily be measured by the feature’s support, which is the number of instances to which the explanation applies divided by the total number of instances.


## 5 Reflections
### 5.1 Proxy features can make models gameable
> The system can only be gamed if the inputs are *proxies* for a causal feature but do not actually cause the outcome. Whenever possible, proxy features should be avoided as they make models gamable.

example: A system may use whether a person wearing shoes is a proxy for whether they are going to a gym, which in turn is a proxy for their health. However, wearing shoes does not cause good health, and thus the system can be gamed by simply wearing shoes without actually going to the gym or improving health.

* proxy: a variable that is correlated with the outcome but does not cause it.

I think this can be transferred to human cognition as well(maybe it originally comes from human cognition). If a human has cognitive bias, they may rely on proxy features to make decisions, which can be gamed by others who understand these biases.

### 5.2 People tend to misjudge probabilities of joint events
> Also remember that people tend to misjudge probabilities of joint events. (Joe is a librarian. Is he more likely to be a shy person or to be a shy person who likes to read books?)

This sentence is interesting but I have little understanding of it.

### 5.3 People can be gamed by default from birth
People tend to prefer contrastive explanations. This make me think: when a human is born, they have no prior beliefs or knowledge about the world. How is he able to understand the world and make decisions? 

I use ChatGPT to answer this question. ChatGPT does not give me a certain answer, but it tells me that humans will build a *simple and fuzzy* model of the world when they are young, which means everyone can be gamed by others by default. However, as they grow up, they will update model based on their experiences and interactions with the world. 

