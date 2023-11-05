# ece528
University Of Michigan - Dearborn - ECE528 - Final Project by utilzing GCP services.

Project Description:
The Preventive Maintenance Warning in Vehicles project aims to leverage AI technology to detect potential issues in vehicles before they become major problems. By analyzing various sensor data and vehicle performance metrics, the AI system can identify patterns and anomalies that indicate the need for maintenance or repairs. This proactive approach helps vehicle owners and service providers to take timely actions, reducing breakdowns and improving overall vehicle reliability. 


![image](https://github.com/maboulsaad82/ece528/assets/126989551/570c34f3-1a8f-401b-af5a-e543c9d31fdd)

![image](https://github.com/maboulsaad82/ece528/assets/126989551/2fa0776b-26a3-465a-b7bf-958af86c022b)



# Still under developing

# Inference and Predection

1- Data Collection and Aggregation:

a. Dataflow (Apache Beam): We will utilize Cloud Dataflow to stream the data from Pub/Sub into a Cloud Dataflow pipeline that can ingest, process, and aggregate the sensor readings from millions of cars in real-time.

import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
from apache_beam.io import ReadFromPubSub

pipeline_options = PipelineOptions()
p = beam.Pipeline(options=pipeline_options)


#To demonstrate, let's assume each car's sensor reading is a dictionary with a car_id, timestamp, and a sub-dictionary sensors containing all the sensor readings. For simplicity, we'll #aggregate the average value of each sensor over the window:


def aggregate_readings(elements):
    car_id = elements[0]['car_id']
    
    aggregated_sensors = {}
    for sensor_name in ["Engine Coolant Temperature [°C]", "Intake Manifold Absolute Pressure [kPa]", "Engine RPM [RPM]", "Vehicle Speed Sensor [km/h]", "Intake Air Temperature [°C]", "Air Flow Rate from Mass Flow Sensor [g/s]", "Absolute Throttle Position [%]", "Ambient Air Temperature [°C]", "Accelerator Pedal Position D [%]", "Accelerator Pedal Position E [%]"]: 
        total_value = sum([e['sensors'][sensor_name] for e in elements])
        average_value = total_value / len(elements)
        aggregated_sensors[sensor_name] = average_value

    return {
        'car_id': car_id,
        'aggregated_sensors': aggregated_sensors
    }


This approach ensures we can handle multiple sensors for each car.


(p
 | "Read from PubSub" >> ReadFromPubSub(topic='projects/our_PROJECT_ID/topics/car-sensor-readings')
 | "Window data" >> beam.WindowInto(beam.window.FixedWindows(300))  # 5-minute windows
 | "Aggregate readings" >> beam.Map(aggregate_readings)
 | "Write to GCS" >> beam.io.WriteToText("gs://our_BUCKET/aggregated_data")
)

p.run()

b. Pub/Sub: We will use Cloud Pub/Sub to collect data from the cars. Each car can push its readings to a Pub/Sub topic every 5 seconds. Pub/Sub can handle large-scale real-time messaging workloads.

gcloud pubsub topics create car-sensor-readings


Aggregation Logic: Given the volume, processing every reading in real-time might be overkill. We could use a windowed aggregation (e.g., average, median, or other metrics of the readings) over a set time period, like 1 minute, 5 minutes, or even an hour. This depends on how quickly we need to detect a breakdown and the characteristics of our model.



***We can store raw or aggregated data to BigQuery for historical analysis and backup. This is useful for refining our model in the future and understanding the patterns leading up to a breakdown

Storing in BigQuery:

from google.cloud import bigquery
client = bigquery.Client()

table_id = "our_PROJECT.our_DATASET.our_TABLE"
rows_to_insert = [
    # our aggregated data
]

errors = client.insert_rows_json(table_id, rows_to_insert) 


***We can consider the important notes below as well:

Optimize Data Transmission: Instead of sending raw sensor readings, we might want to send summarized or compressed data, especially if we're aggregating over longer windows.
Caching: If certain repetitive patterns of data don't require inference (e.g., a car that's turned off), we can introduce caching mechanisms to skip unnecessary inferences.


------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

2- Model Serving and Triggering Inference:

AI Platform Prediction: Once we've trained our model, we can deploy it to AI Platform Prediction. This service lets us serve our trained model for online predictions at scale.

After aggregating the sensor readings using Dataflow, we can send the aggregated data to AI Platform Prediction for inference. If our model predicts a potential breakdown, then we can take necessary actions.

a. Model Serving:

gcloud ai-platform models create our_MODEL_NAME
gcloud ai-platform versions create our_VERSION_NAME --model=our_MODEL_NAME --origin=gs://our_BUCKET/our_MODEL_DIR --runtime-version=2.3 --python-version=3.7


b. Triggering Inference:

Use the AI Platform Prediction REST API or Python SDK to send aggregated data for inference.

Here's a Python SDK example for sending data for inference:

from google.cloud import aiplatform

client = aiplatform.gapic.PredictionServiceClient(client_options={"api_endpoint": "REGION-aiplatform.googleapis.com"})

endpoint = client.endpoint_path(project="our_PROJECT_ID", location="REGION", endpoint="ENDPOINT_ID")

instance = {
    "input_data": "our_AGGREGATED_DATA"  # Replace with our actual data
}

response = client.predict(endpoint=endpoint, instances=[instance])

predictions = response.predictions


c. Save model checkpoint to Google Cloud Storage:

If we're using TensorFlow:

model.save("gs://our_BUCKET/our_MODEL_DIR")


For PyTorch, we'd serialize the model and use the Google Cloud Storage Python SDK to save it as follow:

Firstly, make sure to install the required libraries:

pip install torch google-cloud-storage


Then, we can serialize the PyTorch model and save it to Google Cloud Storage:

import torch
from google.cloud import storage


torch.save(model.state_dict(), 'model.pth')   # Assuming model is our trained PyTorch model

#Upload the saved model to GCS
storage_client = storage.Client()
bucket = storage_client.bucket("our_BUCKET_NAME")
blob = bucket.blob("our_MODEL_DIR/model.pth")
blob.upload_from_filename('model.pth')

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

3- Notification & Action:

Cloud Functions: If our model detects a potential car breakdown, use Cloud Functions to trigger a specific action. This could be sending a notification to the car driver, alerting the car manufacturer, or informing a nearby service center.

Pub/Sub: We will set up another topic specifically for potential breakdown notifications. Our Cloud Function can publish a message to this topic, and subscribers (like a notification system) can listen to this topic to alert the necessary stakeholders.


a. Cloud Functions to send a message to a new Pub/Sub topic:

import base64
import json
from google.cloud import pubsub_v1

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("our_PROJECT_ID", "our_NOTIFICATION_TOPIC")

def alert_breakdown(data, context):
    message = base64.b64decode(data['data']).decode('utf-8')
    parsed_message = json.loads(message)

    # our logic to detect potential breakdown (maybe checking prediction values)
    if "our_CONDITION":  # Replace with our actual condition, a few hypothetical examples of potential breakdown detection logic can be seen below
        publisher.publish(topic_path, json.dumps({"alert": "Potential Breakdown", "details": parsed_message}).encode('utf-8'))


Deploy Cloud Function:

gcloud functions deploy alert_breakdown --runtime python310 --trigger-topic our_TRIGGER_TOPIC_NAME



The exact breakdown logic depends on the nature of our model's predictions and the nature of the sensor data. Here are a few hypothetical examples of potential breakdown detection logic:

1. Threshold-Based Alerting:
If our model outputs a probability score of a breakdown occurring, we can use a threshold to send an alert:


breakdown_probability = parsed_message.get("breakdown_probability", 0)

if breakdown_probability > 0.8:  # If probability is greater than 80%
    publisher.publish(topic_path, json.dumps({"alert": "High Breakdown Probability", "details": parsed_message}).encode('utf-8'))


2. Trend-Based Alerting:
If we're receiving time series data and want to detect a potential breakdown based on abrupt changes:


#Assuming you're receiving a sequence of recent readings
sensor_readings = parsed_message.get("recent_readings", [])

if len(sensor_readings) > 2 and sensor_readings[-1] - sensor_readings[-2] > threshold_value:  # sudden spike in reading
    publisher.publish(topic_path, json.dumps({"alert": "Sudden Spike Detected", "details": parsed_message}).encode('utf-8'))


3. Combination of Sensor Readings:
If we're getting readings from multiple sensors and believe a combination of certain readings might indicate a potential breakdown:


temperature = parsed_message.get("temperature", 0)
pressure = parsed_message.get("pressure", 0)

if temperature > max_temperature and pressure > max_pressure:  # both sensors exceed thresholds
    publisher.publish(topic_path, json.dumps({"alert": "High Temp & Pressure", "details": parsed_message}).encode('utf-8'))


4. Historical Data Comparison:
If we want to compare a current reading with a historical average or baseline (this might involve fetching data from a datastore):


current_reading = parsed_message.get("current_reading", 0)
historical_average = fetch_historical_average()  # hypothetical function

if current_reading > historical_average * 1.5:  # reading is 50% higher than historical average
    publisher.publish(topic_path, json.dumps({"alert": "Reading Above Historical Average", "details": parsed_message}).encode('utf-8'))


5. Consistent Anomaly Detection:
If we want to detect consistent anomalies over a few consecutive readings:


recent_anomalies = parsed_message.get("recent_anomalies", [])  # list of 0 (no anomaly) or 1 (anomaly)

if sum(recent_anomalies[-5:]) >= 4:  # 4 or more anomalies in the last 5 readings
    publisher.publish(topic_path, json.dumps({"alert": "Consistent Anomalies Dete

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

4- Scaling and Performance:

We can set up auto-scaling while deploying resources. For Dataflow, the number of workers can be set, and for AI Platform Prediction, we can set the machine type and auto-scaling parameters. This ensures that if there's a sudden surge in data, your services can handle it.

Monitoring and Logging: Use Cloud Monitoring and Logging to keep track of your system's health, performance metrics, and potential anomalies.

---------------------------------------------------------------------------------------

5. Visualize Data:

a. In BigQuery:

Once our data is in BigQuery, we can use BigQuery's Web UI to run SQL queries and visualize results. Additionally, we can integrate BigQuery with Google Data Studio to create rich, interactive dashboards.

b. After Inference:

Google Data Studio: We can push our inference results back into BigQuery or another database and use Google Data Studio to visualize them.

Here are the detailed steps to visualize our inference results in Google Data Studio based on our setup:

1. Prepare our Data in BigQuery:
Before we can visualize data in GDS, ensure that our inference results are saved in BigQuery. Each row might represent an inference result for a specific car at a particular time.

2. Access Google Data Studio:
Navigate to Google Data Studio.

3. Create a New Report:
Click on the + Blank Report button.
We'll be prompted to add data to our report.

4. Connect to our BigQuery Data:
In the data sources panel, search for and select BigQuery.
Choose our Google Cloud project, then the dataset, and then the table where our inference data resides.
Click on the Connect button on the top right. GDS will retrieve the schema of our table.

5. Add Visualizations to our Report:
Once our data is connected:

Tool Lawet: On the top, we'll see various tools - select the type of visualization we want (e.g., line chart, bar chart, table).

Data Configuration: On the right side, we'll have panels that allow we to configure the data for the visualization:

Dimensions: These are usually categorical variables, e.g., car_id, timestamp, or sensor_type.
Metrics: These are numerical variables that we might want to sum, average, or otherwise aggregate, e.g., sensor_value or prediction_score.
Style Configuration: Below the data configuration, there's a Style tab which lets we adjust the aesthetics of the visualization.

6. Filtering and Interactivity:
we can add controls (e.g., dropdowns) that allow users of our report to filter data dynamically. This is useful if we have lots of data and want to see results for a specific car, time period, or sensor.

Add controls by selecting them from the toolbar at the top.

7. Sharing and Collaborating:
Once we're satisfied with our report, we can share it with others. Click the Share button on the top right and enter the email addresses of the people with whom we want to share.

we can also get an embeddable link to include our report in a website or an application.

8. Regularly Refreshing Data:
By default, GDS will regularly refresh to pull in the latest data from BigQuery. However, if our data updates very frequently and we want near real-time visualizations, consider setting a shorter data cache time.

Tips:
Structured Data: Ensure our data in BigQuery is well-structured. For instance, for time series data, ensure timestamps are consistent.

Optimization: If our dataset is extensive, consider optimizing our BigQuery table for faster queries, e.g., by partitioning.

Custom SQL: In GDS, when connecting to BigQuery, we can use custom SQL queries instead of direct table connections. This allows for more flexible data retrieval, especially if we need to aggregate or preprocess data in a specific way.

Following these steps should allow we to set up a comprehensive and insightful dashboard in Google Data Studio using our inference results from BigQuery.

--------------------------------------------------------------------------------------------------------------------------------

7- Feedback Loop:

It's important to continuously improve our model. Collect feedback when the model correctly or incorrectly predicts breakdowns. Use this feedback to retrain and refine our model regularly.

We can set up additional data pipelines to process feedback data and feed it into our model training process.

---------------------------------------------------------------------------------

8- Security & Privacy:

When dealing with GCP resources, ensure that our data is encrypted at rest and in transit. Use IAM roles and permissions to restrict access.

Ensure that the data being transmitted is encrypted and that any Personally Identifiable Information (PII) is either not collected or is anonymized.
