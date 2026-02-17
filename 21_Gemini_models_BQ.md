# Gemini models in Bigquery

Gemini, an LLM introduced in May 2023, has a family of models:

- Gemini Nano: Smallest tasks
- Gemini Pro & Ultra: Powerful
- Gemini Flash: Low latency, and cost-effective

BQ + AI

BQML (BigQuery Machine Learning): Enables users create and run AI and ML models in BigQuery. It also allows access to the remote GenAI models on VertexAI.

You can then perform AI taks by using pre-trained foundation models like Gemini.
You can also leverage Google Cloud AI services through APIs, such as Cloud Natural Language, Cloud Tranlation and Document AI.

SQL + Python as code.

We could run:

- ML: Predictive AI (e.g. forecast sales, classify products)
- Gen AI: (e.g. Generate responses to customer reviews. Automate marketing campaigns)
- Or a hybrid approach, combining both.

See [image](./image.png) for AI/ML model capabilities.

There are two main steps for using the AI/ML capabilities in Bigquery:

1. Create a model
2. Use the model

## Create a model

This stage comprises 3 iterative steps:

- Data preparation (structured and unstructured)
- Model creation
- Model evaluation (ML.Evaluate)

When the evaluation is below expectation: Retrain or tune the model with new training data.

In terms of deployment, BigQueryML supports three main types of models based on their hosting location:

- Local models (BQML built-in models + models trained in Vertex AI): reside within BigQuery and can be trained either internally in BigQuery or externally in Vertex AI. These built-in models mainly focus on predictive AI tasks. These include classification, like a logistic regression model, regression like a linear regression model, and time series forecasting, like an ARIMA_PLUS model. You are recommended to start with simple options such as logistic regression and linear regression, and use the results as a benchmark to compare against more complex models such as deep neural networks (DNN), which take more time and computing resources to train and deploy.

```sql
CREATE MODEL PROJECT.DATASET.MODEL
OPTIONS (
    MODEL_TYPE = 'LINEAR_REG', ...
)
```

- Remote models (Pre-trained Gen AI models): hosted in Vertex AI and referenced by BigQuery. These include pre-trained Gen AI models like Gemini and Cloud AI services like Natural Language APIs.

```sql
CREATE MODEL PROJECT.DATASET.MODEL
REMOTE WITH CONNECTION
'PROJECT.location.connection_name'
OPTIONS(ENDPOINT = 'https://vertex-ai-endpoint-address')
```

- Remote models (Cloud AI models: Natural language, Translate, Vision)

```sql
CREATE MODEL PROJECT.DATASET.MODEL
REMOTE WITH CONNECTION
'PROJECT.location.connection_name'
OPTIONS(REMOTE_SERVICE_TYPE = 'CLOUD_AI_VISION_V1' | ...)
```

- Imported models: trained anywhere and imported into BigQuery from Cloud Storage. For example, Open Neural Network Exchange (ONNX), Tensorflow, and XGBoost.

```sql
CREATE MODEL PROJECT.DATASET.MODEL
OPTIONS(MODEL_TYPE = 'TENSORFLOW' | ...,
MODEL_PATH = 'gs://bucket/path/model/*'
)
```

## Use a model

Once the model is created, it's time to unleash its potential. The use stage unfolds in three iterative steps:

- Model Serving (ML.predict, ML.generate_text)
- Model explanation (optional)
- Model monitoring

### ML.predict

Simply run a BigQuery ML function against different models, and you're all set. For predictive AI tasks like prediction, classification, and clustering, use the ML.PREDICT function. For time series forecasting, use ML.FORECAST(). For generative AI tasks like content generation, summarization, and rephrasing, use ML.GENERATE_TEXT(). ML.understand_text() is available for language analysis, and ML.translate() handles machine translation.

### Model explanation

Next is model explanation, which aims to understand how each feature contributes to the predicted result.

You can use: ML.explanation_predict for non-time-series models, particularly supervised models such as regression models and DNN.

While ML.explanation_forecast is for time-series models.

### Model monitoring

Model monitoring in BigQuery ML focuses on data to monitor model performance.

This involves: comparing a model's serving data with its training data to prevent data skew, and comparing new serving data with previously used serving data to prevent data drift.

Commonly used functions for data monitoring include ML.DESCRIBE_DATA, ML.VALIDATE_DATA_SKEW, and ML.VALIDATE_DATA_DRIFT.

## Gemini in action

So the plan to conquer the challenge of customer relationship management for Coffee on Wheels is:

- Ingest data
- Create models
- Analyze data
- Take actions

From a technical perspective, the workflow can be broken down into a five-step pipeline: 

- Establish a connection to the generative AI remote models hosted on Vertex AI.
- Construct a dataset for multimodal data using an object table that stores unstructured data such as text and images.
- Create a remote model to reference the endpoint of the Gemini model.
- Analyze customer reviews by extracting keywords, determining sentiment, and producing a report.
- Take action by formulating responses to customer feedback and planning marketing campaigns.

Let’s dive into each of these steps in greater detail.

### Establish a connection to the generative AI remote models hosted on Vertex AI

- BQ explorer -> Connections to external data sources -> Vertex AI remote models, remote functions and BigLake (Cloud Resource). 
- IAM: `Vertex AI User` role assigned to corresponding user / service connection

### Build multimodal datasets

- **Upload text data**

```sql
LOAD DATA OVERWRITE gemini_demo.customer_reviews(customer_review_id INT64, customer_id INT64, ...) FROM FILES (
    format = 'CSV',
    uris = ['gs://qwicklabs-gcp-01-....-bucket/gsp1246/customer_reviews.csv']
);
```

- Object tables make unstructured data accessible in BQ

Object tables stores the reference to the unstructured data that live on Cloud storage. Each row represents an object. Columns correspond to the metadata of this object (e.g. uri, generation, content_type, size, md5_hash, updated...)

Object tables use access delegation, so users can access the object table without directly accessing Cloud Storage, which adds an additional layer of security.

-- **Upload image data using an `object table`**

```sql
CREATE OR REPLACE EXTERNAL TABLE `gemini_demo.review_images` 
WITH CONNECTION `us.gemini_conn`
OPTIONS(
    object_metadata = 'SIMPLE',
    uris = ['gs://...-bucket/gsp1246/images/*']
);
```

What happens next to the unstructured data in an object table? Well, through a technology called `embeddings`, that data is converted to numeric vectors that represent semantic meanings. Combined with structured data, these vectors form the input for AI/ML models, enabling further tasks such as prediction or generation.

### Create Gemini models

Currently, the widely used gemini models include `Gemini-pro` and `Gemini-flash`

As you have learned from the previous lesson, in addition to a regular CREATE MODEL statement, you must specify the remote connection and the endpoint that refers to the Gemini model you want to use.

Diane also needs to create a Gemini vision model to process image data. Once the model is created, use it to analyze customer reviews. The `ML.GENERATE_TEXT()` function enables you to perform generative natural language tasks using data from BigQuery standard tables or object tables.