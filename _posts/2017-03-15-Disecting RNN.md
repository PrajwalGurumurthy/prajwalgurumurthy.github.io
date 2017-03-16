---
layout: post
title: Dissecting the RNN[Recurrent Neural Network]
description: The RNNs are magical in a way they work. This post dissects the internal structure of the neural computation graph of RNNs and tries to give an intuition of how neurons tune themselves to understand sequences in data.
---

>RNN

Regular Neural Networks perform exceptionally well in understanding complex non linear data.There are certain requirements for a different class of applications
where it is necessary to keep track of past information. For instance Image captioning, Language translation, Language modelling, Sentiment analysis,
Video processing and other similar use cases. Regular neural networks lack the ability to capture past/sequence information from data. 
RNNs are built to solve this exact problem. RNNs have the ability to understand sequences in data and predict based on current input plus the previous state.

Refer the following blogs on RNNs to get an in depth and simple understanding of RNNs.

[The Unreasonable Effectiveness of Recurrent Neural Networks by Andrej Karpathy](http://karpathy.github.io/2015/05/21/rnn-effectiveness/)<br>
[Understanding LSTM Networks](http://colah.github.io/posts/2015-08-Understanding-LSTMs/)

<p><i>Note: This remainder section assumes that you have a basic understanding of RNNs</i></p>

You flow through different stages of emotions while working with RNNs.

{% highlight code %}
Stage1: WOW!!
Stage2: WTF!!
Stage3:	Aha!!
{% endhighlight %}

>Its OK if you are in the Stage2.

I personally spent a lot of time in Stage2 trying to get the intuition of WTF is happing inside the network that it is able to understand
sequences in data. But once you get that aha!! moment you will realise how simple RNNs are. In fact the simple RNNs(Vanilla RNNs) are very similar
to the vanilla neural networks except that the hidden states are computed in a different way. The hidden state function is what gives the RNN
the ability to store previous states information and use it along with the current state to predict output.

>Questions that haunted me

{% highlight text %}
How does the gradients flow backwards through time?<br>
How are the weights updated backwards through time?<br>
What information should be maintained while forward propagating through time?
What information should be maintained while backward propagating backwards through time?
{% endhighlight %}

<p><b>I found answers to all the questions once I started writing computations graph through time during forward propagation and then
did a backward propagation backwards through time. That is when you realise how RNNs are a bit different from normal Vanilla NN</b></p>


>Dissection

Before we dissect the RNN lets start by dissecting the vanilla NN so that its much easier to understand the difference between them.

<br>For the sake of example lets consider 3:1:1 neural structure with i/p,hidden and o/p respectively.
We will use RELU as activation functions in hidden units,sigmoid in the output layer and log loss function.

>Lets get the color coding of the graph right



<b>Dissecting the Vanilla Neural Network</b>




