## Part 1: Deployment: Fast API + Docker

Once we developed our ML model, we have to make it accessible by the public or at least the applications that require the prediction results.

To do that, we need to create an API in which the outside can easily access.  Particularly, what we want to achieve is:

>Our **ML model**  --access via--> **API** (e.g., FastAPI)  --access by--> **consumers** (e.g., websites, mobile, dashboard, IoT devices, etc).

This process of making your model accessible is called **deployment into production**.

So let's get started.

### Prerequisites

- Install Docker
- Install FastAPI (pip install fastapi)
- Install uvicorn (pip install uvicorn)

### API

You build an API that acts as an entry point to your app, through HTTP requests such as GET, POST, PUT, DELETE.

### FastAPI

FastAPI is the most popular go-to framework for building robust and high-performance APIs that scale in production environments.

- Simple and easy to use
- Does not come with a webserver; commonly use **uvicorn** which is a lightning-fast ASGI server.
- FastAPI + uvicorn is one of the fastest
- Unlike Django or Flask, it supports asynchronous requests
- Does not come with a view component;  often used together with React/Vue/Angular/HTML for frontend
- Allows data validation(e.g., maximum length, type)
- Supports error messages
- Default UI like Postman

In this example, we will only be using two HTTP methods:

- <code>GET</code>: used to retrieve data from the application
- <code>POST</code>: used to send data to the application (required for inference)

### Let's get started

The directory structure is as follows:

    ml
    +-- classifier.py
    +-- train.py
    +-- iris_v1.joblib
    schema
    +-- iris.py
    app.py
    Dockerfile
    requirements.txt

### 1. Create a new directory called `ml`. 

This directory will contain all the code related to machine learning.

### 2. Train a simple classifier

For simplicity, let’s use Logistic Regression as our algorithm.

Create a `train.py` in your <code>ml</code> directory.  Put this code below:

```python
from joblib import dump
from sklearn import datasets
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression

# import dataset
iris = datasets.load_iris(return_X_y=True)
X = iris[0]
y = iris[1]

# train
pipeline_dict = [('scaling', StandardScaler()),
            ('clf', LogisticRegression())]

pipeline = Pipeline(pipeline_dict)

pipeline.fit(X, y)

# save model for deployment
dump(pipeline, 'iris_v1.joblib')
```


Note that this model is very simple...e.g., no scaling/splitting/gridsearch.  This is intended so we can quickly jump to deployment...

### 3. Define a placeholder classifier

Let's create a placeholder variable to hold the model, when we load, so we can reuse.

Create a `classifier.py` under the `ml` folder with the following code:

```python
clf = None
```

### 4. Define the schema

FastAPI has an automatic data validation, if we provide it with the `BaseModel` definition.

Create a directory `schema`, and create `iris.py` inside with the following code.

```python
from pydantic import BaseModel, conlist
from typing import List

# Without this file won't break your app, but it's good practice

#basically create a schema describing Iris
#mainly for the purpose of automatic data validation
class Iris(BaseModel):    
    #conlist helps imposing list with constraints
    data: List[conlist(float, 
                    min_items=4,
                    max_items=4)]
```

### 5. Define the router

Here we gonna define how url is routed.  You can see this as the *main()* file.

Create `app.py` in the root level.  In this script, we define the app and specify the router(s).

```python
#our classifier
import ml.classifier as clf
from fastapi import FastAPI, Body
from joblib import load

#Iris data structure
from schema.iris import Iris

#define the fastapi
app = FastAPI(title="Iris Prediction API",
            description="API for Iris Prediction",
            version="1.0")

#when the app start, load the model
@app.on_event('startup')
async def load_model():
    clf.model = load('ml/iris_v1.joblib')
    
#when post event happens to /predict
@app.post('/predict')
async def get_prediction(iris:Iris):
    data = dict(iris)['data']
    prediction = clf.model.predict(data).tolist()
    proba = clf.model.predict_proba(data).tolist() 
    return  {"prediction": prediction,
            "probability": proba}
```

### 6. Try run the uvicorn server to see how the API is

We are actually done with the API.  Yes!  It's that simple.

Run the server by:

    uvicorn app:app --port 5000

Go to `http://127.0.0.1:5000/docs`.  Then try input some values and see the response by clicking **Try it out**.

You can also try only three values, and see the errors.

![swagger UI](figures/swagger.png)


### 7. Include Dependencies

Let's prepare ourselve containerize our app.  But before that, let's create a file containing all our dependencies.

At root, create a `requirements.txt` file to specify all of the dependencies required to build this app.

My requirements.txt looks like this:

    fastapi==0.78.0
    numpy==1.23.1
    scikit_learn==0.24.2
    starlette==0.19.1
    uvicorn==0.18.2
    joblib==0.17.0
    pydantic==1.9.1

If you don't know which version you are using, try `pip list`.

### 8. Dockerfile

We also need to create a Dockerfile which will contain the commands required to assemble the image. Once deployed, other applications will be able to consume from our iris classifier to make cool inferences about flowers.

```dockerfile
FROM python:3.8-slim-buster

RUN apt-get update && apt-get install -y python3-dev build-essential

WORKDIR /app

COPY requirements.txt .
RUN pip3 install -r requirements.txt

COPY . .

EXPOSE 5000 

CMD uvicorn --host 0.0.0.0 --port 5000 app:app
```

The first line defines the Docker base image for our application. The `python:3.8-slim-buster` is a popular image — it’s lightweight and very quick to build. 

Our Dockerfile concludes with a `CMD` which is used to set the default command to `uvicorn --host 0.0.0.0 --port 5000 app:app`. The default command is executed when we run the container.

If you don't understand very well, don't worry!  There are many online materials how to make Dockerfile. :-)

### 9. Build and run the container

We are almost there!!

Build the docker image using 

    docker build . -t iris

This step takes a while to finish.

Check whether you have successfully build the image

    docker images

*Note: If you make any mistake, simply* `docker rmi [image_id]`*, and do the build again*.

After the image is built, generate the docker container using 

    docker run --name iris -p 8080:5000 iris

Check whether your image is running

    docker ps -a

*Note: If you want to stop, do* `docker stop [image_id]`*; if you want to remove the container, do* `docker rm [image_id]`*.  Do these until you are satisfied :-)*

This exposes the application to the port 8080. Running the container also kicks off the default command we set earlier — which effectively starts up the app!

### 10. Use the API

So let's try our API.

First, I have to check my docker-machine ip by doing

    docker-machine ip default

Here, my ip is 192.168.99.100

Go to `192.168.99.100:8080/docs`.  Now you can do the same thing.

### Congrats!!

In the next lab, let's deploy to **Heroku**, so everyone in the world can use your API.  Also let's try setup **CI/CD with github actions**.

## Part 2: Deployment: Heroku + Github Action

Let's deploy our app online.  We gonna use **Heroku** which is free but also support paid version.

### Pre-requisites

Make sure you have a completely separate repository holding the app that we did last time.

Here is the separate repository of mine:

https://github.com/chaklam-silpasuwanchai/iris-heroku-example


### 1. Install heroku cli 
(You can do it in any directory)

    brew tap heroku/brew && brew install heroku

If you are using other os, please refer to 

https://devcenter.heroku.com/articles/heroku-cli#install-the-heroku-cli

### 2. Login 

Login to your heroku

(You can do it in any directory)

    heroku login
    heroku container:login

### 3. Create heroku app

(You can do it in any directory; app name can be anything)

    heroku create [app-name]

To check that you have really created the app, you can go to heroku website and check.

![app](figures/app.png)

### 4. Push and deploy

Before we do anything, we have to revise the port variable in `Dockerfile`.  This is because heroku has its own port.

You can check the PORT variable via

    heroku run printenv -a [app-name]

Revise your `Dockerfile` to:

```dockerfile
FROM python:3.8-slim-buster

RUN apt-get update && apt-get install -y python3-dev build-essential

WORKDIR /app

COPY requirements.txt .
RUN pip3 install -r requirements.txt

COPY . .

# EXPOSE 5000 <--we don't need this

CMD uvicorn --host 0.0.0.0 --port $PORT app:app
```

Now, let's push to heroku container register.  Go to the level where the Dockerfile is:

    heroku container:push web -a [app-name]

(Note:  The first time I did this, it freezes.  Not sure why, but once I restarted my mac, it works fine.)

Then let's release to the public

    heroku container:release web -a [app-name]    

Now go to 

    http://[app-name].herokuapp.com/docs

If you want to change the domain name, just simply purchase a domain name and link with it.

(Note: if your app is not running, check the logs:  `heroku logs -a [app-name]`)

### 5. Changing app

Now let's try add something in the `app.py` and see whether the changes propagate

```python
@app.get("/")
async def root():
    return {"message": "Hello World"}
```

Again, we just repeat the two steps:

    heroku container:push web -a [app-name]
    heroku container:release web -a [app-name]    

Go to 

    http://[app-name].herokuapp.com   

You will see the changes.

### 6. Continuous integration with Github action

Now, this process can be automated, which is called **continuous integration** or **CI/CD**.  The idea is that whenever we push the code, it must run certain steps for us, such as test and deploy procedure for us.  There are two popular CI/CD frameworks which are **Jenkin** and **Github action**.  Since **Github action** has received a lot of interest lately, we shall explore it.

First, create a directory `.github` on the root level (at the same level as the root level of the repository)  (Note that the name cannot change because github action looks for this folder)

    mkdir .github

Then inside .github, create a directory called `workflows`

    cd .github
    mkdir workflows

Inside the workflows, create the `main.yml` file

    cd workflows
    touch main.yml

Inside this, we shall define our github action, i.e., everytime we commit and push new code, it should help us automatically deploy to heroku.  The code is:

```yml
name: Deploy

on: push

jobs:
  build:  # any name is ok for this line
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: akhileshns/heroku-deploy@v3.12.12 # this is the action
        with:
          heroku_api_key: ${{secrets.HEROKU_API_KEY}} #must be set in github > settings > secrets
          heroku_app_name: "iris-ait" #must exist
          heroku_email: "chaklam072@gmail.com"
          justlogin: true
      - run: |
          heroku container:login
          heroku container:push web -a iris-ait  
          heroku container:release web -a iris-ait

     #please change iris-ait to your app name
```

Go to your github repository, go to `Settings > Secrets`, set `HEROKU_API_KEY`.

![secrets](figures/secrets.png)


For the api key, run `heroku authorizations:create` for production apps, use `heroku auth:token` for development (you can do this anywhere in the terminal).

![auth](auth.png)


If you want to further tweak, see https://github.com/marketplace/actions/deploy-to-heroku

To see that it is working, we can try change some of our API code like this:

```python
@app.get("/")
async def root():
    return {"message": "We change something"}
```

Then you can push and commit as usual.

You can check whether your `main.yml` is working by going to your github > actions.

![actions](figures/actions.png)

Then try to go to `http://[app-name].herokuapp.com` to see the change.

### Congrats!

Now we don't have to worry about running tedious commands.  Everything we push, these commands will be run.  What you can do more is to incorporate test in the github action.

In the next lab, let's try explore some monitoring tools.

#### References

- https://towardsdatascience.com/deploying-iris-classifications-with-fastapi-and-docker-7c9b83fdec3a
- https://medium.com/analytics-vidhya/serve-a-machine-learning-model-using-sklearn-fastapi-and-docker-85aabf96729b

