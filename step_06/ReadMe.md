> You can download the code of this step [here](../src/step_06.py) or all the steps [here](https://github.com/Avaiga/taipy-getting-started/tree/develop/src).

# Creation and execution of Scenarios

Now that you have seen how to create and run a single pipeline, let's configure a scenario. Remember, a scenario is usually used to answer your business problem. Different scenarios would represent different solutions to your problem. Here, *max_capacity*, *day* and *number of predictions* influences the scenario.

In this example, it will run two pipelines: the first pipeline (*baseline*) and another one (*ml*) that will predict through a different function.

```python      
from statsmodels.tsa.ar_model import AutoReg

# This is the function that will be used by the task
def predict_ml(cleaned_dataset: pd.DataFrame, n_predictions: int, day: dt.datetime, max_capacity: int):
    print("     Predicting with ML")
    # Select the train data
    train_dataset = cleaned_dataset[cleaned_dataset['Date'] < day]
    
    # Fit the AutoRegressive model
    model = AutoReg(train_dataset['Value'], lags=7).fit()
    
    # Get the n_predictions forecasts
    predictions = model.forecast(n_predictions).reset_index(drop=True)
    predictions = predictions.apply(lambda x: min(x, max_capacity))
    return predictions
```

The *predict_ml* task configuration must be created in the same format as before with a function, inputs, and outputs.

<p align="center">
    <img src="predict_ml.svg" width=300>
</p>

```python   
# Create the task configuration of the predict_ml function.
## We use the same input and ouput as the previous predict_baseline task but we change the funtion
predict_ml_task_cfg = tp.configure_task(id="predict_ml",
                                        function=predict_ml,
                                        input=[cleaned_dataset_cfg, n_predictions_cfg, day_cfg, max_capacity_cfg],
                                        output=predictions_cfg)
```

With this new task, the Machine Learning pipeline can finally be configured.

```python   
# Create a ml pipeline that will clean and predict with the ml model
ml_pipeline_cfg = tp.configure_pipeline(id="ml", task_configs=[clean_data_task_cfg, predict_ml_task_cfg])
```

To configure a scenario, you need to use `tp.configure_scenario` and the list of the related pipelines. You could add more pipelines/algorithms with just a couple more lines.

```python   
# Configure our scenario which is our business problem.
scenario_cfg = tp.configure_scenario(id="scenario", pipeline_configs=[baseline_pipeline_cfg, ml_pipeline_cfg])
```

The configuration is now complete. Now, you can create your scenario and execute it. When creating it, Taipy will create your pipelines, and when you submit the scenario, it will run them based on the built-in intelligent scheduling. Taipy knows which tasks to do before which one and will be able to cancel Jobs if a task is repetitive.

```python
# Create the scenario
scenario = tp.create_scenario(scenario_cfg)
# Execute it
tp.submit(scenario)
# Get the resulting scenario
## Print the predictions of the two pipelines (baseline and ml)
print("\nBaseline predictions\n", scenario.baseline.predictions.read())
print("\nMachine Learning predictions\n", scenario.ml.predictions.read())   
```