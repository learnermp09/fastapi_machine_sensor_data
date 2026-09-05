# Predictive Maintenance - Data Validation Pipeline

This project builds a data validation and cleaning pipeline for industrial sensor data. The goal is to take raw machine sensor readings, clean them, validate every field against strict rules, and store only the trustworthy records for later use in a machine learning model.

The dataset used here (`data_predictive_maintenance.csv`) contains sensor readings collected from multiple machines, including temperature, pressure, vibration, RPM, voltage, current, humidity, operating hours, maintenance history, and fault codes. Each record also has a label, `failure_next_7_days`, indicating whether the machine failed within the following week.

## Why this project exists

Raw sensor data is rarely clean. Values go missing, sensors report out-of-range readings, and text fields sometimes contain placeholders like "N/A" or empty strings instead of real data. Feeding this kind of data directly into a machine learning model leads to unreliable predictions.

This notebook treats data validation as its own step, separate from modeling. Every row is checked against a defined schema before it is accepted. Rows that pass validation are considered safe to use downstream. Rows that fail are kept aside along with the reason they failed, so nothing is silently dropped or ignored.

## What the notebook does

1. **Load and inspect the data**
   Reads the CSV file and checks its shape, column types, missing values, and duplicate rows.

2. **Clean placeholder values**
   Replaces common placeholder patterns (empty strings, "N/A", "NULL", "null") with proper missing values, and fixes column data types, including converting the timestamp column to a real datetime type.

3. **Define a validation schema with Pydantic**
   The sensor and machine data is modeled using Pydantic classes, each responsible for one part of a machine reading:
   - `SensorData` validates sensor measurements against realistic operating ranges (for example, temperature between 10 and 102, voltage between 220 and 260) and adds two computed fields: power consumption and a vibration status label (Normal, Warning, or Critical).
   - `MachineInfo` validates the machine identifier and production batch, and rejects any machine ID that does not follow the expected naming pattern.
   - `MaintenanceInfo` validates operating hours, days since last maintenance, and the fault code.
   - `PredictionInfo` validates that the failure label is either 0 or 1.
   - `MachineReading` combines all of the above into a single nested model representing one full sensor reading. It also includes a cross-field check that flags physically implausible combinations (for example, very high temperature together with very low pressure), and a computed health score that combines vibration and operating hours into a single 0-100 rating.

4. **Convert and validate every row**
   Each row of the cleaned DataFrame is converted into the nested structure the Pydantic models expect, then validated. Rows that pass are collected as clean, structured records. Rows that fail are kept separately along with a readable description of what went wrong, tied back to the original row number for traceability.

5. **Store the results**
   Valid records are flattened back into a DataFrame for review, and can optionally be inserted into a MongoDB collection for downstream use.

## Why Pydantic

Pydantic was chosen over manual if-checks because it keeps validation rules close to the data definition, gives clear and consistent error messages, and supports nested models, computed fields, and cross-field checks without extra boilerplate. This makes the validation logic easier to read, test, and extend as the dataset grows.

## Project structure

```
.
├── data_predictive_maintenance.csv     # raw sensor data (not included in repo)
├── pydantic_data_validation.ipynb      # main notebook: cleaning, validation, storage
├── .env                                # MongoDB connection string (not included in repo)
└── README.md
```

## Requirements

- Python 3.10 or later
- pandas
- numpy
- pydantic (version 2)
- pymongo
- python-dotenv

Install them with:

```bash
pip install pandas numpy pydantic pymongo python-dotenv
```

## Running the notebook

1. Place `data_predictive_maintenance.csv` in the project folder.
2. Create a `.env` file with your MongoDB connection string:
   ```
   MONGO_URI=your_mongodb_connection_string
   ```
   The MongoDB step is optional. The notebook will still run and produce valid and invalid record sets even without a database connection.
3. Open `pydantic_data_validation.ipynb` and run the cells in order.

## Current status

This repository currently covers the data validation stage only. Cleaning and validation are treated as a separate, reusable step so that any model trained later is built on data that has already been checked for correctness.

## Planned next steps

The validated data from this stage will be used to build a machine learning classification model that predicts whether a machine is likely to fail within the next 7 days, using the sensor and maintenance features described above.

Once the model is trained, the plan is to wrap it in a FastAPI backend that exposes a prediction endpoint. Users will be able to submit machine sensor readings through this API and receive a failure prediction in response, along with a health score for the machine. This will turn the current data pipeline into a complete, end-to-end system, from raw sensor data to a usable prediction service.
