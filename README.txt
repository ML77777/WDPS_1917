
## Python version ##
For our tests on DAS4, we used python 3.5.2 and locally 3.7.0

------------------------------------------------------------------------------------------------------------
## Packages ##

# Pre-processing #

- WARC package was used to open warc files and easily extract content and key-ID for convience
- This can be installed for PYTHON 3 with for example pip3.5 install --user warc3-wet
- For python 2, which was not tested, you can use pip install warc

- gzip package is used in combination with WARC package as well

- Beautifulsoup package was used to extract the text and make it more clean
- Install command: pip install beautifulsoup4
- If gives error when using it and parsing, then may need to download lxml package: pip3.5 install --user lxml


# NER detection #
- NLTK package components, such as tokenize, POS tag and NE chunker, are used to do NER, in particular we used the following models to download their models one time only:
- pip3.5 install --user NLTK

nltk.download('punkt')
nltk.download('averaged_perceptron_tagger')
nltk.download('maxent_ne_chunker')
nltk.download('words')

- Note: If one gets error about NLTK still, then might have to install the package six: E.g pip3.5 install --user six

# Elasticsearch #
- requests package needed pip3.5 install --user requests

# Trident #
- Trident python bindigns are used to sparql queries
- so set export PYTHONPATH=/home/jurbani/trident/build-python to import trident
- This is the reason why python 3.5 was used
- Numpy also needs to be the latest version in order to work

Other packages that are usually standard:
- difflib package to import SequenceMatcher to do comparison between strings
- operator package to import itemgetter to sort or to get the maximal tuples based on ith element

------------------------------------------------------------------------------------------------------------
## Folder and files ##

- data folder
    - sample.warc.gz  this is the default file (path) to do predictions with run run_entity_linking_file.py
    - sample.annotations.tsv  some annotations of the sample file where we predict on with 560 golden entities
    - own_annotations.tsv     Some more (manually) annotations on top of the sample.annoations.tsv one with 886 Golden entities

- elasticsearch.py :
contains the function to requests a search and returns its results

- functions.py
contains all the functions used in the main file run_entity_linking.py. Functions are to clean the text, to do elasticsearch and some operations and functions to check if entities are discarded

- run_entity_linking.py
The main file to run everything, reads input such as elasticsearch domain and optional labels for filenames and optional paths to data. It defaults to the data repistory that we assume and specified above. When predicting it automatically creates an folder predictions and saves it in there, and writes line by line while predicting (so you will have results when interrupted mid-run). The file has a standard name of predictions_. Results are saved in results folder with same name.

- run_entity_linking_spark.py
The main file to run with spark, with the first argument being the master URL or number of the master node on das4. The rest of the arugments are the same sequence as in run_entity_linking.py

- score_extended.py
Used the same way as score.py originally with golden file as first argument and predictions as second argument. Additionally it shows the amount of mappings and how many were correct and how many incorrect.

## Launch Scripts ##

- run_all_normal.sh
Script that runs all steps without use of spark. It starts an elasticsearch server from folder in /var/scratch2/wdps1917, just copied from the one in urbani path . It assumes defaults paths of the data folder as described above (arguments can be modified)

- run_normal.sh
Same as run_all_normal script but assumes an elasticsearch server is set up beforehand and have to define the node it is running on in the script. This is because our elasticsarch on DAS4 sometimes did not want to start due to the last state not changing yellow.

- prep_commands_spark.sh
There are different ways to start a spark cluster using standalone or yarn manager for example. Here we manually start master and slaves, as with yarn it just kept running for us and did not end.
This script reserves the necessary nodes for master and slave nodes. In this script you can define the x amount of worker nodes in the for loop that you want to reserve. Then it uses preserve -llist to give an overview of the nodes and can also look at the RESERVE IDS to be specified in run_all_spark.sh to run commands on specific nodes. So this is run before run_all_spark.sh

- run_all_spark.sh
After reserving the nodes and following the method to start things manually, we must define the nodes in this script to distinguish which one is master and which ones are slave nodes. We do this by defining the node number of the master and its reserve ID such that we can launch the start-master script (path also to be defined if different) on it. We also must define the reserve IDs of the slave nodes to start the start-slave script and stop it at the end. The reserve IDs can be found by using preserve -llist. 
Also, here we assume the elasticsearch server is started beforehand and we define the node and the port. Having defined all this in this script, we can run this script. It will start master and slave nodes, execute the main script of run_entity_linking_spark.py and stop the master and slave nodes afterwards.
Other way to start the cluster is to run the start-all script on the master and define the slave nodes in conf/slaves file of spark application folder. This method did not work for us while testing because master and worker nodes used different python versions. We tried to solve this with defining PYSPARK_PYTHON and PYSPARK_DRIVER_PYTHON to the directory of the python/3.5.2 which is a shared package. But workers do no have access to this because the module of python/3.5.2 is not loaded in for them. It could be the case that this way of starting master and slave nodes does not give a proper connection, which would explain the missing speedup to be described in the results section. In this default script we have the "_spark" prefix to distinguish from the other.

Note: In run_entity_linking and run_entity_linking_spark, we save the predictions in predictions folder and score in results folder as described before. The score is saved by directing the output with ">" command. To still print the score, we run the score_extended.py seperate outside it again. But here it assumes the same prefix as used before. If the argument in run_entity_linking is changed, one could see another output if old predictions are present. So one can check the scores in results folder just to be sure. 

------------------------------------------------------------------------------------------------------------
## Running ##

If one assumes the default paths and structure from above then can simply run:

Without spark:
- bash run_all_normal.sh or bash run_normal.sh depending on which elasticsearch server to use
- This will produce predictions__.txt in predictions folder and results_predictions__.txt in results folder

we can modify paths as arguments of run_entity_linking.py in run_all.sh if one wants to:

python run_entity_linking.py <Required Elasticsearch domain> <optional label after prefix> <optional warc path> <optional annotations path>

run_entity_linking.py requires 1 argument minimal which is the elasticsearch domain so for example assuming elasticsearch on node001:9200:

python3.5 run_entity_linking.py node001:9200

this means we assume the annotations_path = "./data/sample.annotations.tsv" is and  warc_path is "./data/sample.warc.gz" and we give no additional label


if we want to give a label "test_run" and give other annotations file than we can run:

python run_entity_linking.py node001:9200 "test_run" ./data/sample.warc.gz ./data/own_annotations.tsv

The reason we also give annotations file is because at the end, run_entity_linking computes the score at the end and saves it in the results folder with the same prefix and also writes the amount of time in seconds it took to compute. So here we get predictions/predictions_test_run.txt and results/results_predictions_test_run.txt


With Spark:
As mentioned before we reserve nodes beforehand by running bash prep_commands_spark.sh
Then we note down the node number of the master and the reserve IDs of master and slave nodes in run_all_spark.sh
This is needed to start the master and slave nodes, which needs the master URL to connect.
Then similiar to run_entity_linking.py, but with an master URL or master node number as first argument we run the main script:

python3.5 run_entity_linking_spark.py <Master URL or master node number> <Required Elasticsearch domain> <optional label after prefix> <optional warc path> <optional annotations path>
or
spark-submit run_entity_linking_spark.py <Master URL or master node number> <Required Elasticsearch domain> <optional label after prefix> <optional warc path> <optional annotations path>


Note that we assume 5 hours max in our script file, with the sample file a little more than 1 hour should be enough. Time can be changed if needed more on a node.

Our files on das4 can be found at:

"/var/scratch2/wdps1917/wdps_final_spark" folder on das4 

Or can be found on Github: https://github.com/ML77777/WDPS_1917


------------------------------------------------------------------------------------------------------------
## Original Method ##

First we preprocess the text because we want to make sure what we put in our NER part is really visible text, which is why we make use of the package Beautifulsoup and gzip and warc to extract the correct information that is text. For example we remove HTML tags from text and get the proper TREC-ID from the document to be used in the prediction.
This text is then given to our NLTK tagger, which is a tagger that is relatively fast compared to others, can detect entities reasonablly well and also does this with multiple spans. Furthermore, it is famous package that is supported well and easily installed.

In case the NLTK tagger does recognize a wrong or useless entity, we check this if there are any useless symbols such as "<",">","/",":" or other symbols that might give trouble to elasticsearch.
NLTK can regonize entities well, but also recognizes a lot more than other models (low precision), so to filter these we look at the response when we put it into elasticsearch.
We skip the entity if elasticsearch returns no results or if it does not pass the following 2 criterias:
- First check on the top k labels/terms that elasticsearch returns and look at their score, if none of them are higher than the minimal score of 2, we discard it.
- For each result we do similarity matching with the entity mention and the entity label from elasticsearch to see how well they match. If this value is below 0.82 ratio we also discard it.

If the top k results all pass this test, it will keep on going through each element of the response from elasticsearch. Word labels from this response are matched with an freebase ID. We use an OrderedDict to keep track of the rank of freebase_IDs from the response and the set of word labels that match a certain freebase ID. The idea is that a popular freebase ID will likely have more word labels/terms linked to it than a less known freebase ID.
For the top q results of the response from elasticsearch, usually q >= k, we also keep track of elasticsearch scores and these similarity matching ratios. Which will be ranked based on score and given a certain score value to the corresponding freebase ID.

After having a set of freebase IDs for a word and its set of labels we give score points for the top r freebase IDs from elasticsearch. Score points are given based on the elasticsearch score and similarity ratio ranking as mentioned above and queries.
For example, the elasticsearch scores are ranked and each freebase_id gets a score based on their ranking, where number one gets 1, number two gets 1 - (1/ranks) etc. The same is done for the ratio.
Then we do some general sparql queries such as:

- Title similarity matching with the entity mention, which we 
found important and thus is multiplied times 3 as a weight. With this title matching, we also look subwords ratios to see if we can a higher max ratio. 

- The amount in the set of labels linked to a freebase ID from elasticsearch is also quite important, so we divide the amount of labels by 5 (if 100 labels then divide by 5 is 20), then divide by 20 and times 2 in order let it be maximal score of 2 possible as it is less important than title matching ratio. 

- Some more general queries are amount of equivalant webpages (same score assignment as how many labels), if have twitter, media presence and official website (a standard small value addition of 1/r).

Then we run some more specific queries based on their entity label from the tagger, could be "GPE","GSP","LOCATIOn","PERSON","ORGANIZATION" and "FACILITY" and gets a score addition of 3 if it matches with its label category. If it does not get a matching result, then it moves to a secondary category type. But the addition that it will get if it gets a match is reduced by 1 then. All the queries can be found below that are also present in the run_entity_linking.py file

Then the freebase_id with the highest score is used for prediction

------------------------------------------------------------------------------------------------------------
## results

Originally, for the 1464 documents and using our own annotations labels, we have 886 golden entities, 37899 predictions, where we found 731 mappings and 452 of them are correct mappings. Leading to an precision of 0.011926, recall of 0.51015 and F1 score of 0.0233. The precision is low because we predict a lot but we only have so much annotations. This originally takes around 4900 seconds.

With implementation of spark, we ran it using 3 workers, same results as same method, but the times are less consistent. One run it ran within 3500 seconds and other time in 5200 seconds. We mainly have some doubts if the cluster is correctly setup and making use of workers rather than the program not working. (Not sure how to look at the WEB UI with das4). As of right now, we know 3 ways of starting the cluster and launching application, but for 2 of the 3 ways we do not know how to solve, which is why we went the first method of manually starting the launch scripts. But method 2 should be closely related to method 1 but gives error of not using the same python version as mentioned before.

Thus to be sure to increase the scalability a bit, we also remove some queries and ranking to make it a bit faster:
- We reduced the query label order to maximum of 2 categories. So if an entity does not match with 2 categories it will not go a third or fourth one to execute its queries
- We do not look at subwords of titles anymore to do the similarity ratio matching. Only 1 time comparison between the two strings
- We do not look at twitter and media presence anymore for general queries
- For PERSON queries, we do not query anymore if it is a nndb, celebrity and if in entertainment sector
- We do not sort and give scores for ranking of elastic scores and matching ratio similarity of elasticsearch response anymore.

With the use of spark we then get:
- 37899 predictions, same as before, but slightly less performance results, 731 mappings, 398 correct, precision of 0.0105, recall 0.4492 and F1 of 0.0205.
Finished in 3700 seconds.

Without use of spark:
- It took around 3750 seconds.

Slightly worse results, but do decrease the time to run it.

So one can consider setting up their own Spark cluster instead of using our launch script with Spark and then run our run_entity_linking_spark.py file with its required arguments to see if it can really benefit if workers are connected with certainty.

------------------------------------------------------------------------------------------------------------
## Queries ##

    # General
    sparql_query_title = "select distinct ?obj where { \
        <http://rdf.freebase.com/ns%s> <http://rdf.freebase.com/key/wikipedia.en_title> ?obj . \
        } "  # % (freebase_id)

    sparql_query_n_webpage = "select distinct ?obj where { \
        <http://rdf.freebase.com/ns%s> <http://rdf.freebase.com/ns/common.topic.topic_equivalent_webpage> ?obj . \
        } "  # % (freebase_id)

    # Is social media active (Not consistent for persons) #Avg 2 or 1
    sparql_query_media_presence = "select distinct * where { \
       <http://rdf.freebase.com/ns%s> <http://rdf.freebase.com/ns/common.topic.social_media_presence> ?obj .\
        } "  # % (freebase_id)

    # Not yahoo and disney , Avg 1
    # Is social media active (Not consisten for persons?)
    sparql_query_twitter = "select distinct * where { \
       <http://rdf.freebase.com/ns%s> <http://rdf.freebase.com/key/authority.twitter> ?obj .\
        } "  # % (freebase_id)

    # Avg 1 or 2
    # Is social media active (Not consisten for persons?)  #New york times of Flash player bijv
    sparql_query_website = "select distinct * where { \
       <http://rdf.freebase.com/ns%s> <http://rdf.freebase.com/ns/common.topic.official_website> ?obj .\
    } "  # % (freebase_id)

    # ------------Location-----------------------------------------------------------------------------------------------------

    sparql_query_location = "select distinct * where {\
     { <http://rdf.freebase.com/ns%s> ?rel <http://rdf.freebase.com/ns/location.location>  .  }\
     UNION \
     { <http://rdf.freebase.com/ns%s> <http://rdf.freebase.com/ns/base.biblioness.bibs_location.loc_type> ?type . } \
      } "  # % (freebase_id,freebase_id)

    # ----------Person------------------------------------------------------------------------------------------------------

    sparql_query_person = "select distinct * where { \
       <http://rdf.freebase.com/ns%s> ?rel <http://rdf.freebase.com/ns/people.person> .\
        } "  # % (freebase_id)

    # Scenario schrijver Tony Gilroy is a not a celebrity and not a notable name either.
    sparql_query_person_nndb = "select distinct * where { \
       <http://rdf.freebase.com/ns%s> ?rel <http://rdf.freebase.com/ns/user.narphorium.people.nndb_person> .\
        } "  # % (freebase_id)

    # Jeremy Renner is notable name but does not have this type celebrity
    sparql_query_person_celeb = "select distinct * where { \
       <http://rdf.freebase.com/ns%s> ?rel <http://rdf.freebase.com/ns/base.popstra.celebrity> .\
        } "  # % (freebase_id)

    # Tony Gilroy (and Matt damon) has these properties, politician not:
    sparql_query_entertainment = "select distinct * where {\
     { <http://rdf.freebase.com/ns%s> <http://rdf.freebase.com/key/source.entertainmentweekly.person> ?name  .  }\
     UNION \
     { <http://rdf.freebase.com/ns%s> <http://rdf.freebase.com/key/source.filmstarts.personen> ?name2 . } \
      } "  # % (freebase_id,freebase_id)

    # --------Organization------------------------------------------------------------------------------------------------------

    # Should be the first one if company
    sparql_query_company = "select distinct * where { \
        <http://rdf.freebase.com/ns%s> <http://rdf.freebase.com/key/authority.crunchbase.company> ?obj . \
        } "  # % (freebase_id)
    #Skipped other ones
    #--------Other---------------------------------------------------------------------------------------------------------------

    # Also Flash player etc.
    sparql_query_inanimate = "select distinct * where { \
       <http://rdf.freebase.com/ns%s> ?rel <http://rdf.freebase.com/ns/base.type_ontology.inanimate> .\
        } "  # % (freebase_id)
