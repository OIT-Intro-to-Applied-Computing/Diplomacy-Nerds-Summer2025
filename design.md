# Design Document: Social Media Sentiment Analysis System for UAE & Qatar
## 1. Introduction

This document provides a detailed design for a social media sentiment analysis system focused on public discourse regarding the UAE and Qatar on major US social media platforms. It elaborates on the architectural components, technologies, and workflows required to acquire, store, process, analyze, and visualize sentiment data at scale, ensuring adherence to platform policies and data privacy best practices.  (aside: i have mostly worked on gcp, not aws/azure, but i'm sure we could get them to work just as well with a little wrangling, so the first options i generally have listed for the stack are google products)
## 2. System Goals & Requirements
### 2.1. Core Goals

    Objective: Accurately determine the sentiment (positive, neutral, negative) of social media posts related to UAE and Qatar originating from US social media platforms.

    Insight Generation: Provide actionable insights into public perception, emerging trends, and reactions to events concerning these regions.

    Scalability: Design a system capable of handling high volumes of social media data.

    Reliability: Ensure continuous data ingestion and analysis with robust error handling.

    Data Privacy & Compliance: Adhere strictly to social media platform API terms of service and relevant data privacy regulations (e.g., GDPR, CCPA).

### 2.2. Non-Functional Requirements

    Performance: Data ingestion should be near real-time or batch-processed with minimal latency (e.g., hourly updates). Sentiment analysis should complete within reasonable timeframes.

    Scalability: The system must scale horizontally to accommodate increasing data volumes and user queries.

    Reliability & Fault Tolerance: Components should be designed for high availability and graceful degradation in case of failures.

    Security: Data must be encrypted at rest and in transit. Access controls must be strictly enforced.

    Maintainability: The codebase should be modular, well-documented, and easy to update.

    Cost-Effectiveness: Utilize cloud services efficiently to optimize operational costs.

## 3. Architecture Overview

The system follows a layered architecture, leveraging cloud-native services for scalability and managed operations:

    Data Acquisition Layer: Connects to official social media APIs (X, Facebook, Instagram, Reddit) to extract raw data.

    Data Ingestion & Storage Layer (Data Lake): Stores raw, unstructured social media data in a cloud-based data lake.

    Data Processing & Preprocessing Layer: Cleans, transforms, and prepares the raw data for machine learning, including language detection and normalization.

    Machine Learning Model Layer: Applies sentiment analysis models (pre-trained or fine-tuned) to classify the sentiment of processed text.

    Insights & Visualization Layer: Stores analyzed data in a structured format and provides tools for reporting and dashboarding.
```
+---------------------+     +---------------------+     +-------------------------+
|   Social Media      |     |     Data Ingestion  |     |   Data Processing &     |
|       APIs          |---->|      (Python/SDKs)  |---->|     Preprocessing       |
| (X, FB, IG, Reddit) |     |                     |     | (Spark/Dataflow/Glue)   |
+---------------------+     +----------+----------+     +------------+------------+
                                       |                               |
                                       v                               v
+-----------------------------------------------------------------------------------+
|                            Data Lake (e.g., S3/GCS)                               |
| (Raw JSON, Parquet/Delta Lake for processed data)                                 |
+-----------------------------------------------------------------------------------+
                                       |
                                       v
+-------------------------+     +---------------------+     +---------------------+
|   ML Model Layer        |     |   Analyzed Data     |     |   Insights &        |
| (Sentiment Analysis)    |---->|      Storage        |---->|    Visualization    |
| (LLMs, BERT, AraBERT)   |     | (Data Warehouse)    |     | (BI Tools, Dashboards)|
+-------------------------+     +---------------------+     +---------------------+
```
## 4. Detailed Design
### 4.1. Data Acquisition Layer

This layer is responsible for securely connecting to social media APIs and extracting relevant data.

    Chosen Platforms:

        X (formerly Twitter): Primary source for public discourse. Will utilize the Twitter API v2 (e.g., Filtered Stream for real-time, Search Tweets for historical/batch).

        Facebook: Focus on public pages and groups related to news, travel, or official entities. Will use the Facebook Graph API.

        Instagram: Focus on public posts via hashtags or business accounts. Will use the Instagram Graph API.

        Reddit: Subreddits and posts related to geopolitics, travel, news. Will use the Reddit API.

    Authentication:

        All API integrations will use OAuth 2.0 or Bearer Tokens as required by each platform.

        API keys and secrets will be stored securely in a dedicated secret management service (e.g., AWS Secrets Manager, Google Secret Manager, Azure Key Vault).

    Data Extraction Logic:

        Language Filtering: Request data primarily in English and Arabic.

        Keyword Filtering: Use specific keywords for UAE and Qatar (e.g., "UAE", "Dubai", "Abu Dhabi", "Qatar", "Doha", "Emirates", "Qatari", relevant hashtags like #Expo2020, #WorldCup2022 for historical context, #VisitUAE, #DiscoverQatar).

        Geographical Filtering: Where possible, filter for posts originating from or mentioning locations within the US. This might be challenging as precise user location data is often limited or not publicly available via APIs.

        Frequency:

            Near Real-time (Streaming): For X, use the Filtered Stream API for continuous ingestion of new tweets.

            Batch (Hourly/Daily): For other platforms or historical data, schedule hourly/daily batch pulls using their respective search/feed APIs.

    Error Handling & Rate Limiting:

        Implement exponential backoff for API retries to handle rate limit errors.

        Log all API errors (e.g., authentication failures, invalid requests, service unavailability) to a centralized logging system.

        Monitor API usage against defined rate limits.

    Technology Stack (Example):

        Programming Language: Python

        Libraries: requests for general API calls, platform-specific SDKs (e.g., Tweepy for X, facebook-sdk for Facebook).

        Orchestration: Cloud-native scheduling services (e.g., AWS Lambda + EventBridge, Google Cloud Functions + Cloud Scheduler, Azure Functions + Timer Trigger) or a dedicated orchestrator like Apache Airflow/Cloud Composer for more complex workflows.

### 4.2. Data Ingestion & Storage Layer (Data Lake)

This layer is responsible for storing the raw, unprocessed social media data.

    Cloud Storage Choice:

        Google Cloud Storage (GCS): Recommended for its cost-effectiveness, scalability, and integration with other Google Cloud services, eg pub/sub.

        (Alternatives: AWS S3, Azure Blob Storage)

    Data Lake Structure:

        Raw Zone: Stores raw JSON responses directly from APIs.

            Naming Convention: gs://[bucket-name]/raw/[platform]/[year]/[month]/[day]/[timestamp]_[platform]_[batch_id].json

            Example: gs://social-sentiment-data/raw/twitter/2025/07/02/165322_twitter_batch1.json

        Staging Zone: (Optional, but good practice) For temporary storage of data after initial validation/schema inference before deeper processing.

        Processed Zone: Stores cleaned and structured data in optimized formats (e.g., Parquet, Delta Lake).
        I think we may see a change in the view of the two as the summer progresses-travel and other events shape public opinion.

    Ingestion Pipeline:

        Streaming Ingestion (for X):
            (can adapt for other platforms similarily)
            Source: X Filtered Stream API.

            Messaging Queue: Google Cloud Pub/Sub (or Apache Kafka) to decouple data producers (API clients) from consumers (processing jobs).

            Consumer: A streaming data processing job (e.g., Google Cloud Dataflow, Apache Spark Streaming) that reads from Pub/Sub and writes raw JSON to GCS. Maybe not streaming, maybe we collect data on some metronome, and feed it to the pub, and the sub gathers the data at a less frequent time stamp. (ie every 15-30 minutes grab data from platforms, send it to 'pub', and our 'sub' client grabs it every 30 -hour or more)

        Batch Ingestion (for Facebook, Instagram, Reddit, and historical X data):
            (maybe do a similar thing here with pub/sub model, look at apis and see how they work)

            Source: Python scripts making API calls.

            Destination: Directly write raw JSON files to GCS.

            Orchestration: Cloud Scheduler or Airflow /cron to trigger these Python scripts.

    Data Format:

        Raw Data: JSON (preserving original API response structure).

        Processed Data: Apache Parquet or Delta Lake format for columnar storage, compression, and schema evolution, optimizing for analytical queries.

### 4.3. Data Processing & Preprocessing Layer

This layer transforms raw data into a clean, normalized format suitable for sentiment analysis.

    Processing Framework:

        Google Cloud Dataflow (Apache Beam): Recommended for its serverless, auto-scaling capabilities, suitable for both batch and streaming data processing.  I am a little bit in the weeds here-gotta read about how this works.

        (Alternatives: Apache Spark on Dataproc/EMR/Azure Databricks, AWS Glue)

    Preprocessing Steps (as Dataflow/Spark jobs):

        Read from Data Lake: Read raw JSON files from the raw zone in GCS.

        Schema Inference/Mapping: Convert semi-structured JSON into a structured table format.(for ease of cleaning, just because i only know how to do elementary cleaning with pandas, and all the following can be done with pandas)

        Cleaning:

            Remove URLs (regex).

            Remove mentions (@username) and hashtags (#hashtag) if not needed for sentiment.

            Remove excessive whitespace, special characters, and non-textual elements.

            Handle emojis: Convert to textual descriptions (e.g., using emoji library) or remove.

        Language Detection:

            Use a robust language detection library (e.g., langdetect, fastText pre-trained models for language identification).

            Filter for English and Arabic content. Posts in other languages will be excluded or flagged for separate analysis.

        Tokenization: Split text into words (e.g., using NLTK or spaCy tokenizers).

        Normalization:

            Lowercasing: Convert all text to lowercase.(pandas cleaning in previous step)

            Stop Word Removal: Use language-specific stop word lists (e.g., NLTK for English, Arabic-stop-words for Arabic).

            Lemmatization/Stemming: Reduce words to their base form (e.g., spaCy for English, AraBERT tokenizers for Arabic).

        Feature Engineering (if using traditional ML): Create TF-IDF vectors or generate word embeddings. (Less critical if using LLMs or fine-tuned transformers directly).

        Write to Processed Zone: Store the cleaned and preprocessed data in Parquet/Delta Lake format in the processed zone of the data lake.

### 4.4. Machine Learning Model Layer (Sentiment Analysis)

This layer applies the chosen sentiment analysis models to the preprocessed text.
This will also be a more manual section-we might be able to whip up a script that does some/most of it but labelling data will be sometimes manual.
    Model Selection Strategy:

        Primary Approach: Pre-trained Transformer Models (Fine-tuned): This offers the best balance of accuracy and efficiency.

            English Sentiment Model:

                Base Model: bert-base-uncased or roberta-base.

                Fine-tuning: Fine-tune on a general English sentiment dataset (e.g., SST-2, IMDB) and potentially a smaller, domain-specific dataset related to geopolitics/travel if available.

                Library: Hugging Face Transformers.

            Arabic Sentiment Model:

                Base Model: AraBERT (aubmindlab/bert-base-arabertv2) or MARBERT (UBC-NLP/MARBERT). These models are specifically pre-trained on large Arabic corpora and understand Arabic nuances.

                Fine-tuning: Fine-tune on an Arabic sentiment dataset, ideally one that includes Gulf dialects or social media content.

                Library: Hugging Face Transformers.

        Alternative/Supplemental: Large Language Models (LLMs) via API (e.g., Gemini): (this way will probably be to costly for our use, even with gemini cli, the restriction is ~ 1 million tokens/day-might be enough-different languages require different number of tokens for same query-maybe arabic/english delta not so much, we could try it and see what happens-best case, we feed gemini cli once per day and she gives us back some type of response that is 'accurate' and since we are using a pretrained LLM, how do we know its biases/programming. worth a shot just to see what the result is.)

            Use Case: For ad-hoc analysis, small batches, or as a robust fallback.

            Method: Direct API calls to Gemini (or similar LLM APIs) with carefully crafted prompts to classify sentiment.

            Pros: No model hosting/management overhead.

            Cons: Cost per inference, latency for large volumes, potential rate limits. (agian gemin cli may alleviate this pain point)

    Training/Fine-tuning Environment (if required):

        Cloud ML Platform: Google Cloud AI Platform Training, AWS SageMaker, Azure Machine Learning. These provide managed environments for model training and hyperparameter tuning.

        Data: Use the labeled datasets (English and Arabic) stored in the data lake.

    Model Deployment & Inference:

        Deployment Platform:

            Google Cloud AI Platform Prediction / Vertex AI Endpoints: For deploying and managing custom ML models as scalable APIs.

            (Alternatives: AWS SageMaker Endpoints, Azure ML Endpoints)

        Inference Pipeline:

            Trigger: Triggered by new data arriving in the processed zone of the data lake. (by a volume of data or any data? NOt sure how this ought to work, oh wait a tic- data cleaning etc can be done to all arriving data, then fed to ML model in batches.)

            Read Data: Read preprocessed text from Parquet/Delta Lake files.

            Load Model: Load the appropriate English or Arabic sentiment model.

            Predict Sentiment: Pass the text through the model to get sentiment labels (positive, neutral, negative) and confidence scores.

            Write Results: Store the original tweet ID, text, predicted sentiment, and confidence score back into the data lake (or directly into the data warehouse).

### 4.5. Insights & Visualization Layer


We can maybe start working on this sooner than later, because we will be taking data from the data lake results which we can say are in gross buckets-qatar: good/bad UAE:good bad and make it more granular as we can.  Then display the changing/unchanging sentiment as the class continues.('real time' displays,see dashboard components below)
This layer makes the analyzed sentiment data accessible and visual for end-users.

    Data Warehouse:

        Google BigQuery: Recommended for its serverless, highly scalable analytical database capabilities, ideal for storing structured sentiment results.

        (Alternatives: AWS Redshift, Snowflake, Azure Synapse Analytics)

        Schema: Table for sentiment results including: tweet_id, original_text, language, sentiment_label, sentiment_score, timestamp, platform, keywords_matched, user_id (anonymized if needed), location (if available).

    Data Loading to Warehouse:

        Use a data integration service (e.g., Google Cloud Dataflow, AWS Glue, custom Python scripts) to load the analyzed sentiment data from the data lake's processed zone into BigQuery.

        Schedule this load after the sentiment analysis job completes.

    Business Intelligence (BI) Tools:

        Looker Studio (Google Data Studio): Free, cloud-native, and integrates seamlessly with BigQuery.

        (Alternatives: Tableau, Power BI, Looker)

        Dashboard Components:

            Overall sentiment distribution (pie charts, bar graphs).

            Sentiment trends over time (line graphs).

            Sentiment breakdown by platform.

            Sentiment by keyword/topic.

            Geographic sentiment distribution (if location data is reliable).

            Top positive/negative phrases or keywords.

            Ability to filter by date range, platform, keywords.

    

## 5. Monitoring & Alerting  
This part is extra and gratuitously tossed in here. some of this data might be nice to have to look at, we can keep it and not make it into a board

Continuous monitoring is essential for system health and performance.

    Metrics to Monitor:

        API call success rates and response times.

        Data ingestion volume (number of posts ingested per hour/day).

        Processing job success rates and execution times.

        ML model inference latency and error rates.

        Sentiment distribution trends (to detect anomalies).

        Cloud resource utilization (CPU, memory, storage).

    Monitoring Tools:  Determine how/why to use this-cost vs ?

        Cloud-native monitoring (e.g., Google Cloud Monitoring, AWS CloudWatch, Azure Monitor).

        Dedicated monitoring solutions (e.g., Prometheus + Grafana).

    Alerting:  Maybe not necessary at first

        Set up alerts for critical failures (e.g., API authentication errors, data ingestion halts, model deployment failures).

        Configure alerts for performance degradation (e.g., increased latency, decreased throughput).

## 6. Security & Compliance
Some of these are absolutely necessary

Data security and adherence to legal/ethical guidelines are paramount.

    API Key Management:

        Use dedicated secret management services.

        Rotate API keys regularly.

        Grant least privilege access to services using API keys.
        Use IAM on GCP wisely.  Dont eff it up.
        Who needs access to the CI/CD pipeline and code?And running instances?

    Data Encryption:

        At Rest: All data stored in GCS and BigQuery will be encrypted by default.
        This is the default.

        In Transit: Use TLS/SSL for all data transfers between components and APIs.
        Not super worried about this.
    Access Control (IAM):

        Implement strict Identity and Access Management (IAM) roles and policies.

        Grant only necessary permissions to services and users.

    Data Minimization: Only collect data essential for sentiment analysis. Avoid collecting sensitive personal identifiable information (PII) where possible.
    We will strip any left over from the api calls-most of the platforms should not allow us access to ther PII.

    Anonymization/Pseudonymization: If user IDs or other potentially identifying information are stored, they should be anonymized or pseudonymized to protect privacy.(see data cleaning-we have to rm all this junk)

    Platform Terms of Service: Continuously review and ensure compliance with the evolving ToS of X, Facebook, Instagram, and Reddit. This includes restrictions on data redistribution and commercial use.

    Data Retention Policies: Define and implement data retention policies to comply with privacy regulations and minimize storage costs.  I suppose remove/delete data after project is complete.

## 7. Future Enhancements

    Emotion Detection: Beyond positive/negative/neutral, classify specific emotions (e.g., joy, anger, sadness, surprise).

    Entity Recognition: Identify key entities (people, organizations, locations) mentioned alongside sentiment.=> are there trends here?

    Topic Modeling Refinement: More sophisticated topic modeling to understand the specific subjects driving sentiment.

    Real-time Dashboard: Develop a real-time dashboard for immediate insights from streaming data.

    User Interface for Keyword Management: Allow non-technical users to manage keywords and view analysis results through a custom UI.  This will take some finagling,and deeper thought than i have at 11 pm.

    Cross-Lingual Sentiment Analysis: Explore models that can handle multiple languages simultaneously, especially for code-switched content.

## 8. Conclusion

This design document provides a comprehensive blueprint for building a scalable and reliable social media sentiment analysis system. By adhering to these design principles and leveraging cloud-native services, the system will effectively capture, process, and analyze public sentiment regarding UAE and Qatar, delivering valuable insights while maintaining data security and compliance.