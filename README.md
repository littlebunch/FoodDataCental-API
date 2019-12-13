# fdc-api
Provides a REST server to query and retrieve USDA [FoodData Central](https://fdc.nal.usda.gov/data-documentation.html) datasets.  You can browse foods from different sources, perform simple searches, access nutrient data for individual foods and obtain lists of foods ordered by nutrient content.    

# What's in the repo    
/api -- source for the REST web server    
/docker -- files used for building docker images of the API server     
/ds -- source for the data source interface.  Implementations should also go here     
/ds/cb -- couchbase implementation of the ds interface   
/ds/cdb -- couchdb implementation of the ds interface (work-in-progress)
/model -- go types representing the data models     

# Quick word about datastores
I've done versions of this API in MySQL, Elasticsearch, CouchDB and Mongo but settled on Couchbase because of [N1QL](https://www.couchbase.com/products/n1ql) and the built-in [full text search](https://docs.couchbase.com/server/6.0/fts/full-text-intro.html) engine.  I've heard it scales pretty good as well. :) It's also possible without a great deal of effort to implement a MongoDb, ElasticSearch or relational datastore by implementing the ds/DataSource interface for your preferred platform.       

# Building   
The steps below outline how to go about building and running the applications using Couchbase.  Additional endpoint documentation is provided by a swagger.yaml and a compiled apiDoc.html in the [api/dist](https://github.com/littlebunch/FoodDataCentral-api/tree/master/api/dist) path.  A docker image for the web server is also available and described below.

The build requires go version 12.  If you are using Couchbase, then version 6 or greater is preferred.  Both the community edition or licensed edition will work.  Or, if you don't want to bother with the go stuff, you can launch a server from docker.io (see below under *Running*).

### Step 1: Clone this repo
Clone this repo into any location *other* than your $GOPATH:
```
git clone git@github.com:littlebunch/fdc-api.git
```
and cd to the repo root, e.g.:
```
cd ~/fdc-api
```
      
### Step 2: Build the binary 

The repo contains go.mod and supporting files so a build will automatically install and version all needed libraries.  If you don't want to use go mod then rm go.mod and go.sum and have at it the old-fashioned way.  For the webserver:   
```
go build -o $GOBIN/fdcapi api/main.go api/routes.go
```
You're free to choose different names for -o binary as you like.  

You can also use the [Docker](https://github.com/littlebunch/FoodDataCentral-api/blob/master/docker/Dockerfile) file to create an image for the web server.

### Step 3: Install and build a datastore   
If you want to use [Couchbase](https://www.couchbase.com) then use the ingest utility available at [https://github.com/littlebunch/fdc-ingest](https://github.com/littlebunch/fdc-ingest).     

### Step 4. Start the web server (see below)   

## Configuration     
Configuration is minimal and can be in a YAML file or envirnoment variables which override the config file.   

```
couchdb:   
  url:  localhost   
  bucket: gnutdata   //default  bucket    
  fts: fd_food  // default full-text index   
  user: <your_user>    
  pwd: <your_password>    

```
      
Environment   
```
COUCHBASE_URL=localhost   
COUCHBASE_BUCKET=gnutdata   
COUCHBASE_FTSINDEX=fd_food   
COUCHBASE_USER=user_name   
COUCHBASE_PWD=user_password   
```
## Running    

The instructions below assume you are deploying on a local workstation.   


### Start the web server:    
```
$GOBIN/fdcapi -c /path/to/config.yml  
where    
  -d output debugging messages     
  -c configuration file to use (defaults to ./config.yml )      
  -p TCP port to run server (defaults to 8000)    
  -r root deployment context (v1)    
  -l send stdout/stderr to named file (defaults to /tmp/bfpd.out
 ```
 
Or, run from docker.io (you will need docker installed):
 ```
 docker run --rm -it -p 8000:8000 --env-file=./docker.env littlebunch/fdcapi
```
You will need to pass in the Couchbase configuration as environment variables described above.  The easiest way to do this is in a file of which a sample is provided in the repo's docker path.
   
## Usage    
A swagger.yaml document which fully describes the API is included in the dist path.     

### Fetch a single food  by FoodData Central id=389714: 
```
curl -X GET http://localhost:8000/v1/food/389714 
```
##### returns all nutrient data for a food   
```
curl -X GET http://localhost:8000/v1/nutrients/food/389714  
```
##### returns nutrient data for a single nutrient for a food
```
curl -X GET http://localhost:8000/v1/nutrients/food/389714?n=208 
```  
### Browse foods:   
```
curl -X GET http://localhost:8000/v1/foods/browse?page=1&max=50?sort=foodDescription
curl -X GET http://localhost:8000/v1/foods/browse?page=1&max=50?sort=company&order=desc    
```

### Search foods (GET): 
Perform a simple keyword search of the index.  Include quotes to search phrases, e.g. ?q='"bubbies homemade"'. For more complicated and/or precise searches, use the POST method.   
```
curl -X GET http://localhost:8000/v1/foods/search?q=bread&page=1&max=100    
curl -X GET http://localhost:8000/v1/foods/search?q=bread&f=foodDescription&page=1&max=100   
```

### Search foods (POST):
Perform a string search for 'raw broccoli' in the foodDescription field:   
```
curl -XPOST http://localhost:8000/v1/foods/search -d '{"q":"brocolli raw","searchfield":"foodDescription","max":50,"page":0}'
```
Perform a WILDCARD search for company names that match ro*nd*:
```
curl -XPOST http://localhost:8000/v1/foods/search -d '{"q":"ro*nd*","searchfield":"company","searchtype":"WILDCARD","max":50,"page":0}'
```
Perform a PHRASE search for an exact match on "broccoli florets" in the "ingredients field:
```
curl -XPOST http://localhost:8000/v1/foods/search -d '{"q":"broccoli raw","searchfield":"ingredients","searchfield":"PHRASE","max":50,"page":0}'
```
Perform a REGEX (regular expression) search to find all foods that begin with "Olive" in the foodDescription field:
```
curl -XPOST http://localhost:8000/v1/foods/search -d '{"q":"^olive+(.*)","searchfield":"foodDesciption","searchfield":"REGEX","max":50,"page":0}'
```
Peform a REGEX search to find all foods that have UPC's that begin with "01111" and end with "684"
```
curl -XPOST http://localhost:8000/v1/foods/search -d { "q":"^01111\\d{2,4}684","searchtype":"REGEX","searchfield":"upc"}
```
### Fetch the nutrients dictionary
```
curl -X GET http://localhost:8000/v1/nutrients/browse
```
```
curl -X GET http://localhost:8000/v1/nutrients/browse?sort=nutrientno
```
```
curl -X GET http://localhost:8000/v1/nutrients/browse?sort=name&order=desc
```
### Run a nutrient report sorted in descending order by nutrient value per 100 units of measure 
Find foods which have a value for nutrient 208 (Energy KCAL) between 100 and 250 per 100 grams 
```
curl -X POST http://localhost:8000/v1/nutrients/report -d '{"nutrientno":207,"valueGTE":10,"valueLTE":50}'
```
Find Branded Food Products which have a nutrient value between 5 and 10 MG per 100 grams caffiene 
```
curl -X POST http://localhost:8000/v1/nutrients/report -d '{"nutrientno":262,"valueGTE":5,"valueLTE":10,"source":"BFPD"}'
```
