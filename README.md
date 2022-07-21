<img src="src/logo_full_res.png" alt="Logo" width="200"/>

# Z-Fed
## A ZKP Federated Framework to Support Balanced Learning
### Information

| Field               | Value                                                           |
| ------------------- | --------------------------------------------------------------- |
| Project Title:      | Z-Fed: A ZKP Federated Framework to Support Balanced Learning   |
| Student IDs:        | 21264466, 20210611                                              |
| Student names:      | Stefano Marzo, Royston Pinto                                    |
| Student emails:     | stefano.marzo2@mail.dcu.ie, royston.pinto2@mail.dcu.ie          |
| Chosen majors:      | Artificial Intelligence, Data Analytics                         |
| Supervisor:         | Dr. Rob Brennan, Dr. Lucy McKenna                               |
| Date of submission: | 27-01-2022                                                      |

### Practicum Topic
Development of a data analysis tool for balanced federated machine learning. The proposed framework can be used to classify data more accurately by reducing bias related to the presence of minority groups within the dataset. The project, motivated by recent scientific results, aims to implement unfairness quantification as part of the fundamental principles described in the Ethics guidelines for trustworthy AI authored by the High-Level Expert Group on Artificial Intelligence (AI HLEG) on behalf of the European Commission.
### Research Questions
- To what extent can we mitigate federated data bias according to EU guidelines for data ethics and trustworthy AI?
- To what extent can zero knowledge proof metadata about the proportions of population groups can be used in a federated learning environment to enhance AI trustworthiness?

# Z-Fed Environment
## Zero Knowledge Proof (ZKP) Framework
### Definitions
Within a set of individuals it is possible define categorical features that describe the population. It is required to design these features using the following dictionary structure:
```
features = {
    'FEATURE_1': ['f_1_label_1', ... 'f_1_label_n'],
    ...,
    'FEATURE_m': ['f_m_label_1', ... 'f_m_label_k'],
}
```
In a *Z-Fed* environment, the server responsible of the learning process must be aware of the possible categories i.e. must have full visibility of the `features` structure. 
### ZKP Server
The Zero Knowledge proof server `ZKServer` presents the following features:
 - `password`: necessary for the ZKP token generation
 - `features`: dictionary of possible features
 - `tokens`: data structure to save the tokens for client authentication
 - `client_representations`: data structure to store an encrypted representation of the `features`.
- `zk`: an object used to: 
	 - instantiate the elliptic curve used in ZKP.
	 - instantiate the $salt$ i.e. a private number to encode a $secret$ i.e. `password`.
	 - create a `signature` used for client registration/authentication.
 - `signature`: an object used for ZKP registration/authentication.
 - `groups` a dictionary of counters used to keep track of the number of clients used for training basing on their (encrypted) features.

The `ZKServer` is able to create an authentication token for every possible client `label` value, and verify if a client has the authentication privilage using ZKP.

### ZKP Client
A Zero Knowledge Proof Client `ZKClient` is responsible for representing an individual's tuple `(feature, label)` in the distributed dataset. Every authorized individual will instantiate a number of `ZKClient` equals to the number of different features defined in the `features` dictionary. 
Example:
For a set of feature:
 ```
features = {
	'SHAPE':  ['Square', 'Circle', 'Triangle'],
	'COLOR':  ['Purple', 'Scarlet'],
	'BORDER': ['Solid', 'Dotted', 'Double']
}
```
and an individual:
```
individual = {
	...,
	'SHAPE':  'Circle',
	'COLOR':  'Scarlet',
	'BORDER': 'Dotted'
}
```
The individual's client will instantiate three `ZKClient`*s* with the following tuples:
- `ZKClient_1`: `('SHAPE',  'Circle')`,
- `ZKClient_2`: `('COLOR',  'Scarlet')`,
- `ZKClient_3`: `('Border', 'Dotted')`.

A `ZKClient` will have the following features:
 - `feature`: *described above*.
 - `label`: *described above*.
 - `zk`: an object used to: 
	 - instantiate the elliptic curve used in ZKP.
	 - instantiate the $salt$ i.e. a private number to encode a $secret$ i.e. `label`.
	 - create a `signature` used for server registration.
	 - create proofs used for server authentication.
 - `signature`: an object used for ZKP authentication.

Moreover, a `ZKClient` can produce an encrypted version of the `label` using a $sha256$ hash function that returns the following:

 `encrypted_label` = $sha256($`label` + $salt)$*
 
 *+ *represents a string concatenation operation.*

### ZKP external services

*Z-Fed* environment requires the following external services to perform balanced ZKP learning:

 - **Client Initializer**: every client will connect to this service to get the generated unique $salt$ value.
 - **Server Initializer**: this service will connect to the Client Initializer to get the common $salt$ value, then it will create one `ZKClient` for each possible label value in the `features` dictionary. This way the `ZKServer` can accept a connection from this clients to prepare the internal `groups` structure using the `feature` value and the `encrypted_label` value.

## Federated Learning Framework

### Machine Learning Model

In a *Z-Fed* environment, one can define a federated machine learning model `LearningModel` as follows:
Given a neural network `nn`, a number of epochs `epochs_n`, and a learning rate `learning_rate`,  `LearningModel` exposes the following APIs:

 - `__init__(model, epochs_n, learning_rate)`: initializes the federated learning model.
 - `get_weights()`: returns the neural network learning model `nn` weights.
 - `set_weights(W)`: uses an array structure `W` to update the `nn` weights.
 - `fit(X, y)`: uses the array of features `X` to propagate data and an array of labels `y` to calculate the loss for backpropagation.
 - `predict(X)`: returns the predicted values for an array of features `X`.

### Federated Server
A *self balancing federated server* `FairServer` is defined as follows:

 - `model`: an instance of a `LearningModel` .
 - `workers`: list of addresses to reach workers i.e. federated clients.
 - `queue`: data structure to store workers in case they would represent an imbalance update for the model.
 - `max_gap`: in percentage, represents the balancing gap allowed to accept a worker update .
 - `gap`: a number that represents the balancing gap allowed to accept a worker update .

The `FairServer` presents the following functionalities:

 - Register workers for training.
 - Register features and labels to keep even proportions among subgroups.
 - Enqueue a worker if its update would result in imbalanced training.
 - Train the `model` using all the allowed workers respecting the balance training principles.
 - Return subgroups proportions and count metadata and plotting histograms.

### Federated Client (Worker)

A Federated Client `Worker` is defined as follows:

 - `id`: unique identifier
 - `client_prototype`: object storing the data retrieved from  the *Client Initializer*
 - `server`: the address/api to exchange information with the server
 - `model`: used to store the predictive model and ask for training
 - `x`: the client feature used for propagation
 - `y`: the client feature used to calculate the loss and for backpropagation
 - `secret_features`: dictionary of features used for balancing learning
 - `zkp_clients`: dictionary of `ZKClient`*s* used in ZKP subgroups balancing

`Worker`*s* have the following functionalities:

 - Load model: `server` sends a version of the current `model` to the `Worker`, the latter will save a version of it
 - Train: `Worker` uses this method to train the `model` received using `x` and `y` parameters.
 - Send updated: once the `Worker` trained the model, the updated weights and the feature groups are ready to be sent to the `server`. The update is accepted by the server only if the worker update is balanced.

## Z-Fed Framework 

The ZKP Federated Learning can be inizialized by following the steps below:

 1. The `ClientInitializer` service prepare the shared $salt$ value.
 2. All the `Worker`s that will participate to the learning process will get the $salt$ value needed.
 3. The `ServerInitializer` service get the $salt$ value from the `ClientInitializer` and prepare as many `ZKClient`s as defined in the `features` dictionary (one per label).
 4. A `FairServer` is initialized, tokens are generated using the trusted `ServerInitializer` list of `ZKClient`s, groups are prepared to count examples and enable balanced training.
 5. `Worker`s subscribe to the `FairServer` for the training procedure.
 6. Once reached a number of `Worker`s subscribed, the server start requesting them to train the `LearningModel`.


