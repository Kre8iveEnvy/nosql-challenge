# nosql-challenge
Instructions The UK Food Standards Agency evaluates various establishments across the United Kingdom, and gives them a food hygiene rating. You've been contracted by the editors of a food magazine, Eat Safe, Love, to evaluate some of the ratings data in order to help their journalists and food critics decide where to focus future articles.

## Part 1: Database and Jupyter Notebook Set Up
--Import the dataset with `mongoimport --type json -d uk_food -c establishments --drop --jsonArray establishments.json`
--Import dependencies
from pymongo import MongoClient
from pprint import pprint
--Create an instance of MongoClient
mongo = MongoClient(port=27017)
--confirm that our new database was created
print(mongo.list_database_names())
--assign the uk_food database to a variable name
db = mongo['uk_food']
--review the collections in our new database
print(db.list_collection_names())
--review the collections in our new database
pprint(db.establishments.find_one())
-- assign the collection to a variable
establishments = db['establishments']

## Part 2: Update the Database
--# Create a dictionary for the new restaurant data
new_restaurant = {
    "BusinessName":"Penang Flavours",
    "BusinessType":"Restaurant/Cafe/Canteen",
    "BusinessTypeID":"",
    "AddressLine1":"Penang Flavours",
    "AddressLine2":"146A Plumstead Rd",
    "AddressLine3":"London",
    "AddressLine4":"",
    "PostCode":"SE18 7DY",
    "Phone":"",
    "LocalAuthorityCode":"511",
    "LocalAuthorityName":"Greenwich",
    "LocalAuthorityWebSite":"http://www.royalgreenwich.gov.uk",
    "LocalAuthorityEmailAddress":"health@royalgreenwich.gov.uk",
    "scores":{
        "Hygiene":"",
        "Structural":"",
        "ConfidenceInManagement":""
    },
    "SchemeType":"FHRS",
    "geocode":{
        "longitude":"0.08384000",
        "latitude":"51.49014200"
    },
    "RightToReply":"",
    "Distance":4623.9723280747176,
    "NewRatingPending":True
}
--# Insert the new restaurant into the collection
establishments.insert_one(new_restaurant)
--# Check that the new restaurant was inserted
pprint(establishments.find_one({"BusinessName": "Penang Flavours"}))
--# Find the BusinessTypeID for "Restaurant/Cafe/Canteen" and return only the BusinessTypeID and BusinessType fields
query = {"BusinessType":"Restaurant/Cafe/Canteen"}
fields = {"BusinessTypeID":1,"BusinessType":1}

pprint(establishments.find_one(query,projection=fields))
--# Update the new restaurant with the correct BusinessTypeID
establishments.update_one({'BusinessName':'Penang Flavours'},
 {'$set':{'BusinessTypeID': 1}})
--# Confirm that the new restaurant was updated
pprint(establishments.find_one({"BusinessName":"Penang Flavours"}))
--# Find how many documents have LocalAuthorityName as "Dover"
in_dover = establishments.count_documents({'LocalAuthorityName': 'Dover'})
in_dover
--# Delete all documents where LocalAuthorityName is "Dover"
establishments.delete_many({'LocalAuthorityName': 'Dover'})
--# Check if any remaining documents include Dover
in_dover = establishments.count_documents({'LocalAuthorityName': 'Dover'})
in_dover
--# Check that other documents remain with 'find_one'
pprint(establishments.find_one())
--# Change the data type from String to Decimal for longitude and latitude
establishments.update_many(
    {},
    [
        {'$set':
            {'geocode.latitude':{'$toDouble': '$geocode.latitude'},'geocode.longitude': {'$toDouble': '$geocode.longitude'}
            }}])
--# Set non 1-5 Rating Values to Null
non_ratings = ["AwaitingInspection", "Awaiting Inspection", "AwaitingPublication", "Pass", "Exempt"]
establishments.update_many({"RatingValue": {"$in": non_ratings}}, [ {'$set':{ "RatingValue" : None}} ])
--# Use update_many to convert RatingValue to integer numbers
establishments.update_many({}, [ { '$set': { "RatingValue": {'$toInt':"$RatingValue"},
                                           }} ] )
--# Check that the coordinates and rating value are now numbers
query = {'RatingValue': {'$gte':1}}
fields = {'geocode':1, 'RatingValue':1}

results = establishments.find(query,fields)

for x in range(10):
    pprint(results[x])

## Part 3: Analysis
--Notebook Set Up
 from pymongo import MongoClient
import pandas as pd
from pprint import pprint
import matplotlib.pyplot as plt

--# Create an instance of MongoClient
mongo = MongoClient(port=27017)
--# assign the uk_food database to a variable name
db = mongo['uk_food']
--# review the collections in our database
print(db.list_collection_names())
--# assign the collection to a variable
establishments = db['establishments']
### 1. Which establishments have a hygiene score equal to 20?
--# Find the establishments with a hygiene score of 20
query ={"scores.Hygiene": 20}

--# Use count_documents to display the number of documents in the result
print(f"The is {establishments.count_documents(query)} establishments with a hygiene score equal to 20.")

--# Display the first document in the results using pprint
results = establishments.find(query)
print('')
print("The first document in the results is:")
pprint(results[0])
--# Convert the result to a Pandas DataFrame
hygiene = establishments.find(query)
hygiene_df = pd.DataFrame(hygiene)
--# Display the number of rows in the DataFrame
print(f"Rows in DataFrame: {len(hygiene_df)}")
--# Display the first 10 rows of the DataFrame
hygiene_df.head(10)
### 2. Which establishments in London have a `RatingValue` greater than or equal to 4?
--# Find the establishments with London as the Local Authority and has a RatingValue greater than or equal to 4.
query = {"LocalAuthorityName":{'$regex':'London'},'RatingValue': {'$gte':4}}

--# Use count_documents to display the number of documents in the result
print(f"The is {establishments.count_documents(query)} establishments in London with a RatingValue greater than or equal to 4.")

--# Display the first document in the results using pprint
results = establishments.find(query)
print('')
print("The first document in the results is:")
pprint(results[0])
--# Convert the result to a Pandas DataFrame
london = establishments.find(query)
london_df = pd.DataFrame(london)
--# Display the number of rows in the DataFrame
print(f"Rows in DataFrame: {len(london_df)}")
--# Display the first 10 rows of the DataFrame
london_df.head(10)

### 3. What are the top 5 establishments with a `RatingValue` rating value of 5, sorted by lowest hygiene score, nearest to the new restaurant added, "Penang Flavours"?

--# Search within 0.01 degree on either side of the latitude and longitude.
--# Rating value must equal 5
--# Sort by hygiene score

degree_search = 0.01
latitude = 51.49014
longitude = 0.08384

query = {
    'RatingValue': 5,
    'geocode.latitude':{
        '$gte': latitude-degree_search,
        '$lte': latitude+degree_search
    },
    'geocode.longitude': {
        '$gte': longitude-degree_search,
        '$lte': longitude+degree_search
    }}
sort = [('score.Hygiene',1)]
limit = 5

--# Print the results
pprint(list(establishments.find(query).sort(sort). limit(limit)))
--# Convert result to Pandas DataFrame
top_5_near_penang = establishments.find(query).sort(sort). limit(limit)
top_5_near_penang_df = pd.DataFrame(top_5_near_penang)
top_5_near_penang_df

### 4. How many establishments in each Local Authority area have a hygiene score of 0?
--# Create a pipeline that:
--# 1. Matches establishments with a hygiene score of 0
--# 2. Groups the matches by Local Authority
--# 3. Sorts the matches from highest to lowest
pipeline = [
    {'$match': {'scores.Hygiene':0}},
    {'$group':{'_id': '$LocalAuthorityName', 'count': {'$sum': 1}}},
    {'$sort': {'count': -1}}] 

results = list(establishments.aggregate(pipeline))

--# Print the number of documents in the result
print(f" Number of documents in result: {len(results)}")
--# Print the first 10 results
pprint(results[0:10])

--# Convert the result to a Pandas DataFrame
hygiene_score_0_df = pd.DataFrame(results)
--# Display the number of rows in the DataFrame
print(f"Number of rows in DataFrame {len(hygiene_score_0_df)}")
#rename columns for _id is 'LocalAuthorityName' and count is 'Count of Establishments')
hygiene_score_0_df = hygiene_score_0_df.rename(columns= {'_id': "LocalAuthorityName", 'count': "Count of Establishments" } )
--# Display the first 10 rows of the DataFrame
hygiene_score_0_df.head(10)

