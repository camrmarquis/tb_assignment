# Truebill Takehome

All code for this is on `notebooks/tb.ipynb`. I usually dont work a lot in Jupyter, but for the exploratory scope of this problem set it seemed appropriate.

*Note*: I excluded `data/tb_data_team_homework.csv` from git as to not make you data open to the public :). It will have to be added in there manually in order to run the notebook.

## Part 1

I wanted to attempt a ML based approach over a rules based approach for the grouping task,
even though it meant forgoing a catch-all group. The general steps of my implementation are as follows:

1. Run all text through a simple regex to remove any sequences of non-alphabetical characters
2. Use a HashingVectorizer to build feature vectors based on token frequencies. I went back and forth on whether to use idf weighting here with 
my initial feeling being not to. My thinking was that terms being frequent throughout all description means something in the context of financial transactions. 
When experimenting with idf weighting, I found no real improvement in results. Some thoughts on the hashing vectorizer:
    - When thinking of this as a production system, the hashing vectorizer has some advantages in that can be used in streaming pipelines and is very scalable even on low memory machines.
    This would allow for the design of an event driven training pipeline.
    - There are some issues with this idea when dimensionality reduction comes into play in the next step.
     
3. Perform a singular value decomposition for dimensionality reduction. Here I picked an arbitrary number of components. (More on that in my proposal for part 2) 
4. I wanted to factor in the transaction amount as well so I preprocessed that data separately. A quick look at the distribution showed the amount column followed a power law distribution. 
For clustering purposes, a log transform was applied to the data. 
5. All data was scaled by its maximum absolute value prior to clustering. I used this scaling method because it does not destroy the sparse data structure.
6. Cluster data into 20 groups using k-means with `k=20`. When running an experiment over many values of k, it appears you may
be able to describe this data in a smaller subset of groups (10-15). In future iterations, this info could be used to 
guide the creation of a catch-all group as we can assume ~5-10 of these groups are being forced at k=20.
7. Visual inspection of the created groups shows a solid grouping by key terms. However, the grouping seems to be heavily weighted towards individual vendors such as Groupon, instacart, etc. 
To improve this, I would look at more efficient ways to weight importance towards terms such as `preaproved debit`.'

Overall this method was relatively successful. We achieved solid groupings based on the vendor associated with the transaction. To improve this solution, I think a hybrid rules and AI based approach would
be valuable. A rules engine could pick out key terms known to a certain group automatically, and clustering could be used on those that fall into a catch-all group. Further analysis could be done on how changing the number of components in the 
decomposition effects the clustering results.

## Part 2 

Here I used a simple random forest classifier to perform the initial classification task. I performed a grid search on the hyperparameters to get an optimal fit. I used random forest in this case because of its often high quality performance on small-scale datasets. 
Standard classifier measurements such as accuracy scores and precision/recall/f1 on the class level are useful in evaluating our initial fit. We can calculate these by creating a validation split fom our training data (in addition to the 10% holdout from step 1).

Overall our classifier is very good at providing class predictions for our synthetically labeled dataset. The key word there is that the labels are generated synthetically. A manual (or rule based) labeling approach would make me far more confident that our classifier is accurately performing the classification task we are after.
An additional validation step could be to look at the Levenshtein Distances between strings within classified groups. If a group in our holdout set has a high average partial match score, we could feel more confident that the classifier has learned a proper representation of transaction descriptions.

In a production-scale system, I would approach this task with a different classifier. In particular I would use something that can be partially fit in batches such as a linear classifier with SGD training. I would also consider that an easier task for an individual classifier is to do binary classification on a single class. So you could build 20 individual classifiers.

## Part 3

For this part I will make the assumption that this transactional data is being emitted as an event from Segment, Kinesis, etc. and that those events are eventually stored in some long term DB such as Postgres.

1. Build an initial training script that queries transaction descriptions from long-term storage and train using an adaptation of the logic from notebook. When the model is trained, upload artifacts to persistent storage and manage using an ml-ops tool such as MLflow.
2. Build a rest API to serve as a webhook endpoint for your streaming service. This app will load the latest model version from Mlflow and use it to process incoming transaction async and assign a class label to each. In AWS, these now classified transaction could be placed in an s3 landing spot complete with a GLUE crawler to be placed back into Redshift (postgres).
3. When a user manually corrects a categorization, an event can be emitted to notify the app that there are records available to retraining. The app could track these by potentially caching a reference_id specific to the individual transaction.
4. When the amount of corrected transaction reaches a certain amount, the training job could be re-run on those corrected transactions and the new model/version being updated in MLflow.
5. Using a metrics tracker such as Datadog, you could track the rate at which transaction categorizations are being corrected and use that as an indicator of model drift. 
