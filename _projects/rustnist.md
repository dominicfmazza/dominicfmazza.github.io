---
layout: page
title: rustnist
description: Barebones MNIST detection in Rust using only linear algebra
importance: 1
img: assets/img/rustnist/mnist.png
category: fun
---

## Overview

This project, [rustnist](), takes inspiration from [this video](https://youtu.be/w8yWXqWQYmU) and the corresponding kaggle notebook.
Originally, I endeavored to reimplement the network in rust in a similar fashion, but over time I realized two things:

1. The video and the corresponding kaggle notebook did not correctly implement a classifier that works when I tested it
2. This would be a unique opportunity to further my understanding of both machine learning fundamentals and rust fundamentals

After realizing those two things, my new goal became learning and implementing a simple neural network from scratch,
and to accomplish this I rewrote most algorithms in the network and made the program more modular. I found this process
to be highly rewarding, as it forced me to understand each and every component of my simple one-layer network. This allowed me
to learn quite a few things, notably:

- The implementation of softmax and its uses
- The implementation of ReLU and Leaky ReLU
- Weight initialization best practices for different activation functions
- The underlying linear algebra of a neural network

Overall, this was a stimulating project that taught me how simple neural networks work at a low level and how to debug a
malfunctioning rust codebase, and additionally gave me a chance to try out Rust and its ndarray and random functionality.

## Implementation

In terms of implementation, I heavily leaned into the use of rust structs and traits, as I really wanted to get my feet wet with
the language.

### Model

The "Model" object that I developed was an abstraction that allowed me to define all the layers needed for my network. It contained
the layers I defined myself. It consisted of one hidden layer using a RELU activation, and an output layer using a softmax activation
for classification.

```rust
// 2 layer neural network, with accuracy vector for tracking performance
pub struct Model {
    // dataset struct holding full data and slices
    dataset: Dataset,
    // ReLU layer
    hidden_layer: ReLU,
    // Softmax layer
    output_layer: Softmax,
    // 10 x 2 vector for tracking accuracy
    accuracy: Array2<f32>,
}
```

I specifically built my layer functionality to allow for easy forwards and backwards propogation between layers, as well as
updating parameters.
This definitely was an interesting challenge, as I wasn't quite used to rust's particularly flavor
of ownership, but once I got it to function as intended, it was quite easy to see why rust went with
the borrowing paradigm.

```rust
// forward propogration function, mostly handled in layer
// configuration: specifies whether or not network is training
fn forward_prop(&mut self, configuration: CONFIG) {
  match configuration {
      CONFIG::TRAIN => {
          self.hidden_layer
              .forward_prop(&self.dataset.train_data_slice);
          self.output_layer.forward_prop(&self.hidden_layer.layer);
      }
      CONFIG::TEST => {
          self.hidden_layer
              .forward_prop(&self.dataset.test_data_slice);
          self.output_layer.forward_prop(&self.hidden_layer.layer);
      }
  }
}

// backwards propogation function, mostly handled in layer
fn backward_prop(&mut self) {
  self.output_layer
      .backward_prop(&self.dataset.train_label_slice, &self.hidden_layer.layer);
  self.hidden_layer
      .backward_prop(&self.output_layer.layer, &self.dataset.train_data_slice);
}

// updating of weights and biases
fn update_params(&mut self) {
    self.output_layer.layer.update_params();
    self.hidden_layer.layer.update_params();
}
```

I also implemented training and testing in the model source, as that made the most sense.

```rust
// train the network, and print accuracy every 10 epochs
pub fn train(&mut self, epochs: usize) {
    for i in 0..epochs {
        self.dataset.shuffle();
        for _ in
            0..(self.dataset.training_data.layer.ncols() / self.dataset.slice_range as usize)
        {
            // get new dataset slice
            self.dataset.set_slice(CONFIG::TRAIN);
            // forward
            self.forward_prop(CONFIG::TRAIN);
            // calculate gradients
            self.backward_prop();
            // update
            self.update_params();
            // tally accuracy for batch
            self.set_accuracy(
                self.get_predictions(),
                self.dataset.train_label_slice.layer.clone(),
            );
        }
        if i % 10 == 0 {
            // print accuracy
            println!("\n\n-----------------------------");
            println!("Total Epochs: {}", i);
            println!("Accuracy: {}", self.get_accuracy(),);
        }
        // reset accuracy for next epoch
        self.accuracy = Array2::<f32>::zeros((10, 2));
    }
}

// test network on separate data to cross-validate
pub fn test(&mut self) {
    println!("\n\nTESTING NETWORK");
    self.accuracy = Array2::<f32>::zeros((10, 2));
    for _ in 0..(self.dataset.testing_data.layer.ncols() / self.dataset.slice_range as usize) {
        // set slice from testing net
        self.dataset.set_slice(CONFIG::TEST);
        // forward
        self.forward_prop(CONFIG::TEST);
        // tally accuracy for batch
        self.set_accuracy(
            self.get_predictions(),
            self.dataset.test_label_slice.layer.clone(),
        );
    }
    // print testing accuracy
    print!("Accuracy: {}", self.get_accuracy(),);
}
```

Besides what's shown, the model also handles getting the classification predictions and testing overall accuracy.
For accuracy measurement, I simply compared how many digits were correctly detected per class when compared to
the overall amount of ground truth samples in the dataset.

### Layers

The layer functionality is honestly what was the most extensible in this project. With each layer of the network, I wanted
to abstract away the actual activation of each layer, allowing me to extend the base layer functionality for any activation I choose.

Each layer struct has access to basic layer components, which are all simply just 2D arrays, a learning rate, and a batch size.
I chose to train the network based on flattened data as opposed to the original 2D images, as it made for a network that was
computationally simpler to run by making batch training 2-dimensional.

```rust
/ struct for a layer (loosely defined)
#[derive(Clone)]
pub struct Layer {
    // nodes before activation
    pub preactivation: Array2<f32>,
    // nodes after activation
    pub layer: Array2<f32>,
    // derivative of nodes and their affects on network performance
    pub d_activation: Array2<f32>,
    // weights for each connection in the network
    pub weights: Array2<f32>,
    // derivative of weights
    pub d_weights: Array2<f32>,
    // bias terms for each node
    pub biases: Array2<f32>,
    // derivative of bias terms
    pub d_biases: Array2<f32>,
    // learning_rate of network
    alpha: f32,
    // amount of samples for each forward and backwards pass
    samples: usize,
}
```

For forwards propogation, it was pretty straightforward. Calculate the dot product of the current layer's weights with
the previous layer, and add bias. For backwards propogation, I just simply calculated the layer's weights and biases as if the layer
had already been differentiated.

```rust
// calculate preactivation layer for activating by activation layer
pub fn forward_prop(&mut self, previous_layer: &Layer) {
    self.preactivation = &self.weights.dot(&previous_layer.layer) + &self.biases;
}

// calculate derivative of weights and biases based on derivative of the activation
pub fn backward_prop(&mut self, previous_layer: &Layer) {
    self.d_weights = self
        .d_activation
        .dot(&previous_layer.layer.t())
        .map(|x| x * 1.0 / self.samples as f32);
    self.d_biases = self
        .d_activation
        .sum_axis(Axis(1))
        .map(|x| x * 1.0 / (self.samples as f32))
        .insert_axis(Axis(1));
}

// update weights and biases
pub fn update_params(&mut self) {
    Zip::from(&mut self.weights)
        .and(&self.d_weights)
        .for_each(|a, b| *a -= self.alpha * *b);
    Zip::from(&mut self.biases)
        .and(&self.d_biases)
        .for_each(|a, b| *a -= self.alpha * *b);
}
```

I also added some funcitonality to create a one-hot representation of a layers output for classification,
as well as a function that created a dummy layer to just get the outputs.

The real meat and bones of the layer functionality comes from the simplicity of rust's traits. This allowed me
to implement each layers activation and deactivation separately, and therefore create a layer with any activation.

```rust
pub trait ActivationLayer {
    fn activate(&mut self);
    fn deactivate(&mut self, previous_layer: &Layer);
}
```

#### ReLU Activation

For the singular hidden layer, I chose a leaky relu activation. While simple in nature, ReLU
is one of the most common methods of activation in a neural network, and it proved quite
effective in this case

```rust
// implementation of relu with option to make it leaky
pub struct ReLU {
    pub layer: Layer,
    relu_coefficient: f32,
}

impl ActivationLayer for ReLU {
    // activate using relu piecewise function
    fn activate(&mut self) {
        for ((i, j), item) in self.layer.preactivation.indexed_iter() {
            if *item < 0.0 {
                self.layer.layer[[i, j]] =
                    &self.layer.preactivation[[i, j]] * &self.relu_coefficient;
            } else {
                self.layer.layer[[i, j]] = self.layer.preactivation[[i, j]];
            }
        }
    }
    // calculate gradient of the layer
    fn deactivate(&mut self, previous_layer: &Layer) {
        self.layer.d_activation =
            (&previous_layer.weights.t().dot(&previous_layer.d_activation)) * &self.derivate();
    }
}
```

#### Softmax Activation

For the output layer, I used a softmax activation as this is a classification network.

```rust
// implementation of softmax layer
pub struct Softmax {
    pub layer: Layer,
}

impl ActivationLayer for Softmax {
    // actual softmax function implementation
    fn activate(&mut self) {
        let mut row_sums = Array2::<f32>::zeros((1, self.layer.preactivation.ncols()));
        let mut out = Array2::<f32>::zeros(self.layer.preactivation.raw_dim());
        let mut m = f32::NEG_INFINITY;
        // get max in layer
        for item in self.layer.preactivation.iter() {
            if *item > m {
                m = *item;
            }
        }
        // subtract max from each element, and add it to sum of its row
        for ((i, j), item) in self.layer.preactivation.indexed_iter() {
            out[[i, j]] = (*item - m).exp();
            row_sums[[0, j]] += out[[i, j]];
        }
        // divide each element by sum of its row
        for ((i, j), _) in self.layer.preactivation.indexed_iter() {
            out[[i, j]] /= row_sums[[0, j]];
        }
        self.layer.layer = out;
    }

    // calculate derivative of activation
    fn deactivate(&mut self, previous_layer: &Layer) {
        self.layer.d_activation = (&self.layer.layer - &previous_layer.layer).map(|x| x * 2f32);
    }
}
```

#### Dataset

For the actual MNIST dataset, I extended the functionality of the mnist package that is available
via cargo, and made the dataset into layers itself. This allowed for the dataset to neatly fit into
the network itself.

```rust
// MNIST dataset converted to layers for interfacing with the model
pub struct Dataset {
    // Full dataset
    pub training_data: Layer,
    pub training_labels: Layer,

    // Full testing set
    pub testing_data: Layer,
    pub testing_labels: Layer,

    // Slice of training set
    pub train_data_slice: Layer,
    pub train_label_slice: Layer,

    // Slice of testing set
    pub test_data_slice: Layer,
    pub test_label_slice: Layer,

    // size of the slices, stored for access in other structs
}
```

The dataset struct has the ability to set and get slices based on whether or not I wanted training or testing data.
Additionally, I implemented a random shuffler of the data to minimize overfitting to the training set.

```rust
// function shuffles the training data and labels in the same order
pub fn shuffle(&mut self) {
    let mut rng = rand::thread_rng();
    let shared_length = self
        .training_data
        .layer
        .index_axis_mut(ndarray::Axis(0), 0)
        .len();
    assert_eq!(
        self.training_data.layer.ncols(),
        self.training_labels.layer.ncols()
    );
    for i in 0..shared_length {
        let next = rng.gen_range(0..shared_length);
        if i != next {
            let (mut index, mut random) = self
                .training_data
                .layer
                .multi_slice_mut((s![.., i], s![.., next]));
            std::mem::swap(&mut index, &mut random);

            let (mut index_2, mut random_2) = self
                .training_labels
                .layer
                .multi_slice_mut((s![.., i], s![.., next]));
            std::mem::swap(&mut index_2, &mut random_2);
        }
    }
}
```

### User Customization

Finally, I added the functionality to set different parameters for the network training.
This is done using rust's clap package, which allows easy command-line argument parsing.

```rust
// command-line parsing for hyperparameters
#[derive(Parser, Debug)]
#[clap(author, version, about, long_about = None)]
struct Args {
    /// # of nodes in the hidden layer
    #[clap(short, long, value_parser, default_value_t = 128)]
    layer_size: usize,
    /// Learning rate of the network
    #[clap(short, long, value_parser, default_value_t = 0.01)]
    alpha: f32,
    /// Batch size for BGD
    #[clap(short, long, value_parser, default_value_t = 100)]
    batch_size: isize,
    /// Amount of epochs to train for
    #[clap(short, long, value_parser, default_value_t = 1000)]
    epochs: usize,
}
```

## Conclusion

Overall, this project allowed me to fully learn how batch gradient descent works under the hood. Additionally,
I learned the best practices for multiple different components of networks that are usually handled by
pytorch/tensorflow/whatever library is typically used to create networks, such as:

- The dying ReLU problem and how leaky ReLU solves this
- The correct way to implement a softmax algorithm
- The importance of shuffling your training data

Additionally, it was fun!
