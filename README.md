<div align="center">
  <img width="50%" src="ReinforceLogo2.png"><br><br>
</div>

-----------------

Reinforcement Learning in R. Work in progress!

### Installation

```r
# install.packages("devtools")
devtools::install_github("markdumke/reinforcelearn")
```

### Get started
Reinforcement Learning with the package `reinforcelearn` is as easy as
```r
library(reinforcelearn)

# Create gridworld environment
t = gridworld$transitions
r = gridworld$rewards
env = makeEnvironment(transition.array = t, reward.matrix = r)

# Solve environment using Value Iteration
iterateValue(env)
```

### Create an environment

With `makeEnvironment` you can create a reinforcement learning environment, either from [OpenAI Gym](https://gym.openai.com/) or from the state transition matrix and reward matrix of a Markov Decision Process. The `makeEnvironment` will create an `R6 class`, from which you can access attributes about state and action space and use the `reset` and `step` methods to sample experiences from the environment.

To use a Gym environment you need to have the prerequisites installed: [gym-http-api](https://github.com/openai/gym-http-api)

```r
# Create an OpenAI Gym environment.
CartPole = makeEnvironment("CartPole-v0")

CartPole$reset() # returns initial state observation
# take random actions for 100 steps
for (i in 1:100) {
  action = sample(CartPole$actions, 1)
  CartPole$step(action)
}
CartPole$close()

# You can also create an environment from a MDP
grid = makeEnvironment(transition.array = windyGridworld$transitions,
  reward.matrix = windyGridworld$rewards,
  initial.state = 30L)
```

Environments created with `makeEnvironment` always have an initialization function to create a new instance of the environment, a reset function, which returns an initial state observation and a step function, which takes a step in the environment given an action returning the next state observation, reward and if the episode is finished.

### Run a reinforcement learning algorithm

After you created an environment you can use various reinforcement learning algorithms. For example, for a tabular environment like  gridworld you can use tabular Q-Learning to solve it and find the optimal action value function Q*. You can set various parameters like the learning rate, the number of episodes, the discount factor or epsilon, the ratio of random actions sampled by an epsilon-greedy behaviour policy.

```r
res = sarsa(grid, n.episodes = 1000, seed = 123)
# reshape action value function Q
print(matrix(apply(res$Q, 1, max), ncol = 10, byrow = TRUE))
```

We can then define the optimal policy by taking the argmax over the action value function Q.

```r
optimal.policy = max.col(res$Q)
print(matrix(optimal.policy, ncol = 10, byrow = TRUE))
```

### Use function approximation for complex environments

Table-lookup reinforcement learning is great to understand and play around with algorithms. But when applying RL to large-scale real world problems building one huge table is not feasible anymore. Instead of representing every state-action pair in a big table one would use a function approximator to represent the value function.

With `reinforcelearn` you can specify your own function approximator very flexible suited to a specific task.
Just create a `predict`, `train` and `preprocessState` function and pass these on to the algorithm, e.g. `qlearning2()`. 

Here is a toy example:

```r
# Solve the Windy Gridworld task using a neural network as function approximator
# Define a tensorflow graph for a neural network.
library(tensorflow)
tf$reset_default_graph()
inputs = tf$placeholder(tf$float32, shape(1L, grid$n.states))
weights = tf$Variable(tf$random_uniform(shape(grid$n.states, 
  grid$n.actions), 0, 0.01))
Q = tf$matmul(inputs, weights)

nextQ = tf$placeholder(tf$float32, shape(1L, grid$n.actions))
loss = tf$reduce_sum(tf$square(nextQ - Q))
optimizer = tf$train$GradientDescentOptimizer(learning_rate = 0.1)
trainModel = optimizer$minimize(loss)

# initialize the session and the weights
sess = tf$Session()
sess$run(tf$global_variables_initializer())

# takes the state observation and returns a one-hot vector
preprocessState = function(state_) {
  one_hot = matrix(rep(0L, grid$n.states), nrow = 1L)
  one_hot[1L, state_ + 1L] = 1L
  one_hot
}

# predict returns vector of q values for a given state
predict = function(inputs_) {
  sess$run(Q, feed_dict = dict(inputs = inputs_))
}

# train model, update weights, e.g. gradient descent: this is supervised learning
train = function(inputs_, outputs_) {
  sess$run(tuple(trainModel, weights),
    feed_dict = dict(inputs = inputs_, nextQ = outputs_))
}

res = qlearning2(grid, preprocessState, predict, train, 
  n.episodes = 300, seed = 123)
```

Also have a look at the vignettes for further examples.

Logo is modification of https://www.r-project.org/logo/.
