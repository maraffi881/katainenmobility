# Introduction

This repo represents quick answer to O’Reilly Architectural Katas 2025: AI-Enabled Architecture.

It’s about a mobility company renting electric cars, vans, scooters and bikes by hour (vans and cars for longer duration). Exact specs omitted.

Three key questions or success criteria were mentioned:

The three key issues and hence success criteria for the work where:

* Can we anticipate customer needs/when people want to use service?
* How to priotize which vehicles to swap out batteris first (bikes, scooters).
* Right now, most of our customers just use them on an ad-hoc basis - we’d like more of them to rely on our fleet for regular trips like daily commutes

## Overview of AI Usage

For the battery swap issue, a predictive model for equipment is proposed. Process works both for predictive care (when things go pear shaped) as well for batteries (out-of-juice situation) just with own models (for each key component that can break as well for batteries). It consists of found main parts: 1) capturing data first (collect, validate, make into canonical format and to data warehouse) and 2) then actual training pipeline with calculating additional features from raw material, splitting into training and validation, model training, evaluation until model is declared ready, 3) usage where model is available via API that uses cached, precalculated results, 4) comparing actual realized events against what the model predicted to monitor model degradation.

For anticipating customer needs a customer demand model is proposed. The model predicts for each vehicle type demand on each docking/charging station on hourly basis. The main change is to segment users first into groups (each user gets assigned ClusterId) and then flagging all training data with right ClusterId before doing the actual predictive modelling.

For incentivizing users to be regular commuters (for work etc.) for-uptake model is created. Its one of the customer behavioural models. For the training we naturally need training data sets that can be only created by first thinking of different incentive schemes to convert people to daily users and then based on that, do the training and monitor how well these models work in the wild. Different customer segments are likely to take up different promotions. The clustering of users will most likely be different than for demand.

None of these models needs generative AI. It becomes relevant when we add a chatbot to the mix. Two/three different chatbots are proposed:

* One for users where they can ask about service, availability of vehicles, pricings etc.
* Another for operational staff where they can question state of the network and as the genAI to generate plans how to fix problems
* Third one for management where they can query about anomalies on the system (are there usage anomalies at the moment and how would you describe their nature?). Ultimately we want to allow users to ask “Identify anomalies in fleet and their most likely causes”

GenAI can also be used to generate ideas for promotions to incentivize customers to increase usage. These concepts are then validated by marketing. GenAI can also generate a lot of different wordings (ways to present the same offer) so that we can see which text works best for each customer segment. These are also pre-screened by humans before usage.

Caveat Emptor: It seems I did not read the memo carefully (who has time in this hectic modern era of ours anyways and started with a breath first approach: whole architecture -> how AI fits. When starting ADRs I realized the focus probs should’ve been more depth first (explain why I chose K-Means instead of xyz).

# Select AI Use Cases

## Predictive Maintenance and Battery Swap

![A diagram of a process flow  AI-generated content may be incorrect.](data:image/png;base64...)

Data flows from the vehicles via the vehicle IoT Kit (described in more detail below) to the device agent and via it to connector service that writes it to pub/sub service. Different services pick relevant data for them (event processor(s) events and data processor(s) incoming measurements). Measurements are written to a data store (Big Query).

The data needs aggregation over a fixed time window (e.g., 1 hour or 1 day) to create meaningful features for AI. Typically needed features includes telematics data on usage (odometer readings, trip duration, trip count), stress data (accelerometer, hard braking events, RPM), environmental (location, temperature, time of day, day of week, raining, icy road etc.), component data (Error codes, battery degradation rate (initial capacity} - current capacity), frequency of error codes (error codes from same component frequenty), vehicle details etc.

This is done via scheduled batch jobs and results stored to same data warehouse.

We want to train for each important component own predictive model based on this past history. Models to predict time-to-failure or in case of battery swap model (time-to-empty-battery), to predict when battery runs out of charge.

The training is done using for example Vertex AI if on GCP.

Clients use the model via a Client API. One of the main users for time-to-failure is field force, where the ERP can create work orders before faults happen (see below more on field force architecture).

To speed up the Client API, we want to pre-calculate results regularly as tear and wear tends to happen slowly. There is a batch job that uses the created model and populates a low-latency cache for results and the Client API just pulls results from this in-memory store.

Sometimes the predictive model is wrong and something breaks before the predictions think. For this case we need a comparison service that reads fault events from pub/sub and compares timings with what the model thinks. Discrepancies are aggregated and presented in the Operations Center UI. Administrators can then take approriate actions if the model performance starts degradating.

### Future direction

**![A diagram of a process  AI-generated content may be incorrect.](data:image/png;base64...)**

As an economically prudent company, Katainen MC is looking for ways to reduce costs. One option is to move interference to the edge.

One thinks easily that for cost reasons the IoT kit at vehicles needs to be simple. The opposite can also be possible. We could add a GPU to the IoT kit and think about loading some AI models to be executed on-vehicle. This makes sense in the case for cars and vans.

The architectural change is not too complex. The model is downloaded to the vehicle (car, van) that is beefed with GPU and results are emitted as events/data from the vehicle using normal routes.

We still want to cache results for the predicted time-to-failure but it makes no sense to report this frequently if everything is shipshape. Local models could send this perhaps once a day, only report when time is within configured limit (say within next month a fault is expected to happen) or when battery discharges much faster than last reported. Comparison works as before.

## Predictive Demand Models

Same architecture for training predictive models based on time series of existing data can be used for many other use cases such as. Let’s discuss demand models

Features can include for example historical usage data, time of day, day of the week, weather forecasts, traffic, and local events. The last one would require feed from external sources what type of large gathering (fairs, world-class artist visits, in general number of smaller events coinciding) to be available.

This can be used to build models on overall demand and more fine grained model for demand for each different vehicle types in any given area to docking stations level at hourly level. This solves one of the essential questions that MC as an organisation has.

These models then provide input data to various marketing and dynamic pricing models.

## Range prediction

Same architecture again for range prediction based on charge, weather, rental time and city traffic prediction. Range prediction can be made at customer level possibly if we have customer driving from past usage. First there needs to be a generic model and then customer past behaviour can be used to finetune it. This might however be too costly or make the architecture more complex with relatively little improvement.

The time-to-empty-battery model alone goes a long way to solving another MC’s pressing issues: How to prioritise which vehicles to switch out batteries? (for bikes and scooters). It can be improved by combining information from demand model and range prediction. If in a given area high demand is expected and batteries are not quite at end, as long as customers see the status and predicted travel distance available, people with short needs can still utilize our fleet.

## Customer Behavioural Model

Customer Behaviour Prediction (predicting user action) requires some modifications, particularly in modelling approach and naturally the features and labels will differ. We want to be able to predict at least when they are most likely to use the service (weekday, weekend, time of year) and to what types of promotions they would be most likely to take up.

It makes sense to segment users to user segments before building predictive models. This is done with unsupervised learning. Segmenting allows to build more accurate and specific models for each segment, leading to better results.

Typical features are time dependent (day of week, hour of day, time of last service use, time since last login etc.), context (location type = beach, city centre, weather, local events/holidays, season, even competitor promotions), user data (demographics, how much consumed so far, how recently, what option(s) they use (car, scooter,…), how often)

Architecture is shown in diagram below.

**![A diagram of data processing  AI-generated content may be incorrect.](data:image/png;base64...)**

The process consists of two phases: segmentation and actual model training

Segmentation starts with the raw data about user behaviour. First step is to calculate key metrics needed in clustering (like per event the time-to-next purchase). The calculated values are then scaled and normalized (typically to values between 0 and 1 to prevent any feature with large value to dominate outcome), then unsupervised learning algorithm like K-Means is executed. This defines the number of clusters and generates the final model. Last step is to use the trained model to assign a clusterId to every customer and then populate all rows in data with clusterIds depending from which customer that data originated. The assignment is done by finding the closest cluster centroid for each user (closes point in multidimensional space).

Last part is training the model.

It starts from the enriched data set is taken and additional derived features are calculated that were not needed in segmentation. Next data set is split into training data and validation data. Validation data is used to evaluate how well the trained model performs when it sees data that it has not seen before (not used in training).

Next model is trained using the training set and after that validated using validation data. Backpropagation is used to recalibrate weights in the model until validation reaches required quality. The final model is ready to use.

The model is then used in similar manner to the predictive care example.

Quite a few customer behaviour models can be created in this manner

* Time-to-next-use. How long will it be until this customer uses us again if we do nothing
* For-uptake-customer-per-promo. For each promotion, model predicts (0,1) if customer is likely to take up. E.g., they are likely job commuter (morning and late afternoon). How likely they are to take up a lunch time promotion
* At-risk-customer. One’s that are likely to churn to competitor
* Fraudulent-customer. How likely this user is attempting fraud

For-uptake-customer-per-promo can be used to find customers most likely to convert to daily commuters.

## Customers’ Conversational AI Chatbot

Purpose of this chatbot is to answer customer questions about the service and current status of the vehicles and service in general (how busy it is etc.) You can interact with it using either text or speak to it.

![A diagram of a speech  AI-generated content may be incorrect.](data:image/png;base64...)

Mobile app is the user interface. User either types or speaks the question. User audio goes to its own /transcribe API where speech recognition is performed and text returned to user. User can correct mistakes before submitting

Queries are handled by a processing path.

* System prompt has been given at the beginning as general guide and policy on how the system behaves.
* Quardrails.ai is used to weed out toxic queries and remove personally identifiable information
* The cleaned query is given to a AI model with a set of tools that can be used to gather additional information needed to answer the question, each tool contains description of when it should be called
* LLM decides what tools are needed to answer and return a list of functions with parameters.
* Each function call gets vetted using pydantic that the parameters are valid
* Main functions include llamaindex where all textual material about service, vehicles, locations is indexed. Those queries should land there. AI Batch Jobs have been run prior to make various predictions of near future, for example about expected demand in each area per hour. If customer is interested in results of particular predictive model => function about that should be called. There are several other backend data items that may make sense in future, the tool list is expandable.
* Functions get called and additional context is built.
* Finally, AI model is passed the full set: system prompt, user query, additional context data. LLM formulates the final answer that gets returned to user

## Experimental

In order to save money, Katainen MC may think of moving some behavioural models to the consumers phone (avoiding to need to run them in the cloud costing money). This would need to be opt-in as most users guard their phone battery with near religious zeal. Still, this would allow the consumer to chat with Katainen MC asking for example “make me an offer I can’t refuse,…, no,no,no,no, make a real offer, …” or “tell me the route I was driving last Thursday” (as text listing street addresses at measurement points). Whether this would be worth it, I’ll leave to the capable hands of the upper management.

## OPS Center Chatbot

Same concept is applicable to operations centre where people manage the fleet. Here they use the system to make queries about the state of the system or individual vehicles. As additional the operational staff can ask the system to make a plan for solving a particular problem. The plan is not immediately executed but the staff can review and edit it before submitting to execution. This also means that it has some additional tools at its usage for making the change. Otherwise, it is similar to the previous system.

![A diagram of a system  AI-generated content may be incorrect.](data:image/png;base64...)

System prompt is augmented to differentiate between PLAN and QUERY modes and to return plan in structured format to user.

User interface is augmented so that user can accept/reject or edit the plan. When submitting again, code handling the plan iterates over it (this is not in LLM, but coded as a “driver” for plan). Plan is executed in similar manner but some tools are added to actually affect the state of the system (config changes for example). Results are added to context and finally a LLM is used to formulate final response.# katainenmobility
